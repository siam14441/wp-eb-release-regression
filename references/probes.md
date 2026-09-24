# Probes — proven verification snippets

Copy-pasteable and already used in real runs. Prefer these over clicking: synthetic clicks no-op on
Gutenberg controls and Interactivity API elements.

Browser snippets go inside one `browser_evaluate` call. Always wait for `wp` to exist first.

## Editor readiness gate

```js
async () => {
  for (let i = 0; i < 60; i++) {
    if (window.wp && wp.blocks && wp.data && wp.data.select('core/block-editor')) {
      return { ready: true, waitedMs: i * 250 };
    }
    await new Promise(r => setTimeout(r, 250));   // MUST yield — a sync loop never lets scripts load
  }
  return { ready: false };
}
```

If it returns `not-ready`, re-run rather than concluding anything. A false negative here poisons
every check after it.

## Block registry census

```js
() => {
  if (!(window.wp && wp.blocks)) return { error: 'wp.blocks not ready' };
  const eb = wp.blocks.getBlockTypes().filter(b => b.name.startsWith('essential-blocks/'));
  return {
    total: eb.length,
    pro: eb.filter(b => b.name.includes('/pro-')).length,
    names: eb.map(b => b.name).sort()
  };
}
```

Capture this on N-1 and N and diff the name lists. Anything that disappears is a recovery risk for
every site using it.

## Serialize -> re-parse sweep (all blocks)

The core block-integrity check. Creates each registered block, serializes it, parses it back, and
reports anything that fails to round-trip.

```js
() => {
  const out = { checked: 0, fails: [], threw: [] };
  wp.blocks.getBlockTypes()
    .filter(b => b.name.startsWith('essential-blocks/'))
    .forEach(bt => {
      out.checked++;
      try {
        const html = wp.blocks.serialize([wp.blocks.createBlock(bt.name)]);
        const parsed = wp.blocks.parse(html);
        if (!parsed.length) { out.fails.push({ name: bt.name, reason: 'parsed to 0 blocks' }); return; }
        const bad = parsed.filter(p => p.isValid === false || p.name === 'core/missing');
        if (bad.length) out.fails.push({
          name: bt.name,
          reason: bad[0].name === 'core/missing' ? 'core/missing (not registered)' : 'isValid=false'
        });
      } catch (e) { out.threw.push({ name: bt.name, error: String(e) }); }
    });
  return out;
}
```

## Validate stored content (the real test)

Round-tripping a freshly created block is weaker than validating what is actually in the database.

```js
async () => {
  // Use wp.apiFetch — it carries the nonce. `wpApiSettings` is NOT defined in the block editor.
  // Stop on the first failed page: WP returns 400 rest_post_invalid_page_number past the last page.
  const pages = [];
  for (let page = 1; page <= 10; page++) {
    let batch;
    try { batch = await wp.apiFetch({ path: `/wp/v2/pages?per_page=50&page=${page}&context=edit&status=any` }); }
    catch (e) { break; }
    if (!batch || !batch.length) break;
    pages.push(...batch);
  }
  const bad = [];
  pages.forEach(p => {
    const flat = [];
    (function walk(bs) {
      bs.forEach(b => { flat.push(b); if (b.innerBlocks && b.innerBlocks.length) walk(b.innerBlocks); });
    })(wp.blocks.parse(p.content.raw || ''));
    flat.filter(b => b.isValid === false || b.name === 'core/missing')
        .forEach(b => bad.push({ page: p.id, title: p.title.raw, block: b.name }));
  });
  return { pagesChecked: pages.length, invalid: bad };
}
```

## Build an all-blocks fixture page

```js
async () => {
  const names = wp.blocks.getBlockTypes()
    .filter(b => b.name.startsWith('essential-blocks/'))
    .map(b => b.name);
  const blocks = names.map(n => wp.blocks.createBlock(n));
  wp.data.dispatch('core/block-editor').resetBlocks(blocks);
  await wp.data.dispatch('core/editor').savePost();
  return {
    inserted: blocks.length,
    saved: wp.data.select('core/editor').didPostSaveRequestSucceed(),
    postId: wp.data.select('core/editor').getCurrentPostId()
  };
}
```

