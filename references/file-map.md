# `metro-file-map` — the virtualized file system

Paths starting `packages/` are relative to the metro repo root; `src/` means `packages/metro-file-map/src/`, and other bare paths are relative to it.

## What it is

`metro-file-map` maintains an **in-memory model of every watched file**, kept in sync with disk by a crawler (cold) and a watcher (warm), persisted as a v8 snapshot, and exposed to the rest of Metro through the `FileSystem` interface plus a `'change'` event. Nothing above it should ever call `node:fs` to answer a question about the file *tree*.

It is a fork/descendant of `jest-haste-map`, which is why "haste" appears throughout; Haste is one plugin over a general file map.

### Deliberate `fs` reads above the file map

These sit outside every invalidation mechanism, deliberately; add to them only with a reason as good. **Content reads:** `PackageCache` reads each `package.json` once per path (invalidated from change events); the transform worker reads the file it transforms (reusing the content `getOrComputeSha1` returns when it had to read it); `getSource` re-reads a module's source for source maps; the server streams source and asset files; error and symbolication code frames (including `UnableToResolveError`'s, error path only) read source files; and an asset lists and hashes its scale variants with `readdir` — outside the cache key too (`references/caching.md`, known gaps). **Tree questions:** `IncrementalBundler` checks entry files with `fs.realpath`, `Transformer` asserts each watch folder is a directory, and `lib/metroRuntime.js` / `lib/metroBabelRuntime.js` locate Metro's runtime packages with `require.resolve`, memoized for the process.

## Lifecycle — `src/index.js`

`FileMap.build()`:

1. **Read** the cache (`CacheManager.read()`) unless `resetCache`. `DiskCacheManager` does `v8.deserialize(readFile(path))`.
2. **Construct `TreeFS`** — either `TreeFS.fromDeserializedSnapshot(...)` adopting the cached tree verbatim, or an empty one. No validation pass: "It'd be too expensive to validate at runtime, so trust our cache manager that this is correct."
3. **In parallel**: crawl for a delta against the cached state, *and* initialize plugins from the cached file system. `fileSystem` is not mutated by either. In watch mode, non-Watchman backends are started before the crawl (below).
4. **Apply the delta** (`#applyFileDelta`): on a cache hit, every file outside the configured `roots` (`TreeFS.pathsOutsideRoots`) is first added to the removals, dropping an earlier session's added roots — with a built-in crawler, files under each root's real path (`fs.promises.realpath`) are kept too, because Watchman lists a symlinked root's files under it, while a custom crawler's roots are taken as given, since its files need not be on disk; then removals; then, concurrently, `readlink` symlinks in-band and batch-process files through `FileProcessor`; then `bulkAddOrModify` into `TreeFS`, reporting to a `FileSystemChangeAggregator` whose net view goes to each plugin's `onChanged`.
5. **Validate plugins** (`assertValid()` — e.g. Haste conflicts) *before* persisting.
6. **Snapshot and persist**, then **watch** if `watch`.

**Changes during the build are not lost.** Non-Watchman backends (Native and Fallback — which also serve a custom crawler, or a Watchman crawl that fell back to the node crawler) have no clock to subscribe `since`, so `Watcher.crawl()` starts them before crawling and buffers their events; `Watcher.watch(onChange, clocks)` replays the buffer as live events after the snapshot is persisted. A backend that isn't live as soon as it starts (`isLiveOnStart()` is `false`: `NativeWatcher`, whose FSEvents stream starts on another thread) is first confirmed with a health-check cookie written into each root; an unconfirmed root only warns. Watchman backends start after the persist but subscribe `since` each watch root's crawl clock (`WatcherOptions.clocks`) — the initial subscription only; a reconnect takes a fresh clock. Either way the persisted snapshot may lag disk briefly, and the first `ChangeEvent` catches it up.

