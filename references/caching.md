# Caching and invalidation

## Every cache in Metro, in one table

| Cache | Where | Key | Value | Lifetime | Invalidated by |
|---|---|---|---|---|---|
| **File map snapshot** | `metro-file-map/src/cache/DiskCacheManager.js` | file name from `rootDirHash` (absolute `rootDir`) + `relativeConfigHash` (root-relative `BuildParameters` incl. `CACHE_BREAKER`, plus plugins' `getCacheKey()`) — `lib/rootRelativeCacheKeys.js` | v8-serialized `CacheData` (`TreeFS` tree, clocks, plugin state) | on disk, forever | crawl delta at startup; different config ⇒ different file; files under `addRoot` roots aren't keyed: any that a later auto-save writes are dropped on read and re-crawled |
| **Transform results** | `metro/src/DeltaBundler/Transformer.js` → `metro-cache/src/Cache.js` → `cacheStores` | `stableHash([baseHash, relPath, assetUrlPath, …transform options]) ++ sha1(content)` | `TransformResult` | on disk / remote, forever | content-addressed — never invalidated, only missed |
| **Transform base hash** | `Transformer` constructor → `getTransformCacheKey` | — | `_baseHash` | process | restart only: every file the transformer's `getCacheKey` hashes, Babel config files and Metro's `@babel/runtime/package.json` are read once, so editing them under a running server changes no key |
| **Resolutions** | `DependencyGraph._resolutionCache` | `canonicalize(resolverOptions) → originDir → name + '\0esm'/'\0cjs' → platform` (a virtual result: origin *file* in place of `originDir`) | `BundlerResolution` (errors are thrown, not cached) | process | **cleared wholesale on any file change** |
| **`package.json` contents** | `node-haste/PackageCache.js` | `package.json` path as asked for | parsed JSON + `rootPath` | process (survives `_resolutionCache` clears and graph resets) | `invalidate(path)` from `_onHasteChange` for every changed file; the closest scope is not cached (`references/resolution.md` §2, including a symlink gap) |
| **Normalized `exports` maps** | `metro-resolver/src/PackageExportsResolve.js` `_normalizedExportsFields` | the parsed `exports` object **by identity** (`WeakMap`) | `NormalizedExportsLikeMap` | while the JSON is cached | identity — a re-read `package.json` is a new object |
| **Metro's runtime packages** | `metro/src/lib/metroRuntime.js`, `metroBabelRuntime.js` | — | `metro-runtime`'s and its `@babel/runtime`'s `package.json` paths; that `@babel/runtime`'s version | process (main thread and each transform worker) | never: resolved by Node, outside the file map, so a reinstall needs a restart |
| **Haste map** | `metro-file-map/src/plugins/HastePlugin.js` | name + platform | path + type | process (rebuilt each start from cached per-file data) | `onChanged` — removals before additions; modifications are ignored (`references/file-map.md`, Plugins) |
| **Per-file data** (sha1, deps, haste name) | `FileMetadata` tuple slots | file's canonical path | slot value; a plugin slot is `undefined` until its worker runs | persisted in the file map | file change ⇒ fresh tuple, re-processed by `FileProcessor` (lazy plugins: on their next `getOrComputeFileData`; an eager plugin's never-run slot reads as `null`) |
| **Dynamic roots** | `DependencyGraph.getDynamicRoots()` | `FileMap.getRoots()` array identity | `DynamicRoot[]` | process | a new roots array from `FileMap.addRoot` |
| **Module graph revisions** | `metro/src/IncrementalBundler.js` | `getGraphId(...)`; also revision id | `GraphRevision` | until `endGraph` (an HTTP `DELETE` of the bundle) or a failed initial build — otherwise unbounded | change events accumulate in `DeltaCalculator`; the graph is updated on the next request, minting a new revision id for a non-empty delta |
| **Module ids** | `Server._createModuleId` (`createModuleIdFactory`) | module path | number | process | never — ids must be stable within a server lifetime |
| **Symbolication line index** | `Server/symbolicate.js` | a module's `VlqMap` **by identity** (`WeakMap`) | `LineIndexedMappings` | until the module leaves the graph or is re-transformed | identity — a re-transformed module has a new `VlqMap` |
| **Cache-store hit provenance** | `metro-cache/src/Cache.js` `#hits` | the key `Buffer` **by identity** (`WeakMap`) | the store that hit | until the Buffer is GC'd | n/a |

