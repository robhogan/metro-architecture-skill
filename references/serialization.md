# Cheap serialization

The principle: **the in-memory representation is the wire format.** Serializing should be a copy, not a transformation. Every format in Metro is designed backwards from that constraint.

## 1. The file map snapshot — v8 structured clone

```js
// packages/metro-file-map/src/lib/TreeFS.js
getSerializableSnapshot(): CacheData['fileSystemData'] { return this.#cloneTree(this.#rootNode); }
static fromDeserializedSnapshot({rootDir, fileSystemData, processFile}) {
  const tfs = new TreeFS({processFile, rootDir});
  tfs.#rootNode = fileSystemData;
  return tfs;
}
// packages/metro-file-map/src/cache/DiskCacheManager.js
import {deserialize, serialize} from 'node:v8';
```

What makes this cheap:

- **`v8.serialize` over `JSON.stringify`.** Preserves `Map` and `Set` natively (no entries-array round trip), is dramatically faster on large structures, and avoids the parse-time cliff that dominates cold start with a JSON blob.
- **No validation on read.** Deliberate, and commented as such: "It'd be too expensive to validate at runtime, so trust our cache manager that this is correct." Correctness comes from the cache key (config hash + `CACHE_BREAKER`) selecting a compatible file, not from checking.
- **No construction pass.** The deserialized object *is* the live data structure.
- **Portable by construction.** Root-relative paths with `'..'` for out-of-root, and no system separators anywhere in the tree, so a snapshot survives a different root directory and a different OS. See "Portability is a contract" below.
- **Lazy, sequenced and debounced writes.** `write(getSnapshot, …)` takes the snapshot at write time, on a promise chain; when and how often it writes (auto-save, watch mode only in Metro) is in `references/file-map.md`, "Cache manager".

### Portability is a contract, not a nicety

`TreeFS.js` states it outright: `#rootNode` is "directly portable (for a given project) between different root directories and operating systems", and `rootRelativeCacheKeys` posix-normalizes `roots` so the config hash matches across OSes. Don't argue that "the cache is machine-local": the default `DiskCacheManager` puts `rootDirHash` in the file *name*, so it happens never to share, but the contract exists for custom cache managers (`unstable_fileMapCacheManagerFactory`) that do — shared CI caches, remote stores.

What makes the tree portable is that it stores **almost no paths**: directory nodes are `Map`s keyed on single segments, so separators never appear in keys. A path *string* stored anywhere in the snapshot — a tuple field, plugin data, plugin state — must therefore be root-relative and posix-separated, and converted to system separators where it is read (`MockPlugin` does this: `normalizePathSeparatorsToPosix` on write, `normalizePathSeparatorsToSystem` on read). `Haste` sidesteps the question: its per-file data is a module name, not a path, and its state isn't serialized at all (`getSerializableSnapshot` returns `null`).