In watch mode, `#watch` processes each backend event in arrival order on `#changeQueue` (in-band, one file — the queue is shared with `addRoot` crawls) and *enqueues* it; every `CHANGE_INTERVAL` (30ms) it applies the queue to `TreeFS` **in order** and emits one `ChangeEvent` `{changes, logger, rootDir}` — added/modified/removed files and added/removed directories. A `'recrawl'` flushes the queue, re-crawls that directory via `Watcher.recrawl(subpath, fileSystem)` (never Watchman) and applies it via `#applyFileDelta`.

`CACHE_BREAKER` (currently `'14'`) is a module-level constant in `src/index.js`, and the only release-to-release component of the file-map cache key — Metro's version is not in it. **Bump it in the same commit as any change to the shape or content of cached data for the same file-system state**, including changing which files a plugin's slot is filled for (examples in `references/caching.md`, "Cache breakers and versions").

**Net change.** `lib/FileSystemChangeAggregator.js` (a `FileSystemListener`) folds a batch into its net effect: remove-then-add is a modification, add-then-remove is nothing, and a removed file keeps its pre-batch metadata. The tree holds only non-empty directories, so directory events are **synthesized by `TreeFS`**; backend events with `type === 'd'` are dropped.

## Roots and `addRoot`

`FileMap.getRoots()` returns `FileMapRoot`s (`{absolutePath, rootRelativePath, dynamic}`): the configured `roots`, then any added with `addRoot()` (`dynamic: true`) in the order added. The same array is returned until the roots change, so consumers memoize on its identity (`DependencyGraph.getDynamicRoots()`); `packages/metro/src/lib/dynamicRoots.js` hashes `rootRelativePath` into URL ids (`references/graph.md`, "URLs and roots").

`addRoot(root)` adds a directory after `build()` (a call during it waits for it):

