# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

This is the **Breeze 1.x** client (`breeze.js`, currently 1.7.1). It is obsolete and unmaintained upstream; active development moved to [breeze-client](https://github.com/Breeze/breeze-client) (Breeze 2.x, TypeScript). Recent commits here only touch the test suite, which is still used to exercise the server-side repos (`breeze.server.net`, `breeze.server.node`). Keep changes minimal and in the existing ES5 style.

There is no root `package.json`. The only npm project is `build/`.

## Build

```bash
cd build
npm install
npx gulp            # default: minify, minify.base, copyBreezeExtns, copyForTests, yuidoc, intellisense, dtsgen
npx gulp minify     # just build/breeze.debug.js + breeze.min.js
npx gulp minify.base
npx gulp copyBreezeExtns   # refresh build/adapters/
npx gulp dtsgen            # regenerate build/typings/{breeze,index}.d.ts
npx gulp yuidoc            # regenerate docs/api-docs/ (runs the yuidoc CLI from src/)
npx gulp copyForTests      # copy build/breeze.* -> test/breeze/ (gitignored)
```

Gotchas:
- The build uses **gulp 3**, which does not run on Node 12+ (`primordials is not defined`). Use an old Node via nvm, or override `graceful-fs` to v4 with an npm `overrides` entry.
- `minify` tasks use `gulp-newer`, so they are skipped when the outputs are newer than every source. Delete `build/breeze*.js` to force a rebuild.
- **Build outputs are committed** (`build/*.js`, `build/adapters/`, `build/typings/`, `docs/api-docs/`). After changing anything in `src/` or `typescript/`, regenerate and commit the outputs too.

## Tests

Tests are **browser-only QUnit 1.17** suites in `test/internal/`; there is no Node test runner. Each `test/Index.*.html` / `.aspx` page targets one backend (ASP.NET Core EF6 / EF Core / NHibernate, .NET WebApi, OData v3/v4/WCF, Mongo, Sequelize, Hibernate) that lives in a sibling repo and serves this `test/` folder. The test pages load `breeze/breeze.debug.js`, which is the gitignored copy produced by `gulp copyForTests` — run the build first.

To run tests, open a page with the required URL params:

```
index.aspcore.ef6.html?canStart=true                     # all tests (nothing runs without canStart)
index.aspcore.ef6.html?canStart=true&filter=entityKey    # single test / substring filter
index.aspcore.ef6.html?canStart=true&modelLibrary=knockout   # backingStore (default) | knockout | backbone
```

Test conventions (`test/internal/testFns.js`):
- `setServerVersion(name, version)` in each page sets `testFns.DEBUG_*` flags, picks the `dataService`/`uriBuilder` adapters, and sets `defaultServiceName`.
- Backend-specific tests use `testFns.skipIf("mongo,odata", "reason").test(...)`. Tokens map to the `DEBUG_*` flags; chain `.skipIf(...)` for multiple conditions.
- Every test file is an IIFE taking `testFns`; new files must also be added to the `<script>` list in each relevant `Index.*` page.

## Architecture

### Conceptual model: a LINQ-style query provider for the browser

Breeze is a client-side take on LINQ + Entity Framework. The pieces map directly onto that idea:

- `EntityQuery` (`a45_`) is the fluent query builder (`.from().where().orderBy().expand().select().skip().take()`); it only builds a description, nothing executes until `executeQuery`.
- `Predicate` (`a44_`) is the expression tree behind `where`. It is evaluated by **visitors**, which is how one query runs in different places: `toJSONVisitor` serializes it, `toFunctionVisitor` compiles it to a JavaScript function, and the OData uriBuilder adds `toODataFragmentVisitor`.
- The `uriBuilder` adapter is the "provider" that translates the tree into a server request. `odata` renders `$filter` / `$orderby` / `$skip` / `$top` / `$expand` / `$select` / `$inlinecount` and is the default (used by the classic .NET WebApi, OData, and Mongo backends); `json` ships the tree as JSON (selected in `testFns` for ASP.NET Core, Sequelize, and Hibernate). The server then turns it into a real LINQ / SQL / Mongo query. Adapters that do not set `uriBuilderName` on their `DataService` get the default instance.
- `EntityManager` (`a50_`) is the client-side unit of work and identity-map cache (the `DbContext` analogue): `executeQuery` merges server results into the cache, `executeQueryLocally` runs the same `EntityQuery` against the cache by compiling the predicate to a function, and `saveChanges` / `getChanges` / `rejectChanges` / `exportEntities` / `importEntities` handle change tracking and persistence.
- `MetadataStore` / `EntityType` (`a40_entityMetadata.js`) hold the schema, usually fetched from the server; queries are validated against it before being sent, and `entityAspect` (`a30_`) wraps each entity with its `EntityState`, original values, and validation errors.

### Source is concatenated, not modular

`src/` files are **not** CommonJS/ES modules. The gulp `minify` task concatenates them in filename order into a single UMD/IIFE (`_head.jsfrag` ... `_tail.jsfrag`), so every file shares one `"use strict"` scope and top-level `var`s are visible across files. The numeric prefix is the dependency order:

- `a01_`–`a60_`: core (`coreFns`, `assertParam`, `enum`, `event`, `config`), then entity model (`entityAspect`, `entityMetadata`, `entityManager`), then query/save (`predicate`, `entityQuery`, `mappingContext`, `abstractDataServiceAdapter`).
- `b00_breeze.<interface>.<name>.js`: the "embedded" adapters bundled into `breeze.debug.js`. Each has its own UMD wrapper so the same file also works standalone; `copyBreezeExtns` copies them to `build/adapters/breeze.<interface>.<name>.js`.
- `src/breeze.*.js` with no prefix (`breeze.dataService.mongo`, `breeze.modelLibrary.backbone`, `breeze.bridge.angular`): external-only adapters, copied to `build/adapters/` but never bundled.
- `_tailbase.jsfrag` is used for `breeze.base.debug.js`, which excludes all `b??_` adapters and does **not** auto-initialize any; callers must run `breeze.config.initializeAdapterInstances(...)` themselves.

A file may only reference top-level names from files with a lower prefix. When adding a file, choose a prefix that places it after its dependencies. `__`-prefixed helpers (`__objectForEach`, `__requireLibCore`, ...) are file-private by convention only.

### Adapter / interface registry (`a20_config.js`)

Four pluggable interfaces: `ajax`, `modelLibrary`, `dataService`, `uriBuilder`. An adapter is a constructor whose instances have a `name` property, registered with `breeze.config.registerAdapter(interfaceName, ctor)` and activated with `initializeAdapterInstance(interfaceName, adapterName, isDefault)`. `_tail.jsfrag` sets the defaults (`webApi`, `jQuery`, `odata`, and `ko` if Knockout is present else `backingStore`).

Third-party libraries (jQuery, Knockout, Backbone, Angular, Q, datajs) are **optional**: locate them with `core.requireLib("jQuery;jquery")` / `__requireLibCore` at adapter `initialize()` time and fail only when actually used. Never add a hard dependency.

### Docs and IntelliSense come from YUIDoc comments

`@module` / `@class` / `@method` / `@param` comments in `src/` drive `docs/api-docs/` (via `yuidoc`) and `build/breeze.intellisense.js` (via `build/intellisense/intellisenseGenerator.js`, which reads `docs/api-docs/data.json`). Public API changes need matching doc comments.

### TypeScript definitions are templated

`typescript/breeze.source.d.ts` is the single source. Lines starting with `//~` are Handlebars directives; `gulp dtsgen` strips the `//~` prefix and renders twice: with `modules=false` into `build/typings/breeze.d.ts` (global `breeze.*` namespaces) and with `modules=true` into `build/typings/index.d.ts` (ES module, `breeze.` prefixes removed). Edit only the source file. `typescript/breeze-tests.ts` is a compile-only check of the global definitions.

## Releasing

Version lives in `src/_head.jsfrag` (`version` and `metadataVersion`). The cross-repo release checklist (bower, npm `breeze-client` 1.x, NuGet, breeze.github.io) is in `build/HowToUpdateBreeze.md`.