Symlink targets, the one path inside the tree, follow the same rule (#1687). `metadata[H.SYMLINK]` holds the target **lexically resolved to a root-relative normal path with posix separators, once, when the node is populated** (`RootPathUtils.resolveSymlinkToNormal` on `FileMap#maybeReadLink`'s `readlink`; the Watchman crawler can use `symlink_target`, but `planQuery` doesn't request it), and traversal converts it with `normalizePathSeparatorsToSystem`. As the `TreeFS` header puts it: "The target is the only path stored inside the tree, so it must not use system separators, or the snapshot would not be portable between operating systems."

**Resolve on write, not on read:** Resolution "runs once per symlink, when its node is populated, not on traversal", so traversal reads a plain string and needs no per-node cache of resolved targets. **Root-relative and posix:** A POSIX-built snapshot works on Windows and vice versa, and an absolute target inside the root is stored root-relative, so it follows the project to a new root.

Test portability with a round trip, not a single-platform fixture. `TreeFS-test.js` has the shape to copy (`describe('snapshot portability between operating systems')`): build under `path.posix`, snapshot, `jest.resetModules()`, load under `path.win32` (and the reverse) and look up through the link. A single-platform suite passes either way.

### The constraint this imposes on you

Everything reachable from a cache must be **v8-serializable**. `flow-types.js` encodes it:

```js
type V8SerializablePrimitive = string | number | boolean | null;
type V8SerializableCollection =
  | ReadonlyArray<V8Serializable> | ReadonlySet<V8Serializable>
  | ReadonlyMap<string, V8Serializable> | Readonly<{[key: string]: V8Serializable}>;
```

Plugin state (`getSerializableSnapshot()`) and per-file plugin data are typed against it (`void | V8Serializable`). **No class instances, no closures, no functions, no `undefined`-vs-absent distinctions you rely on.** If your design needs behavior attached to cached data, keep the data plain and rebuild the behavior at load.

`undefined`-vs-`null` *does* survive structured clone (JSON collapses it), and plugin data slots rely on it for lazy plugin workers (#1950): a slot is `undefined` until a worker has run on the file — the built-in crawlers create tuples without plugin slots, though a custom `crawlerFactory` crawler can fill them at `pluginDataIndices` — and `FileProcessor` stores a worker's `undefined` result as `null`, because "`undefined` is reserved to mean that no worker has run."

## 2. Tuples with constant indices

```js
// packages/metro-file-map/src/constants.js
{MTIME: 0, SIZE: 1, VISITED: 2, SHA1: 3, SYMLINK: 4, PLUGINDATA: 5}
```

`FileMetadata` is `[?mtime, size, 0|1 visited, ?sha1, 0|1|string symlink, ...pluginData]`. Plugins with a worker get a numbered slot appended, allocated in construction order.

- Serializes to a compact array, not a keyed object per file, at 100k+ files. The header comment in `metro-file-map/src/index.js` gives the reason: to "save storage space and reduce parse and write time of a big JSON blob" — written for the earlier JSON cache format, and just as true of v8 serialization.
- Directory nodes are `Map`s and file nodes are `Array`s, so the discriminator is free. It tests **shape, not realm** — `isDirectory` is `!Array.isArray(node)`, not `instanceof Map` (#1934): under Jest, `node:v8` is a host module, so `deserialize` returns `Map`s whose prototype is not the sandbox's `Map.prototype`. Any check on deserialized data has the same exposure.
- **Cost:** `H.SHA1` is less readable than `.sha1`, and adding a field is a format change. That tradeoff is made consciously and is documented in the header comment. Make it the same way: use the `H` constants, never a raw index literal.

## 3. Serialize or recompute — measure, then write it down

`HastePlugin.getSerializableSnapshot()` (`packages/metro-file-map/src/plugins/HastePlugin.js`) returns `null`:

```js
// Haste is not serialised, but built from traversing the file metadata
// on each run. This turns out to have comparable performance to
// serialisation, at least when Haste is dense, and makes for a much
// smaller cache.
```

This is the model for the decision. A derived index that can be rebuilt by one pass over data you're already deserializing may not be worth persisting. **Measure both, pick one, and put the reasoning in the code** — the next person will otherwise "fix" it.

`initialize` yields to the event loop every `YIELD_EVERY_NUM_HASTE_FILES` (10,000) Haste modules and packages while rebuilding, so the recompute doesn't block startup.

## 4. Bundle output — strings not ASTs, position-preserving edits

```js
// packages/metro-runtime/src/modules/types.js (readonly modifiers elided)
export type ModuleMap = ReadonlyArray<[number, string]>;
export type Bundle = {modules: ModuleMap, post: string, pre: string};
export type BundleMetadata = {pre: number, post: number, modules: ReadonlyArray<[number, number]>};
```

Module code is a **string produced once by the transform worker and carried unchanged through the graph and the cache**. Metro does not hold ASTs across stages. The graph stores `output: [{type, data: {code, lineCount, map, functionMap}}]` (`JsOutput` in `metro-transform-worker`) — already-generated code, its line count, and its source map as a `VlqMap` (§5).

**The serializer never parses module code, and every edit it makes is position-preserving** — which is what lets the worker's `VlqMap` be used unadjusted:

- `wrapModule` (`Serializers/helpers/js.js`) completes the `__d(...)` call: `addParamsToDefineCall` (`metro-transform-plugins`) inserts the module id, dependency map and, in dev, the verbose name just inside its last `)`.
- With `serializer.unstable_inlineDependencyMap` and `transformer.unstable_dependencyMapReservedName` both set, `inlineModuleIdReferences` replaces each `<dependencyMapReservedName>[i]` with the module id padded to the same length — "a cheap string operation, requiring neither parsing nor any adjustment to the source map" — and throws when an id doesn't fit, "rather than silently emit an incorrect source map".
- Source-map sections are placed by counting lines in the *unwrapped* `data.code` (`getSourceMapInfo` takes `code` from `getJsOutput(module).data`), so a wrapper that added a line shifts every later section.

A new serializer-side rewrite must keep the line count and every mapped column, or produce its own map.

Serializers find Metro-generated placeholders by string search, so source text must not contain them. The default transformer rejects `_$$_METRO_` (`lib/reservedString.js`: "The string reserved for identifiers that Metro generates for a serializer to find without parsing the code.") in source and JSON files, and serializers receive it as `options.reservedString`; its status is in SKILL.md's seams table.

`bundleToString(bundle)` (`packages/metro/src/lib/bundleToString.js`) sorts modules by numeric id and joins `pre`, the modules and `post` with `'\n'`. It also returns `metadata` — UTF-16 code lengths per module, excluding the joiners — but nothing in Metro reads it (`Server` keeps `.code`); outside Metro it is reachable only as `metro/private/lib/bundleToString`. Metro itself places modules in maps by line count.

**The bundle and its map are separate passes over the graph that must agree.** `.bundle` goes through `baseJSBundle` + `bundleToString`; `.map`, `metro build`'s map and `/symbolicate` take `[...prepend, ...Server._getSortedModules(graph)]` — a second sort by the same ids — and place modules by line count. Nothing checks that the two agree on order (the same `createModuleId`), filtering (`isJsModule` + `processModuleFilter`), prepend handling (`modulesOnly`) and the single `'\n'` between modules, and they diverge in at least one place: `Server.build()` (`runBuild`, `metro build`) keeps `prepend` in its map under `modulesOnly`, though `baseJSBundle` drops it from the code, so every section is offset by the prelude's and polyfills' lines. `serializer.customSerializer` replaces only the code side — `.bundle`, and `Server.build()`'s code (and its map, if it returns `{code, map}`) — while `.map`, `/symbolicate` and HMR always use Metro's own placement, so a custom serializer that reorders, filters or adds lines between modules breaks dev-server maps and symbolication. Change one side and you must change the other; test a mapping in the *last* module, where any drift accumulates. By default (`serializer.createModuleIdFactory`), module ids come from a server-lifetime first-seen counter, so order is consistent within a server, not reproducible across runs.

**Serialization is per request, not incremental.** The graph is incremental; serializing it isn't, and nothing is memoized per revision. Every `.bundle` request re-wraps and re-concatenates every module (the `If-Modified-Since` → 304 check runs after serializing); every `.map` request re-reads every file module's source from disk for `sourcesContent` (`getSource()`, `references/graph.md`; skipped with `excludeSource`, empty for assets) and `JSON.stringify`s the whole index map synchronously. Only HMR is proportional to the delta. Per-module work added to a serializer is paid for every module on every reload.

`HmrUpdate` reuses the `[id, code]` pair (`HmrModule` is `{module: [id, code], sourceMappingURL, sourceURL}`), so HMR is a projection of the same structure rather than a second format. (`DeltaBundle` and `BundleVariant` are declared in the same file. Nothing uses `BundleVariant`, and the only user of `DeltaBundle` is `DeltaBundler/mergeDeltas.js`, which nothing outside its test calls.) Each update module's URLs derive from the client's bundle URL — rewritten by `HmrServer` to `modulesOnly=true&shallow=true` (and `runModule=false` unless set), and by `hmrJSBundle` to drop `excludeSource`, set `platform` to the graph's and, for a virtual module, add `virtual=true` — with the pathname set to the `serverRoot`-relative path of the module's *source* path (`getModuleSourcePath`; `path.relative`, not `RootUrlMap`), its extension kept and `.map` or `.bundle` appended (`foo.js.map`), "so that the server resolves the URL to this exact file" rather than a sibling found through `sourceExts`. The `.map` URL is served by the ordinary `.map` endpoint from a one-module graph.

With `includeAsyncPaths` (`lazy=true`), the `__d` dependency map carries `paths`, each async dependency's URL (`getDefaultAsyncDependencyPath`, `helpers/js.js`, overridable per serializer call with `unstable_getAsyncDependencyPath`, which `Server` never sets): `'/'` plus the POSIX `serverRoot`-relative source path with its extension kept, then `.bundle?` and the bundle URL's search params, with `platform`, `modulesOnly=true&runModule=false` and, for a virtual dependency, `virtual=true`. Like HMR URLs, it uses `path.relative`, not `RootUrlMap` ("TODO: This is not the proper Metro URL encoding of a file path"). Each HMR module also carries the ids of its *transitive* inverse dependencies, as an extra `__d` argument, so the runtime can find accept boundaries; an update's size grows with the changed modules' ancestors, not just the delta.

## 5. Source maps — VLQ per module, sections not re-encoding

```js
// packages/metro-source-map/src/source-map.js
type GeneratedCodeMapping   = [number, number];
type SourceMapping          = [number, number, number, number];
type SourceMappingWithName  = [number, number, number, number, string];
export type MetroSourceMapSegmentTuple = …one of the above;
export type VlqMap = {readonly mappings: string, readonly names: ReadonlyArray<string>};
```

Tuples are the decoded working form: 1-based generated line, 0-based column, then 1-based source line, 0-based source column and name. They are **not** what Metro stores for transformed modules.

- **Storage is VLQ.** `JsOutput.data.map` is always a `VlqMap` (since 0.86), which custom serializers must handle (`isVlqMap`). Decoded tuples are the largest single contributor to dev-server heap on large bundles (~10 million small arrays retained for the FBiOS entry bundle); storing VLQ instead roughly halves post-build heap there (1,653.7MB → 809.7MB as measured in #1743). One string plus a names array is cheap to hold, to return from a worker, and to JSON into the transform cache (`FileStore` and `HttpStore` write JSON).
- **Encode the cheapest way available.** Every JS module's `VlqMap` is encoded by `vlqMapFromBabelDecodedMap` from a decoded map, never from tuples. Unminified, that is Babel's eagerly computed `result.decodedMap`. Minified, it is the minifier's `decodedMap`, after the minifier is given Babel's encoded `result.map` ("A minifier may return its map already decoded, which saves decoding it here."); Terser returns its own `decoded_map`, so its map is never encoded, and a minifier that returns only an encoded `map` is decoded with `@jridgewell/sourcemap-codec` (#1984). JSON modules use `vlqMapFromTuples` unminified and the minifier path when minified. `tuplesFromBabelDecodedMap`, `toBabelSegments` and `toSegmentTuple` are deprecated, "Unused by Metro".
- **Each `VlqMap` is self-contained.** It is encoded by a fresh `Generator` against one placeholder source, so its deltas start from zero and its name indices index its own `names`. That is what lets a section carry it unchanged — and why concatenating modules into a *flat* map means rebasing deltas at each module boundary rather than appending strings.
- **Whole-bundle maps are index maps, unconditionally.** `sourceMapGenerator` ends in `fromRawMappingsIndexed`: one section per module with a map, at `{line: carryOver, column: 0}`. `toIndexMapSection` passes each `VlqMap` through untouched, attaching `sources`, `sourcesContent`, `x_facebook_sources` / `x_google_ignoreList` itself (in `fromRawMappingsIndexed`'s words, "VLQ-stored modules pass through verbatim — no decode/re-encode — which is the whole point"), and encodes the tuple maps of Metro's synthetic modules (`map: []`; in practice `__prelude__` from `getPrependedScripts`) with a per-section `Generator`. **The mappings are never re-derived from the generated code.** On FBiOS this halved `.map` serialization (12.0s → 6.0s, where a flat map from VLQ took 13.9s) for a 3.4% larger map, at a compatibility cost: consumers must understand `sections`, with no opt-out. Section offsets come from counting newlines in each module's code (`countLines`).
- The flat `fromRawMappings` (`Generator` + `B64Builder`) decodes `VlqMap` modules one at a time to keep transient tuples short-lived. Its only caller in Metro is the unwired `baseJSBundleWithLazyModuleDefinition` (see SKILL.md's seams table), which calls it per module, decoding and re-encoding each `VlqMap` before `BundleBuilder` offsets it — wiring that serializer in gives up the passthrough.
- The server's path (`sourceMapStringNonBlocking` → `sourceMapGeneratorNonBlocking`) gathers per-module info in a work loop that breaks to `setImmediate` roughly every 50ms because it's on the main thread: "let's give Metro an opportunity to handle requests." The blocking `sourceMapGenerator` runs the same loop synchronously. Building the index map after that is a synchronous passthrough, and the final `JSON.stringify` of the whole map also runs synchronously, without yielding.
- **`/symbolicate` indexes once, then decodes only what it looks up.** It never builds a map: `getExplodedSourceMap` returns each module's own `map` object by reference — the identity the cache below keys on — and places modules by their stored `lineCount`, reading no sources. `packages/metro/src/Server/symbolicate.js` then uses `LineIndexedMappings` (exported only through `metro-source-map/private/*`): construction scans the module's mappings twice (once to count lines, once to advance the source line/column accumulators) into three O(lines) `Int32Array`s, without allocating per segment, so each lookup decodes only the target line (~27x faster cold on `ReactFabric-dev`: 102.5ms → 3.8ms). The index lives in a module-level `WeakMap<VlqMap, LineIndexedMappings>` that "releases it automatically when the module leaves the graph". `/symbolicate` resolves against the graph's *latest* revision (`_explodedSourceMapForBundleOptions` calls `updateGraph` first), not the one the client loaded.
- Function maps (`FBSourceFunctionMap: {names, mappings: string}`) are a compressed string-encoded side-channel, not a per-position object graph. The worker produces them with `functionMapBabelPlugin`, which it passes to the Babel transformer as an input plugin and reads back from `metadata.metro.functionMap`, so a transformer that doesn't apply the given `plugins` yields none.

See `docs/SourceMapFormat.md` for Metro's vendor-specific fields: `x_facebook_sources` (including the function-map encoding) and `x_google_ignoreList`.

### Consuming and composing maps

`Consumer` (`createConsumer` picks `MappingsConsumer` for a flat map, `SectionsConsumer` for `sections`) is what `composeSourceMaps` runs on — the step that composes, say, a Metro bundle map with the `hermesc` map, rather than any stage regenerating from source. Metro itself doesn't call `composeSourceMaps`; it is API for React Native's Hermes build step (`scripts/compose-source-maps.js`). Expo's Hermes export composes with its own `@jridgewell/remapping`-based composer, which #1983 measured at ~680ms on the same input. The same principles apply in Metro's composer (#1978–#1981, #1983, #2095), measured against Metro's composer before them on a 52.6MB bundle (82.4MB map) composed with a 1.84M-segment Hermes map: **2,088ms → 646ms (3.25x), composition memory −15%**, with React Native 0.88 release maps composed with their `hermesc -O` maps byte-identical to the earlier composer's (#2095).

- **Decode once, in place.** `MappingsConsumer` decodes each segment once (#1978, −13.6%), from character codes in a single pass straight into the mappings array — no per-segment slicing, no `vlq` cache, no generator spread (#1979, 1.63x, −16% memory). `metro-source-map` has no `vlq` dependency.
- **Index by line.** `_lineStarts` records the index of "the first mapping on each generated line" while decoding, so `originalPositionFor` binary-searches only the target line by column, last mapping at or before it winning (#1980, −20.5%, ~9MB of index for 1.18M lines).
- **Don't look up what you already have.** `composeSourceMaps` iterates the last map and traces each mapping from *its own* original position through the earlier maps, instead of looking its generated position up again in the same map (#1981). Of consecutive mappings at one generated position it keeps only the last: "only the last covers any code, e.g. `hermesc` emits a function's declaration, then its first instruction." Keeping them all misleads consumers that take the first, such as `metro-symbolicate` and `@jridgewell/trace-mapping`, which then resolve function entries to the wrong place or to `null` (#2095).
- **Encode directly.** The composed mappings go into parallel number arrays (source and name indices, `-1` for none), sorted by an index permutation only if out of order, with identical neighbors dropped, and are encoded with Metro's `B64Builder` — not `source-map`'s `SourceMapGenerator`, which allocates and validates an object per mapping (#1983, −20.6%). `B64Builder.markLines` grows its buffer until a run of line markers fits, so any gap of unmapped generated lines encodes, in `composeSourceMaps` or `Generator`.
- **Section offsets per spec.** `SectionsConsumer.generatedMappings()` — what `composeSourceMaps` iterates when an index map is last in the chain — places a section's leading unmapped segment at the section's offset, not at 1:0, and applies the column offset only on the section's first line (#1976). Metro's own sections start at column 0, so only the first half touches Metro-produced maps.
- **Decoders.** `MappingsConsumer` and `LineIndexedMappings` each carry their own allocation-light single-pass decoder (`_decodeMappings` and `readVlq`, which share no code). The npm `source-map@0.5` `SourceMapConsumer`, which allocates an object per segment, sits behind the deprecated `toBabelSegments`, whose one remaining caller is `decodeVlqMap` (the flat `fromRawMappings` path). The transform worker decodes a minifier's encoded `map` with `@jridgewell/sourcemap-codec`, and only when the minifier returns no `decodedMap`.

### Paths and locations a map points at

A module path leaving Metro goes through `lib/modulePath.js` (`references/graph.md`, "Module paths"), never `path` functions on the graph key: dev names (`verboseName`) use `getModuleDisplayPath`, and URLs — HMR, async bundle paths and `url-server` `sources` — use `getModuleSourcePath` in the form `references/graph.md` gives ("URLs and roots"), so a virtual module at a real file's path gets distinct URLs. In `Absolute` mode a map's `sources` entry is the raw graph path, as is `/symbolicate`'s symbolicated `file`. `getAllFiles` leaves virtual modules out, and their `sourcesContent` is the buffer `getSource()` returns.

With `sourcePaths=url-server` (`SourcePathsMode.ServerUrl`), `sources` are server URL pathnames from `RootUrlMap.urlPathnameOfFilePath` (`/[metro-project]/…`, `/[metro-watchFolders]/<id>/…`) instead of absolute paths. Like a snapshot path, a path written into a map is a contract the reader must be able to invert, and the server inverts it through the same `RootUrlMap`: it serves those pathnames as source files (with `charset=utf-8` in `Content-Type`, #1888), and `/symbolicate` resolves a frame's `file` through it before falling back to `projectRoot` when building code frames (#2018). `HmrServer` builds its own `RootUrlMap` from the same config, so its entry resolution and `serverRoot` match `Server`'s (#2017) — though HMR module URLs (§4) use `path.relative` instead. And a location in transformed code need not exist on disk — a transform can append code past the last line — so the resolution-error code frame in `ModuleResolution.js` skips missing lines rather than throwing over the real error (#1226).

## Checklist for a new serialized format

1. Can the in-memory structure *be* the serialized structure? If not, change the structure.
2. Is everything v8-serializable (`V8Serializable`)? Do you test it by shape, not `instanceof`?
3. Is it portable — no absolute paths, no native path separators, no machine-specific values? Resolve paths once on write, and prove it with a cross-platform round trip.
4. Is it flat where it's repeated per file? Tuple + constant index, not object + keys.
5. Have you bumped the relevant breaker (`CACHE_BREAKER`, plugin `getCacheKey`, `cacheVersion`)?
6. Is deserialization O(size) with no per-item construction, and does it avoid validation by relying on the key instead?
7. Does the write path debounce and sequence, and never block a request?
8. If it's derived data, did you compare persisting it against recomputing it — and record the result in a comment?
9. For source maps: does encoded VLQ pass through, does a lookup decode only what it needs, and does any edit to module code keep every mapped position?
10. For bundle output: is the work per request, and could it be per module and cached instead? Do the bundle and its map still agree?
