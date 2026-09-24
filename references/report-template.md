<!--
RELEASE REGRESSION REPORT TEMPLATE — wp-eb-release-regression

Baseline is the existing eb-qa-reports corpus format. Anyone who has read those reports should
recognise this one immediately. The extra sections exist only because a release run produces things
a ticket report has nowhere to put. Cut any section that does not earn its place in this run.

- Filename: qa-report-release-<free>-<pro>-<YYYY-MM-DD>.md in wp-content/plugins/eb-qa-reports/
  NEVER overwrite an existing report. Collision -> append -2, -3.
- Style: caveman. Short sentences, no padding. "Block render editor. No errors. Settings persist."
  Keep file paths, SHAs, selectors and error text FULL — only prose gets trimmed.
- Status markers: emoji PLUS text, always, so the report stays greppable.
  ✅ PASS · ❌ FAIL · ⚠️ PASS (Code) · ⚠️ CONCERN · 🔍 NOT VERIFIED · 🚫 BLOCKED · ℹ️ INFO
- NO ROUND NUMBERING anywhere — title, filename, body, or agent-notes params. Grep for "round"
  before publishing. Never frame a finding as what an earlier pass missed.
- Screenshots only for visual findings: screenshots/NN-<block>-<defect>.png beside this file,
  referenced with an **Evidence:** line on the finding.
-->

# QA Report: Essential Blocks — Release Regression — Free {{x.y.z}} / Pro {{a.b.c}}

**Date:** {{YYYY-MM-DD}}
**Site:** {{url}} (WP {{ver}}, PHP {{ver}}, theme {{name}}{{, WooCommerce ver, other relevant plugins}})
**Scope:** {{free+pro / all}}
**Source:** {{installed release ZIPs / built from source}}
**Base:** Free `{{ref}}` @ {{sha}} · Pro `{{ref}}` @ {{sha}} · Controls @ {{sha}}
**Release card:** {{fbs-XXXXX or "none given"}}

## Tested

| Component | Branch / ZIP | Base | Files Changed |
|-----------|--------------|------|----------------|
| Free | `{{release-x.y.z}}` @ **{{sha}}** | `{{base}}` ({{n}} commits) | {{n}} (+{{a}} / −{{b}}) |
| Pro | `{{release-a.b.c}}` @ **{{sha}}** | `{{base}}` ({{n}} commits) | {{n}} (+{{a}} / −{{b}}) |
| Controls | {{sha or "pointer unchanged"}} | | |

Build: {{built fresh / installed ZIPs, not rebuilt / rsync-deployed}}.

## Verdict

**{{✅ PASS — ship-ready. / ⚠️ PARTIAL — {{what blocks it}}. / ❌ FAIL — {{what breaks}}.}}**

{{2–5 caveman sentences. What was tested, what holds, what does not. Name any regression explicitly
confirmed present or absent. If PARTIAL, state exactly what would move it to PASS.}}

**Coverage: {{n}} of {{m}} tests confirmed by running ({{browser/curl/DB/file/shell}}). {{n}} code-only. {{n}} deferred. {{n}} N/A.**

### Self-challenge

<!-- Brief and honest. This is the section that catches a wrong call before it ships. -->
- **Strongest case for the opposite verdict:** {{...}}
- **Most likely 24-hour breakage, and did I test that path:** {{...}}
- **Least confident finding, and would more evidence change the call:** {{...}}
- **Unswept axis I would most want covered, and whether it changes the recommendation:** {{...}}

## Release Scope Ledger

<!-- Every card in the release, and what covered it. This answers "did you retest all the things
     from this release scope?" structurally instead of from memory. -->

