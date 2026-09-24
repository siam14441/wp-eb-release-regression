# Test types — the completeness backstop

Axes organise by *where to look*; engines by *how to find*. Neither ever asks "have I done performance
testing at all?" This is the checklist that catches a whole discipline being skipped.

Fill in the disposition for **every** row before writing the verdict.
**A type you did not run is a coverage gap, not a pass.** `"not applicable, because X"` is a finished
row; silence is not.

## Functional

| Type | Method | Disposition |
|---|---|---|
| Smoke / sanity | Site loads, editor opens, blocks appear in inserter, no fatals | |
| Unit-equivalent | No suite exists — substitute isolated Node/PHP harnesses (`engines.md` §5) | |
| Integration | Free + Pro + controls together; WooCommerce, Templately, theme | |
| System / end-to-end | Author a page, save, view it, edit it again, upgrade, view again | |
| **Regression** | Axes 1, 8, 9 — existing content and untouched features still work | |
| Negative / boundary | Shape probing, empty states, zero/one/many, malformed input | |
| Data / CRUD integrity | Block attributes survive save->reload; form entries; options | |
| **Install / upgrade / uninstall** | Axis 2 and axis 20 — the paths real users take | |
| API / contract | REST routes, AJAX actions, the ~35 public hooks (axis 10) | |
| Exploratory | Unscripted use as a real author for 15 minutes; follow anything odd | |

## Non-functional

| Type | Method | Disposition |
|---|---|---|
| Performance / front-end cost | Axis 18 — asset weight, query count, cold vs warm | |
| Load / scale | Many blocks on one page; many feed instances; large queries | |
| **Security** | Axis 4, including the real non-admin capability matrix | |
| Usability | Can an author configure this without docs? Are labels honest? | |
| Accessibility | Axis 17 — keyboard, target size, contrast, screen reader | |
| Cross-browser / device | Widths at minimum; note explicitly if only one browser was used | |
| Compatibility | Axis 14 — themes, plugins, PHP and WP versions | |
| i18n / l10n / RTL | Axis 19 | |
| Reliability / recovery | Fault injection: does it recover, or stay broken until reload? | |
| Compliance / release readiness | Axis 3 — versions, changelog, checksums, packaging | |

## The thin rows — method notes

**Load and scale.** No tooling exists. Approximate: one page with every block, several feed blocks
sharing a token, a Post Grid with a large `per_page`. Watch query count and page weight, not just
whether it renders.

**Cross-browser and device.** Only Chromium via Playwright is available here. Say so. Report viewport
widths tested and state plainly that other engines were not covered rather than implying they were.

**Usability.** Configure a block the way a first-time author would, without reading source. Controls
that do nothing until an unrelated toggle is on, labels that do not match behaviour, and defaults that
produce an empty block are all findings.

**Exploratory.** Reserve real time for it, unscripted, at the end when you know the system. Several of
the strongest historical findings came from noticing something odd rather than from a checklist row.

**Reliability and recovery.** The distinctive question here: when something fails, does it recover for
blocks that are *already mounted*, or only for ones mounted afterwards? A shared-cache reset that only
helps the next mount leaves every block currently on the page broken until reload.
