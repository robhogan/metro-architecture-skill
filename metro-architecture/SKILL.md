---
name: metro-architecture
description: |
    Metro's core architecture and the design principles behind it — the virtualized file system (metro-file-map / TreeFS, crawlers, watcher backends, dynamic roots), fast in-memory resolution (metro-resolver, DependencyGraph, package exports and the `browser` field, the `metro:` scheme), the layered cache system and its invalidation model, incremental graph traversal (DeltaBundler, require.context and virtual modules — `VirtualResolution`, the `virtual:` namespace, `metro:inline`), and cheap serialization (v8 snapshots, tuple metadata, VLQ module maps composed as index source maps, symbolication, HMR). This is the STARTING POINT for designing, reviewing or debugging any change to Metro core — read it before proposing an approach, not after. Triggers on any work in react/metro (formerly facebook/metro) or the `metro-*` packages: "add a resolver option", "why is Metro slow to start", "cache isn't invalidating", "add a field to the file map", "make this incremental", "design X in Metro", "review this Metro PR", "should this be cached", "watch mode isn't picking up changes", "source map is off by a line", plus anything touching TreeFS, HastePlugin, FileMap.addRoot, Watchman / NativeWatcher / FallbackWatcher, DeltaCalculator, Graph, Transformer, serializers, source maps, HMR, server middleware, or cacheStores.
---

# Metro core architecture

