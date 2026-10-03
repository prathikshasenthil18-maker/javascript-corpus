# TypeScript Order Platform -- Monolith (TS_V20_VITE_NPM_MONO)

Tool-evaluation repository for **Node 20**, bundled with **vite**,
managed with **npm**, in a **Monolith** layout.

This is branch **TS_V20_VITE_NPM_MONO** of the consolidated `typescript-corpus`
repository. It is the TypeScript counterpart of `JS_V20_VITE_NPM_MONO`:
same Node family, same Vite bundler, same npm package manager, same monolith
layout.

## Project type

- **Language:** TypeScript 5.9.3
- **Runtime:** Node 20 (verified against 20.20.2)
- **Scenario:** 1 - Monolithic
- **Architecture:** Monolith
- **Module layout:** flat
- **Bundler:** Vite 8.2.2 (Rollup linker + its own esbuild transform)
- **Package manager:** npm 10.8.2
- **Source root:** `src`

Every version in this repository was resolved against the live npm registry
and then executed on a real Node 20.20.2 interpreter. None was written from
memory.

## Branches

| Variable | This branch |
| --- | --- |
| Branch | `TS_V20_VITE_NPM_MONO` |
| Node.js | 20.20.2 (family V20) |
| Bundler | Vite |
| Package manager | npm |
| Bundled npm | 10.8.2 |
| Architecture | Monolith |
| Source root | `src` |

## Supported tools

29 tool families are wired. Each has a folder under `tools/` containing a
`trigger.yaml` manifest, a runner, and its configuration.

| Family | Pinned | Family | Pinned |
|---|---|---|---|
| TypeScript (tsc) | 5.9.3 | ts-morph | 27.0.2 |
| vite | 8.2.2 | ts-prune | 0.10.3 |
| mocha | 11.8.0 | madge | 8.0.0 |
| c8 (coverage, primary) | 12.0.0 | dependency-cruiser | 17.4.3 |
| nyc + ts-node (cross-check) | 18.0.0 | Stryker | 9.6.1 |
| eslint | 10.9.1 | fast-check | 4.9.0 |
| typescript-eslint | 8.68.0 | cdxgen | 12.8.4 |
| eslint-plugin-sonarjs | 4.2.0 | ORT (cdxgen licence proxy) | n/a |
| eslint-plugin-security | 4.0.1 | npm-check-updates | 22.2.9 |
| eslint-scope | 9.1.2 | npm audit / ls | 10.8.2 |
| jscpd | 5.0.16 | OpenTelemetry sdk-node | 0.221.0 |
| Grype | v0.110.0 (binary) | Lizard | pip |
| pydriller | pip | GitHub Advisories + API | REST |
| knip | 6.32.2 | vitest + @vitest/coverage-v8 | 4.1.11 |
| @biomejs/biome | 2.5.10 | | |

### Tools deliberately NOT wired

Skipping these is a finding, not an omission. See [`dataset.json`](dataset.json).

| Tool | Reason |
|---|---|
| OSV-Scanner | `api.osv.dev` unreachable -- 403 at the egress proxy |
| npm downloads API | `api.npmjs.org` unreachable -- 403 at the egress proxy |

Declaring any of these would have produced a metric that cannot be computed.

## Build

```bash
# Corepack packageManager is npm@10.8.2; lockfile is package-lock.json
npm install
npm run build
```

`npm run build` type-checks with `tsc --noEmit`, emits CommonJS + declarations
to `dist/`, then bundles with **Vite 8.2.2 (Rollup linker + its own esbuild
transform)** and **executes the bundle**. Emitting is not proof; running it is.

There is no Makefile. A Makefile would be classified as a native MAKE project
and hide the npm lockfile from Testable's primary Node profile.

## Run

```bash
node dist/src/index.js
```

## Test

```bash
npm test           # mocha over tests/
npm run coverage   # c8 (primary)
npm run coverage:nyc
```

