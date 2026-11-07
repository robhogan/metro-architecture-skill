# Transformation — `metro-transform-worker` and the main-thread side

Paths starting `packages/` are relative to the metro repo root, a path starting with a package name (`metro-transform-worker/src/index.js`, "the worker") to `packages/`, and other bare paths to `packages/metro/src/`.

Transformation turns one module's source into a `TransformResult`, `{dependencies, output}`. It is the only source of graph edges — `buildSubgraph` resolves the `dependencies` a transform reports, then visits them (`references/graph.md`) — and it never sees a module id: dependencies become indices into a map the serializer fills in, so one result serves every graph that transforms the file with the same options.

## Where it runs, and what crosses the boundary

- `IncrementalBundler` calls `getTransformFn` (`lib/transformHelpers.js`) once per graph. It runs `calcTransformerOptions` and returns the `TransformFn` that the graph's `DeltaCalculator` keeps for the graph's lifetime and `buildSubgraph` calls per module. Each call supplies a virtual or `require.context` module's source as `fileBuffer` (`references/graph.md`, `VirtualSource`), reduces `inlineRequires` to a boolean for the module, and sets `type`.
- `Transformer.transformFile` (`DeltaBundler/Transformer.js`) computes an asset's `assetUrlPath`, the key and the sha1, and reads the cache; a hit never reaches a worker. A miss goes through `WorkerFarm` (payload: `references/graph.md`, "Workers") to `Worker.flow.js`, which reads the file unless content was sent, hashes the bytes it transforms so the result is keyed by them (`references/caching.md`, rule 6), and calls the `transformerPath` module's `transform`.

`JsTransformOptions` has three sources:

- **Per request** (`lib/splitBundleOptions.js`): `customTransformOptions` (`transform.*` URL parameters), `dev`, `minify`, `platform`, the unstable `unstable_transformProfile`, and `type` — `'module'`, or `'script'` for the module system and polyfills `lib/getPrependedScripts.js` builds, refined to `'asset'` per module by `resolver.assetExts`. All are in `getGraphId`.
- **Per graph**, from `getTransformOptions`: `experimentalImportSupport`, `inlineRequires`, `nonInlinedRequires` (default `baseIgnoredInlineRequires`, six React and React Native specifiers) and two unstable memoization options. `inlinePlatform` is always `true`.
- **Config**: `config.transformer` minus `getTransformOptions`, `transformVariants`, `unstable_supportsInlineModules` and `unstable_workerThreads`, sent whole as `JsTransformerConfig` with every call and fixed for the server's lifetime.

Options are keyed per module; config, apart from `globalPrefix`, only through the transformer's `getCacheKey` (`references/caching.md`).

`getTransformOptions(entryPoints, {dev, platform}, getDependenciesOf)`, the user hook, runs once per graph, when `getTransformFn` builds the graph's transform function. Its result is therefore per entry, `dev` and platform — inputs `getGraphId` already holds — and is never re-run when files change. Only its `transform` field is read here, and scripts skip it. An `inlineRequires` of `{blockList}` becomes `false` for a module whose absolute path is a key, `true` otherwise. `getDependenciesOf(path)` lists the graph paths a full main-thread traversal from `path` reaches, through a transform function whose construction calls the hook again with `path` as the only entry, so a hook that calls it unconditionally recurses without end.

## `transform`: three paths

`transform(config, projectRoot, projectRelativePath, data, options, assetUrlPath?)` (`metro-transform-worker/src/index.js`) decodes `data` as UTF-8 and branches:

- **Assets** (`type: 'asset'`, not `.json`) go to `transformAsset` before the reserved-string check, since their contents never reach the bundle. `assetTransformer` builds the AST of `module.exports = require(<assetRegistryPath>).registerAsset({…})`, with `assetUrlPath` baked in, from `getAssetData`, which reads scale variants outside the file map (`references/caching.md`, gaps). The AST skips the Babel transformer, then runs through `transformJS` as `js/module/asset`.
- Any other source containing `_$$_METRO_` or the `unstable_dependencyMapReservedName` is rejected (`references/serialization.md` §4).
- **JSON**, by extension, goes to `transformJSON`: no Babel, no dependencies, a string-built `wrapJson` wrapper, a map that maps nothing, and minification under the JS condition.
- **JS** goes to `transformJSWithBabel`, then `transformJS`.

A `js/script` — the module system or a polyfill — skips `collectDependencies` (no dependencies; any `require` stays as written) and is wrapped by `wrapPolyfill` in an IIFE passed the global object.