Then **reload the page** and re-read — validity before reload proves nothing.

```js
() => {
  const bs = wp.data.select('core/block-editor').getBlocks();
  return {
    total: bs.length,
    invalid: bs.filter(b => b.isValid === false).map(b => b.name),
    missing: bs.filter(b => b.name === 'core/missing').length
  };
}
```

See `fixture-gotchas.md` before trusting an empty-looking result — several blocks serialize to nothing
unless specific attributes are set.

## Frontend recovery markers

```bash
curl -s "$URL" | grep -c "unexpected or invalid content"      # must be 0
curl -s "$URL" | grep -oE 'wp-block-essential-blocks-[a-z-]+' | sort | uniq -c
```

## Capability matrix — create the users first

The gap every previous run left open. Create them, test as each, delete them in cleanup.

```bash
WP="wp --path=/path/to/site/app/public"
for R in subscriber author editor shop_manager; do
  $WP user create "qa_$R" "qa_$R@example.test" --role="$R" --user_pass="$(openssl rand -base64 18)" --porcelain
done
```

Then drive each session's cookies through the browser, or hit REST with application passwords, and
re-run every route probe. Record anonymous separately from logged-in-but-unprivileged — they are
different threat models.

```bash
$WP user delete qa_subscriber qa_author qa_editor qa_shop_manager --yes   # cleanup
```

## REST probing

```bash
curl -s -G "$SITE/wp-json/essential-blocks/v1/products" \
  --data-urlencode 'is_frontend=1' \
  --data-urlencode 'attributes={"showSoldCount":true,"preset":"style-1"}' \
  --data-urlencode 'query_data={"per_page":10,"offset":0}'
```

Vary the shape, not just the value: `true` / `"true"` / `1` / `"1"` / `"yes"` / `[true]` /
`{"0":true}` / case variants / duplicate keys / `__proto__` / omitted entirely.

Compare responses by **byte size and headers**, not by eye — identical `content-length` and
`x-wp-total` across a list of post types proves an allowlist is collapsing them all to the fallback.

```bash
for T in post page shop_order wp_template nav_menu_item revision; do
  printf '%s ' "$T"
  curl -s -o /dev/null -w '%{size_download} ' -D- -G "$SITE/wp-json/essential-blocks/v1/queries" \
    --data-urlencode "post_type=$T" 2>/dev/null | grep -i '^x-wp-total' | tr -d '\r'
done
```

Anonymous vs authenticated in the browser:

```js
await fetch(url, { credentials: 'omit' })      // must behave as an anonymous caller
await fetch(url, { credentials: 'include' })   // the legitimate path must still work
```

## Database checks

```bash
WP="wp --path=/path/to/site/app/public"
$WP db query "SELECT option_name, CHAR_LENGTH(option_name) AS len
              FROM wp_options WHERE option_name LIKE '%eb%transient%'
              HAVING len > 191;"                      # keys this long never read back
$WP db query "SELECT option_name FROM wp_options WHERE option_name LIKE '\_transient\_timeout\_eb%';"
$WP db query "SHOW TABLES LIKE '%eb_%';"               # eb_form_entries, eb_search_keywords
$WP post list --post_type=page --format=count
```

## Log and stability baseline

```bash
LOG="/path/to/site/app/public/wp-content/debug.log"
wc -l "$LOG"                                     # before the run
grep -ciE 'essential|eb_' "$LOG"
# ... run tests ...
wc -l "$LOG"                                     # after; diff the tail, not the whole file
```

Where `WP_DEBUG` is off, read Local's `logs/php/error.log` and the nginx error log instead.

## Artifact identity

