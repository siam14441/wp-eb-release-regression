<!--
RESUME STATE FILE — wp-eb-release-regression

Write this at Phase 0, BEFORE any testing, and keep it current as you go. A release sweep is long
enough that a context reset mid-run is likely; this file is what makes that survivable.

Path: wp-content/plugins/eb-qa-reports/_STATE-release-<free>-<pro>.md
If a session resets, read this first.
-->

# QA Session State — Release Free {{x.y.z}} / Pro {{a.b.c}}

**Started:** {{YYYY-MM-DD HH:MM}} · **Release card:** {{fbs-XXXXX or none}}

## Environment

| Item | Value |
|---|---|
| Site | {{url}} (admin `{{user}}`; credentials from `{{source}}` -- never write the password here) |
| WP / PHP / theme | {{...}} |
| Companion plugins | {{WooCommerce x, Templately y, WPML z, ...}} |
| Free installed | {{version}} — {{ZIP / built from `<branch>` @ `<sha>`}} |
| Pro installed | {{version}} — {{ZIP / built from `<branch>` @ `<sha>`}} |
| Controls | {{sha}} — {{pointer changed / unchanged}} |
| Build dir | {{path, or "not built this run"}} |

## Baselines

Free `{{ref}}` @ {{sha}} ({{date}}) · Pro `{{ref}}` @ {{sha}} · Controls `{{ref}}` @ {{sha}}
Release descends from: {{staging / master / other}} — verified {{how}}.

## Scope — {{n}} cards in this release

| Card | Title | Component | Status |
|---|---|---|---|
| {{fbs-XXXXX}} | {{title}} | {{Free/Pro}} | {{pending / tested / blocked}} |

## Explicitly out of scope (skip ledger)

- {{item}} — declared by {{who}}, {{when}}. Do not test, do not report, do not count.

## Changed files

**Free ({{n}}):** {{list}}
**Pro ({{n}}):** {{list}}
**Controls ({{n}}):** {{list}}

## Axis progress

- [ ] 1 Block/content integrity
- [ ] 2 Upgrade path & version skew
- [ ] 3 Build/artifact identity
- [ ] 4 Security + capability matrix
- [ ] 5 Free/Pro interplay
- [ ] 6 Asset dependency graph
- [ ] 7 Stability
- [ ] 8 Per-card functional
- [ ] 9 Block settings UI + editor/frontend parity  (P0 -- every release; record depth: structure n/94, deep on n blocks)
- [ ] 10 Hook contract
- [ ] 11 FSE  (ALWAYS RUN -- never defer)
- [ ] 12 Content-import path
- [ ] 13 Responsive  (ALWAYS RUN -- never defer)
- [ ] 14 Compatibility  (ALWAYS RUN -- install WooCommerce + Astra if absent, never defer)
- [ ] 15 Dynamic/data blocks
- [ ] 16 Integrations
- [ ] 17 Accessibility & UI/UX
- [ ] 18 Performance
- [ ] 19 i18n / WPML / RTL
- [ ] 20 Lifecycle & uninstall
- [ ] Cleanup & state restoration

## Findings so far

<!-- Code-level suspicions awaiting live confirmation, and confirmed findings. Keep both. -->
1. {{suspicion}} — {{status: unconfirmed / confirmed / dismissed, why}}

## Fixtures and users created (must be cleaned up)

| Thing | ID / name | URL | Removed? |
|---|---|---|---|
| {{page}} | {{id}} | {{url}} | {{no}} |
| {{user}} | {{qa_subscriber}} | — | {{no}} |

## Site state I changed (must be restored)

- {{setting/plugin/theme}} — was {{value}}, now {{value}}, restore by {{how}}.

## Gotchas hit this run (reuse these)

- {{thing that cost time}}

---

**SESSION COMPLETE.** Verdict: {{...}} · Report: {{filename}}
