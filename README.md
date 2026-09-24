# wp-eb-release-regression

A Claude Code skill for full pre-release regression testing of the Essential Blocks WordPress
plugin ecosystem — Free, Pro, and the controls submodule.

`wp-eb-test` is ticket-scoped: it derives a checklist from one fix's diff. A release is a different
question. Sites that already have content authored with the *previous* version are about to be
upgraded, so the thing to answer is **"will anything a user already has stop working?"** This skill
sweeps every coverage axis whether or not the release diff touched it, tests the artifact that will
actually ship, and reasons to a ship / no-ship verdict.

## What it covers

Block and content integrity ("Attempt Block Recovery" risk), upgrade paths from the previous release,
build and shipped-ZIP identity, security with a real non-admin capability matrix, Free/Pro interplay,
asset dependency graph, block settings UI (General / Style / Advanced tabs, panels, controls) and
editor-vs-frontend parity, FSE, responsive (375 / 768 / 1440), compatibility (WooCommerce, Astra, page
builders), integrations, accessibility, performance, i18n and uninstall. The verdict is a judgment
weighed from evidence — not a blocker counter.

## Layout

```
SKILL.md                      workflow, the reportability bar, verdict rules, house rules
references/
  environment.md              machine-specific config: checkouts, test sites, credential sources
  codebase-map.md             repos, baselines, build recipe, counts, attack surface
  known-noise.md              known false positives and dev-confirmed expected behaviour
  axes.md                     the 20 coverage axes and their concrete checks
  test-types.md               completeness backstop across testing types
  engines.md                  ways to go deeper when a sweep finds nothing
  probes.md                   proven, copy-pasteable verification snippets, incl. the settings-UI census
  fixture-gotchas.md          traps that produce empty blocks and false passes
  report-template.md          report skeleton and section order
  state-template.md           the _STATE resume file for long runs
  handoff-templates.md        dev messages, future-polish list, card bodies
```

## Setup on a new machine

1. Copy this repo into `~/.claude/skills/wp-eb-release-regression/`.
2. Fill in `references/environment.md`: where your Free/Pro checkouts live, which local test sites
   you have, and where credentials come from.
3. Keep credentials out of this repo. `.gitignore` excludes `defaults.json`, `user.txt`, `.env*`,
   generated reports and screenshots — nothing secret is or should be committed.

Everything else is portable as-is.

## Requirements

- Claude Code with a browser-automation MCP tool (written against Playwright MCP).
- A local WordPress site you may freely modify, with the Free and Pro plugins installed.
- Git, Node/npm and pnpm if you build from source instead of testing installed ZIPs.
- Network access to `api.wordpress.org` and `downloads.wordpress.org` for checksums and N-1 baselines.

## Usage

```
/wp-eb-release-regression                 # everything inferred
/wp-eb-release-regression fbs-XXXXX       # optional release card
```

Optional overrides: `site_url=` `tier=p0|p0+p1|all` `skip=` `source=zip|build` `time_budget=2h`
`axes=` `publish=no`.

## Scope

Read-only against Essential Blocks source: it reports bugs and never fixes, commits or pushes to the
plugin repositories. For a single ticket or one PR, use `wp-eb-test` instead.