```bash
# wp.org checksum manifest
VER=6.4.3
curl -s "https://downloads.wordpress.org/plugin-checksums/essential-blocks/$VER.json" -o /tmp/eb-sums.json
python3 - "$VER" << 'PY'
import hashlib, json, os, sys
base = "/path/to/site/app/public/wp-content/plugins/essential-blocks"
sums = json.load(open("/tmp/eb-sums.json"))["files"]
bad = missing = 0
for rel, meta in sums.items():
    p = os.path.join(base, rel)
    if not os.path.exists(p): missing += 1; continue
    if hashlib.md5(open(p, "rb").read()).hexdigest() != meta["md5"]: bad += 1; print("MISMATCH", rel)
print(f"checked={len(sums)} mismatched={bad} missing={missing}")
PY
```

```bash
# tested build vs shipped ZIP
diff -rq "$BUILD_DIR" "$INSTALLED_DIR" | head -50
find "$BUILD_DIR" -type f | wc -l; find "$INSTALLED_DIR" -type f | wc -l
# not an older release wearing a new label
grep -m1 'Stable tag' "$INSTALLED_DIR/readme.txt"
```

## Block-integrity diffing between two refs

```bash
# $FREE is set in references/environment.md
git -C "$FREE" diff --stat "$OLD_TAG".."$NEW_TAG" -- '*/save.js'        # any change = recovery risk
git -C "$FREE" diff --stat "$OLD_TAG".."$NEW_TAG" -- '*/deprecated.js'
git -C "$FREE" diff "$OLD_TAG".."$NEW_TAG" -- includes/blocks.php | grep -E '^[+-]\s+.[a-z_]+. =>'
# blocks whose save() applies a filter = Pro-off validation risk
grep -rl "applyFilters" "$FREE/src/blocks/"*/src/save.js
```

## Responsive and measurement

```js
() => ({
  overflow: document.documentElement.scrollWidth > document.documentElement.clientWidth,
  scrollWidth: document.documentElement.scrollWidth,
  clientWidth: document.documentElement.clientWidth
})
```

```js
// WCAG 2.5.8 minimum target size, measured not eyeballed
() => [...document.querySelectorAll('.eb-nav-item, .eb-button, [role="link"]')]
  .map(el => { const r = el.getBoundingClientRect();
               return { cls: el.className, w: Math.round(r.width), h: Math.round(r.height),
                        pass: r.width >= 24 && r.height >= 24 }; })
  .filter(x => !x.pass)
```

## Interactivity API — click with poll-retry

Never a single click immediately after navigation; see `known-noise.md`.

```js
async () => {
  const btn = document.querySelector('.wp-block-navigation__responsive-container-open');
  for (let i = 0; i < 20; i++) {
    btn.click();
    await new Promise(r => setTimeout(r, 150));
    if (document.querySelector('.is-menu-open')) return { opened: true, attempts: i + 1 };
  }
  return { opened: false, attempts: 20 };
}
```

## Asset graph

```js
() => ({
  styles: [...document.querySelectorAll('link[rel=stylesheet]')].map(l => l.href).filter(h => /essential|eb-/.test(h)),
  scripts: [...document.querySelectorAll('script[src]')].map(s => s.src).filter(h => /essential|eb-/.test(h))
})
```

```bash
# every referenced asset resolves
curl -s "$URL" | grep -oE '(src|href)="[^"]*(essential-blocks|eb-)[^"]*"' \
  | sed -E 's/.*="([^"]+)".*/\1/' | sort -u \
  | while read -r u; do printf '%s %s\n' "$(curl -s -o /dev/null -w '%{http_code}' "$u")" "$u"; done | grep -v '^200'
```

## FSE / Site Editor  (axis 11 -- always run)

Proven 2026-09-20 on a block theme (Twenty Twenty-Five). Run on `site-editor.php`, where `wp.apiFetch` exists.

**Readiness and block census**

```js
async () => {
  for (let i = 0; i < 80; i++) { if (window.wp && wp.blocks && wp.data && wp.blocks.getBlockTypes().some(b => b.name.startsWith('essential-blocks/'))) break; await new Promise(r => setTimeout(r, 300)); }
  await new Promise(r => setTimeout(r, 2500));
  const eb = wp.blocks.getBlockTypes().filter(b => b.name.startsWith('essential-blocks/'));
  return { total: eb.length, pro: eb.filter(b => b.name.includes('/pro-')).length };   // 94 / 25 at 6.4.4 + 3.2.2
}
```