| Card | Title | Component | Tests | Result |
|---|---|---|---|---|
| {{fbs-XXXXX}} | {{title}} | {{Free/Pro}} | {{#3, #4, #11}} | {{✅ PASS}} |

{{"All N cards in the release scope were tested." or name exactly which were not, and why.}}

## Change Summary

<!-- One bullet per changed file/module, from the diff. Label Free/Pro/Controls.
     Scope derived from the diff, never the changelog. -->
- `{{path}}` ({{new/modified}}) → {{what it does, one line}}.

## Test Results

| # | Test | Where | How | Result |
|---|------|-------|-----|--------|
| **{{Category — e.g. Block integrity / Upgrade path / Security / Free+Pro / Stability}}** ||||
| 1 | {{what was checked → expected}} | {{Free/Pro/Both}} | {{Visual / curl / DB / File / Code / Shell}} | {{✅ PASS}} |

## Fail Detail

<!-- One entry per blocker. Each must clear the 4-part bar. "None." if nothing failed. -->

### F1 — {{title}}

- **Where:** `{{path:line}}` — {{quoted line}} {{or URL + repro steps}}
- **Why it is wrong:** {{the promise it breaks}}
- **What it costs the user:** {{who hits it, how often, what they lose, silent or visible, recoverable}}
- **The fix:** {{concrete}}
- **Regression?** {{Yes — clean on {{N-1}} / No — reproduces identically on {{N-1}}, pre-existing}}
- **Confidence:** {{live-reproduced Nx / code-only, because ...}}
- **Evidence:** `screenshots/NN-{{block}}-{{defect}}.png`

## Non-blocking findings

<!-- Real issues that do NOT gate the ship. These never drag the verdict down. -->
- **{{title}}** — {{what, where, impact}}. Not blocking because {{...}}.

## Future polish

<!-- Valid issues waived for this run, collected for the dev conversation.
     Use handoff-templates.md for the message form. -->
- **{{title}}** — {{issue}} / {{what could be done}}.

## Explicitly out of scope this run

<!-- The skip ledger, echoed. Never silently dropped, never re-raised. -->
- {{item}} — {{who declared it out of scope, and when}}.

## Axis Coverage Ledger

<!-- Axes 11 (FSE), 13 (Responsive) and 14 (Compatibility) must show `swept`. Never `deferred`, never "not touched by diff", never "environment".
     Axis 9 (Block settings UI + editor/frontend parity) is P0: show the depth reached, e.g. "structure 94/94, deep on 6 blocks + fixture set". -->

| Axis | Component | Disposition | Evidence / reason |
|---|---|---|---|
| {{block-integrity}} | {{Free}} | {{swept}} | {{69/69 valid; 0 save.js changed}} |
| {{settings-ui-parity}} | {{Free}} | {{swept}} | {{n/n inspectors match expected tabs; 0 controls removed vs N-1; deep on n blocks}} |
| {{i18n}} | {{Free}} | {{deferred — environment}} | {{WPML not on this site; the WPML site in `references/environment.md` could cover it}} |

{{Tier note if time-boxed: "P2 axes were not swept — run was time-boxed to Nh."}}

## User Check

| Perspective | Verdict | Note |
|-------------|---------|------|
| Content creator | {{✅ PASS}} | {{one line}} |
| Returning user (existing content) | {{✅ PASS}} | {{one line}} |
| Visitor | {{✅ PASS}} | {{one line}} |
| Mobile | {{✅ PASS}} | {{one line}} |
| Accessibility | {{✅ PASS}} | {{one line}} |
| Anonymous attacker | {{✅ PASS}} | {{one line}} |
| Store owner | {{✅ PASS}} | {{one line}} |

## Concerns

<!-- Non-blocking issues outside direct scope. Always include a Security line. -->
- **{{LOW/MED/HIGH}} — {{title}}:** {{what, repro, suggested fix, blocking or not}}.
- **Security: {{✅ PASS / ⚠️ CONCERN}}.** {{capability, nonce, escaping, prepared-query notes}}.

## Test Artifacts

Created and removed by this run:
- {{page/user/option, ID and URL}} — {{deleted ✅ / left in place, why}}

State restored / verified unchanged: {{plugin activation state, settings, tokens, theme, debug.log line count}}

## Next Steps

{{Ship it. / Fix F1 before ship. / Blocked on X.}} {{Non-blocking follow-ups, if any.}}
