# Coverage axes

**Sweep one axis across every component, not one component across every axis.** That ordering is the
point: "can this break existing saved content?" is a single mental model applied to 94 blocks, and
holding it across the whole tree is what surfaces instances. Component-by-component testing re-contexts
constantly and reliably misses them.

Axes are **mandatory and not diff-gated**. A release clears every applicable axis even when this
release's diff never touched it. A clean diff is not evidence of a safe release.

Tiers exist for time-boxed runs (`time_budget=2h`). Drop from the bottom and **say so in the report**.
A dropped tier is a coverage gap, never a pass.

**Three axes are exempt from tiering: 11 (FSE / Site Editor), 13 (Responsive) and 14 (Compatibility: WooCommerce + Astra).** They run in EVERY
release run and are never dropped, deferred or marked "not touched by the diff", whatever `time_budget`
says. Standing instruction from the user (2026-09-20). If the chosen site lacks WooCommerce or Astra, INSTALL them
(procedure in axis 14) -- "deferred -- environment" is not an accepted disposition for these three. They are
documented in the P0 section below.

Record every axis x component cell in the ledger as **swept / deferred / not-applicable**, with a
reason. `"not applicable, because X"` is a finished row. Silence is not.

---

## P0 — a release cannot ship without these

### 1. Block and content integrity
The single most important axis. Existing sites already have content authored with N-1; the question is
whether any of it breaks.

- Diff `save.js` across every block between the two release tags. **Any** change is a recovery risk.
- Diff `deprecated.js` across all 51. A removed or altered deprecation orphans old markup.
- Registry drift: count `essential-blocks/*` references in the built editor bundle both sides; diff
  `includes/blocks.php` and pro's registry; diff the pro block directory set. Nothing may disappear.
- Serialize -> re-parse every registered block; assert `isValid` and that nothing throws.
- A real editor save -> reload -> re-validate fixture with every block on one page.
- Re-validate content that already exists on the site, and all authored patterns (procedure: axis 11 and
  `probes.md` -- read them from the editor settings, not REST).
- Frontend: zero `unexpected or invalid content` recovery markers.
- Any block whose `save()` calls `applyFilters` is a Pro-off validation risk — enumerate them.

### 2. Upgrade path and version skew
Never actually run before this skill; prior sweeps proved recovery risk only structurally.

- Install N-1, author real content with it, upgrade in place, verify: block validity, settings and
  options, license state, custom tables, generated CSS.
- Repeat from N-2 where a site exists at that version.
- **Old-Free + new-Pro** — a documented latent fatal (see codebase-map). Also new-Free + old-Pro.
- Deactivate -> reactivate after upgrade. Then a clean install as the *control*, not the test.

### 3. Build and shipped-artifact identity
Prove the thing being shipped is the thing that was tested.

- wp.org checksum manifest where published: `downloads.wordpress.org/plugin-checksums/essential-blocks/<ver>.json`,
  md5 every installed file. Strongest single check available.
- File-set comparison between the tested build and the final ZIP (count both sides, `diff -rq`).
- Version bumped in all 6 files; changelog entry actually present.
- No `node_modules`, no stray dev artifacts, no dead block directories shipping a `block.json`.
- **The ZIP is not an older release wearing a new label** — compare `readme.txt` stable tags and the
  presence of whole block directories. This has happened.

### 4. Security, including a real capability matrix
The most frequent source of ship blockers.

- **Create real non-admin users** (subscriber, author, editor, shop_manager) and re-run every REST and
  AJAX probe as each, plus anonymous. Every prior run left this code-verified only.
- Every route in codebase-map: auth, capability, nonce, sanitization, escaping, prepared queries.
- Attacker-controlled attribute payloads reaching view templates.
- Editor-only AJAX actions must not have `nopriv` aliases.
- Enumeration and rate limiting, including spoofed `X-Forwarded-For`.
- Fails-closed on unknown input; cache keys validated and bounded.
- **Both directions, always** — malicious input rejected AND the legitimate flow still working.

### 5. Free/Pro interplay
The most frequent user correction: "didnt you test the pro version?"

- Free alone. Free + Pro. Pro deactivated *after* authoring Pro content (block validation).
- Pro blocks register and appear in the inserter; Block Manager entries correct.
- Licensing/activation surface unchanged unless the release touched it.
- Pro tested on its own terms, not merely swept up in the free pass.