## The JS pipeline, in order

### The Babel transformer

`transformJSWithBabel` calls `transform` on the `babelTransformerPath` module (default `metro-babel-transformer`, whose types state the contract) with `filename` (project-relative, system separators), `src`, `plugins` (`functionMapBabelPlugin`, `importLocationsPlugin`) and `options`: every transform option but `inlineRequires`, plus `projectRoot`, `publicPath`, `globalPrefix`, `hermesParser`, `enableBabelRCLookup`, `enableBabelRuntime`, `supportsInlineModules` and, under the default `enableBabelRuntime: true`, `babelRuntimeModuleName: 'metro:babel-runtime'` and `babelRuntimeVersion`, the version of Metro's own `@babel/runtime`. The worker reads `metadata.metro.functionMap` and `metadata.metro.unstable_importDeclarationLocs` from the returned `{ast, metadata}`, so a transformer that drops `plugins` loses function maps and ESM tagging. "Transformers can output null ASTs (if they ignore the file)"; the worker then parses the untransformed source, and otherwise parsing is the Babel transformer's job.

`metro-babel-transformer` parses with `parseSync`, or `flow-parser` under `hermesParser`, as `sourceType: 'module'`, then transforms with the project's Babel config from `cwd: projectRoot`, setting `BABEL_ENV` for the call (`'development'` in dev, else the existing value or `'production'`). Its caller data is `{name: 'metro', bundler: 'metro', platform, inlinePlatform, supportsInlineModules}`, plus `babelRuntimeModuleName` and that version as `enableBabelRuntime` when both are given. Its `getCacheKey` runs once, on the main thread (`references/caching.md`).

### Metro's passes

1. `generateImportNames` picks unbound names for `$$_IMPORT_DEFAULT` and `$$_IMPORT_ALL`, shared by the ESM transform, inline requires, `collectDependencies` and the factory's parameters. `'use strict'` is added when `sourceType` is `module`, which `metro-babel-transformer` sets for every file.
2. **One Babel pass**, with no user config, of up to three plugins in order: `importExportPlugin` when `experimentalImportSupport === true` (it has ESM to convert only if the Babel transformer left it); `inlineRequiresPlugin` when the module's `inlineRequires` is true, skipping `nonInlinedRequires`; and `inlinePlugin`, always. The first two work on `Program` exit, in that order, so inline requires sees the ESM transform's calls; it tags each copy it moves with `METRO_INLINE_REQUIRES_INIT_LOC`, which `collectDependencies` reads to keep `locs` and ESM tagging. `inlinePlugin` replaces `Platform.OS` with the platform and a static `Platform.select({…})` with its matching value in every build, and `__DEV__` and `process.env.NODE_ENV` with `false` and `'production'` only when `!dev`. By default (`transformer.unstable_disableInputAstCloning: true`) the pass mutates the Babel transformer's AST in place.
3. **Constant folding, only when `!dev`**, alone: "Run the constant folding plugin in its own pass, avoiding race conditions with other plugins that have exit() visitors on Program (e.g. the ESM transform)." It removes dead branches and unreferenced function declarations until nothing more is stripped; on an uncloned AST it first clears Babel's scope cache, whose stale scopes otherwise lead it to strip functions the earlier passes started using.
4. **`collectDependencies`**, for modules only (below).
5. **Wrapping.** `wrapModule` makes the program the body of `${globalPrefix}__d(function (global, require, <importDefault>, <importAll>, module, exports, <dependencyMap>) {…})`, the order `metro-runtime`'s `require.js` calls factories with; the serializer inserts the module id and dependency map before its last `)` (`references/serialization.md` §4). The unstable `unstable_disableModuleWrapping` skips wrapping and rewriting.
6. **Generation and minification.** Metro minifies when `minify` is set and `unstable_transformProfile` is neither `hermes-stable` nor `hermes-canary`, first shortening the factory's parameter names (`normalizePseudoGlobals`) if the source is at most `optimizationSizeLimit` bytes (default 150 KiB; it gates nothing else). `@babel/generator` prints the code and map, and the `minifierPath` module (default `metro-minify-terser`) receives `{code, map, filename, reserved, config: minifierConfig}`, `reserved` holding the shortened names and any reserved dependency-map name. Map encoding is `references/serialization.md` §5.

