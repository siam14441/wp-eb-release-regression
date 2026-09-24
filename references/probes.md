# Probes — proven verification snippets

Copy-pasteable and already used in real runs, except sections marked *draft*. Prefer these over clicking:
synthetic clicks no-op on Gutenberg controls and Interactivity API elements.

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

## Block settings UI and editor/frontend parity  (axis 9 -- P0, run every release)

**Status: first-run drafts.** The logic is tested against a mock of EB's `InspectorPanel` DOM, and the class
names come from the shipped `controls.js` (`codebase-map.md`, "Editor settings UI"), but these have not yet been
run in a live editor. Run each on one known block first and confirm the numbers by eye before trusting a whole
sweep, then delete this note.

**1. Expected tabs, from source.** `hideTabs` removes tabs on purpose, so the expected set is per block. Run for
the release ref and for N-1, in both repos. Block names come from `block.json`: Pro names carry a `pro-` prefix
that the folder name does not.

```bash
for R in "$FREE" "$PRO"; do
  git -C "$R" grep -l "hideTabs" -- 'src/blocks/*' | sed -E 's#^src/blocks/([^/]+)/.*#\1#' | sort -u | while read -r d; do
    printf '%s  %s\n' "$(jq -r .name "$R/src/blocks/$d/block.json")" \
      "$(git -C "$R" grep -h "hideTabs" -- "src/blocks/$d" | grep -oE "hideTabs=\{\[[^]]*\]\}" | head -1)"
  done
done
```

Turn the output into the `HIDE` map the census reads, e.g. `{ 'essential-blocks/wrapper': ['styles'] }`. A `hideTabs`
built from a variable will not match the literal grep: check by eye.

**2. Inspector census.** Open the post editor with the all-blocks fixture (see "Build an all-blocks fixture page"),
then open **Settings > Block** once so the sidebar shows the block inspector. Run on N-1 with `LABEL = 'N1'`
during the upgrade-path setup, then after upgrading with `LABEL = 'N'`. Batch with `START` = 0, 15, 30 ... so one
call stays under the tool timeout. It selects each EB block once, clicks each tab, opens panels **one at a time**
(EB closes the others), and stores an inventory in `localStorage`. Blocks that are child-only or invalid at
top level come back `invalid-skipped`: cover them through their parent. Finish starter steps first
(`fixture-gotchas.md`). Blocks that are new in N are missing from the N-1 fixture: insert them on N before the
run, and they appear under `addedByBlock` in the diff.

```js
async () => {
  const LABEL = 'N', START = 0, COUNT = 15;   // LABEL: 'N1' on the previous release, 'N' on the target. Batch with START/COUNT.
  const HIDE = {};   // fill from the hideTabs bash above, e.g. { 'essential-blocks/wrapper': ['styles'] }
  const wait = ms => new Promise(r => setTimeout(r, ms));
  const be = wp.data.select('core/block-editor'), bd = wp.data.dispatch('core/block-editor');
  const seen = new Set();
  const ids = be.getClientIdsWithDescendants().filter(id => {
    const n = be.getBlockName(id);
    if (!n.startsWith('essential-blocks/') || seen.has(n)) return false;
    seen.add(n); return true;
  });
  const KEY = { general: 'General', styles: 'Style', advance: 'Advanced' };   // EB's class is "advance", not "advanced"
  const leaf = els => els.filter(el => !els.some(o => o !== el && el.contains(o)));
  const labelsIn = root => [...new Set(leaf([...root.querySelectorAll('label, legend, h3, h4, [class*="-title"], [class*="-label"]')])
    .map(el => el.textContent.trim().replace(/\s+/g, ' ')).filter(Boolean))];
  const controlsIn = root => root.querySelectorAll('input, select, textarea, button:not(.components-panel__body-toggle)').length;
  const sig = {}, rows = [];
  for (const id of ids.slice(START, START + COUNT)) {
    const name = be.getBlockName(id), row = { name, flags: [] };
    if (!be.isBlockValid(id)) { row.flags.push('invalid-skipped'); rows.push(row); continue; }
    bd.selectBlock(id);
    await wait(400);   // let the sidebar re-render for the new selection
    let panel = null;
    for (let i = 0; i < 20 && !panel; i++) { panel = document.querySelector('.eb-parent-tab-panel'); if (!panel) await wait(150); }
    if (!panel) { row.flags.push(document.querySelector('.block-editor-block-inspector') ? 'no-eb-inspector' : 'sidebar-not-on-block-tab'); rows.push(row); continue; }
    const tabs = [...panel.querySelectorAll('.eb-tab')].map(b => ({ key: ['general', 'styles', 'advance'].find(k => b.classList.contains(k)), text: b.textContent.trim() }));
    const want = ['general', 'styles', 'advance'].filter(k => !(HIDE[name] || []).includes(k === 'advance' ? 'advanced' : k));
    const got = tabs.map(t => t.key);
    if (want.join() !== got.join()) row.flags.push('tabs-differ:want=' + want.join('/') + ' got=' + got.join('/'));
    sig[name + ' > tabs'] = tabs.map(t => t.text).join(',');
    row.panels = {};
    for (const t of tabs) {
      document.querySelector('.eb-parent-tab-panel .eb-tab.' + t.key).click();
      await wait(250);
      const bodySel = '.eb-parent-tab-panel .eb-tab-controls-' + t.key;
      const tabBody = () => document.querySelector(bodySel);
      if (!tabBody()) { row.flags.push('tab-body-missing:' + t.key); continue; }
      const count = tabBody().querySelectorAll('.components-panel__body').length;
      const loose = leaf([...tabBody().querySelectorAll('label, legend, h3, h4')].filter(el => !el.closest('.components-panel__body')));
      loose.forEach(el => { sig[name + ' > ' + t.key + ' > (no panel) > ' + el.textContent.trim().replace(/\s+/g, ' ')] = 1; });
      row.panels[t.key] = count;
      if (!count && !loose.length) row.flags.push('empty-tab:' + t.key);
      for (let k = 0; k < count; k++) {   // one panel at a time: EB's PanelBody closes the others when one opens
        let body = tabBody().querySelectorAll('.components-panel__body')[k];
        if (!body.classList.contains('is-opened')) { body.querySelector('.components-panel__body-toggle').click(); await wait(200); }
        body = tabBody().querySelectorAll('.components-panel__body')[k];
        const title = body.querySelector('.components-panel__body-title').textContent.trim();
        labelsIn(body).filter(l => l !== title).forEach(l => { sig[name + ' > ' + t.key + ' > ' + title + ' > ' + l] = 1; });
        sig[name + ' > ' + t.key + ' > ' + title + ' > #controls'] = controlsIn(body);
      }
    }
    rows.push(row);
  }
  const store = 'ebqa-inv-' + LABEL;
  const merged = START === 0 ? {} : JSON.parse(localStorage.getItem(store) || '{}');
  localStorage.setItem(store, JSON.stringify(Object.assign(merged, sig)));
  return { label: LABEL, blocksTotal: ids.length, doneThrough: Math.min(START + COUNT, ids.length), flagged: rows.filter(r => r.flags.length), clean: rows.filter(r => !r.flags.length).length };
}
```

