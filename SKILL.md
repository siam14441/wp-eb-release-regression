---
name: wp-eb-release-regression
description: >
  Full pre-release regression testing for the Essential Blocks WordPress plugin ecosystem
  (free, pro, and the controls submodule). Sweeps every coverage axis -- block/content integrity
  and "Attempt Block Recovery" risk, upgrade paths from the previous release, build and shipped-ZIP
  identity, security with a real non-admin capability matrix, free/pro interplay, asset dependency
  graph, FSE, responsive, compatibility, integrations, accessibility, performance, i18n and
  uninstall -- then reasons to a ship / no-ship verdict and writes a release report.
  Use this whenever the target is a RELEASE rather than a single ticket: "release regression test",
  "full regression before release", "deep test the release zips", "is the new release ship-ready",
  "test 6.5.0 before we ship", "final check on the zips going to the live store", or a release card
  (an fbs-XXXXX whose scope lists several tickets). Trigger even when the words "regression" or
  "QA" are absent and the user just says "we're releasing today, check everything" or "make sure no
  existing user content breaks". For a SINGLE ticket or one PR, use wp-eb-test instead.
---

# Essential Blocks Release Regression

You are the last gate before Essential Blocks ships to real sites with real content.

| Component | Path | Notes |
|-----------|------|-------|
| **Free** | `essential-blocks/` | 69 blocks register; `includes/blocks.php` lists 79 entries |
| **Controls** | `essential-blocks/src/controls/` | Git submodule; compiled by free's webpack |
| **Pro** | `essential-blocks-pro/` | 25 blocks register, under the `essential-blocks/` namespace |

**Read `references/environment.md` and `references/codebase-map.md` first, every run.** Do not reconstruct repo paths, baselines,
block counts or the build recipe from memory -- they have each been wrong before.

## Why this skill exists, and how it differs from `wp-eb-test`

`wp-eb-test` is ticket-scoped: it derives its checklist from the diff of one fix and spot-checks the
blast radius around it. That is correct for a ticket and insufficient for a release.

A release ships to sites that already have content authored with the *previous* version. So the
question is not "does this fix work" but "**will anything a user already has stop working**." That
reframing drives three differences:

1. **Axes are mandatory, not diff-gated.** Every applicable axis in `references/axes.md` is swept
   even when this release's diff never touched it. A clean diff is not evidence of a safe release.
2. **Sweep one axis across all components**, never component-by-component. The block-recovery
   question is a single mental model applied to 94 blocks; holding it across the whole tree is what
   surfaces instances. Re-contexting per block reliably misses them.
3. **The shipped artifact is the subject**, not the branch. Test what will actually be installed.

## Trigger

```
/wp-eb-release-regression                 # everything inferred
/wp-eb-release-regression fbs-XXXXX       # optional release card
```

Nothing else needs typing, ever. **Infer, never ask:**

| Thing | Where it comes from |
|---|---|
| Credentials | The sources listed in `references/environment.md`, in order. Never ask before checking all of them. |
| Site | Per-machine config -> installed versions -> ask only if genuinely ambiguous |
| Versions | Plugin header + `readme.txt` stable tag of what is installed |
| Baseline | Previous release tag, resolved per-repo (see codebase-map) |
| Scope / axes / depth | `all` / all axes / deep |
| Screenshots | Findings only |
| Report location | `wp-content/plugins/eb-qa-reports/` |

Optional overrides -- accepted, never required, never prompted for:
`site_url=` `tier=p0|p0+p1|all` `skip=` `source=zip|build` `time_budget=2h` `axes=` `publish=no`

If `time_budget` is set, drop tiers from the bottom and **say so in the report** rather than
quietly running less. **Never drop FSE (axis 11), Responsive (axis 13) or Compatibility (axis 14)** -- they are
exempt from tiering and run every time (standing rule).

## The bar

A finding is reportable only when you can fill in all four:

1. **Where** -- `path/File.php:123` with the line quoted, or a URL plus exact reproduction steps.
2. **Why it is wrong** -- the promise it breaks, not "this looks odd".
3. **What it costs the user** -- who hits it, how often, what they lose, whether it is silent.
4. **The fix** -- concrete enough that a developer can act without re-investigating.

Cannot fill all four? You have a **lead**, not a finding. Chase it or log it as a lead. Never pad
a report with leads dressed as findings.

Before filing anything, check `references/known-noise.md`. Several convincing-looking defects in
this codebase are expected behaviour, standing build gaps, or harness artifacts -- each has already
cost a wasted investigation or a retracted finding.

## Workflow

**0. Ground and preflight.**
Read `references/environment.md`, `references/codebase-map.md` and `references/known-noise.md`. If a QA knowledge vault is configured (`references/environment.md`), seed per-block and recurring-defect
checks from its `Bug-Patterns/` and `Blocks/` notes (read-only; treat what you find as a hypothesis to
verify live, not as proof). Resolve the release card into its scope
cards if one was given. Detect site and credentials. Resolve the diff baseline **per repo**. Record
anything the user declared out of scope into the skip ledger. Write `_STATE` immediately
(`references/state-template.md`) and keep it current -- a long run must survive a context reset.

**1. Artifact acquisition and identity.**
Either verify the ZIPs already installed, or build from source (`references/codebase-map.md` has the
exact recipe and its traps). Then prove the artifact is what you think it is: wp.org checksum manifest
where published, file-set comparison, version bumped across all six files, changelog entry present,
no `node_modules` shipped, and -- critically -- **that the ZIP is not an older release wearing a new
label**. That has happened.

**2. Scope derivation.**
Derive scope from the **actual code diff, never the changelog**. Both 6.4.3 changelog lines read only
"Improved: Security enhancements" while nine files changed. Map every changed file to a testable claim,
and every release-scope card to the tests that will cover it. This mapping is what later answers
"did you retest everything in the release scope?"

**3. Upgrade path.**
The axis prior release sweeps never actually ran. On the one site: install N-1, author real content
with it, then upgrade in place and verify nothing broke -- block validity, settings and options,
license state, custom tables. Then the version-skew cases, including old-Free + new-Pro, which is a
documented latent fatal. Clean install is the control, not the test.

**4. Parallel axis sweeps.**
Dispatch code-analyzable axes to subagents (security static analysis, build identity, registry
diffing, hook contract, i18n, performance statics, spec-vs-code). Each returns findings that already
clear the bar. **You keep the browser** -- subagents never drive it, and there is only ever one.

**5. Live verification.**
Fixture pages, block harnesses, editor -> save -> reload -> frontend, FSE, responsive widths, the
capability matrix as each created non-admin user, compatibility toggles. **FSE (Site Editor, patterns,
blocks in a template and a template part), responsive (375 / 768 / 1440, with screenshots) and WooCommerce + Astra
compatibility (installed from wp.org if the site lacks them, then removed) are part of every run** -- `references/axes.md` 11, 13 and 14 have the procedure. `references/probes.md` has
the proven snippets; `references/fixture-gotchas.md` has the traps that silently produce empty blocks
and false passes. Prefer programmatic verification (`browser_evaluate`) over clicking -- synthetic
clicks no-op on Gutenberg controls and Interactivity API elements.

**6. Consolidate and report.**
Reason to a verdict (below). Write the report per `references/report-template.md` into
`eb-qa-reports/`, with screenshots for visual findings beside it. Then stop and ask before publishing
to agent-notes or writing anything to FluentBoards.

**7. Cleanup.**
Delete the fixtures, users, mu-plugin probes and options **you** created. Restore settings you
changed. List what was removed and what was verified unchanged. Never delete anything you did not
create.

## Judging release readiness -- your call, not a rule engine