### 6. Asset dependency graph
A whole feature once died silently here with **zero console errors** across five builds.

- Every registered script/style handle resolves — no 404s.
- Every emitted chunk actually executes; a `splitChunks` vendor bundle not declared as a dependency
  leaves a deferred module body that never runs.
- The baseline frontend asset contract holds: exactly three, everything else gated.
- Per-`$pagenow` script scoping is correct.

### 7. Stability
- `wp-content/debug.log` line count before and after; zero new fatals, warnings or deprecations.
  Where `WP_DEBUG` is off, read Local's `logs/php/error.log` and the nginx error log instead.
- Zero new console errors on editor, FSE and frontend.
- No failed network requests on a normal page load.

### 11. FSE / Site Editor -- ALWAYS RUN, never deferred
Standing instruction. Not diff-gated. Proven procedure (snippets in `probes.md`):

- **Site Editor loads** (`site-editor.php`) with EB active: all 94 blocks registered (69 + 25 Pro),
  0 console errors.
- **Validate every shipped block pattern** from the *editor settings*
  (`getSettings().__experimentalBlockPatterns`, names starting `essential-blocks/`). Patterns register on
  `admin_init`, so the REST `block-patterns` list shows none -- an empty REST result is not "no patterns".
  79 at 6.4.4 / 3.2.2. Expect the count not to drop; assert 0 invalid and 0 `core/missing`.
- **Blocks in a template and in a template part.** Create a custom template + template part + a page that
  uses the template, then author the EB blocks **through the Site Editor and its entity save**
  (`saveEditedEntityRecord`). Never author them by REST: REST-created content skips editor-side hooks and
  proves nothing. Put a distinctive attribute on each (Wrapper background `rgb(255,0,0)` in the part,
  `rgb(0,0,255)` in the template).
- **Finish each block's own starter step**, or it stays empty by design: Post Grid -> "Start Blank"
  (sets `showBlockContent` and `queryData`); Advanced Image -> pick a source. See `fixture-gotchas.md`.
- **Frontend** (`curl -sL`, then the browser): page body + template blocks + part blocks render; 0
  recovery markers; per-template CSS is generated on first visit at
  `uploads/eb-style/full-site-editor/full-site-editor-{id}.min.css` (template and each part get their
  own file); **computed** colours match what was authored, at 375, 768 and 1440.
- Not yet assessed by any run: global-styles conflicts, and whether heavy editor-only bundles stay out of
  FSE (resource `transferSize` is 0 for cached files -- use the network log or file sizes).
- Cleanup: delete the page, template and part (`DELETE ...?force=true`), then the generated CSS and the
  `full-site-editor/` folder you caused to exist.

### 13. Responsive -- ALWAYS RUN, never deferred
Standing instruction. Not diff-gated. Widths **375, 768 and 1440**. `clientWidth` comes out ~15px
smaller with a scrollbar (360 / 753 / 1425): record the real value.

- **Fixture:** one page with nested Wrapper -> Row -> 3 columns (the historical failure shape), Post
  Grid, Advanced Navigation, Accordion, Tabs, Feature List, Info Box, Testimonial, Team Member, CTA,
  Counter, Progress Bar, Countdown, Pricing Table, buttons, plus any block this release touched.
  Leave Post Carousel out unless it is the subject: its slick track overflows on bulk pages (pre-existing).
- **Metrics per width:** document overflow *with offender attribution* (ignore elements an
  `overflow:hidden` ancestor clips); whether columns stack; touch targets under 24x24 grouped by block
  (skip in-sentence inline links -- WCAG 2.5.8 exemption); smallest text size; hamburger visibility and
  size; console errors.
- **Metrics cannot see clipped content.** A Countdown with "Days" and "Seconds" cut off passes the
  overflow check. Always look at screenshots: viewport-sized crops of the tricky blocks at 375. A
  full-page capture of a long page is downscaled to unreadable.
- **Attribute every defect against N-1 before classifying it.** `git diff --stat <prev-tag>
  origin/<release> -- src/blocks/<block>` (0 = unchanged) and `cmp` the shipped `assets/blocks/<block>/*`
  between the wp.org N-1 and the target. Identical = pre-existing: report it, do not gate on it.
