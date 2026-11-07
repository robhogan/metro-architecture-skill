# The incremental graph, the server and configuration

## `Graph` — `packages/metro/src/DeltaBundler/Graph.js`

A live, mutable, **refcounted** module graph that supports incremental add/modify/delete while remaining correct in the presence of cycles.

```js
dependencies: Map<path, Module<T>>   // path: a file's absolute path, '<dir>?ctx=<sha1>' or 'virtual:<virtual path>'
Module = {path, dependencies: Map<key, Dependency>, inverseDependencies: CountingSet<path>,
          output, getSource, unstable_transformResultKey}
#virtualSources: Map<path, VirtualSource>   // modules with no file (below)
```

### Module paths

A module's `path` is its graph key: usually a file's absolute path, but a context module's is `<dir>?ctx=<sha1>` and a virtual module's is `virtual:<absolute virtual path>`, a namespace (`lib/modulePath.js`) chosen so that it "can never equal the absolute path of a real file". Every consumer of a `Module` sees the key (`createModuleId`, `processModuleFilter`, custom serializers), so derive a file path, URL or display name from it through `getModuleSourcePath`, `getModuleDisplayPath` and `isVirtualModulePath`, never with `path` functions.

### Refcounting without a refcount field

`inverseDependencies` is a `CountingSet` (`metro/src/lib/CountingSet.js`) — a multiset whose `delete` only removes an item once the delete calls match the add calls, and whose `size` counts *unique* items. A module's reference count **is** `inverseDependencies.size` (+1 if it's an entry point). No parallel refcount to keep in sync — this is principle §4 ("represent, don't allocate") applied to correctness as well as memory.

The multiset matters because one module can depend on another more than once; naive `Set` semantics free a module that is still referenced. Lazy `async` edges create no node, but are counted the same way in `#importBundleNodes`, so deleting a lazily imported file still finds its importers.

### Cycle collection