Flags: `tabs-differ` (compare with `HIDE`), `empty-tab:<key>`, `no-eb-inspector`, `sidebar-not-on-block-tab`,
`invalid-skipped`, `tab-body-missing:<key>`. Every flag is a candidate, not a finding: check `known-noise.md` and N-1
first.

**3. Inventory diff, N-1 vs N.** Run after both snapshots exist.

```js
() => {
  const get = l => JSON.parse(localStorage.getItem('ebqa-inv-' + l) || 'null');
  const a = get('N1'), b = get('N');
  if (!a || !b) return { error: 'snapshot missing', have: { N1: !!a, N: !!b } };
  const blk = k => k.split(' > ')[0];
  const tally = ks => ks.reduce((o, k) => (o[blk(k)] = (o[blk(k)] || 0) + 1, o), {});
  const removed = Object.keys(a).filter(k => !(k in b));
  const added = Object.keys(b).filter(k => !(k in a));
  const changed = Object.keys(a).filter(k => k in b && a[k] !== b[k]).map(k => ({ k, was: a[k], now: b[k] }));
  return { removedByBlock: tally(removed), addedByBlock: tally(added), changedByBlock: tally(changed.map(c => c.k)),
           removed: removed.slice(0, 40), added: added.slice(0, 40), changed: changed.slice(0, 40) };
}
```

Read `removedByBlock` first: a control that disappeared or moved is the finding. Also scan both snapshots for
`undefined`, `[object Object]` and empty labels.

**4. Editor vs frontend parity.** Use a page with real content (the axis 13 fixture), not the bare all-blocks page.
Run the editor half in the post editor, then the frontend half on the published page of the **same origin**. Set
the width you want in the editor first (Preview > Tablet / Mobile, or `wp.data.dispatch('core/editor').setDeviceType('Tablet')`
where the store has it) and set the frontend viewport to the reported `canvasWidth`. Both halves ignore elements that
belong to a nested block, so an outer Wrapper is not blamed for its children.