- It resolves the path through the file map's own symlinks (`TreeFS.resolveSymlinks`), because the file system holds only real paths, and crawling a root through a symlink replaces the link with a copy of its target.
- A root equal to an existing one does nothing. A root inside an existing one is recorded (still `dynamic: true`) but has nothing of its own to crawl or watch.
- Otherwise it starts a backend for the root (`Watcher.watchRoot`, watch mode only, confirmed live first where the backend isn't live on start) *before* crawling, then — serialized on `#changeQueue` after flushing pending events — recrawls it with `Watcher.recrawl` (node or custom crawler, never Watchman), applies the result with `#applyFileDelta` and emits one `'change'` if anything changed. `end()` during the crawl rejects rather than applying it.
- Added roots are **not persisted and not in the cache key**: their files reach the snapshot only through a later write (auto-save, so in Metro only in watch mode), are dropped on the next read (step 4), and are re-crawled by that session's own `addRoot`. There is no root removal.

Who calls it on main: SKILL.md's "Unused and gated mechanisms".

## `TreeFS` — `src/lib/TreeFS.js`

A tree of `Map`s. Read the OVERVIEW comment at the top of the file; it is the spec.

- **Directory node** = `Map<basename, MixedNode>`. **File node** = `FileMetadata` tuple. `isDirectory` is `node != null && !Array.isArray(node)` — by **shape, not realm**, because `instanceof Map` fails on `Map`s `v8.deserialize`d in another realm (e.g. under Jest).
- Rooted at `rootDir`. Paths outside the root are reached through a `'..'` key, so a snapshot is **portable between root directories and operating systems** — a contract; see `serialization.md` §1.
- **Symlinks** are leaf tuples where `node[H.SYMLINK]` is `1` (unread) or the target **lexically resolved to a normal path with POSIX separators** when the node was populated (`RootPathUtils.resolveSymlinkToNormal`). It is a `normalPath`, not a `canonicalPath` — symlinks in it aren't followed and it need not exist. Traversal restarts at the root with `joinNormalToRelative(target, remainingSuffix)`. Cycles are detected with a lazy `seen` set. Why it's stored this way: `serialization.md` §1.
- `#lookupByNormalPath` is the core traversal and **the hottest code in Metro's resolution path**. It carries an explicit "Benchmark any modifications!" warning. Its options (`collectAncestors`, `observations`, `changeListener`, `followLeaf`, `makeDirectories`, `start`) are opt-in, so the common path allocates nothing — except `followLeaf`, which defaults to `true` and is opted out of.
- Because traversal follows symlinks, a successful lookup returns the **real path for free** (`realPath`, and `canonicalPath`), and a failed one returns `canonicalMissingPath`, the real path of the **first missing segment** — what `Observations.existence` records.
- Outside the `FileSystem` interface, `FileMap` also uses `resolveSymlinks(mixedPath)` (the absolute path with each in-map symlink resolved, whether or not it exists) and `pathsOutsideRoots(roots)`.

### Terminology (use these words in code and review)

| Term | Meaning |
|---|---|
| `mixedPath` | root-relative *or* absolute |
| `relativePath` | root-relative |
| `normalPath` | root-relative, normalized, may have one trailing slash |
| `canonicalPath` | root-relative, normalized, **real** (no symlinks in dirname), no trailing slash |

### The `FileSystem` interface — `src/flow-types.js`

`exists`, `lookup`, `linkStats`, `getSha1`, `getOrComputeSha1`, `getAllFiles`, `matchFiles`, `getDifference`, `getSerializableSnapshot`, `hierarchicalLookup`.

`hierarchicalLookup` is the one to study — it encapsulates "walk up from X looking for `subpath`, stopping at `breakOnSegment`", which covers both "closest `package.json`" and "closest `node_modules/pkg`". One traversal of the start path collects the ancestor nodes; each candidate is then checked from its own node — one `Map.get` when `subpath` is a single segment (Metro's `package.json` walk), else a traversal of only the subpath from the candidate (`#lookupByNormalPath`'s `start`). A symlinked child, or a candidate above the root, takes that subpath traversal too; it starts at the root only when joining collapses segments, and a symlink restarts it at the link's target. For the common root and its ancestors, the remaining path is measured from the start path when it lies under that root, else from the lookup's real path ("if the start path reaches the common root through a symlink", e.g. `app/monorepo -> ..`), and the call returns `null` if neither applies.

### `Observations` — the invalidation seam

`lookup` and `hierarchicalLookup`, and plugins' `getOrComputeFileData` (below), take one optional trailing `observations?: ?Observations`: two canonical-path sets, keyed like `ChangeEvent.changes`, split by the events that invalidate them.

- `existence` — invalidated by an added or removed file or directory. `lookup` records the path found, or the **topmost** missing path, so an absent `node_modules` is one entry, not one per package lookup beneath it. `hierarchicalLookup` records **every candidate probed, hit or miss**.
- `content` — also invalidated by modification. Traversal records each **symlink traversed** (retargeting a link is a modification: "whose content is their target"); `getOrComputeFileData` records the regular file whose plugin data it returns, since that data derives from the file's bytes. The contract also covers directories whose entries were enumerated (`matchFiles` takes no observations; see SKILL.md's "Unused and gated mechanisms").

Why this shape:

- A parameter, not a return value, so a caller accumulates across one resolution's many lookups; omitted, nothing is allocated. Failed lookups dominate resolution, so even a `Set` or a path conversion per lookup is a measurable cost (a few percent of resolution time).
- Two sets because events invalidate them differently: an edit can't change whether a path exists, but it can retarget a symlink. With one flat set a consumer must either treat every probe as edit-sensitive (over-invalidation) or miss a retarget (a stale resolution).
- Canonical paths are what change events carry, so matching is a set lookup.

Status (who passes one, what consumes it) is in SKILL.md's "Unused and gated mechanisms". It is the intended foundation for fine-grained resolution invalidation; build on it rather than inventing a parallel mechanism.

### `getOrComputeSha1` and `getDifference`

`getOrComputeSha1` returns any sha1 already in the tuple (from Watchman, an eager crawl, the snapshot or an earlier lazy computation); otherwise it processes the file — returning the content it read, so the transformer needn't read it again — and emits `'metadata'`. It mutates *the metadata tuple it originally looked up*, deliberately, so that if the file changes mid-computation the result lands on an orphaned tuple rather than caching a stale sha1 against the live one.