Both coverage tools must report non-zero. They deliberately disagree: c8 reads
V8 coverage of the emitted output and remaps it, while nyc instruments the
TypeScript AST directly through `ts-node/register`. Identical numbers would mean
one of them is not an independent second opinion.

`nyc` here never reports through a source-map remap. Doing so silently yields
0% -- the file is remapped to `.ts`, an `--include` written against `dist/**`
stops matching, and the report empties while the process still exits 0.

## Architecture

**Monolith.** One deployable package. `package.json` declares **no**
`workspaces` field, the module tree under `src/` is flat, and there is no
`services/` directory. Those are exactly the properties an auditor reads to
classify a repository, so they are the ones held true here.

```
src/
  index.ts            public surface + sample runner
  models/             domain records and tax table (leaf layer)
  services/           pricing rules, order service, the duplicate pair
  platform/           integrations that use the planted dependency pins
  analysis/           planted fixtures -- never imported by real code
```

`dependency-cruiser` enforces the layering: `models/` may not import
`services/`, and nothing outside `analysis/` may import `analysis/`.

## Planted fixtures

Nothing in `src/analysis/` is production code. Each file exists so exactly one
tool family has something real to find, **using the committed configuration,
with no extra flags**.

| Fixture | Found by |
|---|---|
| `src/services/retail-order-processor.ts` + `wholesale-order-processor.ts` | jscpd -- a duplicate pair, at default thresholds |
| [`src/analysis/complexity-sample.ts`](src/analysis/complexity-sample.ts) | eslint + sonarjs -- cyclomatic 27, cognitive 74 |
| [`src/analysis/sast-fixture.ts`](src/analysis/sast-fixture.ts) | eslint-plugin-security |
| [`src/analysis/taint-fixture.ts`](src/analysis/taint-fixture.ts) | 4 taint flows + 1 sanitised control |
| [`src/analysis/dead-code.ts`](src/analysis/dead-code.ts) | ts-prune, eslint-scope |
| [`src/analysis/call-graph-sample.ts`](src/analysis/call-graph-sample.ts) | madge, dependency-cruiser -- depth 5, fan-out 6 |
| Five pinned dependencies | npm audit, Grype, GitHub Advisories -- see [`tools/grype/PLANTED-CVES.md`](tools/grype/PLANTED-CVES.md) |

The duplicate pair sits in real service code, not in `analysis/`, because
duplication inside a fixtures folder is trivially dismissed.

## Tool entry points

```bash
ts-node tools/tool_integration.ts             # wiring banner
ts-node tools/tool_integration.ts --list      # machine-readable list
ts-node tools/tool_integration.ts --verify    # folder + manifest + runner for every tool
ts-node tools/tool_integration.ts --run jscpd # one tool
ts-node tools/tool_integration.ts --run-all   # every tool, in order
ts-node tools/full_check.ts                   # cross-file consistency audit
```

Every tool can also be run directly: `bash tools/<tool>/run_<tool>.sh`.

CI runs **every one of these runners** and uploads their output as artifacts.
A CI file that only installs and tests would leave the declared tools unproven.

## Layout

```
typescript-corpus/  (TS_V20_VITE_NPM_MONO)
|-- .github/  (1 files)
|-- src/  (15 files)
|-- tests/  (5 files)
|-- tools/  (68 files)
|-- .editorconfig
|-- .gitignore
|-- .jscpd.json
|-- .madgerc
|-- .npmrc
|-- .nvmrc
|-- biome.json
|-- dataset.json
|-- eslint.config.mjs
|-- knip.json
|-- package-lock.json
|-- package.json
|-- tsconfig.build.json
|-- tsconfig.json
|-- vite.config.mts
|-- vitest.config.ts
```

## Verification

Every claim in this README is checked by `ts-node tools/full_check.ts`, which
reads its expectations **from the repository** rather than from a hard-coded
list -- including that `.nvmrc`, `package.json` engines, `dataset.json`, the CI
workflow and all 29 `trigger.yaml` manifests agree on the Node version, the
branch, and the architecture.