Deterministic checks establish **facts**. They never by themselves produce the verdict. Objective
gates stay mechanical because they have exact answers: checksums match, 94/94 blocks re-parse, file
sets identical, zero new fatals in `debug.log`, every registered handle resolves, version bumped in
all six files. Those are evidence rows, not a score.

**The verdict is your judgment, weighing that evidence.** There is no blocker counter and no formula.

Per finding, weigh:

1. **Regression or pre-existing?** Reproduce against N-1 before calling anything a regression. If it
   behaves identically on the previous release, it is pre-existing -- report it, but it does not gate
   *this* ship. Check `known-noise.md` and dev-confirmed-expected behaviour first.
2. **User impact.** Who hits it (anonymous visitor / author / admin), how likely on a default
   configuration, what they lose, whether it fails silently, whether they recover without support.
   Silent and unrecoverable is far worse than loud and fixable.
3. **Severity as calibration, not arithmetic.** Roughly: silent data loss > security exposure to
   lower-privileged actors > fatal or white screen > primary flow broken > degraded flow > polish.
   Use it to sanity-check an instinct, never to compute an answer.
4. **Confidence.** Live-reproduced beats code-only. Reproduced twice beats once. Anything with an
   innocent explanation you have not excluded is a lead.
5. **Coverage.** Which axes went unswept, and could that gap plausibly hide something worse than what
   you found. A clean run with a large hole is not a clean run.

**Self-challenge before writing the verdict.** Answer these briefly, in the report:

- What is the strongest case for the opposite verdict?
- If this ships and something breaks within 24 hours, what is it most likely to be -- and did you
  actually test that path, or assume it?
- Which finding are you least confident about, and would more evidence change the call?
- Which unswept axis would you most want covered, and does its absence change the recommendation?

If the self-challenge changes the call, change the call.

**The three verdicts:**

- `✅ PASS` -- ship. No unwaived finding you believe will hurt a real user on upgrade, and coverage
  good enough to say that honestly.
- `⚠️ PARTIAL` -- ship-ready with named caveats, or blockers explicitly waived, or a P0 axis unswept (FSE, Responsive and Compatibility count as P0: leaving any one unswept caps the verdict at PARTIAL).
  State exactly what would move it to PASS.
- `❌ FAIL` -- at least one finding you judge will hurt real users.

**Standing calibration -- these override your instinct where they conflict:**

- **Non-blockers never drag the verdict down.** Polish, target sizes, minor layout, perf and unreached
  edge cases go under "Non-blocking findings" and "Future polish". They do not make a report fail.
- **Authoring-time prevention counts as a fix.** If the editor now clearly warns against a bad
  configuration, that is a PASS even when runtime behaviour for content that already has that
  configuration is unchanged. Footnote the technical detail; do not let it pull the verdict down.
- **A control removed as a deliberate scope decision is INFO**, not a regression.
- **Code-verified-only is never equal to a live PASS.** Label it `⚠️ PASS (Code)` and say why it
  could not be run.

## Deliverables

| Output | What it is |
|---|---|
| Release report | `eb-qa-reports/qa-report-release-<free>-<pro>-<YYYY-MM-DD>.md` |
| Screenshots | `screenshots/NN-<block>-<defect>.png` beside the report, one per visual finding |
| `_STATE` file | Live resume file; updated as you go, not at the end |
| Dev handoff | Per blocker, in "The issue: / What could be done:" form |
| Future polish | Valid issues waived for this run, collected for the dev conversation |
| Coverage ledgers | Release-scope card ledger + axis x component ledger with dispositions |

## Anti-patterns -- each of these has already cost a real run

- Testing the branch instead of the **merge result** or the **actually-shipped ZIP**.
- Deriving scope from the changelog rather than the diff.
- Testing security only in the negative direction. Always prove the legitimate flow still works.
- Filing harness noise as a defect. Rapid `insertBlock()` produces benign validation warnings and
  Fancy-Chart `NaN` errors -- verify against raw DB content via REST `context=edit` first.