`getDifference(files, {subpath})` turns a listing into a delta (see Crawlers). For a recrawl `subpath` it doesn't follow a leaf symlink, and reports a file or symlink standing where the new directory is as removed — else `bulkAddOrModify` throws "already exists in the file map as a file" and watch mode drifts (e.g. unlinking a package). `subpath: ''` — a recrawl of a watch root equal to `rootDir` — compares only the tree under `rootDir`, skipping its `'..'` branch, so other roots' files aren't reported removed; omitting `subpath` compares the whole tree.

## `FileMetadata` and plugin slots

```js
// src/constants.js
{MTIME: 0, SIZE: 1, VISITED: 2, SHA1: 3, SYMLINK: 4, PLUGINDATA: 5}
```

`FileMetadata` is a variable-length tuple: fixed fields, then one slot per plugin that declares a worker, allocated in construction order from `H.PLUGINDATA`. Plugin names holding a slot must be unique — crawlers address slots by name via `pluginDataIndices`, and a duplicate silently writes into a shadowed plugin's slot (there's an `invariant` for this).

Slots are **tri-state**: `undefined` = no worker has run (fresh tuples from crawlers and watchers carry no slots, so a changed file's data resets); `null` = ran, nothing to store (a worker's `undefined` is stored as `null`). Don't conflate them.

**To store new per-file data, add a plugin with a worker, not a field on the tuple.**

## Plugins — `src/plugins/`

```js
interface FileMapPlugin<SerializableState, PerFileData> {
  name; initialize(opts); assertValid(); onChanged(changes);
  getSerializableSnapshot(); getCacheKey(); getWorker();
}
```

- `getWorker()` returns `{worker: {modulePath, setupArgs}, filter, lazy?}` — a **module path plus JSON args**, not a closure, because it has to cross into a worker thread.
- `filter({normalPath, isNodeModules})` is the only gate on which files an eager worker runs on as they are crawled or changed, `node_modules` included — a plugin that doesn't want them rejects `isNodeModules`, as both built-ins do. `DependencyPlugin` also excludes media, font and `.json` extensions (`workerExclusionList.js`).
- **Reading data: `getOrComputeFileData(mixedPath, observations?)`**, given in `initialize`, returns a `FileDataResult`: `{exists: false}`, `{exists: true, type: 'd'}` or `{exists: true, type: 'f', realPath, data}`. The plugin's `files.lookup` reports only existence and type; `files.fileIterator` still yields each file's `pluginData`. With `observations`, a regular file found is recorded in `content`.
  - For a **`lazy: true`** plugin — skipped at crawl and change time, `filter` never consulted — an `undefined` slot runs the worker synchronously in-band (`FileProcessor.processFileForPlugin`), stores the result (`undefined` → `null`) and emits `'metadata'` so the cache keeps it. A changed file's slot resets, so the worker reruns on the next ask. For small, unpredictable subsets (most `package.json` files are never read).
  - For an eager plugin, an `undefined` slot means the worker never ran on the file (e.g. `filter` rejected it), so it reads as `null` without running.
  - `FileDataPlugin` exposes it as a method; `DependencyPlugin.getDependencies` and `HastePlugin.getModuleNameByPath` use it.
- `getCacheKey()` feeds the file-map cache key. If your plugin's behavior depends on user config or a user-supplied module, that must be reflected here — see `DependencyPlugin`, which calls the custom `dependencyExtractor`'s own `getCacheKey()`. `FileDataPlugin` keys neither `lazy` nor `filter`, so changing either needs a new `cacheKey`: an old snapshot's `undefined` slots read as `null` for an eager plugin.
- `getSerializableSnapshot()` may return `null` to opt out of persistence. **`HastePlugin` does exactly this**, with the reasoning stated in the code: rebuilding the Haste map by iterating cached per-file data "turns out to have comparable performance to serialisation, at least when Haste is dense, and makes for a much smaller cache". That's the right way to make this call — measure recompute-vs-serialize, and write down the answer.
- `initialize` should yield (`await new Promise(setImmediate)`) if it loops over all files — `HastePlugin` yields every 10,000 Haste files.
- `onChanged` gets the net added/modified/removed sets, projected to the plugin's slot. `HastePlugin` **processes removals first so that moves aren't misread as duplicates** — a good example of the ordering hazards in incremental index maintenance. It also **ignores modifications**: the aggregator's `modifiedFiles` carries only the new metadata, so a Haste name changed by editing a file (a Haste package's `name`, or anything a custom `hasteImplModulePath` reads from content) isn't picked up until restart. An index keyed on derived data needs the old value to process a modification.