Then `browser_console_messages` at level `error` -- must be 0.

**Validate every shipped block pattern.** Read them from the editor settings. They register on `admin_init`,
so `/wp/v2/block-patterns/patterns` returns none of them (an empty REST list is a false negative).

```js
() => {
  const s = wp.data.select('core/block-editor').getSettings();
  const all = s.__experimentalBlockPatterns || s.blockPatterns || [];
  const eb = all.filter(p => /^essential-blocks\//.test(p.name));
  const invalid = []; let blocks = 0;
  eb.forEach(p => { (function w(bs) { bs.forEach(b => { blocks++; if (b.isValid === false || b.name === 'core/missing') invalid.push(p.name + ' -> ' + b.name); if (b.innerBlocks.length) w(b.innerBlocks); }); })(wp.blocks.parse(p.content || '')); });
  return { patterns: eb.length, blocksParsed: blocks, invalidCount: invalid.length, invalid: invalid.slice(0, 10) };   // 79 patterns, 0 invalid
}
```

**Template + template part + page, authored through the Site Editor.** Create only the empty shells by REST, then
author the EB blocks in the editor and save with the entity save. Never put EB blocks into the shells by REST.

```js
// 1) shells (run on any admin page with apiFetch)
const theme = wp.data.select('core').getCurrentTheme().stylesheet;
await wp.apiFetch({ path: '/wp/v2/template-parts', method: 'POST', data: { slug: 'qa-fse-part', theme, type: 'wp_template_part', area: 'uncategorized', title: '[QA-RR] FSE part', status: 'publish', content: '<!-- wp:paragraph --><p>placeholder</p><!-- /wp:paragraph -->' } });
await wp.apiFetch({ path: '/wp/v2/templates', method: 'POST', data: { slug: 'qa-fse-tpl', theme, title: '[QA-RR] FSE template', status: 'publish', content: '<!-- wp:template-part {"slug":"header","tagName":"header"} /--><!-- wp:group {"tagName":"main"} --><main class="wp-block-group"><!-- wp:post-title {"level":1} /--><!-- wp:post-content /--></main><!-- /wp:group --><!-- wp:template-part {"slug":"qa-fse-part"} /--><!-- wp:template-part {"slug":"footer","tagName":"footer"} /-->' } });
await wp.apiFetch({ path: '/wp/v2/pages', method: 'POST', data: { title: '[QA-RR] FSE page', status: 'publish', template: 'qa-fse-tpl', content: '<!-- wp:paragraph --><p>body</p><!-- /wp:paragraph -->' } });
```

