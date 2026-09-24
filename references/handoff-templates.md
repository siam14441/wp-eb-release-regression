# Handoff templates

The report is for the record. These are for the conversation that follows it.

## Dev message — one per blocker

The user asks for this form specifically. Plain text, pasteable into chat, no markdown tables.

```
{{Block or area}} — {{one-line symptom}}

The issue:
{{What goes wrong, in plain English, from the user's point of view first. Then where it lives:
path/File.php:123. Then what triggers it. Keep it short enough to read on a phone.}}

What could be done:
{{The concrete fix. If there is more than one option, give the one you would pick and say why in
half a sentence. Note anything that must not break in the process.}}
```

**Plain English matters here.** The user has asked more than once what a finding means "in plain
English" — if the message only makes sense to someone holding the source open, rewrite it.

## Future polish list

Issues that are real but were waived for this release. The user collects these every release to
raise with the dev separately.

```markdown
## Future polish — {{release}}

Valid issues, not blocking this release. Raised for a future cycle.

1. **{{Title}}** ({{block/area}})
   - What: {{symptom}}
   - Where: `{{path:line}}`
   - Why it was not blocking: {{reason — e.g. affects an edge configuration, cosmetic, pre-existing}}
   - Suggested: {{fix}}
```

Rules: only items that would have been findings on their own merits. Never pad the list to look
thorough, and never move a genuine blocker here to make a verdict look better.

## FluentBoards card body

**Ask before creating or commenting on any card.** Approval for one write is not approval for the
next.

```markdown
**Environment:** Free {{ver}} / Pro {{ver}} · WP {{ver}} · PHP {{ver}} · {{theme}} · {{site}}

**Steps to reproduce**
1. {{...}}

**Expected:** {{...}}
**Actual:** {{...}}

**Root cause:** `{{path:line}}` — {{mechanism}}
**Suggested fix:** {{...}}
**Severity:** {{blocker / high / medium / low}} — {{who is affected and how}}
**Regression:** {{introduced in x.y.z / pre-existing since ...}}
```

## Publishing to agent-notes

Ask first. Then:

- Use `<details>` dropdowns for the long tables (Test Results, Axis Coverage Ledger) so the note stays
  readable — the user has asked for this explicitly: "make sure to use dropdown where appropriate so
  that the report is not that crazy large".
- **Grep the draft for `round` before publishing** — no pass numbering in the body, the title, or the
  `filename` parameter. "round-trip" is fine; "round 3" is not.
- Keep severity, repro steps, root cause and suggested fixes in full. This is about framing, never
  about softening substance.
- To update an existing note: show the diff, ask, then write. Never overwrite on an implied yes.

## Summary comment for a release card

```
Release regression for Free {{ver}} / Pro {{ver}}: {{✅ PASS — ship-ready / ⚠️ PARTIAL / ❌ FAIL}}

{{One or two sentences: coverage and the headline result.}}

{{n}} blockers{{, listed below}} · {{n}} non-blocking · {{n}} for future polish
Full report: {{url or path}}
```