> **Provenance.** Derived from `react/metro` at [`4159bf41f092b5ce47252fe47808ddddf44f5ef1`](https://github.com/react/metro/commit/4159bf41f092b5ce47252fe47808ddddf44f5ef1) — the commit after Metro 0.88.0, committed **2026-11-06**, last verified **2026-11-06**. Covers this file and everything in `references/`.
>
> Rules for keeping this honest, which must survive any future update:
>
> - **The pinned hash must be public** — a commit that exists on `main` in `react/metro`, never a local, unlanded or work-in-progress commit. It has to resolve for anyone with a fresh clone.
> - **Describe only landed code.** If the working tree contains unlanded changes to a file this skill cites, document the public version. (Open work that builds per-resolution observation collection on the seams described here is deliberately absent for this reason.)
> - **Give each fact one home.** State a mechanism in the reference that owns it and link to it from elsewhere; restating it is how one commit makes six passages wrong. Volatile status ("nothing on main does X") lives only in [Seams and their status](#seams-and-their-status).
> - **To find what may have drifted**, list the commits since the pin that touch the code cited here. Sapling: `sl log -r "(4159bf41f092::remote/main - 4159bf41f092) and (file('glob:packages/metro*/src/**') or file('glob:docs/**'))"`. Git: `git log --oneline 4159bf41f092..origin/main -- ':(glob)packages/metro*/src/**' docs`.
> - **When you refresh the pin**, re-verify the quoted comments and constants — not just the paths — update both dates, and re-check every row of the seams table.

Metro is a **long-lived, incremental, content-addressed bundler for very large repos**. Every architectural decision in it falls out of that one sentence. Before you design anything here, internalize the problem shape:

- **Scale.** Hundreds of thousands of files in the watched roots; tens of thousands in a bundle. Anything O(project) per request is a bug in waiting. Anything allocating one object per file is a memory regression.
- **Lifetime.** The dev server runs for hours. Cold start happens once; warm rebuilds happen thousands of times. The steady state is *one file changed*.
- **The critical path is I/O and memory, not CPU.** Metro's hot loops are map lookups and string slicing, and its slow paths are `fs` syscalls, worker IPC, and deserializing caches.
- **Correctness is invalidation.** Almost every Metro bug that reaches users is either a stale cache or a resolution that didn't get re-run. Speed that costs correctness is worthless because the workaround — `--reset-cache` — destroys the speed anyway.

## The pipeline

```
watchman / node crawl / custom crawler        packages/metro-file-map/src/crawlers/
  + watcher backends (Watchman, Native, Fallback)        src/watchers/
        │  CrawlResult { changedFiles, removedFiles, clocks }
        ▼
FileMap.build()                               packages/metro-file-map/src/index.js
   ├─ read v8 cache snapshot ─────────────►   cache/DiskCacheManager.js
   ├─ TreeFS ◄── the virtual file system ─►   lib/TreeFS.js
   ├─ FileProcessor (workers: plugins; sha1)  lib/FileProcessor.js
   ├─ plugins: Haste, Dependency, Mock, …     plugins/   (workers eager or lazy)
   └─ addRoot(): dynamic roots, crawled and watched after build
        │  FileSystem interface + 'change' events (net, via FileSystemChangeAggregator)
        ▼
DependencyGraph                               packages/metro/src/node-haste/DependencyGraph.js
   ├─ _resolutionCache (4-level Map tree; virtual results per origin file)
   ├─ PackageCache (parsed package.json contents, per path)
   └─ ModuleResolver ──► metro-resolver       packages/metro-resolver/src/resolve.js  (PURE)
        │  BundlerResolution: sourceFile {filePath} | virtualModule {filePath: 'virtual:…', source, sha1}
        ▼
Transformer                                   packages/metro/src/DeltaBundler/Transformer.js
   ├─ key = stableHash(config ∥ path ∥ assetUrlPath (assets only) ∥ options) ++ sha1(content)
   ├─ Cache(cacheStores)                      packages/metro-cache/src/Cache.js
   └─ WorkerFarm ──► metro-transform-worker
        │  TransformResultWithSource { dependencies, output (code + VlqMap), getSource, unstable_transformResultKey }
        ▼
Graph / DeltaCalculator                       packages/metro/src/DeltaBundler/{Graph,DeltaCalculator}.js
   └─ refcounted, cycle-collecting module graph keyed by module path (file, `?ctx=`, `virtual:`); emits DeltaResult {added, modified, deleted, reset}
        │
        ▼
Serializers                                   packages/metro/src/DeltaBundler/Serializers/
   ├─ baseJSBundle → bundleToString (lib/) → code
   ├─ sourceMapGenerator (VLQ module maps → index-map sections, no re-encode)
   └─ hmrJSBundle (delta only)
        │
        ▼
IncrementalBundler revisions / Server         packages/metro/src/{IncrementalBundler,Server}.js
   └─ RootUrlMap: URL pathname ↔ file for sources, assets and entries   packages/metro/src/lib/RootUrlMap.js
```

Two rules follow from the diagram alone:

1. **Data flows one way and deltas flow with it.** Every stage consumes a delta and produces a delta. If your design only describes a full build, it isn't finished. (Serialization is the exception that proves the rule — see `references/serialization.md` §4.)
2. **Layers below `DependencyGraph` do not know about bundles, and layers above do not ask `fs` about the file tree.** Keep it that way; the few exceptions are listed in §1. Roots beyond the configured `watchFolders` reach the server from the file map (`FileMap.getRoots()`, via `DependencyGraph.getDynamicRoots()`), not from a list the server keeps itself.

## Seams and their status

Landed mechanisms that are partly or wholly unused on main. Build on these rather than inventing parallel ones, and update this table — and only this table — when their status changes. Status as of the pin.

| Seam | Where it's described | Status on main |
|---|---|---|
| `Observations` on `FileSystem.lookup`, `hierarchicalLookup` and plugins' `getOrComputeFileData` | `references/file-map.md` | No caller passes one. Nothing records enumerated directories yet, though the contract covers them. `resolver.unstable_incrementalResolution` (default `false`) reaches `ModuleResolver`'s `Options` and DeltaBundler `Options` and is read by nothing; it is not a `ResolutionContext` member. |
| `FileMap.addRoot` / dynamic roots | `references/file-map.md`, `references/graph.md` | Used once, at startup: `DependencyGraph._addMetroRuntimeRoots` adds Metro's own `metro-runtime` and its `@babel/runtime`. Default config depends on them, since the module system, `asyncRequireModulePath` and `emptyModulePath` default to `metro:runtime/…` specifiers (`references/resolution.md`). Added roots aren't persisted or keyed. No root removal. |
| Lazy plugin workers (`lazy: true`) | `references/file-map.md` | No built-in plugin is lazy. |
| `VirtualSource` (modules with no file) | `references/graph.md` | Two arms: `requireContext` (behind `transformer.unstable_allowRequireContext`, default `false`) and `buffer`, a virtual module's source. |
| Virtual modules (`VirtualResolution`, `virtual:` paths, `metro:inline`) | `references/resolution.md`, `references/graph.md`, `references/serialization.md` | Off by default: `ModuleResolver` throws on a `virtualModule` unless `resolver.unstable_allowVirtualModules` (default `false`), and a `null` `virtualPath` fails an invariant. Metro's only producer is `metro:inline`, which nothing in Metro emits — Babel plugins do, told by `supportsInlineModules` caller data. The server serves no virtual module by URL, though HMR URLs, async bundle paths and `url-server` source paths carry `virtual=true`: `parseBundleOptionsFromBundleRequestUrl` throws ("Virtual modules cannot be requested by URL yet."), failing `.bundle`, `.map` and `/symbolicate` requests for one, and a source request with the parameter passes to `next()`. So a lazily imported virtual module can't be loaded. |
| `serializer.unstable_lazilyDefineModules` / `baseJSBundleWithLazyModuleDefinition` | `references/serialization.md` | Serializer exists; nothing reads the option (default `false`) and `Server` never calls the serializer, though `docs/Configuration.md` documents the option as making "the default serializer" define modules lazily. |
| `reservedString` (`_$$_METRO_`) placeholders | `references/serialization.md` | Passed to serializers as `options.reservedString`; no Metro serializer reads it. The worker emits `_$$_METRO_MODULE_ID` (in `$SHBuiltin.moduleFactory(…)`) only under `customTransformOptions.unstable_staticHermesOptimizedRequire`, for a custom serializer to replace. |

---

## The nine principles

### 1. The file system is a data structure, not a syscall

`TreeFS` (`metro-file-map/src/lib/TreeFS.js`) is an in-memory tree of `Map`s keyed on path segments, rooted at the project root, with `'..'` entries for anything outside it. Every existence check, type check, `realpath`, glob and package-scope walk in the resolution path is a walk over that tree, so **those questions cost zero syscalls.** The bounded exceptions in resolution are `package.json` *contents*, which `PackageCache` reads with `readFileSync` once per path and invalidates from change events, and the origin file `UnableToResolveError` reads for its code frame (error path only).

That single fact is what buys Metro:

- speed (`#lookupByNormalPath` is a few `Map.get` calls plus string slicing — "this code is extremely hot during resolution, being the most expensive part of a file existence check. Benchmark any modifications!"),
- determinism (two identical graphs resolve identically regardless of disk timing),
- testability (fixtures are literal object trees),
- and, crucially, **invalidation for free** — the only way file state changes is a change event, so there is exactly one place to hook.

The corollary: resolution can only see the file map. A path outside its roots is `{exists: false}` however real it is on disk — until something adds it as a root with `FileMap.addRoot`.

**Design rule:** If you need to know something about a file, add a method to the `FileSystem` interface (`metro-file-map/src/flow-types.js`) or a `FileMapPlugin`. Never reach for `node:fs` above the file-map layer. Apart from `PackageCache`, the existing exceptions are deliberate, and all sit outside every invalidation mechanism. Content reads: the transform worker reads the file it transforms (reusing the content `getOrComputeSha1` returns when it had to read it), `getSource` re-reads a module's source for source maps, the server streams source and asset files, error and symbolication code frames read source files, and an asset lists and hashes its scale variants with `readdir` — outside the cache key too (§6). Tree questions: `IncrementalBundler` checks entry files with `fs.realpath`, `Transformer` asserts each watch folder is a directory, and `lib/metroRuntime.js` / `lib/metroBabelRuntime.js` locate Metro's runtime packages with `require.resolve`, memoized for the process. Add to these only with a reason as good.

`hierarchicalLookup` is the model to copy: instead of exposing raw primitives and making callers loop, it pushes the whole "walk up until you find `package.json`, stopping at `node_modules`" pattern into one traversal that collects the ancestor nodes and the real path on the way, then checks each candidate from the node it already holds — one `Map.get` for a single-segment subpath such as `package.json` (#1946: ~2.1x per walk, −13% first-pass resolution). **Push the loop into TreeFS; don't pull nodes out of it — and once you hold a node, don't re-walk from the root.**

### 2. Optimize the warm path; amortize the cold one

Cold start is dominated by crawl + cache deserialization. Warm rebuild is dominated by nothing at all if you do it right — the work should be proportional to the change.

Concretely, cold-start costs are paid *once* and are allowed to be expensive if they make the warm path cheap: running (non-lazy) plugin workers over every file, building the Haste map, resolving each symlink's target to a normal path when its node is populated rather than on every traversal. The converse also holds — don't pay per file for what most files never need: by default a file's sha1 is computed lazily, in-band, the first time it is transformed (`watcher.unstable_lazySha1`, default `true`), not for every crawled file. Warm-path costs are paid thousands of times and must be near-zero: a resolution cache hit, a transform cache hit keyed on content, a graph delta touching three modules.

Not everything warm is incremental: every `.bundle` and `.map` request re-serializes the whole graph, and only HMR is proportional to the delta (`references/serialization.md` §4). Per-module work in a serializer is paid per module, per reload.

**Design rule:** When you add work, state which path it's on. Work on the warm path needs a justification and a measurement. If you find yourself adding something per-file to the warm path, you almost certainly want it precomputed on the cold path and stored in the file map.

`RootPathUtils` exists purely because of this: `path.relative` and `path.resolve` were hot enough to matter, so Metro has specialized versions (~8x and ~20x faster respectively) that exploit the "many paths, few roots" assumption. Watcher event filtering is the same instinct: the only things a backend includes files by are extensions, one basename (`package.json`) and one prefix (the health-check file's), so it checks `Set`s and `startsWith` (`WatcherIncludedFiles`, `watchers/common.js`) rather than running a general glob matcher per event. That's the right kind of optimization — a narrower contract in exchange for a constant factor on a hot path, documented where it lives.

### 3. Deltas all the way down

- Crawlers return `{changedFiles, removedFiles}` against a previous state, not a full listing. Watchman clocks make this a server-side query; otherwise `FileSystem.getDifference()` derives it from a listing — an mtime match is unchanged, and where the crawler supplies a sha1 (Watchman with `unstable_lazySha1: false`, or a custom crawler), a sha1 match against a visited tuple updates the mtime without re-visiting.
- In watch mode, a change made while the file map builds is not lost: non-Watchman backends start before the crawl — confirmed live first where they aren't live on start (`NativeWatcher`) — and buffer their events for replay, and Watchman subscribes `since` the crawl's clock (`references/file-map.md`).
- `FileMap` folds each watch batch into its **net** effect (`FileSystemChangeAggregator`: remove-then-add is a modification, add-then-remove is nothing) and emits one `ChangeEvent` whose `changes` carry added/modified/removed files and added/removed directories. Directory events are synthesized from `TreeFS`'s own mutations, not taken from watcher backends.
- `DeltaCalculator` accumulates those into `_modifiedFiles` / `_addedFiles` / `_deletedFiles` through explicit transitions (`#markAdded`, `#markModified`, `#markDeleted`) and, on the next build, traverses only from those roots and the modules they implicate (importers of deleted files, recreated files the graph still holds, matching `require.context` modules).
- `Graph.traverseDependencies` returns `{added, modified, deleted}` — including as `modified` a virtual module whose importer re-resolved it to different content, though no file event names it (`references/graph.md`).
- HMR serializes only the delta (`Serializers/hmrJSBundle.js`).

**Design rule:** Define the delta type of your new component before its full-build behavior — and define it as **transitions, not just sets**. Two batches of changes compose by replaying one through the transitions, never by set union. When a `DeltaCalculator` build fails, the sets it took are restored and the changes that arrived meanwhile are replayed on top; a union instead leaves a file modified before the build and deleted during it in both `modified` and `deleted`, and the next build transforms a deleted file. **Say what happens to changes that land while a build is in flight**, on both the failing and the passing path (`references/graph.md`).

And define what forces a full revisit: `DeltaCalculator._requiresReset` exists because some changes (any symlink change, and any `package.json` change under `unstable_enablePackageExports`) may invalidate resolutions Metro can't identify. Then every module in the graph is marked modified and revisited — conservative, but still emitted as an ordinary module delta. A conservative revisit is an acceptable answer — an *unsound* delta never is.

### 4. Represent, don't allocate

At 100k+ files, per-file object overhead is the memory budget. Metro's answer is flat tuples with constant indices:

```js
// metro-file-map/src/constants.js
{MTIME: 0, SIZE: 1, VISITED: 2, SHA1: 3, SYMLINK: 4, PLUGINDATA: 5}
// metro-file-map/src/flow-types.js (slot labels and comments elided)
export type FileMetadata = [?number, number, 0|1, ?string, 0|1|string, ...];
```

Not `{mtime, size, visited, sha1, symlink}`. The header comment in `metro-file-map/src/index.js` is explicit that this is to "save storage space and reduce parse and write time".

Other instances of the same instinct, all worth copying:

- Plugins get *numbered slots* appended to the tuple (`dataIdx`, allocated from `H.PLUGINDATA`), not a nested object. A slot is `undefined` until the plugin's worker has run on that file and `null` when it ran with nothing to store — two states the v8 snapshot preserves and JSON does not, which is what lets a plugin's worker run lazily, only on the files it asks for.
- The symlink slot holds the target already resolved to a root-relative normal path with POSIX separators — derived once, stored in the tuple, serializable and portable, rather than kept in a side table and re-derived (`references/serialization.md` §1).
- `CountingSet` (`metro/src/lib/CountingSet.js`) stores `inverseDependencies` as a multiset, so that repeated edges from one parent are counted, while its `size` counts *unique* items — and that size (+1 for an entry point) *is* the module's refcount. No separate refcount field per module.
- Derived data that's expensive but reconstructible lives in a `WeakMap` keyed on the object it derives from, so it costs nothing once that object dies: `metro-resolver` normalizes a package's `exports` once per parsed manifest (`_normalizedExportsFields`), and `Server/symbolicate.js` keeps a per-line index of a module's mappings keyed on its `VlqMap`, which "releases it automatically when the module leaves the graph". This works because `Graph` replaces a module object, and its output, whenever the module changes.
- Paths are stored **root-relative and normalized**, sharing the prefix implicitly via the tree rather than repeating it in every string.
- `bulkAddOrModify` caches the last directory node because bulk file data is clustered by directory — "significantly (~30%) faster than caching all lookups in a Map, and 70% faster than no cache". Note the shape of that comment: a micro-optimization with the measurement attached.

**Design rule:** Adding a field to `FileMetadata` for everyone is a real cost. Ask whether it belongs to a plugin slot (a lazy one, if few files will ever need it), a lazily-computed `WeakMap`, or nothing at all.

### 5. The in-memory shape *is* the cache format

This is the "cheap serialization" principle, and it is the one most often violated by well-meaning changes.

`TreeFS.getSerializableSnapshot()` returns a structural clone of the live tree. `DiskCacheManager` hands it straight to `v8.serialize()`. On read, `v8.deserialize()` produces the tree, and `TreeFS.fromDeserializedSnapshot()` adopts it **as-is** — no validation, no schema migration, no object construction pass:

```js
// metro-file-map/src/index.js — trust, don't validate
// Typed `mixed` because we've read this from an external
// source. It'd be too expensive to validate at runtime, so
// trust our cache manager that this is correct.
```

The consequences you must respect:

- **Everything reachable from a cache must be v8-serializable**: primitives, `Array`, `Map`, `Set`, plain objects. The `V8Serializable` type in `flow-types.js` is the enforcement mechanism — plugin state and per-file plugin data are typed against it. No class instances, no closures.
- **Test deserialized data by shape, not by class.** Under Jest, `v8.deserialize` returns host-realm `Map`s, so `instanceof Map` fails; `TreeFS` tells directory nodes from file tuples with `!Array.isArray(node)`.
- **No JSON for large structures.** JSON of a 100k-entry map is a parse-time cliff at startup; v8 structured clone of the same data is dramatically cheaper and preserves `Map`/`Set` and `undefined`-vs-`null`.
- **The snapshot must be portable** between root directories and operating systems — root-relative paths, and POSIX separators in the one path string inside the tree. Prove it with a cross-OS round trip, not a single-platform fixture (`references/serialization.md` §1).
- **Writes are lazy, sequenced and debounced** — and after the first, Metro writes only in watch mode (auto-save), so a one-shot build never persists lazily computed sha1s or plugin data (`references/file-map.md`, "Cache manager").

The same instinct runs through the bundle output side (`references/serialization.md` §4–5):

- Module code is a string produced once by the transform worker and carried unchanged through the graph and the cache. The serializer never parses it, and its edits (completing the `__d(...)` call, inlining module ids) are **position-preserving**, so the worker's source map is used unadjusted.
- A module's source map is a `VlqMap` (`{mappings, names}`), encoded once in the worker and carried through the cache and the graph as that string. Whole-bundle maps are **index maps** with one section per module, each `VlqMap` passed through verbatim — no decode, no re-encode, never re-derived from the code.
- Consumers follow suit: the `Consumer` decodes each mapping once, in a single pass; `composeSourceMaps` encodes its result directly; `/symbolicate` indexes a module once, then decodes only the line each lookup needs.

**Design rule:** If serializing your structure requires a conversion pass, you designed the structure wrong. Change the structure so the wire format and the working format are the same thing — and bump the cache breaker (see §6).

### 6. Cache keys are composed, content-addressed and total

Metro's transform cache key is the canonical example (`DeltaBundler/Transformer.js`; every term is spelled out in `references/caching.md`):

```
partialKey = stableHash([
  baseHash,                       // ← getTransformCacheKey: metro version, config.cacheVersion,
                                  //   hash of the transformer module file, transformer.getCacheKey(),
                                  //   globalPrefix — computed once, in the Transformer constructor
  posix(projectRelativePath),     // ← path affects transformer behavior, so it's in the key
  assetUrlPath,                   // ← assets only (else null): the URL path baked into an asset
  ...the exact transform options that reach the worker
])
fullKey = partialKey ++ sha1(fileContents)     // ← content-addressed
```

Read the properties off that:

- **Content-addressed, not mtime-addressed.** A touched-but-unchanged file is a cache hit. This is also what makes the remote cache work across machines and CI.
- **Portable.** Project-*relative*, posix-separated paths. An absolute path or a native separator in a cache key silently destroys remote-cache hit rates and Windows/macOS sharing.
- **Total, by intent.** Every input that can change the output belongs in the key — including the transformer's own source (`getCacheKey([require.resolve(transformerPath)])`) and its self-declared key. The `extra` keys check in `transformFile` throws if an unknown transform option appears, precisely so an input can't sneak past the key. Inputs that reach the worker *outside* the options object, like `assetUrlPath`, can't be caught by that tripwire, so they are added to the key by hand. It isn't fully total today — `references/caching.md` lists the known gaps. Don't add to that list.
- **Scoped.** An input goes into the key of exactly the outputs it affects. Only an asset's output depends on `watchFolders` and the file map's roots (through the URL path baked into it), so: "that dependency enters the cache key per asset here and not in the base hash". Putting it in `baseHash` is just as correct and misses every module in the project whenever a root changes.
- **Layered.** `metro-file-map` has its own `CACHE_BREAKER`, bumped "whenever a code change to `metro-file-map` itself would cause a change to the cache data structure and/or content" — Metro's version isn't in that key, so nothing else separates releases; each `FileMapPlugin` contributes `getCacheKey()`; `Cache` composes multiple `cacheStores` and writes through to every store above the one that hit (tracked in a `WeakMap` on the key `Buffer`, which `Transformer` therefore reuses unless the sha1 changed).
- **Doubles as identity.** The full key (as a string) is returned as the module's `unstable_transformResultKey`, so `Graph` can tell a re-transformed module with identical inputs — and so, by purity, identical output — from a real change and keep it out of the delta.

**Design rule:** When you add an input to any stage, you have added it to that stage's cache key. Find the key and update it in the same change, scoped to the outputs it affects. When you change a serialized *format*, or what a cached value means for the same inputs, bump the breaker in the same commit — and since the transform cache has no breaker constant, make sure a module that changes transform output is one the transformer's `getCacheKey` hashes (`references/caching.md`, breakers). A stale-hit bug costs a user hours and gets reported as "Metro is broken".

### 7. Invalidation is a design output, not an afterthought

For every cache you introduce, you must be able to answer three questions in the PR description:

1. **What invalidates it?**
2. **How does it learn?** (a `'change'` event, a key that includes the input, or a process restart — those are the only three. An identity-keyed `WeakMap` over immutable input is the second.)
3. **Is it conservative or precise?** Both are legitimate; being unable to say is not.

Metro contains several points on that spectrum, deliberately:

- **Coarse and correct:** `DependencyGraph._onHasteChange` throws away the entire `_resolutionCache`, invalidates `PackageCache` for every changed path, and rebuilds the module resolver on *any* file change. Resolutions depend on the existence of files that were never resolved to, so precise invalidation is genuinely hard; blowing it away is cheap relative to a rebuild and cannot serve a stale entry. But **clearing is not re-resolving**: an entry is only recomputed when `DeltaCalculator` revisits its importer, and it ignores changed paths outside the graph — so creating `Foo.ios.js` beside an already-resolved `Foo.js` changes nothing until the importer is revisited for another reason.
- **Precise by construction — don't memoize what's cheap to ask:** `PackageCache` caches only parsed `package.json` contents, evicted when a change event names the file; the closest-package question goes to `TreeFS` on every call, so there is no derived answer to go stale. Its old closest-scope memo, with a reverse index and a negative-result set, missed a *nearer* `package.json` and a retargeted directory symlink. #1948 removed it rather than fix it: a correct memo has to record and replay what each entry observed, and the walk had become cheap (`references/resolution.md` §2).
- **Precision-ready, unconsumed:** `lookup`, `hierarchicalLookup` and plugins' `getOrComputeFileData` take an optional `Observations` record, split into `existence` and `content` by the events that invalidate them. Nothing passes one yet; if you're designing fine-grained resolution invalidation, build on it, not on a new seam (`references/file-map.md`).

**Design rule:** Prefer conservative invalidation you can prove over precise invalidation you hope is complete — then make it precise behind a flag, with the coarse path still available. A cache of a file-system *answer* (existence, scope, realpath) must record everything the answer depended on, including paths that *didn't* exist and symlinks traversed; indexing an entry only under the file it found misses both. When you record what something depended on, **classify each observation by the events that can invalidate it**; one flat "invalidated by" set forces a consumer to either over-invalidate on every edit or miss the cases (like a symlink retarget) that only a modification reveals.

### 8. Pure cores, effectful shells

`metro-resolver` is a **pure function** of `(ResolutionContext, specifier, platform)`. It does no I/O and holds no state that can go stale; every capability it needs is a member of the context (`fileSystemLookup`, `getPackage`, `getPackageForModule`, `resolveAsset`, the Haste lookups, `schemeResolvers`, …), bound by `createModuleResolver` to `TreeFS`, `PackageCache`, `HastePlugin` and config, and assembled per call by `ModuleResolver.resolveDependency` (`ModuleResolution.js`).

That purity is why resolution can be memoized at all, why `resolveRequest` can be layered on top without the core knowing, why the resolver has exhaustive unit tests over in-memory fixtures, and why resolution errors can enumerate exactly which candidates were tried (`FileCandidates`, `formatFileCandidates`). It is also why a custom `resolveRequest` should observe the file system only through the context it is given: anything it reads directly is invisible to every invalidation mechanism above.

The transform worker has the same shape: a pure function of `(source, options)` plus explicitly keyed extra inputs — which is exactly what makes its output safe to content-address (§6).

**Design rule:** New capability needed in the resolver? Add a member to `ResolutionContext`, bind it in `createModuleResolver`, and thread it through `ModuleResolution.js` (`references/resolution.md` has the checklist). Do not import `fs`, a cache, or a config object into `metro-resolver`. Same test: could this function be called twice with the same arguments and be required to return the same answer? If yes, keep it pure and cache outside it.

### 9. Parallelize off the main thread, and yield when you can't

The Metro server must stay responsive while building. So:

- Per-file metadata extraction at crawl time (plugin workers such as dependency extraction and Haste names, plus sha1 when computed eagerly) runs batched across `jest-worker` threads via `FileProcessor` — which is also why plugin workers are declared as a `modulePath` + JSON `setupArgs` rather than a closure. When one worker will do, and for every single-file watch-mode change, it runs in-band behind the same interface, so small rebuilds never pay worker startup.
- A plugin whose data only a small, unpredictable subset of files ever needs declares its worker `lazy`: it isn't run at crawl or change time at all, and runs in-band on a file the first time the plugin asks for that file's data through `getOrComputeFileData`, and again after the file changes.
- Transformation runs in `WorkerFarm`.
- Work that is still on the main thread and unbounded yields cooperatively — gathering per-module source-map info breaks to `setImmediate` roughly every 50ms: "let's give Metro an opportunity to handle requests". Bundle serialization, the final map `JSON.stringify` and HMR serialization don't yield; that is a known cost, not a model.
- Lazy work emits a signal so caches can react: computing a sha1 or lazy plugin data emits `'metadata'`, which, with auto-save on (§5), schedules a debounced cache write.

**Design rule:** If your new work is per-file and CPU-bound, it belongs in a worker and its inputs must be serializable. If only a few files will ever need it, make it lazy rather than paying for every file. If it's unavoidably on the main thread and unbounded, it must yield.

---

## Designing a change to Metro core

Work through this before writing code. It's roughly the order a reviewer will ask about.

1. **Which layer owns this?** File map / resolution / transformation / graph / serialization / server. If your change spans layers, that's the first thing to question — most cross-layer designs in Metro are better expressed as an addition to a lower layer's interface. Check the [seams table](#seams-and-their-status) for a landed mechanism to build on.
2. **What's the warm-path cost?** Per rebuild, per module, per resolution — and per request, if it's in a serializer. If it's non-zero, what makes it worth it?
3. **What's the delta behavior?** What happens on one changed file? On a deleted file? On an added file that didn't exist when a resolution failed? On a `package.json` change? On a symlink change? On a change that lands while a build is in flight, when that build fails and when it succeeds? For a module with no file: what is its graph path, and what revisits it when its content changes with no file event?
4. **What's the memory cost at 100k files?** Per-file bytes × file count. Can it be a plugin slot, a lazy plugin, a `WeakMap`, or lazily computed instead?
5. **Does it cross a cache boundary?** If so: is it in the key (scoped to the outputs it affects), is the value serializable in that cache's format (v8 for the file-map snapshot, JSON for transform results in `FileStore`/`HttpStore`), is the breaker bumped, and does it stay portable across machines and OSes?
6. **What invalidates it, how does it learn, conservative or precise?** (§7) If it records dependencies, is each one classified by the events that invalidate it?
7. **Does it need to be pure?** If it's in `metro-resolver`, a scheme resolver (`metroSchemeResolver`, `resolveInlineModule`) or a transform worker, yes. A resolver returning a virtual module must give the same source for the same virtual path from every origin.
8. **Is it configurable, and does the config reach the cache key?** New `metro-config` options need defaults, validation, docs in `docs/Configuration.md`, and — if they affect output — inclusion in the relevant cache key (only `transformer.*` options are keyed automatically, and only by a transformer that hashes its config — `references/caching.md`). A new *per-request* option (a bundle URL parameter) must also go into `getGraphId`; config needn't, as it's fixed for a server's lifetime.
9. **Public API?** Root exports are the stable surface, gated by committed `API.md` snapshots (`yarn build-api-snapshots`, checked in CI); TS defs are generated (`yarn build-ts-defs`). `<pkg>/private/*` is Metro's explicit unstable tier — don't invent another. Prefix options not yet committed to with `unstable_`.
10. **Does it hold on Windows?** Backslashes, drive letters, and roots on another drive (`..\..\D:\store`) reach every path-handling layer. CI runs the Jest suite on Windows; write tests with posix fixtures converted for the host (`posixToSystemPath` in `metro-resolver/src/__tests__/utils.js`) rather than hard-coding separators.
11. **How is it tested?** Unit tests against in-memory fixtures for pure code; integration tests for delta behavior. **Watch-mode behavior needs a test that mutates state and rebuilds** — a single-build test proves nothing about invalidation. Pass `watch: true` explicitly: without it, a `DependencyGraph` watches only when not on CI.

## Anti-patterns

| Anti-pattern | Why it's wrong | Do instead |
|---|---|---|
| `fs.existsSync` / `statSync` above the file map | Bypasses the virtual FS: unwatched, uninvalidated, slow, non-deterministic | `fileSystemLookup` / a `FileSystem` method |
| A `Map` keyed on absolute path, cleared "somewhere" | Unbounded growth, unclear invalidation | Attach to an existing cache with a defined lifetime and invalidation hook |
| Memoizing a `TreeFS` answer beneath the resolution cache | Misses a nearer file added later or a symlink retargeted — the bug #1948 removed | Ask `TreeFS` each time; if that's too slow, record the entry's `Observations` and invalidate on them |
| `JSON.stringify` a large structure to cache it | Parse cliff at startup; loses `Map`/`Set` and `undefined` | v8 serialize the live structure (§5) |
| `instanceof` on deserialized data | Fails across realms (e.g. under Jest) | Test by shape (`Array.isArray`) |
| Absolute paths or `\` separators in a cache key or snapshot | Kills remote-cache hits and cross-platform sharing | Project-relative, posix-normalized |
| mtime in a cache key | Touch-without-change becomes a miss; CI never hits | sha1 of contents |
| A per-module input in the global base hash | Every module misses whenever it changes | Key it only on the outputs it affects (`assetUrlPath`) |
| An object per file "for readability" | Memory at 100k files | Tuple + constant index, or a plugin slot |
| A new transform option not added to the key | Silent stale hits — the worst class of Metro bug | Add it to `partialKey`; the `extra` check will catch you if you don't |
| Merging change sets by union across a build | Order lost: a modify-then-delete is transformed after deletion | Replay later changes through the same transitions |
| One flat "invalidated by" set | Can't tell add/remove-sensitive from edit-sensitive | Separate `existence` and `content` observations |
| Full graph rebuild on an event you could have deltaed | O(project) per keystroke | Route it through `DeltaCalculator` |
| Caching state that can go stale in `metro-resolver` | Breaks purity, testability, and the caller's invalidation | Cache in `DependencyGraph`; keep the resolver pure (an identity-keyed `WeakMap` over immutable input, like `_normalizedExportsFields`, is fine) |
| `path.relative` / `fs` on a `Module.path` | A virtual module's path is `virtual:<path>`: machine-dependent names, failed reads, a virtual module conflated with the file at its virtual path | `getModuleSourcePath` / `getModuleDisplayPath` / `isVirtualModulePath` (`lib/modulePath.js`) |
| A serializer edit that moves mapped code | Every later mapping in the module, or every later section, points at the wrong place | Keep line counts and mapped columns (pad, insert at the end), or produce a map for the edit |
| Precise invalidation with no proof of completeness | Stale resolution, reported as "Metro can't find my file" | Coarse invalidation now, precise behind a flag later |
| A micro-optimization with no measurement | The hot paths here are counter-intuitive | Benchmark it and put the number in a comment, as the existing hot paths do |

## Reference files

Load these when the work goes deep on a subsystem:

- `references/file-map.md` — lifecycle (including changes during the build), dynamic roots and `addRoot`, TreeFS internals, the `Observations` seam, path terminology, plugins and the plugin data API (eager and lazy), crawlers, watchers and their filtering, the cache manager, `FileProcessor`.
- `references/resolution.md` — the resolution algorithm, `ResolutionContext`, package scope, Haste, package exports and `browser` ordering, the `metro:` scheme and its defaults, assets, virtual resolutions and `metro:inline`, errors, and the caching and invalidation layers around it, including their gaps.
- `references/caching.md` — every cache in Metro, its key, its lifetime and its invalidation, in one table; the transform key in full and its known gaps; asset URL paths as a scoped key input; cache stores and remote caching; breakers; the traps; debugging.
- `references/serialization.md` — v8 snapshots and their portability contract, tuple metadata, `V8Serializable`, bundle formats, position-preserving serializer edits and reserved placeholders, bundle/map agreement, VLQ module maps and index source maps, consuming and composing maps, symbolication, HMR and async bundle URLs, module paths in output.
- `references/graph.md` — `Graph`, module paths (file, `?ctx=`, `virtual:`), refcounting and cycle collection, the commit protocol, `VirtualSource` and virtual modules, `DeltaCalculator` (including changes during a build), `Transformer`, workers, `Bundler` lifecycle, `IncrementalBundler` revisions, server composition, `RootUrlMap` and dynamic roots.

Also authoritative, in the repo itself: `docs/Concepts.md`, `docs/Caching.md`, `docs/Resolution.md`, `docs/PackageExports.md`, `docs/SourceMapFormat.md`, `docs/Configuration.md`.

When the code and this skill disagree, **the code wins** — check the file, then fix the skill.