```js
// 2) open  /wp-admin/site-editor.php?postType=wp_template_part&postId=<theme>%2F%2Fqa-fse-part&canvas=edit
//    (and the wp_template equivalent), wait for blocks, insert, then save the ENTITY:
wp.data.dispatch('core/block-editor').insertBlocks([
  wp.blocks.createBlock('essential-blocks/wrapper', { wrpBackgroundbackgroundType: 'classic', wrpBackgroundbackgroundColor: 'rgb(255, 0, 0)' },
    [wp.blocks.createBlock('core/paragraph', { content: 'EB wrapper in the TEMPLATE PART' })])
]);
await new Promise(r => setTimeout(r, 4000));
const id = `${theme}//qa-fse-part`;
await wp.data.dispatch('core').saveEditedEntityRecord('postType', 'wp_template_part', id);
await new Promise(r => setTimeout(r, 3000));
wp.data.select('core').getLastEntitySaveError('postType', 'wp_template_part', id);   // must be falsy
```

For a Post Grid, click the block's **Start Blank** button (native `.click()` on the button inside
`iframe[name="editor-canvas"]`'s `contentDocument`) before saving, or it stores no `queryData` and renders nothing.

```bash
# 3) frontend -- always -L
curl -sL "http://SITE/qa-rr-fse-page/" -o fse.html -w 'http=%{http_code} bytes=%{size_download}\n'
grep -c "unexpected or invalid content" fse.html                          # 0
ls "$SITE/wp-content/uploads/eb-style/full-site-editor/"                  # one css per template and per part, created on first visit
```

```js
// 4) styles actually apply -- assert computed colours, at 375 / 768 / 1440
const bgOf = t => { const p = [...document.querySelectorAll('p')].find(x => x.textContent.includes(t)); return getComputedStyle(p.closest('.eb-wrapper-outer')).backgroundColor; };
```

```js
// 5) cleanup (delete only what you created), then rm the generated css and the full-site-editor/ dir
await wp.apiFetch({ path: '/wp/v2/pages/<id>?force=true', method: 'DELETE' });
await wp.apiFetch({ path: '/wp/v2/templates/' + encodeURIComponent(theme + '//qa-fse-tpl') + '?force=true', method: 'DELETE' });
await wp.apiFetch({ path: '/wp/v2/template-parts/' + encodeURIComponent(theme + '//qa-fse-part') + '?force=true', method: 'DELETE' });
```

## Responsive sweep  (axis 13 -- always run)

Use `browser_run_code_unsafe` so the viewport can be changed between loads. Wait ~2s after load for
hydration. Records overflow with offender attribution, column stacking, small touch targets by block, min
font size, hamburger box, console errors.

```js
async (page) => {
  const url = 'http://SITE/qa-rr-responsive-fixture/';
  const out = {};
  for (const [w, h] of [[375, 800], [768, 1024], [1440, 900]]) {
    const errs = []; const oc = m => { if (m.type() === 'error') errs.push(m.text().slice(0, 140)); }; const op = e => errs.push('PAGEERROR ' + String(e).slice(0, 140));
    page.on('console', oc); page.on('pageerror', op);
    await page.setViewportSize({ width: w, height: h });
    await page.goto(url, { waitUntil: 'load', timeout: 90000 });
    await page.waitForTimeout(2200);
    out[w] = await page.evaluate(() => {
      const vw = document.documentElement.clientWidth;
      const clipped = el => { for (let p = el.parentElement; p && p !== document.documentElement; p = p.parentElement) { const o = getComputedStyle(p); if (/(hidden|clip|auto|scroll)/.test(o.overflowX) && p.getBoundingClientRect().right <= vw + 1) return true; } return false; };
      const blockOf = el => { const b = el.closest('[class*="wp-block-essential-blocks-"]'); return b ? (b.className.match(/wp-block-essential-blocks-[a-z-]+/) || [''])[0].replace('wp-block-essential-blocks-', '') : null; };
      const off = [...document.querySelectorAll('body *')].filter(el => { const r = el.getBoundingClientRect(); return r.width > 0 && r.right > vw + 2 && !clipped(el) && getComputedStyle(el).position !== 'fixed'; })
        .map(el => ({ blk: blockOf(el) || 'non-eb', cls: (typeof el.className === 'string' ? el.className : '').slice(0, 40), right: Math.round(el.getBoundingClientRect().right) })).sort((a, b) => b.right - a.right);
      const cols = [...document.querySelectorAll('.wp-block-essential-blocks-column')].filter(c => c.getBoundingClientRect().width > 0).slice(0, 3).map(c => Math.round(c.getBoundingClientRect().left));
      const tgt = [...document.querySelectorAll('a[href], button, input:not([type=hidden]), select, textarea, [role=button], [role=tab]')].filter(el => { const r = el.getBoundingClientRect(); return r.width > 0 && r.height > 0 && getComputedStyle(el).visibility !== 'hidden' && blockOf(el); });
      const small = tgt.filter(el => { const r = el.getBoundingClientRect(); const inSentence = getComputedStyle(el).display === 'inline' && el.closest('p, li'); return !inSentence && (r.width < 24 || r.height < 24); });
      const by = {}; small.forEach(el => { const b = blockOf(el); by[b] = (by[b] || 0) + 1; });
      const ham = document.querySelector('.eb-advanced-navigation-wrapper .wp-block-navigation__responsive-container-open'); const hb = ham && ham.getBoundingClientRect();
      return { vw, overflow: document.documentElement.scrollWidth > vw + 1, scrollWidth: document.documentElement.scrollWidth, offenders: off.slice(0, 3), columnsStacked: cols.length > 1 && new Set(cols).size === 1, smallTargetsByBlock: by, hamburger: hb && hb.width > 0 ? [Math.round(hb.width), Math.round(hb.height)] : null };
    });
    out[w].consoleErrors = errs.slice(0, 3);
    page.off('console', oc); page.off('pageerror', op);
  }
  return out;
}
```

**Screenshots -- required, because the metrics are blind to clipped content.** Viewport crops of the tricky
blocks at 375, saved into the workspace `.playwright-mcp/` folder, then Read them:

```js
async (page) => {
  await page.setViewportSize({ width: 375, height: 800 });
  await page.goto('http://SITE/qa-rr-responsive-fixture/', { waitUntil: 'load' });
  await page.waitForTimeout(2200);
  for (const [k, sel] of Object.entries({ tabs: '.wp-block-essential-blocks-advanced-tabs', pricing: '.wp-block-essential-blocks-pricing-table', countdown: '.wp-block-essential-blocks-countdown' })) {
    const el = page.locator(sel).first(); if (!(await el.count())) continue;
    await el.scrollIntoViewIfNeeded(); await page.evaluate(() => window.scrollBy(0, -120)); await page.waitForTimeout(500);
    await page.screenshot({ path: `/ABS/PATH/.playwright-mcp/resp375-${k}.png` });
  }
}
```

**Attribute a defect against N-1** (identical source + shipped assets = pre-existing):

```bash
git -C "$FREE" diff --stat "$PREV_TAG" "origin/$RELEASE" -- src/blocks/<block> | tail -1      # empty = unchanged
for f in $(cd "$NEW/essential-blocks" && find assets -path '*<block>*' -type f); do
  cmp -s "$N1_WPORG/essential-blocks/$f" "$NEW/essential-blocks/$f" && echo "IDENTICAL $f" || echo "DIFFERS   $f"; done
