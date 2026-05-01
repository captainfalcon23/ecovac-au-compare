# Prompt: Add a New DEEBOT Model to the Comparison Widget

## Context

You are updating a self-contained HTML comparison widget (`deebot_compare.html`) that covers the current Ecovacs DEEBOT lineup available in Australia. The widget is vanilla JS with no dependencies, and all data lives inline in the `<script>` block. It has grid, list, and compare views with tier-coloured fields, multi-sort, and per-retailer pricing.

---

## What you need from the human before starting

Ask the human to provide the following before touching any code. Do not guess or search speculatively — bad URLs or invented prices will corrupt the dataset.

**Required:**
1. **Ecovacs AU spec page URL** — the `#specification` anchor URL for the new model  
   e.g. `https://www.ecovacs.com/au/deebot-robotic-vacuum-cleaner/deebot-MODEL#specification`
2. **Release year** (approximate is fine — the human usually knows)
3. **Retailer pricing and URLs** — for any of: Ecovacs.com/au, JB Hi-Fi, Harvey Norman, Good Guys  
   Format: retailer / URL / price  
   e.g. `JB: https://www.jbhifi.com.au/... / $1,299`

**Helpful but you can infer if missing:**
4. **Key features blurb** — Ecovacs marketing copy or bullet list (the spec table misses important details like hot air dry temperature, ZeroTangle version, AIVI version, Fresh-Flow, Triple Lift, TruePass, auto re-mop, bagless status, PowerBoost flash-charge rate, carpet care mode, camera resolution). Ask the human to paste the product page's feature highlights if available.
5. **Any caveats** — e.g. "this is actually a US-spec unit sold here without an AU listing", "discontinued", "not stocked at Harvey Norman"

---

## Step 1 — Extract the spec table

You have three options for extraction, in order of preference. Check which tools are available before proceeding.

---

### Option A — Playwright MCP (preferred)

Use if `playwright` tools are available in your toolset.

```javascript
async (page) => {
  const extractSpecs = async () => {
    const buttons = document.querySelectorAll('.style_f-specification__h_s1U button[aria-expanded="false"]');
    buttons.forEach(b => b.click());
    await new Promise(r => setTimeout(r, 1000));
    const container = document.querySelector('.style_f-specification__h_s1U');
    if (!container) return null;
    const specs = {};
    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT);
    const nodes = [];
    let node;
    while ((node = walker.nextNode())) {
      const text = node.textContent.trim();
      if (text && text.length > 1) nodes.push({ text, tag: node.parentElement.tagName });
    }
    for (let i = 0; i < nodes.length; i++) {
      const n = nodes[i];
      if (n.tag === 'DT') {
        const next = nodes[i + 1];
        if (next && next.tag === 'DD') { specs[n.text] = next.text; i++; }
      }
    }
    return specs;
  };
  await page.goto('SPEC_URL_HERE', { waitUntil: 'domcontentloaded', timeout: 25000 });
  await page.waitForTimeout(3000);
  return JSON.stringify(await page.evaluate(extractSpecs), null, 2);
}
```

Close the browser when done.

---

### Option B — Claude in Chrome extension (fallback)

Use if the `Claude in Chrome` tools are available but Playwright is not. Check with `tabs_context_mcp` first.

1. Get an available tab or create a new one with `tabs_create_mcp`
2. Navigate to the spec URL using `computer` (action: `left_click` on the address bar, then `type` the URL)
3. Wait for the page to load, then run the extraction script via `javascript_tool`:

```javascript
// Expand all collapsed accordions first
document.querySelectorAll('.style_f-specification__h_s1U button[aria-expanded="false"]')
  .forEach(b => b.click());
```

Then after a 1-second pause, run:

```javascript
(() => {
  const container = document.querySelector('.style_f-specification__h_s1U');
  if (!container) return 'CONTAINER NOT FOUND';
  const specs = {};
  const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT);
  const nodes = [];
  let node;
  while ((node = walker.nextNode())) {
    const text = node.textContent.trim();
    if (text && text.length > 1) nodes.push({ text, tag: node.parentElement.tagName });
  }
  for (let i = 0; i < nodes.length; i++) {
    const n = nodes[i];
    if (n.tag === 'DT') {
      const next = nodes[i + 1];
      if (next && next.tag === 'DD') { specs[n.text] = next.text; i++; }
    }
  }
  return JSON.stringify(specs, null, 2);
})()
```

Read the return value from `javascript_tool` — it's the full spec object.

---

### Option C — Human-provided screenshots (last resort)

Use if neither Playwright nor Claude in Chrome is available.

Ask the human to:

1. Navigate to the spec page URL (the one ending in `#specification`)
2. Click the **SPEC** tab if it isn't already selected
3. Expand **all** accordion sections on the page (click every `+` or expand button)
4. Take a full-page screenshot — on Windows: Snipping Tool → full page capture; on Mac: `Cmd+Shift+3` or use a browser extension like Full Page Screen Capture
5. Paste or upload the screenshot(s) into the conversation

Then read the spec rows directly from the image. Be thorough — capture every key-value pair visible. If multiple screenshots are needed to cover all sections, ask for them. Once you have all the values, construct the RAW spec object manually in the same key-value format as the other models in the widget.

> **Note:** Screenshot extraction is error-prone for dense spec tables. After constructing the RAW object, read it back to the human and ask them to confirm any fields that look wrong or are hard to read, particularly: suction Pa, battery mAh, runtime minutes, robot weight, and mop diameter.

---

## Step 2 — Understand the data structures you need to update

Open `deebot_compare.html` and locate these constants in the `<script>` block:

| Constant | What to add |
|---|---|
| `RAW` | Full spec key-value object from extraction |
| `PRICES` | `{ ecovacs: NNN, jb: NNN, hn: NNN, gg: NNN }` — only include retailers the human confirmed |
| `RETAILER_URLS` | `{ ecovacs: '...', jb: '...', hn: '...', gg: '...' }` — same retailers only |
| `SPEC_URLS` | The `#specification` URL |
| `RELEASE_YEARS` | Integer year |
| `FEATURES` | Supplementary feature data (see Step 3) |

**`normalize()` is automatic** — it reads from all of the above and produces the model object. You do not need to modify it unless you're adding an entirely new field type.

**`MODELS` filter** — if the model is discontinued in AU, add its name to the `.filter()` exclusion list:
```javascript
const MODELS = Object.entries(RAW)
  .filter(([n]) => n !== 'T50 MAX PRO OMNI' && n !== 'NEW DISCONTINUED MODEL')
  .map(([n, r]) => normalize(n, r));
```

---

## Step 3 — Populate the FEATURES entry

The `FEATURES` object captures things the spec table doesn't include. Add an entry for the new model:

```javascript
'MODEL NAME': {
  aivi_v:        '3.0',   // AIVI version: '2.0', '3.0', or '4.0'. T90 series = 4.0.
  zt_v:          '2.0',   // ZeroTangle version: '2.0' or '3.0'. null if not ZeroTangle-branded.
  te_v:          '2.0',   // TruEdge version: '1.0', '2.0', '3.0'. null if not listed.
  mop_rpm:       null,    // Roller RPM if stated (200 for X11/T90, 220 for T80S). null otherwise.
  hot_dry_c:     63,      // Hot air dry temp in °C. 63 = premium (X/T80S/T90). 45 = mid (T80/T50). null = unconfirmed.
  solution_auto: false,   // Auto cleaning solution dispenser in station (X8 MAX/PRO = true).
  powerboost_pct:null,    // Flash-charge % per 3 min pause (X11 = 6, T90 PRO = 10). null otherwise.
  triple_lift:   false,   // Triple Lift carpet/spill/debris adaptation (X8 MAX PRO, T80S).
  fresh_flow:    false,   // Fresh-Flow power washing — no dirty-water recycling (T90 series).
  truepass:      false,   // TruePass door-track crossing (X11 series).
  camera_res:    null,    // Camera resolution string e.g. '960P' (T50 PRO). null otherwise.
  bagless:       false,   // True only for X11 OmniCyclone and X11 PRO OMNI (no dust bags ever).
  re_mop:        false,   // Auto re-mop on detected stains (X8 PRO, T80 OMNI, T30S PRO).
  carpet_care:   false,   // Dedicated Carpet Care mode (X8 MAX PRO OMNI, X8 PRO OMNI).
  mop_override:  null,    // Override the spec table's mop type label. Use when spec says "Microfiber Roller"
                          // but key features doc says "OZMO ROLLER 3.0". e.g. 'OZMO ROLLER 3.0'.
},
```

**Key rules for FEATURES:**
- `mop_override` is critical for T90 series (spec table says "Microfiber Roller", correct value is "OZMO ROLLER 3.0") and T80S (correct is "OZMO ROLLER 2.0")
- X8 MAX PRO OMNI ZeroTangle is 3.0 per Ecovacs marketing despite spec table saying 2.0 — key features doc takes precedence
- If `hot_dry_c` is unknown, leave null rather than guessing — it's a genuine quality differentiator (63°C vs 45°C)

---

## Step 4 — Check for warnings

Add a notice if needed:

**US-spec unit sold in AU** (like X8 PRO OMNI): the notice bar, grid card tag, and compare column header all contain model-specific `if(m.name==='X8 PRO OMNI')` guards. Copy and adapt for the new model if it applies.

**Discontinued in AU**: add to the MODELS filter exclusion and update the subtitle count in `.hdr-sub`.

---

## Step 5 — Update the subtitle model count

Find and update:
```html
<div class="hdr-sub">11 models · 2025 · Spec comparison</div>
```

Increment the count if adding, or keep as-is if replacing a discontinued model.

---

## Checklist before finishing

- [ ] Spec data extracted via Playwright, Chrome extension JS injection, or human screenshot — method noted
- [ ] `RAW` entry added with full extracted spec object
- [ ] `PRICES` entry added (only confirmed retailers — never guessed)
- [ ] `RETAILER_URLS` entry added (matching retailers)
- [ ] `SPEC_URLS` entry added
- [ ] `RELEASE_YEARS` entry added
- [ ] `FEATURES` entry added with all fields populated or explicitly null
- [ ] `MODELS` filter updated if model is discontinued
- [ ] Any US-spec / caveats handled in notice bar and card tags
- [ ] Model count in `.hdr-sub` updated
- [ ] File copied to `/mnt/user-data/outputs/` and presented to user
- [ ] Browser closed after extraction (Playwright or Chrome extension)

---

## Important notes

- The spec table uses inconsistent key names across model families (X series vs T series). The `normalize()` function handles this — verify at least `suction_pa`, `runtime_min`, `battery_mah`, and `mop_type` resolve to non-null values for the new model before finishing.
- Always ask the human to confirm prices — do not scrape prices speculatively. Ecovacs.com/au, JB Hi-Fi, Harvey Norman, and The Good Guys all carry different selections and the human will have already checked for current discounts.
- The X8 PRO OMNI is a known edge case: no Ecovacs AU listing, specs sourced from ecovacs.com/us, 110V heating module. Any similar model should be flagged the same way.
- If using the Chrome extension, be aware the JavaScript injection approach reads the live DOM — make sure all accordions are expanded before running the extraction script or you'll get an incomplete spec object.
- If using screenshots, cross-reference values against the Ecovacs AU product listing page where possible, as spec table screenshots can be cropped or partially obscured.