- Declaring a timing-sensitive UI failure on the first immediate check. The Interactivity API needs
  about a second to hydrate; a hamburger FAIL had to be publicly retracted over exactly this.
- Skipping Pro, or folding Pro into the free sweep instead of testing it on its own terms. This is
  the single most frequent user correction.
- Silently dropping a waived item instead of recording it in the skip ledger and echoing it in the
  report.
- Reusing a "backup" that is actually a hybrid build as an A/B baseline.
- Reporting a missing asset from a locally-built tree as a release defect. Free's dependency install
  is a workaround, so the tree is not the intended one -- such findings are inconclusive.
- Letting a report grow a "round 3" framing. Never number QA passes anywhere the dev can see.
- Judging responsiveness from the overflow metric alone. It cannot see content that an ancestor clips
  (Countdown labels cut off at 375px passed it). Read screenshots.
- Building FSE fixtures by REST, or inserting blocks programmatically and skipping the block's own starter
  step (Post Grid "Start Blank", Advanced Image source), then filing the empty result as a defect. It was
  a fixture artifact both times.
- Calling an empty asset list or a zero count a pass. `curl` without `-L` on `?page_id=N` fetches an empty
  301; print byte counts so a blank fetch cannot hide.
- Recording compatibility as "deferred -- environment" because the chosen site has no WooCommerce or Astra. Install
  them from wp.org, test, and restore the site (axes.md 14).
- Using WooCommerce's `/shop/` page as a reference while its Coming Soon mode is on. It shows a placeholder to
  anonymous visitors; compare against WooCommerce's own `[products]` shortcode instead.
- Setting an attribute programmatically on a block whose UI control is a `<select>` or toggle, then reporting the
  empty result. Drive the real control (value + `change` event), as a user would.

## House rules

- **Read-only against source.** Never `add`, `commit`, `push`, `merge`, `rebase`, `reset`, `stash`
  or modify plugin source. Found a bug? Report it, never fix it.
- The read-only mirror checkouts (`$QA_HOME/repos/`) are read-only **including `git fetch` and `git pull`**.
- **Never modify `wp-eb-test`, `wp-eb-reproduce`, or any other existing skill.** Borrow conventions
  read-only.
- Reports go in `wp-content/plugins/eb-qa-reports/`, **never** inside `essential-blocks/` -- a WP
  auto-update wiped reports stored there and one was permanently lost.
- Never leave a `node_modules/` inside a plugin folder WordPress may install over; the recursive
  delete exhausts PHP memory and destroys the folder.
- **New report file per run.** Never overwrite an existing report.
- **No round numbering** in the title, filename, body, or agent-notes params. Grep the draft for
  `round` before publishing.
- Ask before: any FluentBoards write, `update_note` on an existing note, and any write-back to the
  knowledge vault. Show a diff, then ask. Approval for one write is never approval for the next.
- Ask before activating/deactivating plugins or switching themes on a site you did not set up.

## Reference files

| File | Read it when |
|---|---|
| `references/environment.md` | **First, every run.** Machine-specific paths, test sites, credential sources |
| `references/codebase-map.md` | **First, every run.** Repos, baselines, build recipe, counts, surfaces |
| `references/known-noise.md` | **Before filing any finding.** Known false positives |
| `references/axes.md` | Phase 0 and throughout -- the 20 axes and their concrete checks |
| `references/test-types.md` | Before the verdict -- completeness backstop across testing types |
| `references/engines.md` | When a sweep is not finding anything, or to go deeper than surface checks |
| `references/probes.md` | Phase 5 -- proven, copy-pasteable verification snippets, incl. Site Editor, patterns, template authoring and the responsive sweep |
| `references/fixture-gotchas.md` | Any time you build a fixture, before trusting an empty result |
| `references/report-template.md` | Phase 6 -- report skeleton and section order |
| `references/state-template.md` | Phase 0 -- the `_STATE` resume file |
| `references/handoff-templates.md` | Phase 6 -- dev messages, future-polish list, card bodies |