The header comment says "Portions of this code are based on the Synchronous Cycle Collection algorithm described in" Bacon & Rajan's *Concurrent Cycle Collection in Reference Counted Systems* (ECOOP '01). Modules are colored black/gray/white/purple; candidate roots go into `#gc.possibleCycleRoots` (a `Set`, so it's duplicate-free without a per-node "buffered" flag); `_collectCycles` runs `_markGray` → `_scan` → `_collectWhite` between the two commit passes (and again after a rollback).

Without it, a deleted entry into a dependency cycle leaks the whole cycle. **If you touch edge add/remove, run the cycle collector's tests** — the invariants are not obvious, and the code says which deviations from the paper were deliberate.

### Commit protocol

`traverseDependencies` builds the new state first (`_buildDelta` → `buildSubgraph`), descending only into **new or modified** modules, and into a virtual module whose edge carries a different SHA-1 from the one the graph holds ("Traverse new or modified paths, and virtual modules with new content.") — any other module already in the graph is a boundary. `buildSubgraph` collects per-path `errors` rather than throwing. The commit is then a subtractive pass (`onlyRemove`), cycle collection, and an additive pass, which "ensures that any errors encountered on the additive pass would also be encountered on a fresh build": an error in a module the same edit made unreachable doesn't fail the build. If the additive pass does throw, the graph **rolls back** to snapshots of the base state and re-throws, so a failed build cannot leave a half-updated graph. Copy this shape: build the new state, then swap, reversibly.

Per module, `_recursivelyCommitModule` diffs dependencies in three steps:

1. remove dependencies that changed or disappeared,
2. add dependencies that changed or appeared,
3. **detect changed or reordered dependency keys** — which must still be committed even when the set of edges is identical, because the serializer emits a module's dependency map in `dependencies` order and the transformed code indexes into it by position (`_dependencyMap[i]`). A stale order points requires at the wrong modules: "Replace dependencies with the correctly-ordered version, matching the transform output". (Module order in the *bundle* is a separate sort by id.)

A module with an unchanged `unstable_transformResultKey` (the transform cache key, so unchanged inputs) and unchanged dependencies is never marked touched, so revisiting an importer, or saving a file unchanged, yields an empty delta: no new revision, and HMR's `update` message carries no modules.

**Touched modules are replaced, not mutated.** `_recursivelyCommitModule` builds a new `Module` object for a module it touches and restores the previous object otherwise, so every module in `added`/`modified` is a new object, and a `WeakMap` keyed on the `Module` (or on its output's `VlqMap`) is invalidated for free. The converse doesn't hold: a module released and re-added within one commit, or restored by a rollback, also gets a new object without appearing in the delta — a cache miss, never a stale hit. `inverseDependencies` is shared between the old and new objects and mutated in place even on untouched modules, so key nothing on it.

Traversal respects `shouldTraverse`: `weak` dependencies and (in `lazy` mode) `async` ones are recorded but not followed.

### Modules with no file — `VirtualSource`

A dependency on a module with no file carries that module's source on the edge as a `VirtualSource` (`DeltaBundler/types.js`). `buildSubgraph` returns them per dependency key, `Graph` keeps them in `#virtualSources` by module path (freed with the module, snapshotted for rollback), and `_buildDelta` hands each traversal root's entry back to the transform, so such a module can be revisited without re-transforming its importer. Either way `transformHelpers.getTransformFn` passes the source to the transform as `fileBuffer`, so the content hash in the cache key covers it and `getSource` returns it. Two arms, which differ in what invalidates them:

- **`requireContext`** — a `require.context` call (behind `transformer.unstable_allowRequireContext`), at a synthetic path `<dir>?ctx=<sha1 of its options>` so two calls over one directory with different options don't collide. Its inputs are "discovered at transform time rather than pinned at resolution": the template is rendered from the file map on the main thread, and `DeltaCalculator` marks the module modified when an added file matches it (`Graph.markModifiedContextModules`); a deleted file reaches it as an ordinary inverse dependency.
- **`buffer`** — a virtual module (status in SKILL.md's "Unused and gated mechanisms"; resolver contract in `references/resolution.md`), whose source is pinned at resolution. Its identity is its path, not its content, and no file event names it, so `traverseDependencies` compares each `buffer` edge's `sha1` with the one the graph holds and revisits the module on a mismatch (`dependenciesEqual` compares only paths). It is transformed as, and resolves its dependencies from, its bare virtual path, so transformers and resolvers never see `virtual:`. Two edges in one traversal that resolve one virtual path to different content are an error.

**A new kind of module with no file is a new arm of this union, not a second side channel** — with a graph path no file can have, its source carried on the edge into `fileBuffer`, and a stated answer to what revisits it when its content changes.

## `DeltaCalculator` — `packages/metro/src/DeltaBundler/DeltaCalculator.js`

One per graph. Subscribes to the file map's `'change'` event and accumulates `_addedFiles` / `_modifiedFiles` / `_deletedFiles` between builds. `getDelta` awaits any in-flight `_currentBuildPromise` ("delta server doesn't support concurrent builds"), then swaps the sets for fresh ones, so changes that arrive mid-build wait for the next delta.

Details worth internalizing:

- **Event folding.** `#markAdded` / `#markModified` / `#markDeleted`: deleted+added ⇒ modified; added+modified ⇒ still added; added+deleted ⇒ no change; modified+deleted ⇒ deleted. The transitions keep the three sets disjoint. Getting this wrong leaves a stale module in the graph, or tries to transform a deleted file.
- **Failure must not lose changes — or reorder them.** If the build throws, the pending sets are restored: otherwise "asking for a delta after an error will produce an empty Delta, which is not correct". Changes that arrived during the build happened after the ones it took, so they are **replayed through the same transitions** on top of the restored sets, not merged by union (a union can leave a file in both `modified` and `deleted`). `_requiresReset` is cleared when taken and not restored, because the restored base already holds every module a reset marked.
- **Success can leave stale state too.** A file deleted and recreated before its importer re-resolves it leaves the build green and the old module in the graph, so the next build treats an added file the graph already holds as modified: "It is modified, not added, since it never left the graph." Any accumulate-then-drain design must say what happens to events that land during a drain, on both paths.
- **`#shouldReset`** — some changes invalidate resolutions we can't identify, so they force a full revisit: **symlink changes**, and (with `unstable_enablePackageExports`) any `package.json` change. A conservative reset is the right answer when you can't enumerate the affected modules; an unsound delta never is.
- `getDelta({reset})` with `reset: true` returns the whole graph as `added` with `reset: true` — how the initial build is produced (`DeltaBundler.buildGraph`, and `IncrementalBundler`'s initial revision). Server and HMR updates call `updateGraph(revision, false)`.
- `DeltaCalculator` emits `'change'` for every file map event, whether or not the file is in its graph, so each `HmrServer` client group runs an update cycle (debounced by 50ms) even for changes outside its graph.

## `Transformer` — `packages/metro/src/DeltaBundler/Transformer.js`

The main-thread half of transformation; what the worker does is `references/transformation.md`, and the cache-key composition is `references/caching.md`. Architecturally:

- The transform is a **pure function of (source, options)** executed in a worker. That purity is the precondition for content-addressing its result. Anything else the output depends on is an explicit, keyed input: an asset bakes in its URL path, so `assetUrlPath` is computed on the main thread, keyed "per asset here and not in the base hash", and passed to the worker.
- The base hash is computed once, in the constructor, so everything it covers is read only at startup.
- `getOrComputeSha1` returns the file **content alongside the hash** when it had to read it, so a cache miss doesn't re-read the file.
- Cache writes are **fire-and-forget** with an error reporter, so a slow or broken *write* never blocks the build. Reads are awaited, and a read error is reported (`cache_read_error`) and re-thrown, failing that module.
- `getSource` is a **thunk** on the result, not eager content: sources are materialized on demand (source maps, RAM bundle info). It re-reads the file from disk, so it can differ from what was transformed if the file changed in between — unless the caller supplied `fileBuffer` (a context module's rendered template, or a virtual module's source), which it returns as-is.
- `fileBuffer` lets a caller of `Bundler.transformFile` supply a file's contents.
- The `WorkerFarm` is created last: "Start workers last - nothing can end them if the constructor throws."

## Workers — `WorkerFarm.js`, `Worker.flow.js`

`jest-worker` (or in-band when `maxWorkers <= 1`) over `metro-transform-worker` (or `config.transformerPath`); `unstable_workerThreads` switches to threads instead of processes. The worker logic lives in `Worker.flow.js`; `Worker.js` is a babel-register shim around it.

- Worker config must be **serializable and stable** — it crosses a process boundary, and it reaches the cache key through the transformer's own `getCacheKey(transformerConfig, …)` (a custom `transformerPath` without one contributes only `globalPrefix`, which `getTransformCacheKey` adds itself). `getTransformOptions`, `transformVariants`, `unstable_supportsInlineModules` and `unstable_workerThreads` are explicitly stripped before being passed, and `unstable_supportsInlineModules: true` is added back when `resolver.unstable_allowVirtualModules` is on ("Babel plugins can emit different code for this, so it must be part of the transform cache key." — `references/caching.md`).
- **Output must not depend on the worker** — or on anything else unkeyed: `Worker.flow.js` deletes the per-worker `JEST_WORKER_ID` for this reason, and the rest of the environment is a known gap (`references/caching.md`).
- The transformer module is addressed by **path**, and the hash of that file is in the cache key.
- Keep payloads small. Every `transform` call sends the path, options, `projectRoot`, the **whole `transformerConfig`**, `assetUrlPath`, and the content only if the main thread already has it — so the size of the transformer config is a per-file IPC cost. Never send a graph or a closure.
- `config.stickyWorkers` shards transforms by file path (`computeWorkerKey`); the farm uses `workerSchedulingPolicy: 'in-order'` so warm workers take the initial fan-out.

## `Bundler` — `packages/metro/src/Bundler.js`

Owns the `DependencyGraph` (file map, watchers) and then the `Transformer` (workers). `ready()` includes adding, crawling and watching Metro's runtime roots (`references/file-map.md`, "Roots and `addRoot`").

- **Initialization errors propagate.** `_initializedPromise` re-throws, so `ready()` and everything behind it rejects with the original error, and a no-op `.catch` is attached "so callers can await the original promise later without triggering an unhandled rejection".
- **Teardown tolerates partial initialization.** `end()` swallows the initialization error (as does `DependencyGraph.end()`), ends the transformer if there is one, then the dependency graph in a `finally`. A teardown that awaits `ready()` throws after a failed start and leaks whatever did start.

## `IncrementalBundler` and `Server`

- `_revisionsByGraphId: Map<GraphId, Promise<GraphRevision>>` — one live graph per distinct bundle configuration (see `getGraphId`). **Promises are stored, not values**, so concurrent requests for the same graph share one build. A failed initial build is evicted.
- `_revisionsById: Map<RevisionId, Promise<GraphRevision>>` — `updateGraph` mints a new revision only for a non-empty delta. Its date drives `Last-Modified`/304, its id goes out as `X-Metro-Delta-ID`, and `HmrServer` keys client groups by it, re-keying on each update. **Revision ids expire on the graph's next change:** A non-empty update deletes the old id before minting the next (an empty one keeps it), so anything still holding an old id gets `RevisionNotFoundError` (e.g. an HMR group whose graph a bundle request updated first). Look graphs up by `GraphId`; treat a revision id as a token for the response that carried it.
- **Graph lifetime.** A revision lives until `endGraph` — an HTTP `DELETE` of the bundle URL — and each graph keeps a `DeltaCalculator` subscribed to file changes. `DeltaBundler.buildGraph`'s own note warns that its graph "needs to be ended when finished"; the API paths `Server.build()`, `getRamBundleInfo()` and `getOrderedDependencyPaths()` build through it and don't end theirs.
- **A graph is identified by its resolved entry path.** `Server` resolves a bundle URL's entry (decoding URL prefixes via `RootUrlMap`) before `getGraphId`. `HmrServer` repeats that resolution to find the graph a bundle request built — it never builds one — so it constructs its own `RootUrlMap` from the same inputs. API and CLI entry files are disk paths: `path.resolve` against the server root, no URL decoding.
- `Server._createModuleId` is a single factory for the server's lifetime, shared with `HmrServer` — numeric module ids must stay stable across deltas, or an HMR update references the wrong module.
- Serializers (`Serializers/`) are the last stage and read the graph without mutating it, though they assign module ids as a side effect: `baseJSBundle`, `hmrJSBundle` (delta only), `sourceMapString`/`sourceMapObject`, `getExplodedSourceMap` (for `/symbolicate`), `getAssets`, `getRamBundleInfo`. They run per request (`references/serialization.md` §4).

### A bundle request, end to end

`Server.processRequest` (`packages/metro/src/Server.js`) runs `_processRequest` and hands any rejection to `next`. For a `.bundle` URL:

1. **Rewrite.** `_rewriteAndNormalizeUrl` passes `req.url` through `config.server.rewriteRequestUrl`, with `jscSafeUrl.toNormalUrl` on its input and output.
2. **Route** by pathname. The suffixes `.bundle`, `.map` and `.assets` go to `_processBundleRequest`, `_processSourceMapRequest` and `_processAssetsRequest`, all made by `_createRequestProcessor`, so a `.map` request builds or updates the bundle's graph. `/assets/…` goes to `_processSingleAssetRequest` and `/symbolicate` to `_symbolicate`. Any other pathname `RootUrlMap` maps to a file goes to `_processSourceRequest`, which streams it if the file map holds it and its extension is in `sourceExts`, `additionalExts` or `assetExts`, and answers 404 otherwise; an unmapped pathname goes to `next()`.
3. **Parse.** `parseBundleOptionsFromBundleRequestUrl` throws on a `virtual=true` URL, into `next`. Booleans take `true` or `1`: `dev` and `runModule` default to `true`, and `minify`, `excludeSource`, `inlineSourceMap`, `lazy`, `modulesOnly` and `shallow` to `false`. `platform` defaults to the pathname's platform suffix, `sourcePaths` to `absolute` (or `url-server`), and `unstable_transformProfile` to `default` (or `hermes-stable`, `hermes-canary`). `resolver.<key>` and `transform.<key>` fill `customResolverOptions` and `customTransformOptions` with strings. `entryFile` is the pathname (or `bundleEntry`), decoded and stripped of its extension; `sourceUrl` (JSC-safe) and `sourceMapUrl` derive from the request URL.
4. **Split.** `splitBundleOptions` yields `resolverOptions` (`customResolverOptions`, `dev`), `transformOptions` (`customTransformOptions`, `dev`, `minify`, `platform`, `unstable_transformProfile`, `type: 'module'`), `serializerOptions` (`excludeSource`, `inlineSourceMap`, `modulesOnly`, `runModule`, `sourceMapUrl`, `sourceUrl`, `sourcePaths`) and `graphOptions` (`lazy`, `shallow`).
5. **Entry.** `_resolveRelativePath` resolves `entryFile` as a dependency of the server root, so `sourceExts` and platform extensions apply. A failure is answered at once, 404 for `UnableToResolveError` and 500 otherwise, with no reporter event.
6. **Graph.** `getGraphId` (`references/resolution.md` §3); a `DELETE` ends the graph. Otherwise a revision from `getRevisionByGraphId` goes to `updateGraph(revision, false)`, or `initializeGraph` builds one ("`IncrementalBundler` and `Server`").
7. **Serialize** with `serializer.customSerializer` (any `map` it returns is dropped) or `bundleToString(baseJSBundle(…)).code`, on every request (`references/serialization.md` §4).
8. **Respond.** An `If-Modified-Since` string-equal to the revision date's `toUTCString()` gets a 304. Otherwise a 200 carries `Content-Type: application/javascript; charset=UTF-8`, `Last-Modified`, `X-Metro-Delta-ID` (the revision id), `X-Metro-Files-Changed-Count` and `Content-Location` (`sourceUrl`). Under `Accept: multipart/mixed`, progress parts come first and the status travels in `X-Http-Status`, because "the headers have already been sent".
9. **Errors.** `formatBundlingError` turns a build failure into JSON typed by its error class, else `InternalError`. Only `ResourceNotFoundError`, for an entry `fs.realpath` rejects, gets a 404, so an unresolvable dependency is a 500.

**Reporting.** `config.reporter` is any `Reporter`, `interface {update(event: ReportableEvent): void}` (`lib/reporting.js`), `TerminalReporter` by default. The rule for Metro code is to report, not print: "add a new type of ReportableEvent instead, and implement a proper handler in the reporter(s)". Each request processor emits, under one `buildID`, `bundle_build_started` (its `bundleDetails.bundleType` is `bundle`, `map` or `assets`), `bundle_transform_progressed`, then `bundle_build_done`, or `bundle_build_failed` and `bundling_error`. `HmrServer` emits `bundling_error` for a failed update. Startup, the file map, workers and caches report through the same interface (`initialize_*`, `dep_graph_*`, `watcher_*`, `transformer_load_*`, `cache_read_error`, `cache_write_error`, …). `Server.build()` emits no build events; `metro build` sends started, progressed and done, never failed, to its own `TerminalReporter` through `runBuild`'s `onBegin`, `onProgress` and `onComplete`. `ReportableEvent` is exported from `metro`, so a new event changes its API snapshot.

**A new bundle URL parameter** needs:

1. Parsing in `parseBundleOptionsFromBundleRequestUrl`, and a place in `BundleOptions` (`shared/types.js`) and `Server.DEFAULT_BUNDLE_OPTIONS`, which the API paths spread.
2. One `splitBundleOptions` bucket. Output-only means `serializerOptions`, so variants share a graph.
3. If it changes resolution or transformation, a `getGraphId` entry. A transform input also needs `calcTransformerOptions` (`lib/transformHelpers.js`) to pass it to the worker, and the transform key (`references/caching.md`).
4. If it changes output, the serializer options built in `_processBundleRequest`, `_serializeGraph` (`Server.build()`) and `getRamBundleInfo`, and the map paths if the map must agree (`references/serialization.md` §4). `HmrServer` finds graphs through the same parsing and `getGraphId`, and async and HMR bundle URLs, also in §4, keep the bundle URL's parameters.
5. Docs in `docs/GettingStarted.md`, "URL and bundle request".
6. Counterparts in `RunBuildOptions` and `runBuild` (`index.flow.js`, whose defaults differ: `dev: false`, `minify: true`, `platform: 'web'`), `RequestOptions`, `shared/output/bundle.flow.js`, `commands/build.js`, `docs/API.md` and `docs/CLI.md`.

### Server composition

`Server.processRequest` is one connect handler; `createConnectMiddleware` and `runServer` (`metro/src/index.flow.js`) wrap it. The stack, outermost first:

1. `runServer`'s `unstable_extraMiddleware` (deprecated) and `unstable_priorityMiddleware`
2. `config.server.unstable_priorityMiddleware`
3. `config.server.enhanceMiddleware(processRequest, server)` — deprecated in its type and docs, with no runtime warning
4. `config.server.unstable_middleware`
5. `runServer`'s `unstable_middleware`

Each entry is a handler or a `[path, handler]` pair, mounted with connect's prefix semantics. `unstable_onServerCreated(server)` is called from config, then from `runServer`, before the HTTP server listens. When configs merge, a later config's middleware wraps an earlier one's (its `unstable_priorityMiddleware` runs first, its `unstable_middleware` last), while a later `unstable_onServerCreated` or `enhanceMiddleware` replaces the earlier one. Priority handlers see a request before `rewriteRequestUrl`; post-Metro handlers only receive requests Metro passed to `next()`. Extend the server through these, not by wrapping `processRequest`.

### URLs and roots — `packages/metro/src/lib/RootUrlMap.js`

The one map between URL pathnames (`/[metro-project]/…`, `/[metro-watchFolders]/<index>/…`) and files, for source, asset and entry requests, `ServerUrl` source paths and symbolication.

A **dynamic root** (`metro/src/lib/dynamicRoots.js`) is any root added to the file map with `FileMap.addRoot`. It takes an 8-hex-digit SHA-1 prefix of its project-relative path in place of the index — independent of other roots and their order, and portable across machines, as are the asset cache keys that include it. Roots are read through a callback on every use (`DependencyGraph.getDynamicRoots()`, memoized on the identity of `FileMap.getRoots()`).

- Configured roots are matched first, then dynamic roots outermost-first. Because `addRoot` records a root even inside an existing one, a dynamic root can lie inside `projectRoot` without changing any file's URL. Appending a configured root never changes the URL of a file under an existing configured root, nor what an existing URL refers to, though a dynamic root inside it moves to its index; adding a dynamic root that *contains* an existing dynamic root moves that root's files to the new id.
- On main, `DependencyGraph._addMetroRuntimeRoots` adds Metro's own `metro-runtime` and its `@babel/runtime` (what `metro:runtime` and `metro:babel-runtime` resolve into) before `ready()` resolves, so the list is never empty in practice.
- Because roots arrive during initialization, `Server` waits for `ready()` before giving up on an unknown `[metro-watchFolders]` id in a source request.
- A virtual module's URL is the one a file at its virtual path would have, plus `virtual=true`, so it must never be served as that file.

Asset URL paths likewise have one derivation, `getAssetUrlPath` (`metro/src/Assets.js`), run per asset in `Transformer`. Metro's transformer bakes in the path it receives (or the project-relative path when it gets none) and reports it as `unstable_assetUrlPath` on the asset's output ("A temporary migration helper, to be removed."); `getAssets` (what a build tool saves) reads it back, falling back to the project-relative path ("Transformers other than Metro's use the project-relative path."). So a wrapper transformer that forwards only five arguments still gets matching paths in the bundle and in `getAssets`, and `getAssets` needs no roots.

## HMR and Fast Refresh

The server is `packages/metro/src/HmrServer.js`, the WebSocket at `/hot`; the client is `metro-runtime`'s `modules/HMRClient.js` and the dev branch of `polyfills/require.js`. Messages are typed in `metro-runtime/src/modules/types.js`: clients send `register-entrypoints`, `log`, `log-opt-in` and `heartbeat`, and the server sends `bundle-registered`, `update-start`, `update`, `update-done` and `error`.

### Server

- **Registration.** For each bundle URL in `register-entrypoints`, `_registerEntryPoint` applies `rewriteRequestUrl`, resolves the entry and looks the graph up by `getGraphId`. It never builds one ("`IncrementalBundler` and `Server`" above), and sends `error` (`GraphNotFoundError`) if there is none.
- **Client groups** are keyed by revision id (`_clientGroups`). A client joins the group for its graph's current revision, or creates one, which subscribes to the graph's `DeltaCalculator` (`DeltaBundler.listen`) and keeps the rewritten URL that update URLs derive from (`references/serialization.md` §4). Registration then runs an update cycle with `isInitialUpdate: true`, delivering changes the graph hasn't taken yet, and sends `bundle-registered`.
- **Update cycle.** `'change'` goes through `debounceAsyncQueue` (50ms, one cycle at a time per group). Each cycle sends `update-start`, `update` or `error`, then `update-done` with the event's `changeId`. `_prepareMessage` calls `updateGraph(revision, false)`, re-keys the group by the new revision id, and sends that id with `hmrJSBundle(delta, …)`'s added and modified `HmrModule`s and deleted ids.
- **Errors.** A throw while updating is reported as `bundling_error` and sent as `error` (`formatBundlingError`); the group keeps its revision and `DeltaCalculator` the failed changes, so the next cycle retries. React Native's client closes the socket on either not-found error and asks for a reload.

### Client

- **`HMRClient`** is a transport that re-emits each message. If enabled, an `update` evaluates each added, then each modified, module's code (`global.globalEvalWithSourceUrl(code, sourceURL)` if defined, else `eval`); if not, updates merge until `enable()`. Deleted ids stay registered. The embedder (React Native's `Libraries/Utilities/HMRClient.js`) sends `register-entrypoints`.
- **Applying a module.** An update's `__d` registers a new id and, for a registered one, calls `__accept` (`metroHotUpdateModule`). A match for `unstable_forceFullRefreshPatterns` (on the verbose name) forces a full refresh; an uninitialized module just swaps its factory. Otherwise the runtime walks up the update's inverse dependencies to modules that called `module.hot.accept()` or are React Refresh boundaries (every export a likely component type), forcing a full refresh at a module with no parents ("No root boundary") or on a cycle. It re-runs, child first, every module from the changed one up to and including the boundaries. A throw restores that module's exports and stops the update ("This error has already been reported via a redbox."); a boundary that stops qualifying, or whose export signature changes, passes the update to its parents if all are boundaries, else forces a full refresh; then `performReactRefresh()`, debounced by 30ms. A full refresh is `window.location.reload()` if available, else `Refresh.performFullRefresh(reason)`.
- **React Native's part.** `Refresh` is `global[<prefix>__ReactRefresh]` (or `require.Refresh`), which React Native installs over `react-refresh/runtime`; with none, only `module.hot.accept()` ends a walk. The `$RefreshReg$` / `$RefreshSig$` registrations come from `react-refresh/babel`, which the Babel transformer must add, not Metro; Metro's transform options carry no `hot` flag for it to key on.

### Reviewing a change that touches this

- HMR serializes only the delta, with Metro's `wrapModule` and ids whatever `customSerializer` does; work added to `hmrJSBundle` is paid per changed module per edit, plus a walk of its ancestors.
- Module ids must stay stable for the server's lifetime: update ids address the client's registry, inverse dependencies its parents, and React Refresh families are keyed on `<prefix> <moduleId>`.
- The delta must be exact: a module missing from `modified` never re-runs, and one included needlessly re-runs its side effects up to a boundary, or reloads the app if it reaches the entry.
- Run `require-test.js`, `fast-refresh-integration-test.js`, `hmrJSBundle-test.js` and `HmrServer-test.js`.

## Configuration — `metro-config`

**Loading.** `loadConfig(argv, defaultConfigOverrides)` (`packages/metro-config/src/loadConfig.js`):

1. `resolveConfig(argv.config, argv.cwd)` resolves an explicit path (`require.resolve`, then against `process.cwd()`), or searches from `cwd` up to the home directory, inclusive, in the order `docs/Configuration.md` lists; with no file, the config is `{}`. `loadConfigFile` `require`s the file, falls back to `import()`, and awaits the export.
2. The base is `mergeConfig(await getDefaultConfig(rootPath), defaultConfigOverrides)`, where `rootPath`, the default `projectRoot`, is the file's directory (the parent of `.config`; `cwd` with no file). A function export is called with the base; its result, or a plain export, is merged over it.
3. Validation (below), then `overrideConfigWithArguments` applies CLI arguments, unvalidated: `port`, `projectRoot`, `watchFolders`, `assetExts`, `sourceExts`, `platforms`, `maxWorkers`, `transformer` (as `babelTransformerPath`), `resetCache`, and `verbose: false` (a no-op reporter).
4. `projectRoot` is prepended to `watchFolders`, the file map's roots, "as Metro assumes that is the case".

`runMetro` (behind `runServer`, `runBuild` and `createConnectMiddleware`) and `buildGraph` merge their config over `getDefaultConfig(config.projectRoot)` again, without step 4.

**Merging.** `mergeConfig(base, ...configs)` folds left to right; a function receives the merge so far, and a promise makes the result one. `mergeConfigObjects` spreads the top level and each section, so an array, function or nested object within a section (`resolver.extraNodeModules`, `transformer.minifierConfig`) is replaced whole, except:

- `resolver.schemeResolvers`, `watcher.watchman`, `watcher.healthCheck` and `watcher.unstable_autoSaveCache` merge key by key.
- `server.tls`: `false` replaces, an object merges into the base's, `null` or `undefined` keeps it.
- `server.unstable_middleware` and `unstable_priorityMiddleware` concatenate, without duplicates ("Server composition").
- A `cacheStores` function is called with `metro-cache`'s exports.
- `resolver.dependencyExtractor`, `resolver.hasteImplModulePath` and `transformer.babelTransformerPath`, when set, are resolved like an explicit config path.

A new rule needs a test in `src/__tests__/mergeConfig-test.js`, and must leave a complete config unchanged when `runMetro` merges it over the defaults again.

**Validation.** Only `loadConfig` validates: `jest-validate` checks the merged config against `validConfig()` (`src/defaults/validConfig.js`: the defaults, with stand-ins for a few `null` or `undefined` ones), with `recursiveDenylist: ['reporter', 'resolver', 'transformer']` and `deprecatedConfig: {}`. A type mismatch throws (`null` and `undefined` pass; any function matches a function); a key absent from the example warns as unknown; `resolver`, `transformer` and array contents go unchecked; nothing warns of deprecation.

**Types and API.** In `src/types.js`, `ConfigT` is the complete config consumers receive; `InputConfigT` (alias `MetroConfig`), partial per section and in `watcher`'s merged objects, is what config files and `mergeConfig` accept. `packages/metro-config/API.md` snapshots the exports (`yarn build-api-snapshots`).

**A new option needs:**

1. A type in its section's `*ConfigT`, or for most transformer options in `metro-transform-worker`'s `JsTransformerConfig`, which `TransformerConfigT` spreads.
2. A default in `getDefaultValues` (`src/defaults/index.js`), or a validated section warns.
3. A `validConfig` example, if its default is `null`, `undefined` or of another type than its values.
4. A merge rule, if later configs should extend or compose it.
5. A `docs/Configuration.md` entry, labeled Experimental for an `unstable_` name.
6. Regenerated API snapshots.
7. A place in each cache key whose output it changes (`references/caching.md`): `transformer.*` through the transformer's `getCacheKey`, file-map input through `BuildParameters` or a plugin's `getCacheKey()`.

## Adding something to this layer

- **New per-module data?** It must come out of the transform (so it's cached with the code) or be derivable from the graph. Don't compute it per-request in the serializer — serializers run over every module on every request.
- **New edge type?** Update `dependenciesEqual`, the three-step commit diff, and the GC — and add cycle tests.
- **New kind of module with no file?** A new `VirtualSource` arm, carried on the edge into `fileBuffer` so the content hash covers it, under a graph path no file can have (a `lib/modulePath.js` namespace, or a suffix like `?ctx=`), with a stated answer to what revisits it when its content changes — `dependenciesEqual` compares paths, so a content change at a fixed path needs its own check, as the `buffer` arm's SHA-1 comparison is. Derive every file path, URL and name from its key through `lib/modulePath.js`.
- **New change source?** Route it through `DeltaCalculator` so it participates in folding, reset and error recovery. Don't add a second subscription to the file map.
- **New bundle variant?** It needs a distinct `getGraphId`, or it will silently share a graph revision with a different configuration.
- **New long-lived resource?** Acquire it last, and release it in `end()` even after a failed start. A graph is one: end it when you're done with it. **New URL mapping?** Put it in `RootUrlMap`, so `Server` and `HmrServer` agree. **New server behavior?** A middleware hook, not a wrapper around `processRequest`.
- **Test incrementally.** A test that builds once proves nothing here: mutate, rebuild, assert the delta; then delete, rebuild, assert again — and land a change mid-build, on a failing and a passing build.
