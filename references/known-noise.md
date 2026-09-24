# Known noise — check before filing any finding

Every entry here already produced a wrong finding, a retracted report, or hours of wasted
investigation. Encoding them is as valuable as the axes themselves.

**None of these are excuses to stop looking.** If the symptom differs from what is described, or the
stated cause does not actually apply this time, it is a real finding.

## Build and asset artifacts

**`assets/vendors/js/bundle.babel.js` 404 — but only on locally-built trees.** It is registered
unconditionally in three PHP files and emitted only conditionally by webpack, so a local
`npm run build` can omit it and produce a 404. **The published wp.org package does ship it** (it is
present in the 6.4.3 checksum manifest), so a 404 seen on a site running the real release is *not*
this known gap and must be investigated. Verify which build the site is running before dismissing it.

**Missing assets after a local build are inconclusive.** Free's dependencies install through a
workaround (`npm install --legacy-peer-deps` + `ajv@8 --no-save`) because its `pnpm-lock.yaml` is
unusable, so the tree is not the intended one. Reproduce against the official ZIP before calling any
missing-asset finding a defect.

**A build that "wrote nothing" is usually fine.** webpack 5's `compareBeforeEmit` skips writing
byte-identical output, so mtimes do not change and the build looks like a no-op. Verify by wiping
`assets/` and rebuilding — a real build rewrites every file.

**`scripts/release.sh` drops a promo link.** Its `--force` path rewrites `includes/Admin/Admin.php`
from a two-link template to a single-`%s` link while the `sprintf()` call still passes two arguments.
Logged in the 6.4.1 and 6.4.2 runs and **still unfixed** — known, not new.

## Editor harness artifacts

**Rapid `insertBlock()` produces benign validation warnings** and Fancy Chart `NaN` console errors.
These are harness noise from inserting many blocks in quick succession, not defects. Verify against
the raw stored content (REST with `context=edit`) before filing anything based on in-editor state.

**Dirty-on-open is usually core, not the block.** Before reporting "the post is modified as soon as it
opens", run the control: a page containing only a core paragraph. If that is also dirty, it is core
`blockMeta` behaviour. Compare `getEditedPostContent()` against `getCurrentPost().content` and their
byte lengths.

## Timing

**The Interactivity API needs about a second to hydrate.** A click immediately after navigation often
no-ops, and the first immediate DOM check is **not a true negative**. A hamburger-menu FAIL was
published and then had to be retracted over exactly this. Always poll-and-retry before recording a
timing-sensitive UI failure — especially before shipping a report that contains it.

**Synthetic Playwright clicks and hovers no-op** on Gutenberg sidebar controls built as ARIA
`role="radio"`/`role="checkbox"` buttons, and on Interactivity API frontend elements. Drive state via
`wp.data.dispatch(...).updateBlockAttributes()` or a native `.click()` inside `browser_evaluate`.
Real CSS `:hover` cannot be sustained reliably across tool calls here — report hover-only interactions
as **unverified**, never as pass or fail.

## Platform facts that look like bugs

**WordPress polyfills the PHP 8 string functions.** `str_contains`, `str_starts_with` and
`str_ends_with` are defined in `wp-includes/compat.php` (loaded at `wp-settings.php:36`, well before
plugins at line 574). "This fatals on PHP 7" claims are almost always false. Check `function_exists`
**after** `wp-load`, not before — checking before is the trap. Also watch short-circuit evaluation:
a call on the last operand of a long `||` chain may never be reached.

**`offsetTop` is not document-relative on a block theme.** Always measure with
`getBoundingClientRect()` plus scroll position. Mixing `window.scrollY` (document space) with
`offsetTop` (offsetParent space) is itself a real bug class — but do not reproduce the same mistake
in your measurement code.

## Dev-confirmed expected behaviour

**Advanced Video Overlay + Pro deactivation** — deactivating Pro after saving a post containing Advanced Video Overlay content
causes a validation error when the post is reopened. Discussed with the dev and **expected**. Do not
test it, mention it, footnote it, or count it in coverage, for that ticket. This is scoped to Advanced
Video Overlay specifically; it is *not* a blanket exemption for Pro-deactivation validation errors
elsewhere, which remain a legitimate axis.