The order shapes the graph. Dependencies are collected after folding, so in a non-dev build a `require` in a branch made dead by `__DEV__`, `NODE_ENV` or `Platform.OS` is no dependency, while in dev, with nothing folded, both arms of a `Platform.OS` check are. A module's edges vary with `dev` and `platform`, and both are in the key and the graph id.

## `collectDependencies`

`ModuleGraph/worker/collectDependencies.js` registers each dependency in one traversal and rewrites its call into "ID-independent code", an index into the factory's dependency-map parameter, returning `{ast, dependencies, dependencyMapName}`.

| Source form | `asyncType` | `isESMImport` | Rewritten to |
|---|---|---|---|
| `require('x')`, or an `importDefault`/`importAll` call | `null` | from import locations | `<callee>(_dependencyMap[i])` |
| `import … from 'x'`, `export … from 'x'` | `null` | `true` | none; normally converted earlier |
| `import('x')` | `'async'` | `true` | `require(<asyncRequireModulePath>)(_dependencyMap[i], _dependencyMap.paths)` |
| `__prefetchImport('x')`; unstable `require.unstable_importMaybeSync('x')` | `'prefetch'`; `'maybeSync'` | `true` | that module's `.prefetch(…)`; `.unstable_importMaybeSync(…)` |
| `require.resolveWeak('x')` | `'weak'` | `false` | `_dependencyMap[i]`, the id alone |
| `require.context(…)`, behind `unstable_allowRequireContext` | `null` | `false` | `require(_dependencyMap[i])`, with `contextParams` |

Every call form but `import()` must name an unbound global. The three forms that call `asyncRequireModulePath` (default `metro:runtime/modules/asyncRequire`) add a never-optional sync dependency on it. A `require` gets `isESMImport` when its source location is an import or re-export declaration `importLocationsPlugin` recorded, so a `require` the Babel transformer made from an `import` still resolves as one (`references/resolution.md`).

**Specifiers.** Apart from `require.context`, a dependency call takes exactly one argument, which must `evaluate()` confidently to a string. Any other count throws `InvalidRequireCallError`, as does a non-literal in any form but `require`. A non-literal `require` follows `transformer.dynamicDepsInPackages`: under `'throwAtRuntime'` (the default), in a file whose project-relative path has a `node_modules` segment, it becomes an IIFE that throws `Dynamic require defined at line <n>; not supported by Metro` when reached; elsewhere, or under `'reject'`, the module's transform fails.

**Optional dependencies.** Under `transformer.allowOptionalDependencies` (default `true`, minus its `exclude` names), a call is optional when its first enclosing block, past at most two other statements, is a `try` statement's `block` (not a `finally`, `catch` or nested block), or when a non-nullish `.catch(handler)` or `.then(_, handler)` sits in an unbroken chain on it, as on an `import()`. One non-optional occurrence makes the dependency non-optional. An optional dependency that fails to resolve gets no graph node and a `null` map slot: "An unresolved dependency, which should cause a runtime error when required."

**Keys.** `getKeyForDependency` joins with `\0` the specifier, `import` or `require`, any `asyncType` and, for `require.context`, its parameters; `data.key` is that string's base64 SHA-1. Occurrences with one key share an entry and its `locs`; the same specifier through `require` and `import`, or sync and async, is two dependencies, which can resolve differently. `index` is first-occurrence order, `dependencies` is in index order, and the code reads `_dependencyMap[index]`. `Graph` relies on both halves (`references/graph.md`, "Commit protocol"):

- **The key is the edge's identity.** `Graph` diffs a re-transformed module's edges by key, so a key that changes for an unchanged edge removes and re-adds it and puts the module in the delta. Keys derive only from what an import says, not its position, so an edit that leaves a module's imports alone changes none of them. They must be unique within a module (`buildSubgraph` throws "Found duplicate dependency key") and deterministic, since they are cached.
- **The position is the runtime contract.** The serializer emits the map in `dependencies` order and the code indexes it by position, so `dependencies[i]` is what `_dependencyMap[i]` means, and a reorder is committed even when every edge is unchanged.

`dependencyMapName` is an unbound name from `generateUidIdentifier('dependencyMap')`, or the unstable `transformer.unstable_dependencyMapReservedName`, which the worker rejects in source and keeps unmangled for serializers' string search (`references/serialization.md` §4). In dev, each rewritten call also carries the specifier as a debug name.

## Purity