- Run the FSE template page from axis 11 at all three widths too.
- Hamburger: poll-retry a native click on `.wp-block-navigation__responsive-container-open` inside
  `.eb-advanced-navigation-wrapper`; the open state is `.is-menu-open` on
  `.wp-block-navigation__responsive-container`. (`.eb-mega-menu-hamburger-open` is not the toggle.)

### 14. Compatibility: WooCommerce + Astra -- ALWAYS RUN, never deferred
Standing instruction (2026-09-20). Not diff-gated, and not skippable because the site lacks them: install them.
Proven procedure (snippets in `probes.md`):

- **Record the baseline first** -- pages, posts, users, media, custom templates, plugins, themes, uploads
  files, `debug.log` line count -- so cleanup can be proven.
- **Install from wp.org by file copy.** WooCommerce latest stable, verified against
  `plugin-checksums/woocommerce/<ver>.json`; Astra via `api.wordpress.org/themes/info/1.1/` (use `curl -g`,
  the URL has brackets). Activate WooCommerce from the Plugins screen (that is what builds its tables).
  Activate Astra by navigating to its `themes.php?action=activate&...` link (the button is hidden until hover).
  Templately is normally already active on the site and stays active.
- **Seed a real store:** 6 published simple products, plus a draft, a private and a catalog-**hidden** one, each
  carrying a marker string; 3 completed orders through `/wc/v3/orders` so `total_sales` is non-zero (a store of
  zeros once masked a data exposure). WooCommerce starts in **Coming Soon** mode: anonymous shop, cart,
  checkout and product pages show "Great things are on the horizon". Test store pages as the logged-in admin, or
  use the Store API. It is not an EB defect.
