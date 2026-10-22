# Actual 26.10.0 upgrade audit

Date: 2026-10-22. Worktree: `actual/`, branch `codex/actual-upgrade`.
Local-only work: no push, deployment, NAS connection, production request, or credential discovery.

## Result

Use the official 26.10.0 application/API. The parent-context feature is already present in this release, so no duplicate cherry-pick or custom runtime overlay is needed for that feature.

| Fork change | Status | Evidence / reason |
| --- | --- | --- |
| `ec7a7ee38` Expose parent transaction text to split rule formulas | Obsolete as a downstream patch; upstream includes it | PR [#8898](https://github.com/actualbudget/actual/pull/8898), merged 2026-09-13; listed in [26.10.0 release notes](https://actualbudget.org/blog/release-26.10.0/). Release tree includes it through `203034455` (release update #9012). |
| `fda3d318e` Categorize split parent context as enhancement | Obsolete | The released notes already place #8898 under Enhancements. No pending duplicate release note added. |
| `46c6463ba` earlier 26.8.1 parent-context variant | Superseded | Same downstream feature; do not replay the older implementation. |

`ec7a7ee` is not a Git ancestor of v26.10.0, but ancestry alone would give the wrong result here. The tag contains all four parent fields, literal text handling, cleanup, formula completions/badges, and both new test files. The two new test files have zero diff against ec7a7ee. `action.ts` and `rule.ts` also have zero diff; transaction-rules additionally includes upstream lazy balance calculation. These are real rules/API/editor changes, not Docker-only setup.

## Fork/ref inventory

Inspected local Git refs without fetching or changing other branches/worktrees. Remote-tracking refs may be stale. `fork/master` is `db1b0ea97`, `origin/master` is `4a7ba9687`; origin has 13 commits beyond fork/master. The fork/master-to-origin delta is upstream application work, not a private Docker customization. Many fork refs are copied PR branches (`claude/*`, `jfdoming/*`, `cross-version-sync*`, etc.); their names alone do not establish ownership.

An all-ref author search (`git log --all --author='krzysztof\|Krzysztof'`) identifies only the three commits above as authored by Krzysztof. The upstream feature branch, including `fork/feat/split-parent-formula-context-upstream`, ends at fda3d31; its feature parent is ec7a7ee, based on db1b0ea97. The older local feature branch ends at 46c6463. No other author-identified private code patch was found. Local origin/master lacks the feature, but the requested v26.10.0 release tree and public merged PR establish its inclusion. No branch other than the requested upgrade branch was modified.

## Server/API changes relevant to this upgrade

Source comparison: `v26.9.0..v26.10.0`, checked against the official [release notes](https://actualbudget.org/blog/release-26.10.0/).

- Budget migration `packages/loot-core/migrations/1788468782000_add_messages_pending.js` creates the internal pending-sync-message table, keyed by dataset/row/column. Newer migrations are additive under the compatibility policy. Back up budget/server data before opening it with the new client/API; rollback must restore compatible data as well as images.
- Cross-version synchronization defers changes requiring newer migrations. API/CLI users receive a warning when newer changes await an update. Improved compatibility is not a guarantee of full/fresh results from arbitrary older SDKs.
- Server folder migration changes from `.js` to `.ts`; the migration loader preserves its persisted `.js` title, avoiding a repeat migration.
- `ACTUAL_OPENID_CLIENT_SECRET_FILE` and `ACTUAL_GITHUB_TOKEN_FILE` are new optional configuration inputs. Their trimmed contents override regular config/env values. This audit reads only implementation/tests, never secret files. Existing OIDC authentication logic has no change in this release diff; no mandatory environment renaming was found.
- API adds synced preference setting and duplicate-transaction merging. Parent split-rule fields are now included in the released engine. No mandatory incompatible API call replacement was identified for the MCP tools.
- Server Dockerfile also fixes inclusion of the Peggy build workspace. Full server image construction was not tested here.

The allowed snapshot `project/ACTUAL_MCP_UPGRADE_PLAYBOOK.md` explicitly requires matching Actual Server and `@actual-app/api` versions and documents older SDK failures such as `invalid-schema` or missing SQLite tables. Keep server, MCP API/core, and the Enable Banking derivative image at 26.10.0 together. Protocol tolerance may defer new data; it does not replace that operational rule.

## Local verification

Runtime: repo `.nvmrc` asks for v24.18.1, engines require >=22.18.0. System Node was v26.3.1 and had no Corepack. Installed isolated tooling under the working root `.tooling/`: Node v24.22.1 (same required major), Corepack 0.34.5, repo-pinned Yarn 4.17.1. Corepack/Yarn caches were redirected into the working root. No Node-version blocker remains.

| Check | Result |
| --- | --- |
| Corepack Yarn dependency install | Initial global cache attempt was sandbox-blocked; local-cache retry linked dependencies but exited 1 building `sharp@0.32.6`. Build failure is an unmodified dependency installation limitation; no successful full install is claimed. |
| `yarn workspace @actual-app/core test:node src/server/transactions/transaction-rules.test.ts src/server/rules/split-parent-context.test.ts src/server/accounts/split-formula-import.test.ts` | PASS: 3 files, 46 tests |
| `yarn workspace @actual-app/core test:node src/server/migrate/additive-migrations.test.ts src/server/migrate/migrations.test.ts src/server/sync/deferred.test.ts` | PASS: 3 files, 57 tests |
| `yarn build:api` | PASS: API/core/CRDT/CLI build pipeline, 4 successful tasks |
| `yarn build:browser --skip-translations` | PASS: web/plugins pipeline; avoids translation clone/pull and checkout mutations |

No full monorepo test run, server container build, browser interaction, OIDC login, bank connection, or production migration was performed. Ignored local logs are in the working root `.tooling/actual-*.log`.

## Reproducing web/API artifacts

Run commands from this repository root with Node 24 and Corepack (Yarn selected by packageManager):

```sh
corepack yarn install
corepack yarn build:browser --skip-translations
corepack yarn build:api
```

Outputs: `packages/desktop-client/build/` and `packages/api/dist/`. The NAS recipes formerly copied a custom web build into `/app/node_modules/@actual-app/web/build` in the official server image and copied custom API dist into MCP. With 26.10.0, official web/API already include parent-context; prefer official artifacts over repeating those overlays. For a future additional custom change, copy these build outputs using equivalent recipe Dockerfiles, remove macOS `._*` files, and preserve package/version identity.

The MCP still needs its documented SDK request-timeout patch, independently of parent-context. Do not overwrite its patched `dist/index.js` with an unpatched custom API bundle after installation. The Enable Banking image is FROM the MCP image and inherits its API; rebuild/retest it as part of any later approved deployment. Deployment still needs backups, image builds on the target architecture, coordinated image changes and fresh readback against the actual budget. None is authorized or performed in this task.
