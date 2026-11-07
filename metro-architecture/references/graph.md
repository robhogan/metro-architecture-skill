# The incremental graph, transformation and workers

## `Graph` — `packages/metro/src/DeltaBundler/Graph.js`

A live, mutable, **refcounted** module graph that supports incremental add/modify/delete while remaining correct in the presence of cycles.

```js
dependencies: Map<path, Module<T>>   // path: a file's absolute path, '<dir>?ctx=<sha1>' or 'virtual:<virtual path>'
Module = {path, dependencies: Map<key, Dependency>, inverseDependencies: CountingSet<path>,
          output, getSource, unstable_transformResultKey}
#virtualSources: Map<path, VirtualSource>   // modules with no file (below)
```

### Module paths

A module's `path` is its graph key, in one of three forms: a file's absolute path, a context module's `<dir>?ctx=<sha1>`, or a virtual module's `virtual:<absolute virtual path>`, in the namespace `lib/modulePath.js` reserves so that it "can never equal the absolute path of a real file, and anything that mistakes it for one fails rather than reading the wrong file". Every consumer of a `Module` sees the key — `createModuleId`, `processModuleFilter`, custom serializers, `Absolute` source paths. Derive a file path, URL or name from a key through the helpers there, never by applying `path` functions to it: `getModuleSourcePath` is the path a module "is resolved from, transformed as, and addressed by in URLs" (a virtual module's bare virtual path, any other key unchanged), `getModuleDisplayPath` gives the dev-name form (`virtual:<rootDir-relative POSIX path>` for a virtual module), and `isVirtualModulePath` is the test. Where serializers use each is in `references/serialization.md`, "Paths and locations a map points at".

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

`buildSubgraph.js` does the resolve+transform fan-out for the paths being visited. A dependency on a module with no file carries that module's source on the edge as a `VirtualSource` (`DeltaBundler/types.js`), a tagged union of two arms that differ in where their content comes from, and so in what invalidates it:

- **`{type: 'requireContext', requireContext}`** — a `require.context` call (behind `transformer.unstable_allowRequireContext`). `deriveAbsolutePathFromContext` gives the context module a synthetic path, `<dir>?ctx=<sha1 of its options>`, so two calls over the same directory with different options don't collide. Its inputs are "discovered at transform time rather than pinned at resolution": `transformHelpers.getTransformFn` enumerates the file map on the main thread and renders a template. So it has its own invalidation — `DeltaCalculator` marks a context module modified when an added file matches it (`Graph.markModifiedContextModules`), and a deleted file reaches it as an ordinary inverse dependency.
- **`{type: 'buffer', source, sha1}`** — a *virtual module*, from a `virtualModule` resolution (contract and gate in `references/resolution.md`, "Virtual resolutions"). Its graph path is `virtual:<virtual path>`, and its source is pinned at resolution: "The source travels with the edge that produced it, so it lives exactly as long as the module is reachable." Its identity is its path, not its content, and no file event names it, so its content changes only when an importer re-resolves it.

Either way, `getTransformFn` passes the source to `Bundler.transformFile` as `fileBuffer`, so the content hash in the cache key covers it and `getSource` returns it.

- **Edge to graph.** `buildSubgraph` returns `ModuleData.virtualSources` keyed by dependency key. `_addDependency` copies each into `#virtualSources`, keyed by module path, and deletes the entry when an added or changed edge carries none; `_freeModule` deletes it, and `_moduleSnapshot` includes it so a rollback restores it.
- **Revisiting without the importer.** `_buildDelta` passes `#virtualSources` to `buildSubgraph`, which hands each root's entry to the transform, so a module with no file that is a traversal root (an importer of a deleted file, a matching context module, every module on a reset) is re-transformed from the source the graph holds without re-transforming its importer. In `TransformFn = (modulePath, ?VirtualSource)`, a `buffer` module's `modulePath` is its bare virtual path (`getModuleSourcePath`), and its own dependencies resolve from that path, so transformers and resolvers never see `virtual:`. A context module's `?ctx=` path is passed unchanged.
- **Content changes.** `dependenciesEqual` compares paths, so an edge whose virtual module changed content looks unchanged. Instead, `traverseDependencies`' traversal filter compares a `buffer` edge's `sha1` with the one `#virtualSources` holds, and on a mismatch revisits the module and commits it as `modified`. The new source is stored only after a successful commit; a rollback keeps the old one. `metro:inline` names each module after a hash of its source, so an edit there yields a new path instead — an add, a delete and a modified importer.
- **Conflicts.** `buildSubgraph` records an error for the importer when two edges in one traversal resolve one virtual path to different SHA-1s ("A resolver must produce the same source for the same virtual path."). The check lives for one `buildSubgraph` call, so an importer that isn't revisited is never compared.
- **Gap: a root ignores new content.** Every root is marked visited before any transform completes, and a root with no file is visited from its stored source. If an importer revisited in the same traversal resolves it to different content, the filter returns `true` before comparing (the path is already modified) and the visited check skips it, so the new content is neither transformed nor stored, and nothing reports it. On a reset every module is a root, so a virtual module whose content changes in that build keeps its old content until its importer is next revisited while it is not itself a root. (Traced in code, not reproduced. `metro:inline` is immune, since its paths are content-hashed.)

**A new kind of module with no file is a new arm of this union, not a second side channel** — with a graph path no file can have, its source carried on the edge into `fileBuffer`, and a stated answer to what revisits it when its content changes.

## `DeltaCalculator` — `packages/metro/src/DeltaBundler/DeltaCalculator.js`

One per graph. Subscribes to the file map's `'change'` event and accumulates `_addedFiles` / `_modifiedFiles` / `_deletedFiles` between builds. `getDelta` awaits any in-flight `_currentBuildPromise` ("delta server doesn't support concurrent builds"), then swaps the sets for fresh ones, so changes that arrive mid-build wait for the next delta.

Details worth internalizing:

- **Event folding.** `#markAdded` / `#markModified` / `#markDeleted`: deleted+added ⇒ modified; added+modified ⇒ still added; added+deleted ⇒ no change; modified+deleted ⇒ deleted. The transitions keep the three sets disjoint. Getting this wrong leaves a stale module in the graph, or tries to transform a deleted file.
- **Failure must not lose changes — or reorder them.** If the build throws, the pending sets are restored: otherwise "asking for a delta after an error will produce an empty Delta, which is not correct". Changes that arrived during the build happened after the ones it took, so the taken sets become the base and the later changes are **replayed through the same transitions** (set by set: added, then modified, then deleted — safe because the sets are disjoint). Merging by plain union is unsound: a file modified before the build and deleted during it ends up in both `modified` and `deleted`, and the next build tries to transform a deleted file. The restored base is the same set object the failed build augmented in place — with every graph module if a reset was pending, plus importers of deleted files and matching context modules, and with recreated files moved from added to modified — which is why `_requiresReset` is cleared when taken and never restored.
- **Success can leave stale state too.** A file deleted and recreated during a build leaves that build green and the old module in the graph, and an added path is not itself a traversal root. So the next build *moves* an added file the graph already holds from added to modified: "It is modified, not added, since it never left the graph." Moving rather than copying keeps the sets disjoint for the failure path — in both sets, a deletion replayed after a failed build cancels the addition and leaves a deleted file modified, so every later build fails to get its SHA-1. Any accumulate-then-drain design must say what happens to events that land during a drain, on both paths.
- **`#shouldReset`** — some changes invalidate resolutions we can't identify, so they force a full revisit: **symlink changes**, and (with `unstable_enablePackageExports`) any `package.json` change. A conservative reset is the right answer when you can't enumerate the affected modules; an unsound delta never is.
- `getDelta({reset})` with `reset: true` returns the whole graph as `added` with `reset: true` — how the initial build is produced (`DeltaBundler.buildGraph`, and `IncrementalBundler`'s initial revision). Server and HMR updates call `updateGraph(revision, false)`.
- `DeltaCalculator` emits `'change'` for every file map event, whether or not the file is in its graph, so each `HmrServer` client group runs an update cycle (debounced by 50ms) even for changes outside its graph.

## `Transformer` — `packages/metro/src/DeltaBundler/Transformer.js`

The cache-key composition is covered in `references/caching.md`. Architecturally:

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
- **Output must not depend on the worker.** `Worker.flow.js` deletes `process.env.JEST_WORKER_ID` when it loads: "jest-worker assigns the ID of the worker to JEST_WORKER_ID in a worker thread, regardless of the `env` it is given. Transform output must not depend on the worker that produced it, and a Babel plugin may inline any environment variable." The rest of the environment reaches workers unkeyed (`references/caching.md`, known gaps).
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
- A virtual module's URL "is the one a file at its virtual path would have, plus this parameter" (`virtual=true`, `VIRTUAL_MODULE_URL_PARAM`), wherever one is derived: `ServerUrl` source paths (`Server._getModuleSourceUrl`, through `RootUrlMap` on `getModuleSourcePath`), HMR URLs and async bundle paths. The pathname may be a real file's, so such a URL must never be served as that file; what the server does with one is in SKILL.md's seams table.

Asset URL paths likewise have one derivation, `getAssetUrlPath` (`metro/src/Assets.js`), run per asset in `Transformer`. Metro's transformer bakes in the path it receives (or the project-relative path when it gets none) and reports it as `unstable_assetUrlPath` on the asset's output ("A temporary migration helper, to be removed."); `getAssets` (what a build tool saves) reads it back, falling back to the project-relative path ("Transformers other than Metro's use the project-relative path."). So a wrapper transformer that forwards only five arguments still gets matching paths in the bundle and in `getAssets`, and `getAssets` needs no roots.

## Adding something to this layer

- **New per-module data?** It must come out of the transform (so it's cached with the code) or be derivable from the graph. Don't compute it per-request in the serializer — serializers run over every module on every request.
- **New edge type?** Update `dependenciesEqual`, the three-step commit diff, and the GC — and add cycle tests.
- **New kind of module with no file?** A new `VirtualSource` arm, carried on the edge into `fileBuffer` so the content hash covers it, under a graph path no file can have (a `lib/modulePath.js` namespace, or a suffix like `?ctx=`), with a stated answer to what revisits it when its content changes — `dependenciesEqual` compares paths, so a content change at a fixed path needs its own check, as the `buffer` arm's SHA-1 comparison is. Derive every file path, URL and name from its key through `lib/modulePath.js`.
- **New change source?** Route it through `DeltaCalculator` so it participates in folding, reset and error recovery. Don't add a second subscription to the file map.
- **New bundle variant?** It needs a distinct `getGraphId`, or it will silently share a graph revision with a different configuration.
- **New long-lived resource?** Acquire it last, and release it in `end()` even after a failed start. A graph is one: end it when you're done with it. **New URL mapping?** Put it in `RootUrlMap`, so `Server` and `HmrServer` agree. **New server behavior?** A middleware hook, not a wrapper around `processRequest`.
- **Test incrementally.** A test that builds once proves nothing here: mutate, rebuild, assert the delta; then delete, rebuild, assert again — and land a change mid-build, on a failing and a passing build.