## Key composition — the rules

Metro's transform key is the reference implementation (`Transformer.transformFile`):

```
baseHash   = stableHash([ getTransformCacheKey({cacheVersion, projectRoot, transformerConfig}) ])
             // ↳ sha1 of 'metro-cache' ∥ metro version ∥ config.cacheVersion
             //   ∥ md5(transformer module file contents) ∥ Transformer.getCacheKey?.(transformerConfig) ∥ globalPrefix
             //   computed once, in the Transformer constructor
             // transformerConfig = config.transformer − getTransformOptions, transformVariants,
             //   unstable_supportsInlineModules, unstable_workerThreads
             //   + unstable_supportsInlineModules: true when resolver.unstable_allowVirtualModules
partialKey = stableHash([ baseHash, posix(projectRelativePath),
                          assetUrlPath,   // assets only, else null
                          customTransformOptions, dev, experimentalImportSupport, inlinePlatform,
                          inlineRequires, minify, nonInlinedRequires, platform, type,
                          unstable_memoizeInlineRequires, unstable_nonMemoizedInlineRequires,
                          unstable_transformProfile ])
fullKey    = Buffer.concat([partialKey, sha1(contents)])   // contents: the file, or the caller's fileBuffer
```

The default transformer's `getCacheKey` (`metro-transform-worker`) hashes its own entry file and seven helpers (`getMinifier`, `assetTransformer`, `collectDependencies`, `generateImportNames`, `JsFileWrapping`, `reservedString`, `importLocationsPlugin`), the *entry files* of the Babel transformer and minifier, Metro's transform plugins (the `metro-transform-plugins` index and its five plugin files), Metro's own `@babel/runtime/package.json` when `enableBabelRuntime === true` (the default — so upgrading it misses every transform), `stableHash` of the rest of the transformer config (as in the block above), and the Babel transformer's `getCacheKey` (for `metro-babel-transformer`, the contents of the root Babel config files). A new JSON-serializable `transformer.*` option is keyed for free; other config isn't, until it is injected into the transformer config or added to `getTransformCacheKey` (as `globalPrefix` is). `resolver.unstable_allowVirtualModules` is keyed the first way: Metro sets `unstable_supportsInlineModules` from it, overriding any configured value, and the worker passes it to Babel as the caller's `supportsInlineModules`, so a plugin emits `metro:inline` specifiers only when they resolve ("Babel plugins can emit different code for this, so it must be part of the transform cache key.").

