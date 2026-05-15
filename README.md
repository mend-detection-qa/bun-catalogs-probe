# bun-catalogs-probe

Mend SCA detection probe — Tier 4, entry #21.

## Pattern bundle

- **D4** (`catalog` — default shared version map, Bun 1.2+): The root `package.json` declares a top-level `"catalog"` object mapping package names to version ranges. Workspace members reference it with the `"catalog:"` version token in their own `dependencies`. Mend must expand `"catalog:"` to the concrete resolved version recorded in `bun.lock`.
- **D5** (`catalogs` — named catalogs, Bun 1.2+): The root `package.json` also declares a `"catalogs"` object with named sub-maps (`"react18"` here). Workspace members reference a named catalog with `"catalog:<name>"` (e.g., `"catalog:react18"`). Mend must resolve the catalog name, find the version range in the correct sub-map, and expand to the concrete resolved version.

## Bun 1.2+ requirement

`pm_version_under_test: "1.2.0"`

The `catalog` and `catalogs` fields were introduced in Bun 1.2 (released 2025-01-14). Any Mend parser built against Bun 1.0.x or 1.1.x will not have these fields in its schema, and catalog-token version strings (`"catalog:"`, `"catalog:react18"`) will be treated as unknown specifiers. The expected failure modes are:

1. Deps appear with the literal version string `"catalog:"` or `"catalog:react18"` (token not expanded).
2. Deps are dropped entirely (token treated as invalid).
3. Default catalog (`catalog:`) expands correctly, but named catalog (`catalog:react18`) does not (two-phase failure).

This probe is the designated instrument for open question Q3 in `docs/BUN_COVERAGE_PLAN.md §9`.

## Why bundled

D4 and D5 share the same catalog-expansion code path inside Bun's manifest parser. The root `package.json` must be read to build the expansion table (both `catalog` and `catalogs` live there), and then each workspace member's `dependencies` must be re-resolved by substituting catalog tokens for the concrete version ranges before lockfile lookup. Bundling both in one probe means a parser that partially handles `catalog:` but ignores `catalog:<name>` surfaces as one compound partial failure — comparator can distinguish the named vs unnamed failure axis.

The standalone pattern `workspace-cross-reference` (probe #4, `bun-workspaces-probe`) already covers the basic `workspace:*` cross-reference; this probe focuses exclusively on the catalog expansion layer on top of workspaces.

## Project layout

```
bun-catalogs-probe/
├── package.json                  root (workspaces + catalog + catalogs)
├── bun.lock                      text JSONC (Bun 1.2+, lockfileVersion: 1)
├── index.ts                      minimal stub
├── packages/
│   ├── app/
│   │   └── package.json          uses "catalog:" and "catalog:react18"
│   └── lib/
│       └── package.json          uses "catalog:"
└── expected-tree.json
```

## Catalog to expanded-version table

| Token | Catalog source | Version range | Resolved version | Used by |
|---|---|---|---|---|
| `"catalog:"` | `root.catalog.zod` | `^3.25.0` | `zod@3.25.68` | `app`, `lib` |
| `"catalog:react18"` (react) | `root.catalogs.react18.react` | `^18.0.0` | `react@18.3.1` | `app` |
| `"catalog:react18"` (react-dom) | `root.catalogs.react18.react-dom` | `^18.0.0` | `react-dom@18.3.1` | `app` |

### Transitive chain

```
react@18.3.1
  └── loose-envify@1.4.0
        └── js-tokens@4.0.0

react-dom@18.3.1
  └── scheduler@0.23.2
        └── loose-envify@1.4.0  (deduplicated)
              └── js-tokens@4.0.0  (deduplicated)

zod@3.25.68
  (no runtime deps)
```

Total: 3 direct catalog-expanded deps (zod, react, react-dom) + 3 transitives (scheduler, loose-envify, js-tokens) + 2 workspace members (app, lib) = 8 unique nodes.

## Mend config

No `.whitesource` is emitted for this probe. Bun is NOT in Mend's `install-tool` supported list — `scanSettings.versioning` cannot pin a Bun toolchain, and there is no UA pre-step toggle for Bun. The UA javascript.md resolver file has no registered resolver for `bun.lock`; Bun is not a known ecosystem in the UA's lock-file dispatch table. Detection must rely entirely on static `bun.lock` parsing.

Because there is no `bun` install-tool entry, the `scanSettings.versioning` block would be silently ignored even if written. This limitation is documented in `docs/BUN_COVERAGE_PLAN.md §4` and is itself a probe target (probe #24, `bun-not-in-install-tool-probe`).

## Failure modes targeted

| Failure | Symptom in Mend output | BUN_COVERAGE_PLAN.md ref |
|---|---|---|
| `catalog:` token not recognized | `zod` appears with version `"catalog:"` | D4 |
| `catalog:react18` token not recognized | `react`/`react-dom` appear with version `"catalog:react18"` | D5 |
| Catalog field not parsed | All catalog-sourced deps absent from tree | D4 + D5 |
| Named catalog lookup unimplemented | `zod` correct, `react`/`react-dom` absent or unversioned | D5 |
| Deduplication broken | `zod` appears twice (once per workspace member) at different or same version | D4 |
| Transitives missing | `scheduler`, `loose-envify`, `js-tokens` absent | Cascade from D4/D5 |
| JSONC parser crash | Entire tree empty (trailing commas / `//` comments cause `JSON.parse` failure) | §4 row 1 |

## Key assertions for autotest config

When wiring this probe into `automation_config_files/bun-catalogs-probe/autotest_config.json`:

- `validate_scan_output.security_scan_report_conclusion: "success"`
- `dependency_tree.check_update_request_sca_results: true`
- `dependency_tree.expected_package_managers.contains: ["bun"]`
- `dependency_tree.expected_direct_dependencies_number.eq: 5` (app + lib + zod + react + react-dom from app's perspective; adjust if Mend reports per-workspace sub-trees with `eq: 3` for app and `eq: 1` for lib)
- `dependency_tree.expected_transitive_dependencies_number.gte: 2` (scheduler, loose-envify, js-tokens — allow tolerance)
- `dependency_tree.direct_dependencies_names_list.contains: ["zod", "react", "react-dom"]`
- `dependency_tree.transitive_dependencies_names_list.contains: ["scheduler", "loose-envify"]`

Note: if Mend reports literal version strings like `"catalog:"` for any dep, the dep will fail the version-match check in the comparator. Record observed values in the probe results note field.

## Resolver notes

Bun does NOT appear in the UA `javascript.md` resolver file. The lock-file dispatch table maps:

- `yarn.lock` -> YarnLockCollector
- `package-lock.json` / `npm-shrinkwrap.json` -> NpmLockCollector
- `pnpm-lock.yaml` -> PnpmLockCollector
- `bun.lock` -> **not listed**

The UA will likely fall back to npm ls or filesystem scan when `bun.lock` is the only lockfile. Neither fallback understands `catalog:` tokens. This means:

1. The catalog expansion table (built from `package.json` `catalog`/`catalogs` fields) is never consulted.
2. Deps declared with `"catalog:"` are reported with the literal token as a version, or dropped.
3. This probe's `expected-tree.json` encodes the CORRECT expanded tree — the comparator will flag the gap.

## Tracked in

`docs/BUN_COVERAGE_PLAN.md` §11.4 entry #21 (D4 + D5, Bun 1.2+)