# Environment — the machine-specific configuration

Everything that differs from one QA machine to the next lives in this file and nowhere else:
checkout locations, test sites, and where credentials come from. Fill it in once per machine.
The rest of the skill refers to these names (`$FREE`, `$PRO`, `$CONTROLS`, `$QA_HOME`) instead of paths.

**Never write a password, token, application password or license key into this file or anywhere in
this repository.** Point at where the secret lives; do not paste the secret.

## Checkouts

Keep build checkouts **outside** `wp-content/plugins/` — a manual "Add Plugin" upload over a folder
containing `node_modules` can exhaust PHP memory mid-delete and destroy the folder.

| Variable | Meaning | Example |
|---|---|---|
| `$FREE` | Free build checkout, folder named exactly `essential-blocks` | `~/eb-release-build/essential-blocks` |
| `$PRO` | Pro build checkout, folder named exactly `essential-blocks-pro` | `~/eb-release-build/essential-blocks-pro` |
| `$CONTROLS` | Controls submodule inside `$FREE` | `$FREE/src/controls` |
| `$QA_HOME` | Workspace holding read-only mirrors, the knowledge vault and reports | `~/EssentialBlocks-QA` |
| Read-only mirrors | Reference clones used for `log` / `diff` / `show` only | `$QA_HOME/repos/{essential-blocks,essential-blocks-pro}` |
| Knowledge vault | Optional. Distilled QA notes used to seed checks at Phase 0 | `$QA_HOME/knowledge/` |

## Credentials

Look in this order, and **never ask before checking every source**:

1. A credentials file inside the target site, e.g. `wp-content/plugins/user.txt`.
2. A per-machine config file, e.g. `$QA_HOME/defaults.json`.
3. Ask the user.

Re-check per site. A credentials file that exists on one site says nothing about another.

Shape of the per-machine config file (values are placeholders — keep the real file out of version control):

```json
{"site_url":"http://<your-site>.local/","wp_user":"<qa-user>","wp_pass":"<from your password manager>",
 "scope":"all","build":"auto","test_areas":["editor","fse","frontend"],"analysis":"deep",
 "check_dependents":true,"screenshots":"no"}
```

Prefer a dedicated QA administrator on a throwaway local site. Do not reuse a real account or a
password used anywhere else.

## Test sites

One site per run. Pick the site whose environment covers the most axes for the release. Record what
each of yours provides — the skill uses these capabilities to decide which axes can run and which are
recorded `deferred — environment`.

| Capability | Why the skill wants it | Your site |
|---|---|---|
| Instrumented default (Query Monitor, an object cache, a plugin-rollback tool, a forms plugin) | Makes the upgrade-path axis cheap and gives query/perf evidence | `<site>` |
| WPML + String Translation | Only way to cover the i18n axis | `<site>` |
| Site already on N-1 Free/Pro | Upgrade path by in-place update; also a WP-version axis | `<site>` |
| Older PHP (for example 8.0) | PHP-version axis; two-versions-back upgrades | `<site>` |
| Elementor + Essential Addons | Page-builder coexistence | `<site>` |
| Older Free/Pro (N-2, N-3) | Upgrade-source coverage | `<site>` |
| Block theme (Twenty Twenty-Five), no WooCommerce / WPML | FSE, responsive, upgrade path by file swap | `<site>` |

Note per site: installed Free / Pro versions, active theme, whether `wp-cli` is available, and any
pre-existing QA content or users that a run must not delete.

WooCommerce + Astra are installed per run from wp.org when a site lacks them, then removed again
(`references/axes.md` 14).

Axes the chosen site cannot cover are recorded `deferred — environment`, naming the site that could.
Never claim them as passed, never silently drop them.

## Report location

Reports go in `wp-content/plugins/eb-qa-reports/` on the test site, **never** inside `essential-blocks/`
(a WordPress auto-update wipes anything stored there). If you keep a separate archive of reports,
copy each finished report there as well.