- **WooCommerce checks:**
  1. Coexistence: EB census still 94 / 25; `debug.log` delta. WooCommerce's activation-time DB-order error and
     `_load_textdomain_just_in_time` notice are its own; confirm no recurrence on 5 normal loads, and that EB never
     uses the `woocommerce` text domain.
  2. Anonymous `/essential-blocks/v1/products`: no draft or private product, no sold count or `total_sales`.
  3. `/essential-blocks/v1/queries` with `rest_base` = `product` / `product_variation` / `shop_order`: collapses
     to the default output, no leak by ID or forged `post_status`.
  4. EB Woo Product Grid and Pro Woo Product Carousel render, add-to-cart works, carousel initialises. Compare
     against WooCommerce's own `[products limit="20"]` loop: it omits Hidden products; EB's blocks list them
     until that is fixed (a known pre-existing defect: `known-noise.md`).
  5. Post Grid, Pro News Ticker and Pro Timeline Slider, including **dynamic** mode. Switch dynamic through the
     real control: it is a native `<select>`, so set its value and dispatch a `change` event. Setting the
     attribute programmatically skips the handler and renders nothing.
  6. EB's five single-product blocks (images, price, rating, details, add-to-cart): create a custom
     `single-product` template shell by REST (it overrides WooCommerce's plugin template), author the blocks
     through the Site Editor, view a product page. Test a product **with** an image and one **without**.
  7. Baseline 3-asset contract on a plain page, a product page and the cart.
- **Astra checks:** the 25-block fixture from axis 13 at 375 / 768 / 1440 (overflow, stacking, computed Wrapper
  background not overridden), the editor loads with 0 console errors, the EB hamburger opens and closes, and the
  **Customizer** (`customize.php`): count EB stylesheets and EB inline `<style>` tags -- must be **0** (the historical
  leak vector); EB scripts loading there is expected (the Widgets panel runs the block editor). Ignore core's
  `widget-types` `ERR_NETWORK_IO_SUSPENDED`. Pricing Table and Countdown outcomes depend on the theme's padding
  (320px column under Astra vs 300px under Twenty Twenty-Five): read them against `known-noise.md`.
- **Restore exactly.** Delete your content by ID. Switch back to the original theme. Uninstall WooCommerce through
  WordPress with its remove-all-data flag: a temporary `mu-plugin` containing
  `define('WC_REMOVE_ALL_DATA', true);`, then REST `PUT status:inactive` and `DELETE /wp/v2/plugins/woocommerce/woocommerce`.
  Then delete what its uninstall leaves: the "Refund and Returns Policy" page, `uploads/wc-logs/`,
  `uploads/woocommerce_uploads/`, and orphaned cron hooks (a one-shot `mu-plugin` clearing hooks matching
  `/^(woocommerce_|wc_|wc-|action_scheduler_|as_)/`). Delete Astra with `wp.updates.deleteTheme({slug:'astra'})` on
  `themes.php` while another theme is active. Remove the temporary `mu-plugin` files and the `mu-plugins/` folder if you
  created it. Verify against the baseline. Unremovable without DB access: small leftover option rows
  (`theme_mods_astra`, Astra settings, Action Scheduler) -- say so in the report.
- Not covered by this axis: Elementor and other page builders (use the Elementor + Essential Addons site from `references/environment.md`).

---

## P1 — expected in any honest release run

### 8. Per-card functional tests
Apply `wp-eb-test`'s diff-driven method once per release-scope card: map each changed file to a
testable claim and test that claim. This is what answers "did you retest all the things from this
release scope?"

### 9. Editor/frontend parity
Standing rule, not diff-gated. For every control in scope, editor behaviour must match frontend
behaviour. A control that works in one and not the other is a bug — unless its own label says it is
frontend-only.

### 10. Public hook and filter contract
~35 public hooks form the extension API. Assert names, arity and payload **shape** are unchanged —
a feed filter silently began passing `null` for every failure case. Run pro's `audit-free-pro-parity`
for the free->pro hook contract.

### 11. FSE / Site Editor
Promoted to P0 -- see above. Always run.

### 12. Content-import path
Pages created by REST, WP-CLI or the importer get `_eb_block_lists` but **no `_eb_attr`**, because that
meta is written by the editor on save — so they render completely unstyled until re-saved. Test at
least one imported/REST-created page per release.

### 13. Responsive
Promoted to P0 -- see above. Always run.

### 14. Compatibility
Promoted to P0 -- see above. Always run.

### 15. Dynamic and data blocks
Zero / one / many items. Empty states, pagination, filters, sorting. Post Grid, Woo Product Grid and
the feeds are the recurring offenders.

### 16. Third-party integrations
Facebook and Instagram feeds, Google Maps, AI. **Openverse and NFT are out of scope** by user
decision — they still get the generic all-block integrity sweep, no deep integration testing.

Watch: token caching and reset, request dedupe, cache-key composition, negative-caching TTLs,
transient key length against the `option_name` VARCHAR(191) limit, and whether a failure recovers for
blocks that were *already mounted* rather than only for new ones.

---

## P2 — drop first under time pressure, and say so

### 17. Accessibility and UI/UX
Keyboard reachability and focus return; WCAG 2.5.8 24x24 minimum target size (measured with
`getBoundingClientRect()`, not judged by eye); contrast including non-text UI at 3:1; screen-reader
announcement of state changes. The user asks for "ui ux issues if any" most releases.

### 18. Performance and front-end cost
Asset weight and count on a representative page; query counts; cold vs warm cache timing; whether a
cache ever warms at all; N feeds causing N token requests.

### 19. i18n / WPML / RTL
`.pot` completeness and correct text domain; `wpml-config.xml` translatability per block — exposing a
structural value (an element ID) as translatable rich text lets WPML's AI translate it and break every
reference. RTL layout sanity.

### 20. Lifecycle and uninstall
Activation, deactivation, reactivation. `uninstall.php`: are the custom tables and options actually
removed, and is that the documented intent? Leftover transients and orphaned rows.

---

## Standing, every run

**Cleanup and state restoration.** Delete the fixtures, users, probes and options *you* created.
Restore settings you changed. Report what was removed and what was verified unchanged. Never delete
anything you did not create.

---

## Ledger format

```markdown
| Axis | Component | Disposition | Evidence / reason |
|---|---|---|---|
| block-integrity | Free | swept | 69/69 serialize+reparse valid; 0 save.js changed |
| block-integrity | Pro | swept | 25/25 valid; wrapper dir ships no block.json |
| upgrade-path | Free+Pro | swept | 6.4.2 -> 6.4.3 in place, 102-block page 0 invalid |
| i18n | Free | deferred — environment | WPML not on this site; the WPML site in `references/environment.md` could cover it |
| lifecycle | Pro | not applicable | uninstall.php unchanged this release |
```