```

## Media Library modal

`wp.media` frames open only from a **native** `.click()` inside `browser_evaluate`; a Playwright locator click
lands but opens nothing. The modal lists only the allowed type (an uploaded PDF is not offered to an
image-only picker).

```js
async () => {
  const btn = [...document.querySelectorAll('.interface-interface-skeleton__sidebar button')].find(b => /Upload Image|Replace Image/i.test((b.getAttribute('aria-label') || '') + ' ' + b.textContent));
  btn.click();
  for (let i = 0; i < 40 && !document.querySelector('.media-modal'); i++) await new Promise(r => setTimeout(r, 250));
  await new Promise(r => setTimeout(r, 1500));
  const modal = document.querySelector('.media-modal');
  (modal.querySelector('li.attachment[data-id="<id>"] .attachment-preview')).click();
  await new Promise(r => setTimeout(r, 800));
  modal.querySelector('button.media-button-select').click();
}
```

## Compatibility: WooCommerce + Astra  (axis 14 -- always run)

Proven 2026-09-20 (WooCommerce 11.1.1, Astra 4.13.12, WP 7.1.1).

**Download and verify (no wp-cli needed)**

```bash
curl -sL "https://api.wordpress.org/plugins/info/1.0/woocommerce.json" -o woo-info.json     # .version
curl -sL -o woocommerce.zip "https://downloads.wordpress.org/plugin/woocommerce.<ver>.zip"
curl -sL "https://downloads.wordpress.org/plugin-checksums/woocommerce/<ver>.json" -o woo-sums.json   # md5 every file
curl -sgL "https://api.wordpress.org/themes/info/1.1/?action=theme_information&request[slug]=astra" -o astra-info.json   # -g: brackets
rsync -a woo/woocommerce/ "$SITE/wp-content/plugins/woocommerce/" ; rsync -a astra/astra/ "$SITE/wp-content/themes/astra/"
```

Activate WooCommerce from `plugins.php` (click its activate link); activate Astra by **navigating to the
`href`** of `a[href*="action=activate"][href*="stylesheet=astra"]` (the on-card button is hidden until hover and a click is intercepted).

**Seed a store (run on an admin page: cookie auth works for `/wc/v3`)**

```js
const mk = (name, price, extra) => wp.apiFetch({ path: '/wc/v3/products', method: 'POST', data: Object.assign({ name, type: 'simple', regular_price: String(price), status: 'publish', short_description: name, description: `<p>${name}</p>` }, extra || {}) });
// 6 published + draft + private + catalog-hidden, each with a marker string in short_description
await mk('[QA] probe-hidden-product', 97, { catalog_visibility: 'hidden', short_description: 'QA-MARKER-HIDDEN' });
await wp.apiFetch({ path: '/wc/v3/orders', method: 'POST', data: { status: 'completed', line_items: [{ product_id: <id>, quantity: 5 }] } });   // makes total_sales non-zero
```

**Anonymous probes (curl -G, then grep for the markers)**

```bash
curl -sG "$SITE/wp-json/essential-blocks/v1/products" --data-urlencode 'is_frontend=1' \
  --data-urlencode 'attributes={"showSoldCount":true,"preset":"style-1"}' --data-urlencode 'query_data={"per_page":20,"offset":0}'
