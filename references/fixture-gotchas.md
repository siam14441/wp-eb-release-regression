# Fixture gotchas — read before trusting an empty result

Every one of these cost real time, and several produced *false passes*: a block that renders nothing
because the fixture was built wrong looks exactly like a block that renders nothing because it is
broken. When a programmatically created block comes out empty, suspect the fixture first.

## Blocks that serialize to nothing unless you set the right attribute

**`showBlockContent` gates `save()`.** Several blocks begin `save()` with
`if (!showBlockContent) return null`. `createBlock()` leaves it `false`, so the block serializes
self-closing and renders nothing. Set it explicitly.

**Image Gallery's images attribute is `sources`, not `images`.** Setting `images` does nothing.

**Post Grid needs real queryable content.** A programmatic Post Grid renders empty when the query
returns nothing — check the site actually has posts of that type before calling it a bug.

**Accordion and other container blocks need `innerBlocks`.** With none, they render 0-height. Use
`replaceInnerBlocks()` or pass children to `createBlock()`.

## Attribute naming

**Responsive range attributes are suffixed `Range`** — `itemSpacingRange`, `markerSizeRange`,
`scrollSpeedRange`. Setting the bare name silently does nothing.

**Pro blocks use the `essential-blocks/` namespace with a `pro-` slug prefix** —
`essential-blocks/pro-one-page-navigation`, never `essential-blocks-pro/...`. Read `block.json`'s
`name` field rather than inferring from the folder. The wrong namespace produces an "Unsupported
block" that looks exactly like a licensing failure — and a dashboard "Activate your license" nag can
be an unrelated red herring. No license key is needed to build, edit or render blocks locally.

**Undeclared attributes are dropped on parse** and never re-serialized. If an attribute is not in the
block's schema, setting it programmatically has no lasting effect.

## Getting the correct expected markup

WordPress's **"Attempt Block Recovery"** button on a validation-failed block is the fastest reliable
way to obtain the real expected `save()` output and a working inspector — far quicker than
reverse-engineering markup by hand.

## Measurement

**`offsetTop` is not document-relative on a block theme.** Measure with `getBoundingClientRect()` plus
scroll position. Mixing `scrollY` (document space) with `offsetTop` (offsetParent space) is a genuine
bug class in this codebase — do not reproduce it in your own probe.

**Anchor IDs are not painted on the editor DOM** for core blocks. Resolve them through
`getClientIdsWithDescendants()` and the block attributes instead of querying the DOM.

## Navigation fixtures

A `core/navigation` block with a `ref` attribute points at a shared `wp_navigation` entity. Editing its
`innerBlocks` on the page changes local editor state only and is **silently discarded on save or
reload**. For an isolated test page, strip `ref` or build a fresh `core/navigation` with none.

## Editor state vs stored truth

In-editor state is not the artifact users get. Before filing anything based on what the editor shows,
confirm against the raw stored content via REST with `context=edit`. Rapid insertion produces benign
warnings that look alarming and mean nothing (see `known-noise.md`).

## Site hygiene

**One fixture page per purpose, freshly created.** Do not reuse a page from an earlier run — a page
corrupted by a previous repro produces confusing results, which has happened.

**Label fixtures so cleanup is unambiguous**, e.g. `[QA-RR] all-blocks 6.4.3`. You delete only what
you created; anything unlabelled is someone's real content.

**Seed WooCommerce with non-zero `total_sales`.** A store where every product has zero sales masked a
real data-exposure finding — the value being absent looked like the value being protected.

**Keep scratch files out of the WordPress install.** Use the session scratchpad, never the site root.

## Starter steps a programmatic block skips  (added 2026-09-20)

A block created with `createBlock()` skips the step a real user does first. The result looks like a bug and
is not.

- **Advanced Image:** `imgSource` has no default (`// default: "custom"` is commented out in
  `attributes.js`). Until a source is chosen the canvas shows "Please Select an Image Source" and the
  inspector shows only the core "Advanced" panel. Set `imgSource: 'custom'` (or click the choice), then
  the placeholder and the General / Style / Advanced tabs appear.
- **Post Grid:** starts on a template chooser ("Choose" / "Start Blank") with `showBlockContent: false`.
  No `queryData`, empty canvas, bare inspector, empty frontend -- in the post editor *and* the Site
  Editor, on 6.4.3 and 6.4.4 alike. Click **Start Blank** (native click) and `queryData` fills in
  (`source: post`, `per_page: "6"`), posts render, the full inspector appears.