```js
() => {
  const PROPS = ['display', 'flexDirection', 'flexWrap', 'justifyContent', 'alignItems', 'gap', 'textAlign', 'color', 'backgroundColor', 'backgroundImage',
    'fontFamily', 'fontSize', 'fontWeight', 'fontStyle', 'lineHeight', 'letterSpacing', 'textTransform', 'opacity', 'boxShadow', 'visibility',
    'borderTopWidth', 'borderTopStyle', 'borderTopColor', 'borderTopLeftRadius', 'borderTopRightRadius', 'borderBottomLeftRadius', 'borderBottomRightRadius',
    'marginTop', 'marginBottom', 'paddingTop', 'paddingRight', 'paddingBottom', 'paddingLeft'];   // no width/height/left/right margin: they depend on container width
  const record = (root, win, owns) => {
    const seen = {}, out = {};
    [root, ...root.querySelectorAll('[class*="eb-"]')].forEach(el => {
      const tok = [...el.classList].find(c => c.startsWith('eb-'));
      if (!tok || !owns(el)) return;   // skip elements that belong to a nested block
      const key = tok + '#' + (seen[tok] = (seen[tok] || 0) + 1), cs = win.getComputedStyle(el);
      out[key] = Object.fromEntries(PROPS.map(p => [p, cs[p]]));
    });
    return out;
  };
  const be = wp.data.select('core/block-editor');
  const frame = document.querySelector('iframe[name="editor-canvas"]');   // absent when the editor is not iframed
  const doc = frame ? frame.contentDocument : document, win = doc.defaultView;
  const byName = {}, missing = [];
  be.getClientIdsWithDescendants().filter(id => be.getBlockName(id).startsWith('essential-blocks/')).forEach(id => {
    const el = doc.querySelector('[data-block="' + id + '"]');
    if (!el) { missing.push(be.getBlockName(id)); return; }
    (byName[be.getBlockName(id)] = byName[be.getBlockName(id)] || []).push(record(el, win, e => e.closest('[data-block]') === el));
  });
  localStorage.setItem('ebqa-par-editor', JSON.stringify({ canvasWidth: doc.documentElement.clientWidth, blocks: byName }));
  return { canvasWidth: doc.documentElement.clientWidth, iframed: !!frame, blockTypes: Object.keys(byName).length, notInCanvas: missing };
}
```

```js
async () => {
  await new Promise(r => setTimeout(r, 2500));   // Interactivity API and sliders hydrate after load
  const PROPS = ['display', 'flexDirection', 'flexWrap', 'justifyContent', 'alignItems', 'gap', 'textAlign', 'color', 'backgroundColor', 'backgroundImage',
    'fontFamily', 'fontSize', 'fontWeight', 'fontStyle', 'lineHeight', 'letterSpacing', 'textTransform', 'opacity', 'boxShadow', 'visibility',
    'borderTopWidth', 'borderTopStyle', 'borderTopColor', 'borderTopLeftRadius', 'borderTopRightRadius', 'borderBottomLeftRadius', 'borderBottomRightRadius',
    'marginTop', 'marginBottom', 'paddingTop', 'paddingRight', 'paddingBottom', 'paddingLeft'];
  const record = (root, win, owns) => {
    const seen = {}, out = {};
    [root, ...root.querySelectorAll('[class*="eb-"]')].forEach(el => {
      const tok = [...el.classList].find(c => c.startsWith('eb-'));
      if (!tok || !owns(el)) return;
      const key = tok + '#' + (seen[tok] = (seen[tok] || 0) + 1), cs = win.getComputedStyle(el);
      out[key] = Object.fromEntries(PROPS.map(p => [p, cs[p]]));
    });
    return out;
  };
  const saved = JSON.parse(localStorage.getItem('ebqa-par-editor') || 'null');
  if (!saved) return { error: 'run parity-editor first, on the same origin' };
  const norm = v => String(v).replace(/url\([^)]*\)/g, 'url()');
  const issues = {}; let clean = 0;
  for (const [name, editorInstances] of Object.entries(saved.blocks)) {
    const roots = [...document.querySelectorAll('.wp-block-' + name.replace('/', '-'))];
    const rep = { editorCount: editorInstances.length, frontendCount: roots.length, onlyEditor: 0, onlyFrontend: 0, diffs: [] };
    editorInstances.forEach((ed, i) => {
      if (!roots[i]) return;
      const fe = record(roots[i], window, e => e.closest('[class*="wp-block-essential-blocks-"]') === roots[i]);
      Object.keys(ed).forEach(k => { if (!(k in fe)) rep.onlyEditor++; });
      Object.keys(fe).forEach(k => { if (!(k in ed)) rep.onlyFrontend++; });
      Object.keys(ed).filter(k => k in fe).forEach(k => Object.keys(ed[k]).forEach(p => {
        if (norm(ed[k][p]) !== norm(fe[k][p])) rep.diffs.push({ instance: i, el: k, prop: p, editor: ed[k][p], frontend: fe[k][p] });
      }));
    });
    if (rep.editorCount !== rep.frontendCount || rep.onlyEditor || rep.onlyFrontend || rep.diffs.length) { rep.diffCount = rep.diffs.length; rep.diffs = rep.diffs.slice(0, 6); issues[name] = rep; } else clean++;
  }
  return { editorCanvasWidth: saved.canvasWidth, frontendViewportWidth: document.documentElement.clientWidth, clean, issues };
}
```

Output: per block, `editorCount` vs `frontendCount` (a mismatch is either a block that fails to render on the
frontend or a class-name change: look), `onlyEditor` / `onlyFrontend` element counts, and the first property
differences. Expected differences are in `known-noise.md`. Then attribute each real difference against N-1 (the
axis 13 procedure) and take a side-by-side screenshot as evidence.

**5. Cleanup.** The snapshots live in the browser profile for that site's origin, not on the site:

```js
['ebqa-inv-N1', 'ebqa-inv-N', 'ebqa-par-editor'].forEach(k => localStorage.removeItem(k));
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