Output, dependencies included, is a pure function of the file's bytes and project-relative path, the `JsTransformOptions`, the transformer config, an asset's `assetUrlPath` and whatever the transformer's `getCacheKey` hashes (SKILL.md §4). The cache depends on that (`references/caching.md`), and so does `Graph`: `transformOutputMayDiffer` keeps the previous module when `unstable_transformResultKey` is unchanged, so new output under an unchanged key is discarded, even with `cacheStores: []`. That rules out:

- **Other files.** A transform receives one file; whatever else it reads is in no key and triggers no invalidation (the existing cases are `references/caching.md`'s known gaps). Information from other modules belongs in the graph or the serializer, or arrives as a keyed option.
- **Environment and worker state.** `process.env` is unkeyed and `JEST_WORKER_ID` is deleted (`references/graph.md`, "Workers"). Likewise no state may carry from one file to the next: which files a worker sees is scheduling.
- **Machine paths.** Babel gets a project-relative `filename`, but with `cwd: projectRoot` a plugin's `state.filename` is absolute, and `projectRoot` is in the options. Embedding either bakes one machine's path into a shared cache entry (`references/caching.md`, rule 2).
- **Nondeterminism.** Nothing may derive from time, randomness or a counter across files.

## Contracts for a custom transformer

The `transformerPath` module (default `metro-transform-worker`), required by path in every worker and on the main thread, exports `transform(config, projectRoot, projectRelativePath, data, options, assetUrlPath?)`, resolving to a `TransformResult`, and optionally `getCacheKey(config, {projectRoot})`, called once on the main thread; without it, the config contributes only `globalPrefix` to the key (`references/caching.md`, gaps).

A `TransformResult` is `{dependencies, output}`:

- `dependencies[i]` is `{name, data: {key, asyncType, isESMImport, isOptional?, locs, contextParams?}}`, with keys as above; `locs[0]` places a resolution error's code frame.
- `output` has exactly one entry whose `type` starts with `js/` (`getJsOutput` asserts it; serializers ignore the rest): `js/module`, `js/script` (emitted verbatim) or `js/module/asset` (how `getAssets`, `getAllFiles` and source maps recognize an asset). Its `data` holds `code`, a finite `lineCount` equal to `code`'s line count (`/symbolicate` places modules by it, `.map` sections by counting `code`'s lines), `map` (a `VlqMap` of `code`, `references/serialization.md` §5) and `functionMap` or `null`.

Metro also assumes that a `js/module`'s last `)` closes its `__d(` call, since `addParamsToDefineCall` inserts at `code.lastIndexOf(')')`, so no trailing comment may contain one; that `map` and `lineCount` describe `code` exactly, since serializer edits preserve positions (`references/serialization.md` §4); that the result is JSON-serializable for `FileStore` and `HttpStore`; and that it contains `_$$_METRO_` only for a serializer: "Custom transforms should not generate it unless they expect Metro to process it."

## Changing the transform — checklist

1. **Per-request option:** `JsTransformOptions`, the destructuring and `partialKey` in `Transformer.transformFile` (the `extra` check throws otherwise), `splitBundleOptions` and URL parsing, and `getGraphId`. `customTransformOptions` has all four.
2. **Config option:** A JSON-serializable `transformer.*` field reaches the worker and the default key unaided (`references/caching.md`, rule 5). Strip it in the `Transformer` constructor if the worker doesn't need it; add a default, validation and docs (SKILL.md's design checklist, step 8).
3. **`getTransformOptions` output:** `ExtraTransformOptions.transform`, `calcTransformerOptions` and the key; it is fixed per graph.
4. **Plugin or worker helper, or new output for the same input:** The changed file must be in `getTransformPluginCacheKeyFiles` or the worker's `getCacheKey` list in the same commit, since the transform cache has no breaker (`references/caching.md`). Place a plugin after any plugin whose `Program`-exit output it consumes, after the inline pass if it needs inlined values, and before `collectDependencies` if it removes code; state its `dev`, `minify` and option gating.
5. **Dependency kind:** A distinct `asyncType` or qualifier in `getKeyForDependency`, so it can't collapse into an existing edge, and a `DependencyTransformer` method (with a no-op in the worker's `disabledDependencyTransformer`); then the graph (`references/graph.md`, "Adding something to this layer"), the serializer's `paths`, which cover every non-null `asyncType`, and `metro-runtime`.
6. **Tests:** `metro-transform-worker/src/__tests__/index-test.js`, `ModuleGraph/worker/__tests__/collectDependencies-test.js` and `metro-transform-plugins/src/__tests__/`, in dev and not, minified and not, as script and module.