- **Post Grid attributes by hand:** `version` must be `'v2'` for the featured post to render (no schema
  default; the editor sets it on mount). `featuredPostId` is a JSON *string* of an option object,
  `"{\"value\":12}"` -- a bare `"12"` or `12` renders nothing. Partial `queryData` renders nothing: copy a
  mounted block's full `queryData` and override only what you test.
- **Accordion:** `accordion-item` created without title text renders empty titles -- just "+" icons.
- **Standalone `accordion-item`** is declared `parent: ['essential-blocks/accordion']`, `inserter: false`;
  a top-level one reports invalid on every version. Fixture artifact, not a finding.

## Tooling traps  (added 2026-09-20)

- **`curl` on `?page_id=N` needs `-L`.** It 301s to the permalink; without `-L` you get an empty body, and
  grepping an empty body "finds nothing" and looks like a pass. Print `%{size_download}`.
- **Playwright: leaving an editor page raises `beforeunload`.** A `page.on('dialog')` handler inside
  `browser_run_code_unsafe` is pre-empted by the tool. Do one navigation per call and clear the prompt with
  `browser_handle_dialog`. Two scripts running at once fight over the single tab and return nothing useful.
- **Synthetic click does not open `wp.media`.** Native `.click()` inside `browser_evaluate` does.
- **Sessions expire** after a lot of REST user create/delete churn; expect a login redirect
  (`reauth=1`) and log in again.
- **Deleting a post does not delete EB's generated CSS.** `uploads/eb-style/eb-style-{id}.min.css` and
  `frontend/frontend-{id}.min.css` stay behind, and FSE runs create `full-site-editor/`. Remove only the
  ones for IDs you created.
- **wp.org N-1 baseline:** download `https://downloads.wordpress.org/plugin/essential-blocks.<ver>.zip`
  and verify it against `plugin-checksums/`. A zip in `~/Downloads` labelled with that version can be a
  pre-release build (6.4.3's failed 29 files).

## WooCommerce / Astra traps  (added 2026-09-20)

- **WooCommerce ships in Coming Soon mode.** Anonymous visitors see "Great things are on the horizon" on the shop,
  cart, checkout and product pages, so `curl` of `/shop/` shows none of your products and proves nothing. Use the
  logged-in browser, the public Store API (`/wp-json/wc/store/v1/products`), or a `[products]` shortcode on a normal page.
- **The Store API is not a reference for "Hidden".** It lists catalog-hidden products. The `[products]` shortcode loop
  omits them; use that to judge whether EB's blocks honour visibility.
- **Post editor vs Site Editor starter step.** In the post editor, a `createBlock()` Post Grid / News Ticker / Timeline
  Slider comes up ready (`showBlockContent: true`). In the Site Editor the same block shows its starter chooser until
  "Start Blank". Same block, different mount behaviour; it is by design.
- **Some EB controls are native `<select>` elements** (News Ticker and Timeline Slider "Content Source": Custom /
  Dynamic). A locator click does nothing and a dispatched click on the option does nothing. Set the value and dispatch
  `change`.
- **Theme card "Activate" button is hover-only**; a click is intercepted by the theme name overlay. Navigate to the
  link's `href` instead.
- **WooCommerce uninstall leaves residue** even with `WC_REMOVE_ALL_DATA`: the "Refund and Returns Policy" page,
  `uploads/wc-logs/`, `uploads/woocommerce_uploads/`, an orphaned cron hook (`woocommerce_marketplace_cron_fetch_promotions`),
  The `mu-plugins/` folder you create for the flag is yours to remove. Option rows cannot be listed or removed without DB access -- say so.
- **Deleting a template also leaves EB's generated CSS** (`eb-style/frontend/frontend-{id}.min.css` and
  `full-site-editor/full-site-editor-{id}.min.css`, keyed by the template's post ID).
- **`curl` with `[` `]` in the URL needs `-g`** (the wp.org themes API uses `request[slug]=`).
- **A product with no image renders no `.eb-product-image_slider`**, which is what triggers the Product Images JS
  error. Give a product a real image (`PUT /wc/v3/products/<id>` `images:[{id:<media id>}]`) to test the working path.
