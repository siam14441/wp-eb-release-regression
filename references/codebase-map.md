# Codebase map — read first, every run

Facts here have each been wrong-by-assumption at least once. Counts and versions drift between
releases: **re-verify the numbers, trust the structure.**

## Repositories

Paths are machine-specific and live in `references/environment.md` (`$FREE`, `$PRO`, `$CONTROLS`, `$QA_HOME`).

| What | Location | Remote |
|---|---|---|
| Free (build) | `$FREE` | `WPDevelopers/essential-blocks` |
| Pro (build) | `$PRO` | `WPDevelopers/essential-blocks-pro` |
| Controls | submodule at `essential-blocks/src/controls` | `EssentialBlocks/controls` |
| Read-only mirrors | `$QA_HOME/repos/{essential-blocks,essential-blocks-pro}` | same |

The installed plugins under `wp-content/plugins/` are **ZIP builds, not git repos**. Pro's ZIP ships
no `src/` at all (Free's does), so you cannot diff Pro's installed copy against source the same way.

The read-only mirrors are **read-only including `git fetch` and `git pull`** — fetch writes into
`.git/`, which counts. Only `status`, `log`, `diff`, `show`, `branch -a`, `grep`, `Read`. If you need
fresher refs than are local, ask.

## Baselines — resolve per repo, never assume

| Repo | Baseline | Trap |
|---|---|---|
| Free | `origin/master` | `origin/main` is stale (Sept 2024, ~2455 commits behind). Diffing it buries the release. |
| Pro | `origin/main` | There is no `master` branch. |
| Controls | `origin/master` | Verify the submodule pointer equals the release head exactly. |

For a point release, the previous release **tag** (`v6.4.2`, `v3.2.0`) is the right baseline.

```bash
git -C "$REPO" rev-parse --verify origin/master 2>/dev/null || git -C "$REPO" rev-parse --verify origin/main
git -C "$REPO" log -1 --format=%ci origin/master   # check the date before trusting a ref
```

**Releases are cut from `staging`, not `master`.** The flow is `release-* → staging → master`. During
6.4.3, `origin/master` did not contain the security work — cutting from master would have shipped a
release with none of it. Confirm which branch the release actually descends from before diffing.

Controls submodule pointer changes appear in the parent diff as `-/+Subproject commit <sha>`:

```bash
DIFF=$(git -C "$FREE" diff "$BASE" -- src/controls)
OLD=$(echo "$DIFF" | awk '/^-Subproject commit/{print $3}')
NEW=$(echo "$DIFF" | awk '/^\+Subproject commit/{print $3}')
git -C "$CONTROLS" log --oneline "$OLD..$NEW"
```

## Build recipe and its traps

**Directory names are load-bearing.** Both webpack configs hardcode path regexes
(`/\/essential-blocks\/src\/(blocks\/[\w-]+)\//` and the pro equivalent), and pro's
`webpack.config.js` dereferences `match[1]` with **no null guard**. Clone into folders named exactly
`essential-blocks` and `essential-blocks-pro` or pro's build dies with
`Cannot read properties of null (reading '1')`.

**Free's `pnpm-lock.yaml` is unusable** — lockfileVersion 5.4 (pnpm 7 era) so pnpm 11 refuses it, and
it is stale (`package.json` wants `wordpress-icon-picker ^1.2.4`, lockfile says `^1.2.2`). It is in
`.gitignore` while still tracked, so bumps were never committed.

```bash
# Free
npm install --legacy-peer-deps
npm install ajv@8 --no-save     # npm hoists ajv@6, breaking ajv-keywords
npm run build
# Pro
pnpm install && npm run build
```

**Consequence:** the resulting dependency tree is not the intended one. Treat any missing-asset
finding from a locally built tree as **inconclusive, not a defect**.

**Controls needs no separate build.** Free's webpack compiles `src/controls/src/index.js` into
`assets/admin/controls/controls.js` as the global `window.EBControls`; both plugins treat
`@essential-blocks/controls` as an external pointing at that global. Build order is **free, then pro**.

The submodule URL is SSH and fails without a key:

```bash
git -C "$FREE" config submodule.controls.url https://github.com/EssentialBlocks/controls.git
git -C "$FREE" submodule update --init --recursive
```

**Deploy** by `rsync` excluding `node_modules` and `.git`. Keep checkouts **outside**
`wp-content/plugins/` — a manual "Add Plugin" upload over a folder containing `node_modules`
exhausted PHP memory mid-delete and destroyed the folder.

## Counts (verify per release)

| Thing | Value |
|---|---|
| `includes/blocks.php` entries | 79 = 47 free + 11 new + 21 pro catalog stubs |
| Blocks that actually register | 94 = 69 free + 25 pro |
| `deprecated.js` files | 47 at 6.4.3 (was 51 at 6.4.0) — **count drifts, always recount** |
| `src/blocks/` dirs (free ZIP) | 69 — more than registered; inner blocks have dirs, no registry entry |
| `includes/Blocks/` classes | free 64, pro 29 (21 registered + 8 inner/child) |
| Block patterns (editor settings, `essential-blocks/*`) | 79 at 6.4.4 / 3.2.2 -- from 12 Free + 12 Pro JSON files in `patterns/`; registered on `admin_init`, so not visible through REST |

The 21 pro entries in free's `blocks.php` are **catalog stubs** (`'is_pro' => true`, no `'object'`)
driving the Block Manager UI and upsell. Real pro registration lives in `essential-blocks-pro/includes/blocks.php`.

**Pro blocks register under the `essential-blocks/` namespace**, not `essential-blocks-pro/` — only
the slug gets a `pro-` prefix (`essential-blocks/pro-one-page-navigation`). Check `block.json`'s
`name` field; never infer the namespace from the folder.

Block categories: content 21 · creative 18 · dynamic 14 · woocommerce 7 · form 7 · marketing 5 ·
social 4 · layout 3.

## Attack surface

**REST — all free routes under `essential-blocks/v1`, registered via `API/Base.php:19`, dispatched by
`API/Server.php`:**

| Route | Method | Class | Permission |
|---|---|---|---|
| `/v1/products` | GET | `API/Product.php:19` | `__return_true` |
| `/v1/queries` | GET | `API/PostBlock.php:19` | `__return_true` |
| `/v1/queries` | POST | `API/PostBlock.php:24` | `verify_post_permission` |
| `/v1/roles` | GET | `API/Common.php:14` | `__return_true` |

`Base::verify_post_permission()` is a **stub that returns true** — the "protected" POST path is
effectively unauthenticated. Pro adds a public Protected-Content unlock route
(`Utils/ProtectedContent.php:190`, `permission_callback => '__return_true'`) and licensing endpoints
(`Deps/WPDeveloper/Licensing/Api.php:134`).

**AJAX — 30+ actions.** `Integrations/ThirdPartyIntegration.php:29`'s `add_ajax()` registers **both**
`wp_ajax_` and `wp_ajax_nopriv_` when passed a string; only the array form (`['public' => true]`)
is explicit. Editor-only actions reaching `nopriv` has been a real vulnerability class here.

Sources: `AI/AI.php` · `Form.php` · `GlobalStyles.php` · `GoogleMap.php` · `NFT.php` · `OpenVerse.php`
· `Facebook.php` · `Instagram.php` · `Data.php` · `Pagination.php` · `PluginInstaller.php` ·
`AssetGeneration.php`, plus admin-side `add_action` handlers.

**Hooks — ~35 public actions/filters** form the extension contract. `essential_blocks::init` is what
Pro's bootstrap gates on. The feed filters (`essential_blocks/facebook_feed/{graph_fields,posts,
query_args,raw_response}`) have silently changed payload shape before.

**Custom tables:** `{prefix}eb_form_entries`, `{prefix}eb_search_keywords`.

## Free/Pro coupling — a known latent fatal

Pro's bootstrap checks Free is **active** but never checks its **version**, while
`pro/includes/blocks.php` does:

```php
if ( class_exists( 'EssentialBlocks\\Blocks\\FacebookFeed' ) ) {
    $pro_blocks['facebook_feed'] = [ 'object' => ProFacebookFeed::get_instance() ];
}
```

`ProFacebookFeed extends FacebookFeed`. On a Free older than the version that introduced the parent,
this fatals. `ESSENTIAL_BLOCKS_REQUIRED_VERSION` is defined (6.0.4) but not enforced at this path.
**Always test old-Free + new-Pro.**

## Baseline frontend asset contract

Free must ship **exactly three** baseline assets on every page; everything else is gated:
inline `essential-blocks-global-styles` (`Core/Scripts.php:138`), `dashicons` (`:283`),
`blocks-localize` (`:572`). The repo skill `verify-baseline-assets` enforces this; regression class
is "an asset starts loading unconditionally with its toggle off".

## Editor settings UI (InspectorPanel)

Every EB block's sidebar is one `InspectorPanel` from the controls submodule (`InspectorPanel.General`,
`.Style`, `.Advanced`, `.PanelBody`). Verified against the shipped `assets/admin/controls/controls.js`
(6.4.4). **Re-verify the selectors if the controls submodule pointer changed this release.**

| Thing | Fact |
|---|---|
| Container | `.eb-panel-control` > `.eb-parent-tab-panel` |
| Tab buttons | `.eb-tab.general` "General", `.eb-tab.styles` "Style", `.eb-tab.advance` "Advanced". The class is `advance`, not `advanced`; the label is "Style", not "Styles" |
| Tab bodies | `.eb-tab-controls-general`, `.eb-tab-controls-styles`, `.eb-tab-controls-advance`. Only the active tab is in the DOM |
| Hidden tabs | `hideTabs={['styles']}` (values `'general'`, `'styles'`, `'advanced'`) removes a tab **on purpose**. Blocks that do it: Free Wrapper, Row, Column, Flex Container, Shape Divider (Style); Pro Animated Wrapper, Form Multistep Wrapper, Form reCAPTCHA, Stacked Cards (Style), Mega Menu Item (Style and Advanced). List as of the Aug 2026 mirror heads; regenerate it, never trust it (`probes.md`) |
| Advanced tab | Always renders the shared advanced controls (margin, padding, background, border ... driven by each block's `advancedControlProps`), then any block-specific `InspectorPanel.Advanced` content |
| Panels | WordPress `PanelBody` (`.components-panel__body`, `.is-opened`). **Children are not in the DOM while a panel is closed** |
| Accordion behaviour | Opening a panel writes a shared `panelName` to the `essential-blocks` store and every other panel closes. Read one panel at a time |
| Remembered tab | The last tab is kept per block in the `essential-blocks` store, so a block can reopen on Style or Advanced, not General |
| Responsive controls | Desktop stores the bare attribute; Tablet stores `TAB<attr>`, Mobile stores `MOB<attr>`. A reset control restores the schema default |

## Version-bump surface — 6 files

`essential-blocks.php` · `includes/Plugin.php` · `readme.txt` · `package.json` ·
`.config/release.jsonc` · `src/admin/dashboard/components/TabGeneral.js`

A changelog entry has been lost to a merge-conflict resolution before — verify it is present, not
just that the version bumped.

## Version chain

Free `6.1.3 → 6.2.1 → 6.3.0 → 6.4.0 → 6.4.1 → 6.4.2 → 6.4.3`
Pro `2.9.4 → 3.0.0 → 3.1.0 → 3.2.0 → 3.2.1`

## Test sites and credentials

Machine-specific. See `references/environment.md` for the capability each site must provide, how to
choose one per run, and where credentials come from. Never ask for credentials before checking every
source listed there.

## In-repo documentation (not shipped in the ZIP)

`essential-blocks/.claude/` is the authoritative spec, and it is stripped by `.distignore` — so
**testing the ZIP alone hides it**:

- `docs/architecture/` — `asset-loading.md` (the baseline-asset contract), `data-storage.md`,
  `hooks.md`, `render-pipeline.md`, `free-pro-extension-points.md`
- `docs/features/` — 39 docs across admin, ai, editor, forms, i18n, integrations, pipeline, pro,
  styling, telemetry
- `docs/glossary.md`, per-block `src/blocks/<block>/CLAUDE.md`
- `releases/release.md` — the release audit trail, including which stale claims in `release-plan.md`
  to ignore

**Repo skills worth reusing as sub-checks** (both read-only audits):
`essential-blocks/.claude/skills/verify-baseline-assets` and
`essential-blocks-pro/.claude/skills/audit-free-pro-parity` (needs free checked out as a *sibling*
of pro; verifies every documented free→pro hook is fired and listened to with matching name, arity
and priority).

## Knowledge vault — `$QA_HOME/knowledge/` (optional)

Read at Phase 0 to seed checks, if you keep such a vault (see `references/environment.md`). **Read-only** unless the user approves a write-back.

- `Bug-Patterns/` (highest reuse): `Security-Checklist.md` (10-item recurring fix taxonomy),
  `Unscoped-Admin-CSS-Leaks-Into-Host-Pages.md`, `REST-Partial-Attribute-Payload-Warnings.md`,
  `Async-Recovery-Only-Covers-New-Mounts.md`, `Timing-Sensitive-UI-Checks-Need-Wait-Before-Fail.md`
- `Blocks/` — 11 per-block histories (Advanced Navigation, Advanced Search, Advanced Video Overlay,
  Data Table, Facebook Feed, Image Gallery, Instagram Feed, One Page Navigation, Protected Content,
  Table of Contents, Woo Product Grid)
- `Releases/`, `Third-Party-Compatibility/`, `Environment-Setup/`

Treat everything found there as a **hypothesis to verify live**, never as proof.

## What does NOT exist

- **No automated tests.** `phpunit.xml.dist` exists in both repos; `tests/` does not. No Jest, no
  Playwright suite, no e2e, no `test` script.
- **No meaningful CI.** Free's `.github/workflows/test.yml` is build-only; Pro has no `.github/` at all.
- So there is no "run the project's own gates first" step. Substitute: build success, `npm run lint:js`,
  `.phpcs.xml.dist`, WP Plugin Check, and this skill's own smoke sweep. **Manual passes are the only
  safety net this product has** — which is the whole reason this skill exists.