## Site-state confounders

**Stale content from an unmerged branch masquerades as a regression.** Test pages authored during an
earlier session may contain markup no shipping version produces. Confirm the markup could have come
from a released build before treating it as a regression.

**A "backup" may be a hybrid.** At least one backup folder held N-1 plugin headers with a
hand-deployed newer PHP file — useless as an A/B baseline. Rebuild the baseline from a tag instead of
trusting a backup directory's name.

## Known pre-existing visual and responsive defects  (added 2026-09-20)

Real defects, not noise -- report them, but as **pre-existing**, and never as a regression, once you have
confirmed the block's source and shipped `assets/blocks/<block>/*` are identical to the previous release
(`probes.md`, "Attribute a defect against N-1"). Re-verify each release; do not assume.

- **Pricing Table overflows the viewport on phones and tablets.** `.eb-pricing-item` renders 362px inside a
  300px `.eb-pricing-wrapper`: 392 vs 360 at 375, 761 vs 753 at 768, none at 1440. Default settings. Present
  in 6.4.3 and 6.4.4.
- **Countdown boxes clip at 375px.** `.eb-cd-inner` is `display:flex; flex-wrap:nowrap;
  justify-content:center`; the four boxes need 322px in a 300px container, so "Days" loses its left edge and
  "Seconds" reads "Second". The overflow metric does not see it -- only a screenshot does. Present in 6.4.3
  and 6.4.4.
- **Post Carousel's slick track extends the document** on bulk fixture pages (`scrollWidth` ~53,000 at
  375). Post Carousel JS, CSS and slick vendor files are identical across 6.4.3 / 6.4.4.
- **Small touch targets:** Post Grid meta links 18-22px tall; Advanced Navigation menu links 12px tall at
  tablet and desktop. INFO / polish (WCAG 2.5.8 exempts targets constrained by line height).
- **Log noise on the bulk 94-block page:** `views/post-meta.php` `Undefined variable $blockId` (a standalone
  `post-meta` block, no Post Grid context) and the Pro reCAPTCHA `WP_Scripts::add` dependency notice. Both
  appear on N-1 too.

## Compatibility pass: pre-existing defects and noise  (added 2026-09-20)

Real defects (report as pre-existing after confirming the block's source and shipped `assets/` are unchanged from N-1):

- **EB's WooCommerce blocks ignore "Hidden" catalog visibility.** Free `includes/API/Product.php` `query_builder()` has
  no `product_visibility` filter, so Woo Product Grid and Pro Woo Product Carousel list hidden products that
  WooCommerce's own `[products]` loop omits. Draft and private products are correctly excluded.
- **Product Images throws on products with no image.** Free `src/blocks/product-images/src/frontend.js:13-14` calls
  `gallery.querySelectorAll` on a null `gallery` (`TypeError: Cannot read properties of null (reading
  'querySelectorAll')`). With a real product image: no error.
- **`via.placeholder.com` is offline and hardcoded in Pro**: Woo Product Carousel editor
  (`src/blocks/woo-product-carousel/src/edit.js:310,509`), Timeline Slider editor components, and frontend PHP
  (`views/post-partials/timeline-markup.php:71`, `views/timeline-slider.php:130`). A console error in the editor;
  a broken image on the frontend when an item has no picture.
- **Timeline Slider dynamic mode logs `Undefined variable $queryData`** (`views/timeline-slider.php:39`, no `isset` guard).
  Log noise only.

Noise, not defects:

- **WooCommerce activation log entries:** a `wp_woocommerce_attribute_taxonomies doesn't exist` DB error and
  `_load_textdomain_just_in_time` notices for the `woocommerce` domain, only during the activation request. EB never
  uses that domain.
- **Astra Customizer:** EB's scripts load there (the Widgets panel runs the block editor) but no EB stylesheet or inline
  style does. Core's `widget-types` request can fail with `ERR_NETWORK_IO_SUSPENDED` in a headless browser.
- **`/shop/` looks empty** while WooCommerce is in Coming Soon mode.