# also /essential-blocks/v1/queries with query_data.rest_base = product | product_variation | shop_order
# A "sold" hit is usually the word inside the add-to-cart URL (showSoldCount=...). Look for "N sold" / total_sales numbers.
```

**Woo's own reference loop** -- add a `core/shortcode` block `[products limit="20" columns="3"]` to the test page and
read `.woocommerce-loop-product__title` from the frontend. It omits Hidden, draft and private products.

**Pro News Ticker / Timeline Slider dynamic mode -- drive the native `<select>`**

```js
const sel = [...document.querySelectorAll('.interface-interface-skeleton__sidebar select')].find(s => [...s.options].some(o => /^Dynamic$/i.test(o.textContent.trim())));
Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, 'value').set.call(sel, 'dynamic-content');
sel.dispatchEvent(new Event('change', { bubbles: true }));   // fires React's onChange; wait ~10s for the posts to load
```

**Single-product template.** WooCommerce registers a plugin template `single-product`; `POST /wp/v2/templates` with
`slug:'single-product'` creates a custom override (source `custom`; deleting it restores the plugin one). Author
`essential-blocks/{product-images,product-price,product-rating,product-details,add-to-cart}` into its main group with
`replaceInnerBlocks`, save with `saveEditedEntityRecord`, then view `/?p=<product id>` in the browser (logged in) and
capture the page error stack:

```js
page.on('pageerror', e => errs.push(String(e.message) + ' @ ' + String(e.stack || '').split('\n')[1]));
// the EB Product Images error names assets/blocks/product-images/frontend.js
```

**Astra Customizer leak check** (on `customize.php`, after it loads):

```js
[...document.querySelectorAll('link[rel=stylesheet]')].filter(l => /essential-blocks/.test(l.href)).length   // must be 0
[...document.querySelectorAll('style')].filter(s => /eb-|essential-blocks/.test(s.id || '')).length          // must be 0
```

**WooCommerce teardown**

```php
// wp-content/mu-plugins/qa-wc-remove-data.php  (temporary; remove the file and the folder afterwards)
<?php if ( ! defined( 'WC_REMOVE_ALL_DATA' ) ) { define( 'WC_REMOVE_ALL_DATA', true ); }
```

```js
await wp.apiFetch({ path: '/wp/v2/plugins/woocommerce/woocommerce', method: 'PUT', data: { status: 'inactive' } });
await wp.apiFetch({ path: '/wp/v2/plugins/woocommerce/woocommerce', method: 'DELETE' });   // runs uninstall.php with the flag
// theme: on themes.php, with ANOTHER theme active:
wp.updates.deleteTheme({ slug: 'astra', success: r => console.log(r) });
```

Then remove what the uninstall leaves (see axes.md 14) and diff the site against the recorded baseline.