Built-ins: `HastePlugin` (Haste names → paths, plus a duplicates index), `DependencyPlugin` (per-file dependency strings; created only when `resolver.dependencyExtractor` is set), `MockPlugin` (Jest manual mocks), `FileDataPlugin` (exported base class for plugins that "store per-file data via a worker and have no separate serializable state").

## Crawlers — `src/crawlers/`

Contract: `(CrawlerOptions) => Promise<CrawlResult>` where the result is a **delta** against `previousState`, plus `clocks` when the crawler has them (Watchman).

- **Watchman** (`crawlers/watchman/`) — the fast path. Requires capabilities `field-content.sha1hex`, `relative_root`, `suffix-set`, `wildmatch`. Uses clockspecs so the server computes the delta; `planQuery.js` builds the query. Returns sha1s directly only with `computeSha1` on (`watcher.unstable_lazySha1: false`; not Metro's default). `planQuery` deliberately doesn't request `symlink_target`, so symlinks are read with `readlink` like any other crawler's. A failure retries once with the node crawler.
- **node** (`crawlers/node/`) — fallback. Walks the tree, then uses `FileSystem.getDifference()` to derive a delta: mtime match ⇒ unchanged, else changed. It supplies no sha1, so `getDifference`'s sha1-match path (a matching sha1 on a `VISITED` tuple updates the mtime **without revisiting**) applies only to sha1-supplying crawlers. Its hot path filters by extension before `lstat` and builds each normal path from its parent's — by concatenation, so each root is first normalized with `path.resolve` — calling `absoluteToNormal` only outside `rootDir`, where `'..'` paths can collapse back in (~33% faster).
- **Custom** — `crawlerFactory({buildParameters, pluginDataIndices})` replaces crawling only, recrawls included (honor `subpath` and narrowed `roots`). No fallback, no Watchman probe — so watching uses Native/Fallback. It may fill plugin slots and set `VISITED` to skip workers.

## Watchers — `src/watchers/`

`WatchmanWatcher` > `NativeWatcher` (macOS only: recursive `fs.watch` over FSEvents) > `FallbackWatcher` (Linux, Windows: `fs.watch` per directory), behind `AbstractWatcher` / `WatcherBackend`, one per root. Events are `{event: 'touch'|'delete'|'recrawl', relativePath, root, metadata, clock}`. `'recrawl'` means the backend can't say what changed under a directory and the file map must resynchronize it from disk — Watchman never emits one.

- **Filtering.** `Watcher` passes `included` (`WatcherIncludedFiles`: an extensions `Set`, `basenames` `{'package.json'}`, health-check `basenamePrefixes`), checked by `isIncluded` (`watchers/common.js`) with `Set` lookups and `startsWith`. Nothing else here is configurable, so a general glob matcher buys nothing on a per-event path. Only regular files are checked; an extension matches after *any* `.`, so `.env` matches `env`. The node crawler uses `path.extname`, so such dotfiles are watched but not crawled. That is the backend's filter, not the file map's: backends also test `ignored` (the flattened `blockList`) per event (`doIgnore`), and `FileMap.#watch` filters again — dropping directory events, symlinks unless `enableSymlinks`, paths `ignorePattern` matches, and regular files whose path doesn't `endsWith` one of `extensions` — so a `package.json` admitted by the backend's basename rule reaches the map only when an extension matches it too (`json` is a default source extension).
- **`NativeWatcher`** isn't live on start, so `Watcher` confirms each root's watch (Lifecycle, "Changes during the build"). It starts each event's `lstat` at once but emits through `#emitQueue` in `fs.watch` arrival order, so `delete app` can't overtake `delete app/moved-in`. A directory `rename` becomes a `recrawl`: a moved-in tree emits nothing for its contents.
- **`FallbackWatcher`** crawls with its own bounded `recReaddir` (32 concurrent visits; extra `lstat`s only queue on libuv's pool) and **watches each directory before listing it**, or files written in between stay invisible. It recrawls a replaced directory and a Windows overflow (no filename), drops errored handles (so stopping can't hang on them), and closes every handle under a deleted directory — a leftover blocks re-watching the path.
- **Errors.** `Watcher` never subscribes to `onError`, so a backend's `emitError` is an unhandled `'error'`: swallow benign codes or emit `recrawl` instead.

The **health check** (`healthCheck` options, `Watcher.js`) periodically writes a cookie file and waits to observe the event, emitting `'healthCheck'` (reported by `DependencyGraph` as `watcher_health_check_result`) — the same cookie mechanism that confirms watches at startup — because a silently dead watcher is otherwise indistinguishable from "nothing changed", and that's a whole genre of Metro bug report.

`watchmanDeferStates` defers processing during e.g. source-control operations; a health-check timeout reports them via `getPauseReason()`.

## Cache manager — `src/cache/DiskCacheManager.js`

- Path = `<dir>/<prefix>-<rootDirHash>-<relativeConfigHash>` from `lib/rootRelativeCacheKeys.js` over `BuildParameters` — so config changes select a *different* cache file rather than invalidating one.
- `write(getSnapshot, {changedSinceCacheRead, eventSource, onWriteError})`: writes once, immediately, if state changed since the cache was read; the snapshot is taken **lazily** at write time and writes are sequenced on a promise chain. A write failing with `ENOENT` creates the cache directory (recursively) and retries once.
- **Auto-save** (on by default in `DiskCacheManager`; Metro enables it only when watching, via `watcher.unstable_autoSaveCache`): each `'change'` or `'metadata'` event re-arms a debounce (`watcher.unstable_autoSaveCache.debounceMs`, default 5s; `DEFAULT_AUTO_SAVE_DEBOUNCE_MS` when unset), so a long-lived watch session keeps its cache warm, and `end()` flushes. Without it nothing after the first write is persisted, so a one-shot build never saves lazily computed sha1s or lazy plugin data.
- Implement `CacheManager` yourself to relocate or share the file map (e.g. seeding CI from a build artifact). `NoopCacheManager` disables it entirely.

## `FileProcessor` — `src/lib/FileProcessor.js`

Batches changed files across `jest-worker` **worker threads** (`enableWorkerThreads: true`, sized by `maxFilesPerWorker` (100) and `maxWorkers`), running sha1 computation and each non-lazy plugin worker whose `filter` accepts the file. When one worker suffices it runs **in-band** behind the same interface, avoiding worker startup cost entirely — a pattern worth copying for any optional parallelism. A single watch-mode change is always in-band; a recrawl or `addRoot` crawl goes through `processBatch` and can start a farm. `forkOptions.execArgv` is cleared so workers don't inherit Node args (notably avoiding Babel registration when running Metro from source). `processFileForPlugin` (the lazy path) runs one plugin's worker in-band and bypasses `filter`.

`worker.js` reads each file **at most once**, lazily, via a shared `getContent()` closure passed to every plugin worker and to the sha1 computation — one read, many extractions. Content is returned to the parent only if `maybeReturnContent` was requested, so it isn't copied across the thread boundary needlessly. The lazy sha1 path (`getOrComputeSha1` → `processRegularFile`) shares this request shape, so it also re-runs every eager plugin worker whose `filter` accepts the file.

## `RootPathUtils` — `src/lib/RootPathUtils.js`

Specialized `path.relative` / `path.resolve` for "many paths, few roots", ~8x and ~20x faster than `node:path` respectively on real FB-scale inputs. Requires well-formed input (native separators, no doubled separators) and falls back to `node:path` for awkward cases. This is the model for a legitimate micro-optimization: narrow documented contract, measured, with a fallback.