1. **Content, not mtime.** `sha1` comes from `getOrComputeSha1`, which returns any sha1 already in the tuple (from Watchman, an eager crawl, the snapshot or an earlier lazy computation) and otherwise hashes the file. When a caller supplies `fileBuffer` — a `require.context` module's rendered template, or a virtual module's source — `Transformer` hashes that instead. A virtual module is transformed and keyed at its bare virtual path, so its key is that of a file at that path with that content, and they share an entry — sound by purity, since the transformer sees identical inputs. A touched-but-unchanged file is a hit; CI and dev machines share entries.
2. **Portable.** Project-relative, `posix`-separated. An absolute path or a `\` in a key silently destroys remote-cache hit rate — the bundle still builds, so nobody notices for months. A portable key doesn't make the *output* portable, though: a Babel plugin that embeds `state.filename` (absolute, resolved against `projectRoot`) bakes the producing machine's path into a shared entry.
3. **Total, by intent.** Everything that can change the output belongs in the key, including the transformer's own file contents (`getCacheKey([require.resolve(transformerPath)])`) and its self-declared key. `transformFile` destructures the known options and **throws if any `extra` key remains** — a deliberate tripwire so a new option can't reach the worker without being keyed.
4. **Inputs outside `TransformOptions` are keyed by hand.** `assetUrlPath` reaches the worker as its own argument, "so that it is not spread into BabelTransformerArgs". The tripwire can't see it, so it's added to `partialKey` explicitly.
5. **`stableHash` = `md5(JSON.stringify(value, canonicalize))`.** `canonicalize` sorts plain-object keys, so key order in a config object can't change the hash. It does nothing else: a `Map`, `Set` or `RegExp` stringifies to `{}`, and functions and `undefined` vanish from objects and become `null` in arrays, so different values can hash identically. Convert them first.
6. **Reuse the key `Buffer` instance.** `Cache` tracks which store answered in a `WeakMap` keyed on the `Buffer`, so it only writes through to stores *above* the one that hit. `Transformer` only rebuilds `fullKey` if the worker reported a different sha1, precisely to preserve that identity.
7. **The key is also the output's identity.** `Transformer` returns `fullKey.toString()` as `unstable_transformResultKey`, and `Graph`'s `transformOutputMayDiffer` uses it to drop a re-transformed module with unchanged inputs and dependencies from the delta. A missing key always counts as changed.

### Known gaps in totality

Each is an input that can change output without changing the key. Don't add to this list; shrink it.

- **An asset's sibling scale variants** (`icon@2x.png`, …), which `Assets.js` lists with `fs.promises.readdir` and hashes — outside the file map and outside the key, which holds only the asset's own path and sha1. Adding a variant changes neither, so a cached transform of the base asset can go stale.
- **Dependency closures.** Beyond the entry files of the transformer, Babel transformer and minifier, only a fixed list is hashed (for the default transformer, seven helpers and Metro's transform plugins). The rest of what they run isn't: `metro-source-map`, `Assets.js` and `Bundler/util` behind `assetTransformer`, `createInlinePlatformChecks` behind `inline-plugin`, the implementation behind `metro-minify-terser`'s re-exporting entry, the modules `transformer.assetPlugins` names (only their paths are keyed), `@babel/*`, `terser`, and Babel presets and plugins. Metro's packages depend on each other at exact versions, so the Metro version separates their releases; the gap bites a patched or linked Metro, and every third-party range. Upgrading a preset without bumping `cacheVersion` can serve stale output.
- **Transformer config under a custom transformer.** `getTransformCacheKey` keys only `globalPrefix` from the transformer config itself. Everything else, including the `unstable_supportsInlineModules` Metro injects, is keyed only if the transformer's `getCacheKey` hashes it, as the default one does.
- **File-relative `.babelrc`.** `metro-babel-transformer`'s `getCacheKey` calls `loadPartialConfigSync` with no `filename`, so Babel finds only root config files (`babel.config.*`).
- **Environment.** Workers inherit the server's `process.env` unkeyed: `BABEL_ENV` (which `metro-babel-transformer` sets to `'development'` in dev and defaults to `'production'` otherwise), `NODE_ENV` as presets read it, and any variable a plugin inlines. The exception is `JEST_WORKER_ID`, which jest-worker sets per worker thread and `Worker.flow.js` deletes, so output can't depend on which worker produced it (#2152).
- **Anything read once at startup** (the base hash row above) is fixed for the server's lifetime even if it is keyed.

### Asset URL paths — the per-asset key input

An asset's output bakes in its URL path (`httpServerLocation`), and an asset outside `projectRoot` is addressed as `[metro-watchFolders]/<index>/…` (`Assets.getAssetUrlPath`). Rather than put `watchFolders` in `baseHash` and miss every module when it changes, `Transformer` computes the path on the main thread for assets only:

```js
// metro/src/DeltaBundler/Transformer.js
// Assets are the only modules whose output depends on watchFolders and
// dynamic roots, via the URL path baked into them, so that dependency
// enters the cache key per asset here and not in the base hash.
```

- Root-relative and posix, so portable. "The first root to contain the asset wins, in index order", so appending a watch folder never changes an existing asset's URL or key.
- Metro's transformer reports the path it baked in as `unstable_assetUrlPath` on the asset's output, and `getAssets` reads it back rather than recomputing it (`references/graph.md`, "URLs and roots"). The path rides in the cached `TransformResult`, which is sound only because it is keyed. Scales, files and hash are still re-read by `getAssets`, so a stale cached transform (the scale-variant gap) can list different variants from the ones a release build copies.
- An asset under no configured root falls back to a **dynamic root** id — 8 hex digits of a sha1 of the root's project-relative posix path (`metro/src/lib/dynamicRoots.js`), independent of other roots and their order, so still portable. How often this path is taken depends on which dynamic roots exist (SKILL.md's seams table); Metro's own runtime packages ship no assets.

## Cache stores — `metro-cache/src/stores/`

```js
interface CacheStore<T> {
  name?: string;
  get(key: Buffer): ?T | Promise<?T>;
  set(key: Buffer, value: T): void | Promise<void>;
  clear(): void | Promise<void>;
}
```

- `FileStore({root})` — `root/<first byte hex>/<rest hex>`. A `Buffer` value is stored raw behind a leading `BINARY_MARKER` (`0x01`, a byte no JSON document begins with); anything else is JSON. `ENOENT` or a `SyntaxError` on read is a miss, not an error, and a zero-filled file left by a failed write fails to parse, so it misses too — **a corrupt local entry degrades to a miss, never a crash.**
- `AutoCleanFileStore` (deprecated) — `FileStore` + periodic sweep.
- `HttpStore({endpoint, timeout, family, cert, ca, key, …})` or `{getOptions, setOptions}` — gzipped `GET`/`PUT`, with opt-in retries (`maxAttempts` defaults to 1). On read only a 404 is a miss: any other status outside `additionalSuccessStatuses` (which read as a 200), a network error, or a body that fails to gunzip or parse rejects.
- `HttpGetStore` is the read-only variant for dev machines: it turns HTTP and network errors into a miss plus one warning, but an undecodable body still rejects.
- `Cache(stores)` reads in order, first hit wins, and writes to all stores **before** the one that hit. Write failures are collected into an `AggregateError` and surfaced via `cache_write_error`, and writes are fire-and-forget, so a failing remote never breaks the build through `set`. A `get` that *throws* fails the transform (`cache_read_error`) — so with a plain `HttpStore`, an unreachable remote fails every transform that misses the stores before it (with the usual local `FileStore` first, every local miss). Stores must turn expected failures into misses.

Typical config: local `FileStore` first, remote second (`docs/Caching.md`). Default: one `FileStore` at `os.tmpdir()/metro-cache`.

## Cache breakers and versions

| Knob | Scope | Bump when |
|---|---|---|
| `CACHE_BREAKER` in `metro-file-map/src/index.js` (currently `'14'`) | file map snapshot | the file-map data structure or its content changes for a given FS state — Metro's version isn't in this key, so nothing else separates releases |
| `FileMapPlugin.getCacheKey()` | that plugin's data | the plugin's extraction logic or its user-supplied module changes |
| `FileDataPlugin`'s `cacheKey` option | that plugin's data | as above — and when flipping `lazy` or changing `filter`, neither of which `FileDataPlugin` keys itself |
| `MockPlugin`'s `CACHE_VERSION` (`RawMockMap.version`) | mock map | mock map format changes (a mismatch rebuilds from files) |
| metro `package.json` version | transform cache | automatic, every release |
| `config.cacheVersion` | transform cache | user-facing escape hatch, e.g. after upgrading a Babel preset; also the way to namespace a shared remote cache |
| `Transformer.getCacheKey()` | transform cache | a custom transformer's behavior changes |
| Files the transformer's `getCacheKey` hashes (above) | transform cache | automatic for an edit to a listed file. There is no transform-cache breaker constant, so a change to transform output in an unlisted module is separated only by the next release's version: add the module to the list in the same commit, as #1749 did for `collectDependencies` |

`transformer.globalPrefix` is also in `baseHash`, but it isn't a cache knob: it prefixes the module-system globals (`${globalPrefix}__d`), so it changes the bundle, which is why it's keyed.

What counts as a content change, by example: storing a symlink's target lexically resolved to a normal POSIX path (#1687), leaving plugin slots `undefined` until a worker has run, so that `null` is a real result (#1950), and resolving a cross-drive symlink's relative target on the symlink's own drive (#2161). Each changed what a cached tuple means for the same file-system state, so each bumped the breaker. Changing which files an eager plugin runs on is the same kind of change: a snapshot written before it holds `undefined` slots that now read as `null`.

**Bump the breaker in the same commit as the format change.** A stale hit is the worst failure mode Metro has: it produces a wrong bundle, silently, and the user's only recourse is `--reset-cache`, which they'll then apply forever.

## Designing invalidation

Answer SKILL.md §7's three questions — what invalidates it, how it learns (a key, a `'change'` event, or a restart), conservative or precise — in the PR description. SKILL.md §7 also walks through Metro's own answers, from the wholesale `_resolutionCache` clear to the unconsumed `Observations` seam.

### The traps

- **Negative results.** "This file doesn't exist" / "there's no `package.json` here" must be invalidated by an *addition*. Most stale-resolution bugs are a missing negative-result invalidation.
- **Caches of file-system answers.** A cache between the resolver and `TreeFS` must store the observations that produced each entry and replay them on a hit, or a later resolution served from it can never be invalidated. An entry indexed only under the file it found misses a nearer file added later and a symlink retargeted above it. That cost is why #1948 removed `PackageCache`'s closest-package memo rather than fix it.
- **Path as asked vs canonical path.** Change events carry canonical paths. A cache keyed on a path through a symlink is never evicted by them (`PackageCache`'s known gap). Key on the canonical path, or record the traversed link.
- **Ordering.** `HastePlugin.onChanged` processes removals before additions so that a moved file isn't briefly seen as a duplicate. Any index keyed on something other than path has this hazard (`MockPlugin` must not reassign the active mock when a *non-active* duplicate is removed) — and needs the *old* derived value to process a modification, which the aggregator doesn't provide.
- **Directory events.** Adding a directory can change resolution (a new `node_modules`), even with no file event you care about. `ChangeEvent` carries `addedDirectories` / `removedDirectories` for this reason.
- **Symlinks.** A target is resolved once, "when its node is populated, not on traversal" (`RootPathUtils.resolveSymlinkToNormal`), and stored in the tuple. It is lexical, not a real path, so links inside it are still followed at lookup time and can't stale it. Any symlink event makes `DeltaCalculator` reset the graph.
- **Mid-flight mutation.** `getOrComputeSha1` mutates the tuple it originally read, so if the file changes during computation the result attaches to an orphaned tuple instead of poisoning the live one. Copy this pattern when computing derived data asynchronously.
- **Changes during a build** — in the file map (`references/file-map.md`, Lifecycle) and in `DeltaCalculator` (`references/graph.md`). Both replay what arrived during the build rather than merging sets.
- **Realms.** Under Jest, `v8.deserialize` returns host-realm `Map`s, so `TreeFS` identifies directory nodes by shape, not `instanceof Map`. Don't `instanceof`-test cached data.
- **Config that never reaches a key.** A new `metro-config` option that changes output but isn't in the transform key is a stale-hit bug waiting for a user; a new per-request option also needs `getGraphId`.
- **Inputs read beside the file.** Any transform input read outside the file map — like an asset's scale variants — has the same exposure; see "Known gaps in totality".

## Debugging a suspected cache bug

1. `--reset-cache` — if it fixes it, it's invalidation, not logic. It skips the file-map snapshot and `clear()`s every store, but `HttpStore.clear()` is "Not implemented": a bad remote entry survives it.
2. `DEBUG='Metro:*'` — `Metro:Transformer` logs the base hash; `Metro:FileMapCache` logs the cache path on write; `Metro:FileMap` logs lazily-processed files.
3. Delete only the file-map cache (path from `DiskCacheManager.getCacheFilePath()`) to separate file-map staleness from transform staleness.
4. `cacheStores: []` disables the transform cache entirely (`Cache.isDisabled`, which also skips computing the global cache key).
5. Check whether the changed input is in the key at all — that is nearly always the answer — and whether it's one of the known gaps.
