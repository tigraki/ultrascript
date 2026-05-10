# Ultrascript — Spec (HTML version)

**v1.1.0**

A single-file browser-based teleprompter built for real recording workflows. No server, no accounts, no install. Open the file, paste your script, record. The design and UX of this version serves as the reference for the full Next.js build.

**heypaul.xyz**

---

## What it is

A professional teleprompter that runs entirely in the browser as a single HTML file. The reading surface is also the editing surface — there is no setup screen, no modes to switch between, no launch step. You paste or type directly into the text, adjust settings in the bottom bar, and hit Start.

Everything is stored in `localStorage`. Nothing leaves the device.

---

## Implementation

| | |
|---|---|
| Format | Single HTML file, ~3860 lines |
| CSS | Custom properties (CSS variables), all sizing in `rem` |
| JS | Vanilla ES5-compatible, `'use strict'` |
| Persistence | `localStorage` |
| Fonts | System fonts only — no CDN, works offline |
| Compatibility | All modern browsers, macOS/iOS/Windows/Android |

---

## Structure

```
ultrascript.html
├── <style>          CSS variables + full styling
├── <body>
│   ├── #wordmark    Top-left product mark (fades on START, returns on pause)
│   ├── #scroll      Fixed scroll container (inset 0, bottom 2.75rem)
│   │   └── #prose   contenteditable reading/editing surface
│   ├── #cue-overlay Optional cue indicator overlay (z-index 5, off by default)
│   │   ├── #cue-band   semi-transparent band style
│   │   ├── #cue-line   1px line style
│   │   └── #cue-col    column wrapper for the Left mark
│   │       └── #cue-left  vertical bar in prose's left padding
│   ├── #cd-overlay  Countdown SVG ring overlay (z-index 50)
│   ├── #bar         Fixed bottom control bar (height 2.75rem)
│   └── #info-overlay  Settings panel overlay (z-index 100)
│       └── #info-panel
│           ├── #info-head   Ultrascript wordmark (linked to ultrascript.vercel.app) + version label, ✕ close
│           ├── #info-tabs   Guide / Preferences / Script / Multi-speaker / Science
│           ├── #info-body   Scrollable tab content
│           └── #info-foot   heypaul.xyz
└── <script>         All JS, 'use strict'
```

---

## Theme system

Two themes: dark (default) and light. Applied via `html[data-theme="dark|light"]`. System preference detected via `matchMedia('(prefers-color-scheme: dark)')` and updated live on OS change when set to Auto.

### Theme vs. custom color scheme

The theme system and the color scheme are orthogonal but interact:

- **`setTheme(t)`** — called when the user picks Auto / Light / Dark. Clears any custom color overrides (sets `customColorsActive = false`, removes inline `color` / `background` from `#prose`, `#scroll`, and `body`), resets `fgColor` / `bgColor` to the theme defaults, hides the override note, and calls `syncCueToTheme()`. Always takes full control.

- **`applyColors(fg, bg)`** — called when a color preset or typed hex/rgba is applied. Derives `data-theme` from bg luminance (`lum = 0.299r + 0.587g + 0.114b`, threshold 0.4): light bg → `'light'`, dark bg → `'dark'`. Sets `customColorsActive = true` and shows `#color-override-note` so the user knows their custom scheme overrides the theme buttons. Calls `syncCueToTheme()` after the theme attribute is updated. RGB extraction goes through `colorToRgb(c, fallback)` (handles 3/6/8-digit hex and `rgb()` / `rgba()`) rather than the strict `hexToRgb` — `parseColor` preserves alpha and emits `rgba(...)` strings when the user types a translucent value into the BG input, and that value also flows back through `applyColors` on the next reload via `tp_bg`. The bg input's `input` handler does the same re-derivation through `colorToRgb` for the same reason.

- **`customColorsActive`** — JS boolean. `true` when a custom scheme is in effect. `setTheme()` clears it; `applyColors()` sets it. The `save()` function only writes `tp_fg` / `tp_bg` to localStorage when this flag is true; otherwise it removes those keys.

- **`#color-override-note`** — a `<p>` in Settings → Preferences (initially `display:none`). Shown by `applyColors()` to tell the user their custom color scheme is overriding the theme. Hidden by `setTheme()`.

- **`syncCueToTheme()`** — auto-pairs the cue color to the effective theme. If `cueColorCustomized === true`, exits immediately (respects the user's pin). Otherwise: light `data-theme` → `autocue-red` preset; dark → lookup `CUE_DEFAULTS_BY_SCHEME[fg|bg]` or fall back to `qtv-white`. Called by `setTheme`, `applyColors`, the bg color input handler (after luminance re-derivation), and the `matchMedia` OS theme change listener (only when `theme === 'auto'` AND `customColorsActive === false` — otherwise the OS would override the luminance-derived `data-theme` set by `applyColors` and desync the bar/chrome from the prose colors).

### CSS variables

| Token | Dark | Light | Used for |
|---|---|---|---|
| `--bg` | `#111` | `#f5f2eb` | Page background |
| `--text` | `#f0ede6` | `#1a1814` | Prose text |
| `--text2` | `#3a3a3a` | `#ccc` | Placeholder text |
| `--bar` | `#161616` | `rgba(232,229,221,0.98)` | Control bar background |
| `--bar-bd` | `rgba(255,255,255,0.08)` | `rgba(0,0,0,0.08)` | Bar border / dividers |
| `--bar-text` | `#f0f0f0` | `#111` | Bar primary text |
| `--bar-dim` | `#666` | `#888` | Bar secondary text |
| `--seg-bg` | `#1e1e1e` | `#e4e1da` | Segment / button backgrounds |
| `--seg-bd` | `#333` | `#ccc9c0` | Segment borders |
| `--seg-off` | `#555` | `#999` | Inactive segment labels |
| `--seg-on` | `#f0f0f0` | `#111` | Active segment text |
| `--seg-on-bg` | `#2e2e2e` | `#fff` | Active segment background |
| `--acc` | `#aaa` | `#444` | Accent (sliders, active borders) |
| `--go-bg` | `#efefef` | `#1a1a1a` | START button background |
| `--go-fg` | `#111` | `#f5f2eb` | START button text |
| `--overlay` | `rgba(0,0,0,0.6)` | `rgba(0,0,0,0.35)` | Panel backdrop |
| `--panel` | `#181818` | `#eeeae0` | Info panel background |
| `--panel-bd` | `rgba(255,255,255,0.08)` | `rgba(0,0,0,0.08)` | Panel borders |

---

## Prose (reading surface)

```css
#prose {
  padding: calc(var(--reading-anchor) * 100vh) 2.5rem 55vh;
  /* top:    places the first line at the reading-line anchor — defaults
              to 33vh (broadcast), or 38vh in Eye-rest mode. Reading starts
              immediately, no warm-up scroll.
     bottom: 55vh trailing whitespace so the last line scrolls comfortably
              above the bar. */
  font-family: Georgia, 'Times New Roman', serif;
  font-size: 1.75rem;
  line-height: 3.0;
  letter-spacing: .015em;
  max-width: 36rem;
  margin: 0 auto;
}
```

`contenteditable="true"`, `spellcheck="false"`, `autocorrect="off"`, `autocapitalize="off"`, `autocomplete="off"`. The autocorrect / autocapitalize / autocomplete trio specifically guards against iOS Safari rewriting the script under the user — without them, iOS happily auto-corrects words mid-paste and re-capitalizes after periods, producing real script edits the user didn't make. Clicking anywhere on the reading surface focuses it for editing. Clicking outside or pressing `Esc` blurs it, restoring keyboard shortcut and scroll behavior.

Placeholder text via `::before` pseudo-element when empty.

`max-width` scales proportionally with font size: `Math.round((fs / 1.75) * 36) + 'rem'` — so larger text gets a wider column, maintaining ~55–65 characters per line.

---

## Control bar

Fixed to the bottom of the viewport. `role="toolbar" aria-label="Ultrascript controls"`. Height auto with `min-height: 2.75rem`; wraps to multiple rows on narrow viewports via `flex-wrap: wrap`. `overflow: visible` (no scroll) — the v1.1 lean control set fits every viewport without needing a scrollable bar. Auto-hides during playback.

### Controls (left to right)

The v1.1 layout is intentionally minimal — only the controls a presenter needs while reading live their are on the bar. Everything else has moved to the **Settings** panel.

| Element | Type | Notes |
|---|---|---|
| `WPM` | Slider + clickable value | Range 40–300 on the slider, 40–500 via click-to-type. Value clickable. |
| spacer | `<div class="bar-spacer">` | `margin-left: auto`, `flex-shrink: 0`. Pushes the right cluster all the way right. |
| `0:00` | Timer | Hidden at rest (`display: none` in markup). Revealed by `doStart()`. Hidden, zeroed, and refreshed (`secs = 0; updTmr()`) on pause — the pause branch is the only place the timer resets. Sits to the left of START during playback. |
| `START / PAUSE` | Button | Primary action. White on dark, dark on light. |
| `Multi` | Toggle button | Multi-speaker mode. Turning ON: runs the full detection pipeline AND opens Settings to the Multi-speaker tab (so the detected speakers surface immediately for review/recolour). Turning OFF: strips colours; does not open the panel. |
| `Settings` | Button | Opens the Settings panel. Text button, same pill style as `Multi`. (`#btn-info` ID retained for backwards compatibility with focus-restore code; `aria-label="Open settings"`, `title="Settings"`.) |

### Removed from the bar in v1.1

| Element | New home |
|---|---|
| `⏱` Countdown toggle | Settings → Preferences (rendered as a labelled row: small "Countdown" label on the left, `⏱  Countdown` pill on the right) |
| `SIZE` slider + value | Settings → Preferences |
| `LINE` slider + value | Settings → Preferences |
| `Reset` button | Settings → Preferences (relabelled `Reset to defaults`, styled as a red-bordered destructive button via `#btn-reset` ID rule) |
| `Auto / Light / Dark` segmented control | Settings → Preferences |
| Visual dividers (`.div`) | Removed entirely — the wrapping flex layout reads cleaner without them |

### Auto-hide behavior

- The hide timer is queued by `scheduleHide()`, which is a no-op unless `running === true`. It fires 3 seconds after the timer was queued and fades the bar out (`opacity: 0`, `transform: translateY(100%)`).
- Direct start: the 3-second hide timer begins the moment `START` is pressed (inside `doStart`).
- Countdown start (⏱ on): the bar stays visible during the 3-second countdown (because `running` is still `false`); the hide timer is queued only after the countdown finishes and `doStart` runs. Total time from `START` press to bar hidden ≈ 6 seconds.
- Wakes instantly on: `mousemove`, `touchstart`, `keydown` (listeners on `document`), and `wheel` (folded into the `passive:false` wheel handler on `#scroll`, so there's only one wheel listener). Wake while paused just keeps the bar visible — `scheduleHide` won't queue a timer because `running` is `false`.
- On `PAUSE` (the only "stop" path — there is no separate Stop control): bar is shown and the pending hide timer is cleared.
- Transitions: `opacity .4s ease, transform .4s ease`.
- Scroll container (`#scroll`) transitions `bottom` between `2.75rem` (bar visible) and `0` (bar hidden) with `.4s ease`.

### Click-to-type values

WPM, Size, and Line height values are clickable spans. On click:
- Span replaced with `<input class="val-input">` pre-filled with the current value formatted by the field's `fmt(v)` function (WPM: integer; Size: 2 decimals; Line: 1 decimal)
- Input auto-focused and selected
- `Enter` or `blur` commits; `Escape` cancels (and stops propagation so the global Esc handler doesn't also fire)
- On commit: value clamped, slider updated, span restored using the same `fmt(v)` (so the displayed number doesn't visually shift after editing — e.g. `1.75` stays `1.75`), `computeLineHeight()` called for Size and Line, **`recomputeSpeedFromWPM()` called for Size and Line so the engine speed tracks the WPM the user set**, `save()` called
- Each field rounds the typed value to its display precision before storing — WPM to integer (`Math.round`), Size to 0.05 (`Math.round(v * 20) / 20`), Line to 0.1 — so `5.55` ↔ `5.6` etc. and the on-screen number always matches the value that's actually scrolling
- WPM accepts values 40–500 (slider visually pegs at 300; click-to-type runs the full range). Slider min was widened from 80 to 40 so saved low-WPM values aren't silently clobbered to 80 the moment the slider is touched.

---

## Scroll engine

### State variables

| Variable | Purpose |
|---|---|
| `offset` | Current scroll position in px |
| `lastTs` | Previous `requestAnimationFrame` timestamp |
| `isManual` | True while user is manually scrolling |
| `manualTmr` | Timeout handle for manual scroll resume |
| `accumulator` | Sub-pixel scroll accumulator |
| `lineHeightPx` | Computed line height in px |
| `lastBoundaryTs` | Timestamp when last line boundary was crossed (the only state the curve engine reads) |
| `_maxOffset` / `_layoutDirty` | Cached `scrollHeight − clientHeight` and a dirty flag — avoids forced reflow on every accumulator-overflow frame. Invalidated by prose input, font / line / font-family change, window resize, and `doStart`. |

### Tuned constants

Promoted out of inline literals so a future drift is easy to spot:

| Constant | Value | Used for |
|---|---|---|
| `SPEED_PX_PER_MS_UNIT` | `0.00576` | px per ms per speed unit (calibration) |
| `COUNTDOWN_DURATION_MS` | `3000` | full sweep of the countdown ring |
| `COUNTDOWN_RING_CIRC` | `326.7` | ring circumference (`2π × r=52`); also written into the SVG `stroke-dasharray` at startup |
| `MANUAL_SCROLL_RESUME_MS` | `900` | inactivity window before auto-scroll resumes |
| `BAR_HIDE_AFTER_MS` | `3000` | idle window before the control bar fades out |
| `CUE_BAND_HEIGHT_LINES` | `1.5` | `band` cue style height in line-heights |
| `READING_ANCHOR` | `0.33` | reading-line anchor as a fraction of `#scroll`'s height. JS mirror of the `--reading-anchor` CSS variable; both updated together via `setReadingAnchor()`. Default `0.33` is the documented Autocue/QTV broadcast standard; `0.38` ("Eye-rest") is offered as an alternative for slow / deliberate reads. Used by the prose `padding-top`, all four cue-indicator styles, the cue-fade gradient stop, `skipToNextSpeaker()`'s reading-Y math, and the manual-resume backdating fraction. |
| `WORDS_PER_LINE_AVG` | `6.4` | average words per line for English text in a 36rem column at 1.75rem default font (~35 chars per line / 5.5 chars per word). Used **only** for converting between the engine's `speed` (px/ms) and the user-facing `wpm`. Surfaced as a constant so the calibration is auditable; tweak if your audience's typical scripts use significantly longer or shorter words. |

### `computeLineHeight()`

```js
var rootPx = parseFloat(getComputedStyle(document.documentElement).fontSize) || 16;
var fntRem = parseFloat(document.getElementById('fnt-sl').value) || 1.75;
var lh     = parseFloat(document.getElementById('lh-sl').value)  || 3.0;
lineHeightPx = fntRem * rootPx * lh;
```

Called on: `doStart`, font size change (slider + click-to-type, via `setFontSize`), line height change (slider + click-to-type), reset, font picker change, window resize, and at the end of the initial localStorage `load()` so `lineHeightPx` reflects saved settings before the first frame runs.

### Scroll loop

Runs every frame via `requestAnimationFrame`. `dt` capped at 32ms to prevent lurching after tab suspension or manual scroll pause.

```js
var delta = speed * 0.00576 * dt * mult;
accumulator += delta;
if (accumulator >= 1) {
  var px = Math.floor(accumulator);
  accumulator -= px;
  // boundary detection + DOM update
}
```

DOM `scrollTop` only updated on whole-pixel increments to avoid unnecessary reflows. The auto-scroll path only ever increases `offset` (`px ≥ 1` per step), so it clamps the upper bound only — `max = scrollHeight − clientHeight`, floored at 0 so empty prose / very short scripts can't produce a negative `max`. The wheel handler, which can move `offset` either direction, clamps both ends to `[0, max]`.

`max` is read from a layout cache (`_maxOffset`) rather than being measured on every accumulator-overflow frame. The cache is invalidated by anything that could change `scrollHeight` or `clientHeight`: prose input, font-size / line-height / font-family changes, window resize, and `doStart` (so a settings change made during pause is picked up on resume). This avoids one forced reflow per scroll step.

`doStart()` re-syncs `offset` from `#scroll.scrollTop` before the first frame. While the user was editing prose, the browser may have shifted `scrollTop` to follow the caret (despite `overflow:hidden` — iOS/Android and contenteditable caret tracking still move it). The paste handler already re-syncs on paste; the resync inside `doStart` covers the edit-and-press-START path so playback resumes from where the user is actually looking, not from a stale `offset`.

The entire body of `loop()` is wrapped in try/catch. The `raf = requestAnimationFrame(loop)` call is moved to the **top** of the function so the handle is always live for the catch to cancel. On an unexpected throw inside the loop, the catch cancels the queued frame, nulls `raf`, sets `running = false`, tears down side-state that would otherwise resume on the next START (`isManual = false`, `clearTimeout(manualTmr); manualTmr = null`, `_skipAnimGen++` to invalidate any in-flight skip animation step), calls `showBar()`, and renders a red `showErrorToast` at the top of the viewport for 8 seconds. Without the wrap, a throw would silently terminate the RAF chain while leaving `raf` non-null — `doStart`'s `if(!raf)` guard would then incorrectly conclude the loop was still running and decline to restart it, leaving the prompter permanently frozen with no signal.

The user-driven pause branch (`btnGo` click handler when `running === true`) does the same side-state teardown — `clearTimeout(manualTmr); manualTmr = null; isManual = false; _skipAnimGen++;` — alongside the visible pause work (resets `secs`, hides `#tmr`, restores `#wordmark`, cancels the RAF loop). Without this teardown a pause that landed inside the 900ms `MANUAL_SCROLL_RESUME_MS` window would leave `isManual = true`, blocking the next START's loop integration until the pending `manualTmr` callback fired up to 900ms later; the pending callback would also backdate `lastBoundaryTs` from the post-resume offset and produce a one-frame velocity blip on resume; and any in-flight skip animation step would keep writing `scrollTop` after pause until it finished its own ease-out.

A secondary catch (`window.onerror`) covers errors outside the loop — event handlers, the `load()` IIFE, etc. — and surfaces the same red toast.

### Manual scroll

`wheel` event on `#scroll` (passive: false, `preventDefault()`):
- Updates `offset += deltaY * 0.6`, then clamps to `[0, max]` (where `max = scrollHeight − clientHeight`, floored at 0)
- Sets `isManual = true`
- After 900ms inactivity: clears `isManual`, resets `lastTs`, then **backdates** `lastBoundaryTs` so the curve resumes at the correct sub-line phase rather than restarting the ease-in. With `frac = (offset % lineHeightPx) / lineHeightPx` and `lineDuration = lineHeightPx / (speed * 0.00576)`:

  ```js
  lastBoundaryTs = performance.now() - frac * lineDuration;
  ```

  Falls back to `null` (multiplier = 1.0 until next boundary) only if `lineHeightPx === 0`.

---

## Mobile / touch support

### Responsive bar (`@media (max-width: 600px)`)

On screens ≤ 600px wide, the single-row bar wraps to multiple rows:

```css
#bar { height: auto; min-height: 2.75rem; flex-wrap: wrap;
       padding: .375rem .625rem; gap: .375rem .5rem;
       overflow-x: hidden; align-items: center; }
```

Tap-target minimums: `#btn-go` has `min-height: 2.25rem`; `#btn-multi` and `#btn-info` get `min-height: 2.25rem` too; the WPM slider is narrowed to fit (`#spd-sl: 5.5rem`). A second media query for landscape mobile (`max-width: 900px and max-height: 500px`) tightens paddings further so the bar stays single-row whenever possible. The `#tmr` timer's appearance/disappearance during play/pause is handled by the existing flex-wrap layout — no extra rules needed.

The info panel becomes a **bottom sheet**: `#info-overlay` gets `align-items: flex-end`; `#info-panel` gets `border-radius: .75rem .75rem 0 0`, `width: 100%`, `max-height: 92vh`.

### Dynamic bar height via `--bar-h`

The CSS custom property `--bar-h` is the single source of truth for bar height. `#scroll`, `#cue-overlay`, and `#cd-overlay` all use `bottom: var(--bar-h)` in CSS — no imperative `style.bottom` writes are scattered around the codebase.

`--bar-h` is kept current by three paths:

1. **`showBar()`** — removes `.hidden` from the bar, then sets `--bar-h` to `bar.offsetHeight + 'px'`. `offsetHeight` is accurate here because the hide animation uses `transform: translateY()`, which doesn't affect layout size — the bar's box is always laid out at its natural height.
2. **`hideBar()`** — adds `.hidden` to the bar, then sets `--bar-h` to `'0px'`.
3. **ResizeObserver** (registered at startup) — fires whenever the bar's rendered size changes (initial load, orientation change, mobile wrap to a second row). Updates `--bar-h` only when `barVisible === true` so it doesn't fight `hideBar()`.

The CSS `transition: bottom .4s ease` on `#scroll` and `#cue-overlay` still applies — the `var(--bar-h)` change propagates through CSS so the transition fires automatically.

### Touch scroll handler on `#scroll`

Drives `offset` from touch gestures identically to the wheel handler, including the 900ms manual-resume backdating logic.

- `touchstart` (passive: true): records `lastTouchY = e.touches[0].clientY`
- `touchmove` (passive: false, calls `e.preventDefault()`): computes `deltaY = lastTouchY - e.touches[0].clientY` (positive = scroll down), updates `lastTouchY`, updates `offset`, clamps to `[0, max]`, sets `scrollTop`. If `running`: sets `isManual = true`, resets `manualTmr` with the same 900ms backdate:
  ```js
  var frac = (offset % lineHeightPx) / lineHeightPx;
  var lineDuration = lineHeightPx / (speed * SPEED_PX_PER_MS_UNIT);
  lastBoundaryTs = performance.now() - frac * lineDuration;
  ```
- `touchend` (passive: true): clears `lastTouchY = null`

`wakeBar()` is also called inside `touchmove` so the bar wakes during a touch-scroll gesture. The existing `document.addEventListener('touchstart', wakeBar)` covers the initial tap.

---

## Dialogue mode

Multi-speaker colour mode. A `Multi` button in the bar toggles it. When on, the script is parsed for speaker labels and each speaker's `NAME:` label is wrapped in a coloured `<span>`.

### State variables

| Variable | Type | Purpose |
|---|---|---|
| `multiOn` | bool | Master on/off for dialogue mode |
| `dialogueSpeakers` | array | `[{name, key, color}]` — ordered by first appearance |
| `DIALOGUE_PALETTE` | array | 8-colour array assigned round-robin to speakers |

`DIALOGUE_PALETTE`: `'#5b9cf6'` (blue), `'#f97066'` (coral), `'#34d399'` (green), `'#fbbf24'` (amber), `'#c084fc'` (purple), `'#fb923c'` (orange), `'#22d3ee'` (cyan), `'#f472b6'` (pink).

### Bar button — `#btn-multi`

Pill button on the bar (right cluster, between START and Settings). Active state (`.on` class): `color: var(--bar-text)`, `border-color: var(--acc)`. Clicking:

1. Toggles `multiOn` and `.on` class.
2. If turning ON: runs `normalizeProse()` → `splitMidBlockLabels()` → `parseDialogue()` → `renderDialogue()` → `renderSpeakerList()`, **then opens the Settings panel pre-switched to the Multi-speaker tab** so the user sees the detected speakers immediately and can recolour or exclude false positives in one gesture. Tab-switch is done by direct class toggling (`.itab[data-tab="multi"]` + `#tab-multi` get `.on`); the panel itself is opened via `openInfoPanel()` so the focus trap and initial-focus pass run normally. Then `save()`.
3. If turning OFF: calls `renderDialogue()` (strips colours), then calls `save()`. **Does not open the panel** — turning Multi off is a "clean up the canvas" gesture; surfacing UI would fight that.

### Paste handler

`#prose` has a `paste` event listener that calls `e.preventDefault()` and re-inserts plain text via the modern `Selection` / `Range` API (`sel.deleteFromDocument()`, `range.insertNode(textNode)`, `range.collapse(false)`). Falls back to the deprecated `document.execCommand('insertText')` only for older browsers that don't support the Range API. This strips all inline styles, font overrides, and rich formatting from the clipboard before insertion — the primary guard against "tiny text" caused by pasting from Google Docs, web pages, or other rich sources.

After insertion, a `setTimeout(0)` re-syncs `offset` from `scrollTop` because iOS/Android can auto-scroll the container to follow the caret even when `overflow: hidden`.

### Parser — `parseDialogue()`

Walks top-level `<p>` / `<div>` blocks in `#prose`. When no block elements exist (the state right after a paste — the paste handler inserts a single text node with embedded `\n` characters) the parser falls back to `prose.textContent.split('\n')`. **Must be `textContent`, not `innerText`** — a raw text node renders with CSS whitespace collapse (`white-space: normal` default) so `innerText` would collapse `\n` into a single space and `split('\n')` would return only one entry, causing all speakers after the first to be silently lost. `textContent` returns the raw source text with `\n` preserved. The block-iteration path keeps `innerText` because already-structured DOM should respect `<br>` and block boundaries when extracting label text. `normalizeProse()` uses the same `textContent` rule for the same reason.

All three formats are detected via `getMatchInfo()`.

**Speaker regexes:**

The regexes use a hand-rolled Unicode letter range so non-Latin scripts (Greek `ΑΛΕΞ:`, Cyrillic `АЛЕКС:`, Arabic, Hebrew, CJK, Hangul, etc.) parse the same way Latin scripts do. ES5 doesn't support the `u` flag with `\p{L}`, so the range is enumerated explicitly via `\u…` escapes:

```js
var UL  = 'A-Za-zÀ-ɏͰ-ϿЀ-ӿԀ-ԯ֐-׿؀-ۿ一-鿿가-힯';
var UN  = UL + '0-9';
var WORD = '[' + UN + ']+';
var WORD_GAP = '(?:[\\- ][' + UN + ']+)';

var SPEAKER_RE        = new RegExp('^(' + WORD + WORD_GAP + '{0,1}):\\s+\\S');
var SPEAKER_ALONE_RE  = new RegExp('^(' + WORD + WORD_GAP + '{0,1}):\\s*$');
var SPEAKER_BRACKET_RE = new RegExp('^\\[(' + WORD + WORD_GAP + '{0,2})\\]');
```

Range coverage: `À-ɏ` covers Latin Extended-A/B (accented Latin); `Ͱ-Ͽ` Greek; `Ѐ-ӿ` and `Ԁ-ԯ` Cyrillic and Cyrillic Supplement; `֐-׿` Hebrew; `؀-ۿ` Arabic; `一-鿿` CJK Unified Ideographs; `가-힯` Hangul Syllables. The casing distinction (uppercase-only) is dropped because most non-Latin scripts don't have a cased pair — `getMatchInfo()` still normalises the captured name to uppercase via `toUpperCase()` for deduplication, which is a no-op for case-insensitive scripts.

`splitMidBlockLabels()` uses the same `UN` range with a `(?:^|[^UN])` lookalike anchor (start-of-string or a non-letter/digit character) to replicate the ASCII `\b` semantics across Unicode. The label start position is recovered from the match via `m.index + m[0].length - m[1].length - 2` (subtracting the captured label length plus `:\s`).

**`getMatchInfo(text)`** — tries all three in order, returns `{name, key, format}` or `null`. `format` is `'colon'` | `'alone'` | `'bracket'`.

| Format | Example | Regex | Match? |
|---|---|---|---|
| Inline colon — ALL-CAPS | `ALEX: text` | `SPEAKER_RE` | ✓ |
| Inline colon — ALL-CAPS hyphenated | `VOICE-OVER: text` | `SPEAKER_RE` | ✓ |
| Inline colon — Title Case | `Dr Smith: text` | `SPEAKER_RE` | ✓ |
| Inline colon — Greek | `ΑΛΕΞ: κείμενο` | `SPEAKER_RE` | ✓ (v1.1 Unicode) |
| Inline colon — Cyrillic | `АЛЕКС: текст` | `SPEAKER_RE` | ✓ (v1.1 Unicode) |
| Inline colon — CJK | `田中: テキスト` | `SPEAKER_RE` | ✓ (v1.1 Unicode) |
| Three-word label | `VISUAL INSERT NEEDED: text` | `SPEAKER_RE` | ✗ (capped at 2 words) |
| All-lowercase | `alex: text` | `SPEAKER_RE` | ✓ (v1.1 — uppercase-only constraint dropped to support uncased scripts; downstream parser dedupes by `toUpperCase()`) |
| Standalone label | `HOST:` (alone on line) | `SPEAKER_ALONE_RE` | ✓ |
| Bracket | `[Alex] text` | `SPEAKER_BRACKET_RE` | ✓ |
| Bracket multi-word | `[Dr Smith] text` | `SPEAKER_BRACKET_RE` | ✓ |
| Single Title-Case common word | `However: text` | `SPEAKER_RE` | ✓ (accepted edge case) |

Speaker names are normalised to uppercase for deduplication (`key = name.toUpperCase()`) so `Alex` and `ALEX` from different formats resolve to the same speaker. `dialogueSpeakers` stores `{name, key, color}`.

**Color preservation across re-parse.** Before resetting the speaker list, `parseDialogue()` indexes the existing `dialogueSpeakers` by `key`. When a re-detected speaker matches a previous key, its color is carried forward — the user's custom swatch choices survive every re-parse, every Multi off→on toggle, and the load-then-Multi-click sequence. Only newly detected keys consume a fresh palette color. The display `name` always reflects the current parse so casing edits in the script ("Dr Smith" → "DR SMITH") propagate to the panel; the key (uppercase) ensures the same speaker matches across casings.

### `splitMidBlockLabels()`

Pre-processing step that runs *before* `parseDialogue()` and `renderDialogue()` on every Multi turn-on and Re-parse. Walks every block (`p, div`) in `#prose`, scans its `textContent` with a permissive global label regex, and splits any block whose text contains a speaker-label-like sequence somewhere other than position 0 into multiple `<div>` blocks — one per speaker turn.

Why this exists: pasted scripts often pack multiple speaker turns into a single paragraph (`"...the obvious question. RHIAN: Yes. TORJE: Hmm."`). Without this step, `parseDialogue()` only catches the FIRST speaker per block (since `SPEAKER_RE` is anchored to `^`), and every subsequent label is silently invisible. With this step, each turn becomes its own block, the existing block-prefix matchers work correctly, and the rendered output puts each speaker on its own line.

The regex used here is permissive (looser than `SPEAKER_RE`) — it just finds candidate label positions to split at; `getMatchInfo()` does the strict validation later during `parseDialogue`. Any false-positive split (e.g. a stray `Note:` mid-paragraph) becomes its own block but won't be wrapped as a speaker because it won't show up in `dialogueSpeakers`.

Click-handler pipeline (Multi turn-on and Re-parse both):
1. `normalizeProse()` — flat text → `<div>` blocks (no-op if blocks already exist)
2. `splitMidBlockLabels()` — split any block with mid-paragraph labels
3. `parseDialogue()` — collect unique speakers
4. `renderDialogue()` — wrap label spans
5. `renderSpeakerList()` — refresh the panel UI

### `updateSpeakerStyles()`

Injects (or updates) a `<style id="sp-styles">` element in `<head>`. One CSS rule per speaker:

```
#prose .sp-label[data-sp="TORJE"] { color: #5b9cf6; }
```

`data-sp` stores the normalized uppercase speaker key. Using a stylesheet instead of inline `style=""` attributes means colors survive the browser's contenteditable sanitizer, which can strip inline styles during user input events.

Called by: `renderDialogue()` (always), color swatch `input` handler, ✕ exclusion handler.

### Renderer — `renderDialogue()`

Safe for `contenteditable` — only the first text node of each matching block is modified.

**Turning ON:**
1. Calls `updateSpeakerStyles()` to inject/refresh the CSS color rules.
2. Calls `normalizeProse()` to ensure block elements exist (pasted content may be flat text + `<br>`).
3. For each block, `getMatchInfo(b.innerText.trim())` is called:
   - If an existing `.sp-label` span is found: if `data-sp` matches a speaker key in the active set, return (CSS handles color). If not in the set, unwrap with `replaceChild(textNode, span)` and fall through.
   - Otherwise: use `TreeWalker(NodeFilter.SHOW_TEXT)` to get the first text node. Split character is `]` for bracket format, `:` for colon/alone. Wrap label in `<span class="sp-label" data-sp="{NORMALIZED_KEY}">`, leave remainder in a sibling text node.

`data-sp` stores the **normalized uppercase key** (e.g. `"DR SMITH"`), not the display name. The CSS selector targets it directly.

**Turning OFF:** `querySelectorAll('.sp-label')` → for each span, `replaceChild(textNode, span)`. Clears the `#sp-styles` stylesheet. No `innerHTML` writes, no content lost.

### Speaker list — `renderSpeakerList()`

Renders in the Multi-speaker tab of the Settings panel. One row per speaker: native `<input type="color">` swatch + name label + **✕ exclude button**.

Swatch `input` handler: updates the speaker's `color` property (via an IIFE closure over the object reference, not the array index), calls `updateSpeakerStyles()` to refresh CSS rules, updates the name label's inline color in-place, and calls `save()`. **Does not** rebuild the speaker list during dragging — rebuilding the list while the native color picker is open closes it immediately. Swatch `change` handler (fires when picker closes): rebuilds the full list so index closures stay fresh.

✕ button handler: `dialogueSpeakers.splice(i, 1)`, then calls `renderDialogue()` (which unwraps stale spans for the removed speaker and clears its CSS rule) + `renderSpeakerList()` + `save()`. This is the recommended way to exclude false-positive speaker detections.

**Empty state:** `#dialogue-empty` visible when `dialogueSpeakers.length === 0`; `#dialogue-speakers` and `#btn-reparse` hidden.

**Re-parse button (`#btn-reparse`):** Re-runs `parseDialogue()` + `renderDialogue()` + `renderSpeakerList()`. Appears when speakers exist. Useful after editing the script post-parse.

**Compact spacing toggle (`#btn-compact-speakers`):** Controls whether blank lines between speaker turns render as full line-height gaps. Default on. When on, `#prose` gets the `.compact-speakers` CSS class, which applies `display: none` to any direct child `<div>` that contains only a `<br>` (the normalized representation of an empty line). This means raw scripts that naturally use blank lines to separate speaker turns look tight and broadcast-appropriate without requiring special formatting. Turn off for scripts where blank lines are intentional pauses. State variable: `compactSpeakers`. Persisted as `tp_compact_speakers`. `applyCompactSpeakers()` syncs the class and button state.

### Multi-speaker tab

Standalone tab in the Settings panel (`id="tab-multi"`). Wired into the existing tab switcher via the same `data-tab` convention — no JS changes to the switcher needed. The bar's Multi button deep-links here on activation (see *Bar button* above).

Contains three `isec` sections:

**Formats** — instructions and format examples for all three speaker label formats, plus **Copy AI prompt button** (`#btn-copy-prompt`) — copies `MULTI_FORMAT_PROMPT` to the clipboard. Falls back to a `<textarea>` + `select()` + `execCommand('copy')` for non-HTTPS or older browsers. Button label changes to "Copied!" for 1.8s, then reverts.

**Compact spacing** — heading + one-line blurb + `#btn-compact-speakers` pill (right-aligned). See *Compact spacing toggle* above.

**Speakers** — speaker list (`#dialogue-speakers`), re-parse button (`#btn-reparse`), and empty state (`#dialogue-empty`).

### `MULTI_FORMAT_PROMPT`

A multi-line string constant (joined from an array) that instructs an AI to reformat a raw script for the Multi parser. Seven numbered rules:

1. Speaker labels start a NEW line, followed by colon + space.
2. **NEVER put two different speakers in the same line or paragraph** — if a speaker change happens mid-sentence in the source, break the sentence so each speaker starts on its own fresh line. Includes a WRONG/RIGHT example. (Even though the parser handles mid-paragraph labels via `splitMidBlockLabels`, clean source produces clean output and avoids ambiguous parses.)
3. One blank line before every speaker label.
4. Don't change the dialogue text — preserve every word, contraction, punctuation mark, pause marker.
5. Stage directions / scene notes / non-dialogue lines stay on their own lines without a label.
6. Convert any pre-existing label format (e.g. `[Name]` brackets, lowercase `name:`) to the CAPS-colon form.
7. **Block false-positive speaker matches** — the parser detects `Capitalised Word(s): content` patterns *anywhere* in a paragraph (via `splitMidBlockLabels`), not just at line start. So mid-sentence phrases like *"the question we put to Finanstilsynet was: does this work?"* or *"as Stephen King wrote: ..."* get tagged as new speaker turns. The rule asks the model to replace the colon with em-dash (`—`) whenever the capitalised lead-in clearly isn't a character name. Covers mid-sentence references, attributions/quotations, definitions, and the line-start list/section cases (`First:`, `Note:`, `Example:`, `However:`, `Tip:`, `Warning:`, `Step 1:`, `Q:`, `A:`, headings). Spacing is contextual — closed (`Word—content`) inside a sentence, spaced (`Word — content`) for line-leading list items. Mitigates the entire `Capitalised Word:` false-positive class at source-format time without changing the parser's permissive grammar.

Stored as a JS constant. The copy button always reads from the same source — no runtime mutation.

### Invariant

Dialogue rendering is purely additive (wraps one `<span class="sp-label">` per matching block) / subtractive (unwraps that span via `replaceChild`). The rest of each paragraph is never touched. `contenteditable` editing works identically in both modes.

---

## Acceleration engine

### Principle

Time-domain acceleration: the curve is driven by elapsed time since the last line boundary crossing, not by pixel position. This makes it accurate regardless of font, window size, or line height — it never drifts.

### `computeLineHeight()` → `lineHeightPx`

One line = `fontSize_rem × rootFontPx × lineHeight_unitless`.

### Boundary detection

In the loop, after computing `newOffset`:
```js
var prevLine = Math.floor(offset / lineHeightPx);
var newLine  = Math.floor(newOffset / lineHeightPx);
if (newLine > prevLine) syncBoundary(ts);
```

`syncBoundary(ts)` records `lastBoundaryTs = ts`. That's the entire state the curve engine needs — the position at the boundary doesn't have to be remembered because the curve is driven purely from elapsed time and the (constant) expected line duration.

First boundary seeded on `doStart` using `performance.now()` so the curve is active from line 1.

### `getVelocityMultiplier(ts)`

```js
var lineDuration = lineHeightPx / (speed * 0.00576);
var elapsed = ts - lastBoundaryTs;
var t = Math.max(0, Math.min(elapsed / lineDuration, 1.0));
return bezierMultiplier(t);
```

`t` is clamped to `[0, 1]`. The upper bound shouldn't be exceeded in practice (boundaries re-sync on each line crossing); the lower bound is defensive against negative `elapsed` values from RAF / `performance.now()` clock-origin mismatches and the manual-resume backdate's floating-point edge cases. Without `Math.max(0, …)`, a slightly-negative `t` produces a 1-frame velocity blip near 0.86 instead of the 0.80 floor.

### `bezierMultiplier(t)` — asymmetric piecewise curve

Three segments, each using `smoothstep`. The constants are picked so the curve is C0-continuous at both seams (no velocity jump at t=0.4 or t=0.65) **and** the time-weighted average equals exactly 1.0:

| Segment | Range | Shape | Min → Max |
|---|---|---|---|
| A — ease in | `t: 0 → 0.4` | `0.80 + 0.3312 * smoothstep` | 0.80 → 1.1312 |
| B — peak | `t: 0.4 → 0.65` | flat | 1.1312 |
| C — ease out | `t: 0.65 → 1.0` | `0.76 + 0.3712 * (1 - smoothstep)` | 1.1312 → 0.76 |

The ramp-up lift (0.3312) is smaller than the ramp-down lift (0.3712) because the eye starts slightly higher (0.80) than it ends (0.76). Both segments meet the peak (1.1312) cleanly.

**Normalization proof (average = 1.0):**
- Seg A: avg `0.80 + 0.3312 × 0.5 = 0.9656`, width 0.40 → contribution `0.38624`
- Seg B: `1.1312`, width 0.25 → contribution `0.28280`
- Seg C: avg `0.76 + 0.3712 × 0.5 = 0.9456`, width 0.35 → contribution `0.33096`
- Total: `0.38624 + 0.28280 + 0.33096 = 1.00000 ✓`

The speed you set is genuinely what you get on average — set 6, you get speed 6 over each line, not 5.4 or 6.3.

### WPM as the primary unit

The user-facing speed value is **WPM** (words per minute) — the universal teleprompter unit. Internally the engine still drives off a `speed` value (px/ms calibration unit, see `SPEED_PX_PER_MS_UNIT`), but `speed` is **derived** from `wpm`, the current font / line height, AND the live measurement of how many words actually fit on a line via `wpmToSpeed()` whenever any of those change. Persisted as `tp_wpm`.

This means: if you set 160 WPM and then bump the font from 1.75rem to 2.5rem (or rotate from desktop landscape to phone portrait, or switch from Georgia to Courier monospace), the displayed WPM stays 160 and the engine recomputes the underlying px/ms rate so the perceived read pace remains constant. Bigger fonts, narrower viewports, and wider-glyph fonts no longer secretly slow or speed up the read.

### Words-per-line — live measurement

Pre-1.0.1 used a fixed constant `WORDS_PER_LINE_AVG = 6.4` calibrated for the desktop 36rem column at Georgia 1.75rem. That broke badly in two real cases:

- **Mobile portrait** — the column is constrained by the viewport (~19rem on a 390px phone), giving ~4 words/line. Engine paced lines as if each carried 6.4 words, so 160 WPM displayed = ~105 WPM actual (35% under-pace).
- **Monospace fonts (Courier)** — fixed-width chars fit fewer words per line at the same column width than proportional Georgia.

`getWordsPerLine()` measures live capacity from the rendered prose box and the active font:

```js
ctx.font = computedFontSize + ' ' + computedFontFamily;
var avgWordPx = ctx.measureText('The quick brown fox jumps over the lazy dog ').width / 9;
var colWidth = prose.clientWidth - paddingLeft - paddingRight;
return colWidth / avgWordPx;
```

Canvas `measureText` runs sub-millisecond; `getWordsPerLine()` is only called from user-driven recompute paths (slider input, click-to-type, font picker, profile apply, reset, window resize) — never inside the RAF loop — so caching adds complexity without measurable benefit. Falls back to the historical `WORDS_PER_LINE_AVG = 6.4` constant only if prose hasn't laid out yet at call time (very early load) or the measurement is non-finite.

`recomputeSpeedFromWPM()` is now also called from the **font picker** click handler (font family affects measurement) and the **window resize** handler (viewport affects column width).

### Calibration

At defaults (60fps, root 16px, font 1.75rem, line 3.0, desktop column ≈ 6.5 measured words/line):
- `lineHeightPx = 1.75 × 16 × 3.0 = 84px`
- ms/frame = `1000 / 60 ≈ 16.67`
- 160 WPM with ~6.5 measured words/line means: `6.5 / (160 / 60) ≈ 2.44 sec/line` → engine speed = `84px / 2.44s ≈ 34 px/sec` → `speed ≈ 5.97`

Closed-form WPM formula:

```
WPM = words_per_line × 21.6 × speed / (fs × lh)
```

Where `words_per_line` is the live `getWordsPerLine()` measurement. The `21.6` is `60 sec/min × 0.00576 px/ms × 1000 ms/sec / 16 px/rem`. Inverse:

```
speed = WPM × (fs × lh) / (words_per_line × 21.6)
```

### WPM scale

| WPM | Use case (industry-grounded) |
|---|---|
| 100–130 | **Conference / Keynote** — slow, deliberate. State-of-the-Union pace. |
| 130–150 | **Podcast / Creator / YouTube** — conversational |
| 150–175 | **News anchor** — broadcast / monologue. **160 = default.** |
| 175–200 | Brisk news read, sermons in narrative mode |
| 200–250 | Fast read, near the upper bound of comfortable listening |
| 250+ | Scrub / rehearsal — speed-read for memorization |

Sources: average English speakers run ~125–150 WPM in conversation; news anchors per multiple industry sources read 150–175 WPM; sermon pacing research cites 140–160 WPM with theology-vs-narrative variation; conference / TED speakers run 100–130 WPM.

### Approximation note

WPM display is **approximate**, within ±~5% across all devices and fonts. The main remaining variance is script vocabulary: the canvas measurement uses a fixed pangram, so long-word scripts (legal, medical, technical) read slightly slower than displayed and short-word scripts (informal, conversational) slightly faster.

**`letterSpacing` on the canvas context.** `#prose` applies `letter-spacing: .015em` in CSS. Without setting `ctx.letterSpacing` to match before calling `measureText`, the canvas sees words ~4% narrower than the browser actually renders them, overestimates words-per-line by ~4%, and produces a small systematic under-pace (160 WPM displayed ≈ 154 WPM actual). The fix sets `ctx.letterSpacing = cs.letterSpacing` immediately after `ctx.font` when the property is present (Chrome 99+, Firefox 101+, Safari 17.2+); older browsers fall back silently and retain the small bias. This is the only known systematic offset.

The displayed number is a planning estimate, not a guarantee. The engine's internal pixel rate is exact — only the WPM display is approximate.

---

## Countdown

Triggered when `⏱` toggle is on and `START` is pressed. Blocks `Space` key and `START` click during countdown.

SVG ring (`r=52`, circumference `326.7px`) driven by `requestAnimationFrame` over 3000ms:
```js
ring.style.strokeDashoffset = 326.7 * progress;
var n = Math.ceil(3 - progress * 3); if (n < 1) n = 1;
numEl.textContent = n;
```

Ring drains from full to empty. Number counts 3 → 2 → 1; the `n < 1` clamp prevents "0" from flashing on the final frame before the overlay is removed. `doStart()` called on completion. Font: system sans-serif (not Georgia — visually distinct from script text).

---

## Typography

### Defaults

| Setting | Default | Range |
|---|---|---|
| Font family | Georgia | Georgia · Courier New · Arial · Verdana · System |
| Font size | 1.75rem | 1–5rem (slider and typed) |
| Line height | 3.0 | 2.0–5.0 |
| Letter spacing | 0.015em | Fixed |

### Font stacks

| Label | Stack |
|---|---|
| Georgia | `Georgia, 'Times New Roman', serif` |
| Courier | `'Courier New', Courier, monospace` |
| Arial | `Arial, Helvetica, sans-serif` |
| Verdana | `Verdana, Geneva, sans-serif` |
| System | `system-ui, sans-serif` |

All system fonts. No network requests, works offline.

---

## Color schemes

### Presets

| Name | Text | Background |
|---|---|---|
| Default | `#f0ede6` | `#111111` |
| Broadcast | `#00dd00` | `#000000` |
| Amber | `#f5c97a` | `#1a1008` |
| High contrast | `#ffffff` | `#000000` |

Presets shown as `2.5rem × 2.5rem` swatches with `Aa` label. Active preset has `--acc` border.

### Custom colors

Two inputs (Text, BG) each with HEX / RGB tab toggle:
- HEX: accepts `#rrggbb`, `rrggbb`, `#rgb` / `rgb` (3-digit expanded), or 8-digit `#rrggbbaa` (alpha preserved as `rgba(...)` for the cue input round-trip — fg/bg don't use alpha but the parser is uniform)
- RGB: accepts `r, g, b`, `r g b`, or `rgba(r,g,b,a)` (alpha preserved)
- `parseColor()` normalizes 6-digit forms to `#rrggbb` and alpha-bearing forms to `rgba(r,g,b,a)`
- Live preview via color swatch
- Entering a custom color deselects all presets

Applying a preset resyncs both inputs to match current mode (HEX or RGB).

Colors applied to: `#prose` color, `#scroll` background, `body` background.

---

## Cue indicator

An optional reading-position marker overlaid on the scroll area. Off by default. All controls live in the Preferences section of the Settings panel.

### Anchor

The reading-line anchor is configurable via the Settings → Preferences "Reading line" segmented control:

| Setting | Value | Inspiration |
|---|---|---|
| **Broadcast** (default) | 33% | Documented Autocue/QTV standard used by major newsrooms (BBC, CNN, Sky, etc.) |
| **Eye-rest** | 38% | Slightly lower, comfortable for slow / deliberate reads |

Single source of truth: the CSS variable `--reading-anchor` (a unitless number, default `0.33`) and its JS mirror `READING_ANCHOR`. `setReadingAnchor(v)` updates both together. Persisted as `tp_anchor` in localStorage.

Consumers (all use the variable, never a literal):
- `#prose` `padding-top: calc(var(--reading-anchor) * 100vh)` — places the first line at the anchor
- `#cue-line` `top: calc(var(--reading-anchor) * 100%)`
- `#cue-band` and `#cue-left` top set inline by JS: `calc(var(--reading-anchor) * 100% - <halfHeight>px)`
- `#scroll.cue-fade` mask gradient stops at `calc(var(--reading-anchor) * 100%)`
- `skipToNextSpeaker()` reading-Y: `clientHeight * READING_ANCHOR`

When the bar hides, `#scroll`'s `bottom` transitions from `2.75rem` to `0`, and `#cue-overlay` mirrors the same transition so the anchor stays correct.

### DOM

```html
<div id="cue-overlay" aria-hidden="true">
  <div id="cue-band" class="cue-ind"></div>
  <div id="cue-line" class="cue-ind"></div>
  <div id="cue-col">
    <div id="cue-left" class="cue-ind"></div>
  </div>
</div>
```

`#cue-overlay` is `position:fixed`, `pointer-events:none`, `z-index:5`. `#cue-col` mirrors `#prose`'s centered column (same `max-width` scaled by font size) so the Left mark sits in the prose's left padding regardless of viewport width.

### Styles (one active at a time)

All four styles anchor to the configurable reading line, which is `var(--reading-anchor) × 100%` of `#scroll`'s height (default 33%, or 38% in Eye-rest mode — see the *Anchor* section above). The table below uses `A` as shorthand for `var(--reading-anchor) * 100%` to keep the descriptions readable.

| Style | Rendering | Sizing |
|---|---|---|
| `line` | 1px horizontal line across the scroll area at the reading anchor (`A`) | fixed 1px height |
| `band` | semi-transparent horizontal stripe centered on the reading anchor | height = `lineHeightPx × 1.5`, top = `calc(A - height/2)` |
| `left` | small vertical bar in the prose's left padding, centered on the reading anchor | height = `lineHeightPx`, top = `calc(A - lineHeightPx/2)`, width `.1875rem` |
| `fade` | CSS mask on `#scroll` — content above the reading anchor fades to transparent | `linear-gradient(to bottom, transparent 0%, black A)`; no overlay shown |

`band` and `left` dimensions follow `lineHeightPx`, so `applyCue()` is re-called whenever font size or line height changes (slider, click-to-type, reset). The vertical anchor follows `--reading-anchor` automatically — no JS update needed when the user toggles Broadcast / Eye-rest because the inline `top: calc(...)` strings reference the live CSS variable.

### Color — named presets

Five presets, each with its own alpha tuned for legibility against typical broadcast color schemes. Stored as full `rgba(...)` strings.

| Preset | `data-cp` | Value |
|---|---|---|
| Autocue Red | `autocue-red` | `rgba(220,50,50,0.35)` |
| QTV White | `qtv-white` | `rgba(255,255,255,0.20)` |
| Amber | `amber` | `rgba(245,201,122,0.18)` |
| Invisible | `invisible` | `rgba(255,255,255,0.07)` |
| Green | `autocue-green` | `rgba(0,200,80,0.30)` |

Naming note: only **Autocue Red** is grounded in documented Autocue broadcast practice. The other four are house presets — *QTV White* references the QTV manufacturer; *Amber* and *Invisible* are descriptive; *Green* pairs naturally with the Terminal scheme. The `data-cp` for Green stays as `autocue-green` because that key is persisted in `localStorage` (`tp_cue_preset`) and used by `CUE_DEFAULTS_BY_SCHEME` and the Terminal profile — renaming it would silently invalidate every existing user's saved cue setting. Internal id frozen, user-facing label corrected.

Swatches reuse the existing `.cpre` class with an additional `.cue-pre` modifier — neutral dark `#1a1a1a` background with a translucent fill stripe (`.cue-pre-fill`) and a 3-letter label (`.cue-pre-lbl`). The dark backing means the color reads truthfully regardless of the active light/dark theme.

### Scheme-aware default

When the user picks an FG/BG color scheme preset, the cue color auto-pairs:

| Scheme (fg / bg) | Cue preset |
|---|---|
| Default `#f0ede6` / `#111111` | QTV White |
| High Contrast (yellow) `#ffeb3b` / `#000000` | QTV White |
| Terminal `#00dd00` / `#000000` | Green |
| Amber `#f5c97a` / `#1a1008` | Amber |
| Pure white `#ffffff` / `#000000` | QTV White |
| Custom (any other combo) | QTV White (safe neutral) |

Mapped via `CUE_DEFAULTS_BY_SCHEME` keyed `fg|bg`. Auto-pairing only fires when `cueColorCustomized === false`.

### Custom override

Below the preset row, the same `.crow` HEX/RGB input pattern used for FG/BG. The HEX/RGB tab toggle uses the same `.ctabs` class. Editing the input:
- Sets `cueColor` to the parsed value (6-digit hex when no alpha, `rgba(...)` when alpha is present)
- Sets `cuePreset = 'custom'`, `cueColorCustomized = true`
- Deselects all preset swatches
- Does **not** deselect the active FG/BG scheme preset

`cueColorCustomized` pins the user's choice — once set, future scheme switches will not override it. Picking a named preset clears the flag (`cueColorCustomized = false` plus `cuePreset = '<name>'`), so subsequent scheme switches resume auto-pairing.

When `cueColor` is an `rgba(...)` preset string, `syncColorInput()` displays it as 8-digit hex (`#rrggbbaa`) when the HEX tab is active and as `r, g, b, a` when the RGB tab is active — so the input always honours the user's chosen format. `parseColor()` round-trips the 8-digit hex back to `rgba(...)` so alpha survives a typed-and-confirmed edit. Picking a preset still stores the rgba string verbatim in `cueColor`.

### State variables

| Variable | Purpose |
|---|---|
| `cueOn` | Master on/off |
| `cueStyle` | `'line' \| 'band' \| 'left' \| 'fade'` |
| `cueColor` | Active CSS color string (rgba for presets, hex for custom) |
| `cuePreset` | Active preset name or `'custom'` |
| `cueColorCustomized` | If true, scheme switches will not change the cue color |
| `cueMode` | HEX/RGB tab for the custom input |

### Render path — `applyCue()`

Single function that reads `cueOn`, `cueStyle`, `cueColor`, and `lineHeightPx` and updates the DOM. Called on:
- Cue toggle click
- Cue style picker click
- Cue preset click and custom input change
- Scheme switch (via `applyColors` → `setCuePreset`)
- Font size change (`setFontSize`)
- Line height change (slider + click-to-type)
- Reset
- End of `load()` so saved state is visible on first paint

Sets `--cue-color` on `<html>`. The three overlay styles read it via `var(--cue-color)`. Fade style adds `.cue-fade` to `#scroll` and hides the overlay.

---

## Profiles

Named one-click setups that load a complete configuration (font, size, line height, speed, theme, reading-line position, cue indicator). Surfaced inside the Settings → Preferences section as a row of `.ppick` buttons. Existing controls remain editable after — the picker switches to **Custom** the moment any control diverges from the active profile's expected values, so the highlighted pill always reflects truth.

### Inspirations

Each profile is grounded in documented industry conventions, not invented numbers. Where a profile maps to a specific real-world rig or use case, that's noted. Profiles are named by use case rather than vendor (legal hygiene + faster recognition for users).

| Profile | Inspiration | Font | Size | Line | WPM | Theme | Anchor | Cue |
|---|---|---|---|---|---|---|---|---|
| **Newsroom** | Autocue/QTV broadcast standard — BBC, CNN, Sky, the canonical anchor read | Arial | 1.75 | 3.0 | 165 | White on `#111` | 33% | QTV White, line |
| **High Contrast** | Autocue's documented yellow-on-black alternative for difficult readers (per Autocue FAQ) | Arial | 1.75 | 3.0 | 165 | `#ffeb3b` on `#000` | 33% | QTV White, line |
| **Studio Voiceover** | Audiobook / commercial booth pace, generous line height for breathing | Georgia | 2.0 | 3.5 | 140 | White on `#111` | 33% | QTV White, line |
| **Creator** | Podcast / YouTube conversational pace | Georgia | 1.75 | 3.0 | 145 | White on `#111` | 33% | Amber, line |
| **Vlog** | Phone-to-face recording — slightly faster, smaller font for closer distance, energetic | Verdana | 1.5 | 2.8 | 155 | White on `#111` | 33% | Amber, line |
| **Educator** | Classroom / lecture / explainer — slow explanation pace, generous line height for absorption, larger font for distance reading, light theme matches academic display environments, eye-rest anchor for absorption time | Verdana | 2.0 | 3.5 | 135 | Light theme | 38% | QTV White, line |
| **Conference** | Keynote / TED — slow deliberate pace, light theme, eye-rest anchor | Arial | 2.5 | 3.0 | 120 | Light theme | 38% | Off |
| **Speed Read** | Fast scrub through a script for memorization or structure review (was named "Rehearsal" — renamed because actors hear "rehearsal" and expect slow + deliberate, the opposite of what this profile is) | Georgia | 1.5 | 2.8 | 250 | White on `#111` | 33% | QTV White, line |
| **Terminal** | Retro CRT vibe — *not* a broadcast convention, an aesthetic choice. P1 phosphor heritage, IBM 5151 / DEC era | system-ui | 1.75 | 3.0 | 165 | `#00dd00` on `#000` | 33% | Green, line |

WPM column is the user-facing value; the engine speed is derived per-profile from `wpm × fs × lh / (WORDS_PER_LINE_AVG × 21.6)` so each profile delivers its target WPM at its own font/line settings.

### `applyProfile(name)`

Sets every control on the page to the named profile's values. Wraps the body in `_applyingProfile = true` so the auto-detect "switch to Custom" logic doesn't fire mid-application. Calls underlying setters (`setFontSize`, `applyColors`, `setReadingAnchor`, `setCuePreset`) so all derived state (max-width scaling, theme luminance, cue position, persistence) flows through normal channels.

### `refreshProfilePicker()`

Called from every settings-change handler (sliders, font picker, color presets, color inputs, cue toggles, anchor segment). Compares the live state to each profile via `profileMatches(p)` (a per-field equality check with floating-point tolerance) and toggles `.on` on the matching button — falls back to highlighting Custom if nothing matches. Suppressed during `_applyingProfile` and `_loading`. Forced once at the end of `load()` to highlight the right pill on first paint.

### Persistence

The active profile is *not* persisted as a string. Instead, the profile is derived from the underlying state on every paint via `refreshProfilePicker()`. This avoids drift between "claimed profile name" and "actual settings" — saving / restoring the underlying values is sufficient.

---

## Info panel legibility

The Settings panel's body text uses **`var(--bar-text)`** (full-contrast) rather than `var(--bar-dim)` (muted gray). Hierarchy comes from font-size and weight, not color contrast, so the descriptions, science prose, and helper text all read at the same brightness as section labels. Section headers (`.isec h3`, `.sci-body h4`) use `var(--acc)` (a slight accent tint) so they still scan as dividers without dimming.

Elements that *keep* `--bar-dim` deliberately:
- Toolbar slider labels (`.lbl`) and the version label — they're part of the chrome, not the panel
- Inactive tab buttons (`.itab`) — that's the active/inactive hierarchy
- Color row sub-labels (`.clbl`) — small typographic field labels next to inputs
- The ✕ close icon and the per-speaker exclude `✕` — icon-style, dim-on-rest is correct
- Cue preset 3-letter labels (`.cue-pre-lbl`) — kept gray over the dark-neutral swatch backgrounds for color contrast against the translucent stripe

This is documented to prevent future "cleanups" from re-dimming the body text.

---

## Settings panel

Opened by the `Settings` button (`#btn-info`). Closed by `✕`, clicking outside the panel, or pressing `Escape`. `Escape` is handled by the global `keydown` listener: if the info overlay has the `open` class, it removes that class and `preventDefault()`s before any other Esc behavior can run.

### Header

`#info-head` carries `justify-content: space-between`. On the left: a `<span id="info-logo">` containing an "Ultrascript" link to `https://ultrascript.vercel.app` (target `_blank`, `rel="noopener"`) plus an inline `<span id="version-label">` populated on load. The link inherits the panel text color via `color: inherit` so it reads as a wordmark, not as a hyperlinked accent; hover lifts to `var(--acc)`. On the right: the `<button id="btn-close-info">✕</button>` dismiss control. The version label was hoisted from the footer in this revision so the panel surfaces product identity in one place — the footer now just hosts the `heypaul.xyz` credit.

### Tabs

Five tabs (was three): `Guide` (default), `Preferences`, `Script`, `Multi-speaker`, `Science`. The previous combined `Settings` tab was split — Preferences, Script, and Multi-speaker each get their own top-level tab so users land directly on the section they want without scrolling past unrelated controls. `data-tab` values are `guide`, `preferences`, `script`, `multi`, and `science`. Each tab maps to a `<div class="tab-content" id="tab-{data-tab}">`. The tab-switcher JS is unchanged — it composes the target id from `data-tab` generically.

**Guide** (unchanged content; the Countdown row points at *Settings → Preferences* since the toggle moved there)
- Getting started: paste/type, edit on fly, stop editing, auto-saved
- Playback: Start/Pause, Faster/Slower keys, manual scroll, countdown (now in Settings → Preferences)
- Pace reference (WPM)
- Controls: WPM in the bar; Size / Line / Reset in Settings → Preferences

**Preferences** (`id="tab-preferences"`). Order is intentional — Reset to defaults sits at the very top so it's discoverable as a "start over" escape hatch, and Countdown follows it because it's a one-tap switch the user reaches for as part of pre-record setup. Both are session-shape controls; everything below is canvas styling.

1. **Reset to defaults** — `#btn-reset`. Red-bordered button (`border: 1px solid #c0392b`, `color: #c0392b`) — visually signals a destructive action without a confirmation dialog. Restores WPM 160, size 1.75rem, line height 3.0.
2. **Countdown** — labelled row with the small "Countdown" label on the left and the `#btn-cd` pill (label `⏱  Countdown`) on the right. `aria-pressed` mirrors the on/off state. Persists via `tp_cd` (unchanged). Moved from the previous Script section in this revision because Countdown is appearance-of-the-experience rather than script-management — it lives next to the other "how the read feels" controls now.
3. Theme — Auto / Light / Dark segmented control.
4. Profile — 9 named profiles + Custom.
5. Font — 5 options, each rendered in its own typeface.
6. Size — slider + clickable value.
7. Line height — slider + clickable value.
8. Color scheme — 5 preset swatches + custom Text/BG inputs with HEX/RGB toggle.
9. Cue indicator — on/off toggle, style picker, 5 named color presets, custom HEX/RGB override.
10. Reading line — Broadcast (33%) / Eye-rest (38%) segmented control.

**Script** (`id="tab-script"`). One section, two buttons in a row. No Countdown here in v1.1 — that toggle moved to Preferences. The `Detect speakers` button that existed in earlier revisions was removed; speaker detection now runs automatically when Multi is toggled on via the bar, making a separate one-shot button redundant.

- `Load from file…` (`#btn-import-script`) — opens the file picker. Accept attribute `.txt,.md,.json,text/plain,text/markdown,application/json`. See *Load from file* below for format-specific handling.
- `Download script` (`#btn-download-script`) — writes a dated `.txt`.

The snapshot recovery banner (when applicable) still surfaces as a top-of-viewport banner on load, independent of which tab is active.

**Multi-speaker** (`id="tab-multi"`). Same content as before, now its own tab so the bar's Multi button can deep-link straight to it.
- Instructions and format examples for all three speaker label formats (inline colon, standalone colon, bracket)
- Copy AI prompt button (`#btn-copy-prompt`)
- Speaker list (`#dialogue-speakers`) with colour swatch pickers (one row per detected speaker)
- Re-parse button (`#btn-reparse`)
- Empty state (`#dialogue-empty`)

**Science** (unchanged)
- Acceleration curves: time-domain model, asymmetric curve, normalization
- Speed and WPM: context, typical professional ranges
- Typography: line height rationale, font choices and their broadcast history
- How it's built: single HTML file, localStorage, no tracking

### Footer

`heypaul.xyz` — centered, links out. The version label was moved out of the footer to the header in this revision; the footer is now a single-link credit.

---

## Keyboard shortcuts

| Key | Action | Condition |
|---|---|---|
| `Space` | Start / Pause | Not editing text, not during countdown, info panel closed |
| `W` or `↑` | WPM +5 | Not editing text, info panel closed |
| `S` or `↓` | WPM −5 | Not editing text, info panel closed |
| `Tab` or `→` | Skip to next speaker (soft eased scroll) | Multi mode on, not editing text, info panel closed |
| `Esc` | Close info panel | Info overlay is open (priority over the others below) |
| `Esc` | Blur prose (stop editing) | Prose is focused, info panel closed |
| `Enter` | Commit click-to-type value | Val input is focused |
| `Esc` | Cancel click-to-type | Val input is focused (handler stops propagation, so neither prose-blur nor info-close fire) |

When the info panel is open, the global `keydown` handler short-circuits after the Escape branch so panel-internal keyboard behavior (the focus trap's Tab handling, focused-button Space/Enter activation, the hex/rgb input arrow keys) doesn't collide with the global hotkeys. Without this guard, Tab inside the panel would also fire `skipToNextSpeaker()` while `multiOn` is true, and Space on a focused `.cpre` swatch would also toggle START/PAUSE.

`skipToNextSpeaker()` animates the scroll to the next label using a cubic ease-out (`1 − (1 − t)³`) over a duration that scales with distance (220ms minimum for a one-line skip, capped at 420ms for multi-paragraph jumps). During the animation, `isManual = true` suspends the auto-scroll integrator so the loop's velocity contribution can't fight the eased value, and `manualTmr` is cleared so a stale wheel/touch resume timer can't fire mid-skip. On completion: lands exactly on `newOffset`, clears `isManual`, nulls `lastTs` so the loop's `dt` resets cleanly, and sets `lastBoundaryTs = performance.now()` so the acceleration curve starts a fresh line at the new position. A generation counter (`_skipAnimGen`) cancels older animations when a newer skip starts, preventing two RAF chains from writing to `offset` simultaneously. `prefers-reduced-motion: reduce` users keep the previous instant jump.

---

## Persistence (`localStorage`)

All settings saved on every change. Prose text is debounced (250ms trailing edge) so fast typing doesn't trigger a per-keystroke `localStorage.setItem` of the entire script. Keys:

| Key | Type | Default |
|---|---|---|
| `tp_text` | string | `''` |
| `tp_wpm` | number (WPM) | `160` |
| `tp_spd` | (legacy) number | one-time migration only — read on load if `tp_wpm` is absent (converted to WPM at the now-restored fs/lh), then `removeItem`'d on the first save so it can never shadow `tp_wpm` again |
| `tp_fnt` | number (rem) | `1.75` |
| `tp_lh` | number | `3.0` |
| `tp_theme` | `'auto'` \| `'light'` \| `'dark'` | `'auto'` |
| `tp_cd` | `'0'` \| `'1'` | `'0'` |
| `tp_fg` | CSS color string (hex or rgba) | only written when `customColorsActive === true`; removed otherwise. `parseColor` emits `rgba(...)` when the user types a translucent value into the FG input, so `tp_fg` can carry alpha. Luminance-derivation paths route through `colorToRgb` to handle this. |
| `tp_bg` | CSS color string (hex or rgba) | only written when `customColorsActive === true`; removed otherwise. Same rgba caveat as `tp_fg`. |
| `tp_font` | CSS font-family string | `"Georgia, 'Times New Roman', serif"` |
| `tp_cue_on` | `'0'` \| `'1'` | `'0'` |
| `tp_cue_style` | `'line'` \| `'band'` \| `'left'` \| `'fade'` | `'line'` |
| `tp_cue_color` | CSS color string (hex or rgba) | `'rgba(255,255,255,0.20)'` |
| `tp_cue_preset` | preset name or `'custom'` | `'qtv-white'` |
| `tp_cue_color_custom` | `'0'` \| `'1'` | `'0'` |
| `tp_multi` | `'0'` \| `'1'` | `'0'` |
| `tp_compact_speakers` | `'0'` \| `'1'` | `'1'` (default on) |
| `tp_dialogue` | JSON array of `{name, key, color}` | `'[]'` |
| `tp_anchor` | number (`0.33` \| `0.38`, clamped to `[0.1, 0.6]` on read) | `0.33` (broadcast standard) |
| `tp_text_snap` | string | last snapshot of the prose text — written every 5 minutes when `length ≥ 50` and content has changed; survives a corrupted `tp_text` write. See *Resilience*. |
| `tp_text_snap_ts` | number (`Date.now()` ms) | timestamp of the last snapshot write — used to format the recovery banner's "X min ago" label and to track which snapshot was dismissed. |
| `tp_text_snap_dismissed_ts` | string (matches `tp_text_snap_ts`) | timestamp of the snapshot the user dismissed via the recovery banner — suppresses the banner until a newer snapshot is written. |

All `localStorage` calls wrapped in `try/catch`. On `setItem` failure (private mode quota, storage disabled, etc.) the catch calls `notifySaveFailure()`, which debounces the toast with a 2-second window: a burst of keystroke-driven failures coalesces into one toast (`"Couldn't save — your script may not survive a reload."`, auto-dismisses after 5 seconds), but if saving keeps failing in subsequent windows the toast re-surfaces — saves never go permanently silent after the first occurrence. Existing on-screen save toasts are removed before a new one is rendered, so re-surfacing doesn't stack DOM nodes.

`save()` is suppressed while `load()` is restoring state (via a `_loading` flag). Without this guard, the in-line `setTheme()` and `applyColors()` calls during load would call `save()` before cd / font / cue state had been restored to globals, overwriting the user's saved values with init defaults.

`tp_fg` and `tp_bg` are only written to `localStorage` when `customColorsActive === true`. When a theme button is selected (or on a default-color session), those keys are actively removed. This prevents a false "color override" note from appearing on reload when no custom scheme was ever set.

On load: WPM slider clamped to `Math.min(wpm, 300)` since the click-to-type allows values above the slider's visual max (40–500).

---

## Resilience

Six layered guards on top of the primary `tp_text` save, each closing a specific failure mode that `tp_text` alone doesn't cover. All offline, no network, no third-party APIs.

### 1. Snapshot slot

Every `SNAPSHOT_INTERVAL_MS` (5 minutes), `writeSnapshot()` writes the current prose to `tp_text_snap` along with a `tp_text_snap_ts` timestamp — but only when (a) the text has actually changed since the last snapshot (cheap memo via `_lastSnapshotText`), (b) the text is at least 50 chars (don't shadow real content with a transient empty state), and (c) `_loading === false`. Failures are caught and silently swallowed: the primary `notifySaveFailure()` toast already covers quota signals, no need to double-report.

`checkSnapshotRecovery()` runs at the very end of `load()`. If `tp_text` came back empty/short (`length < 50`) and the snapshot is substantial (`length ≥ 100`) and the user hasn't already dismissed this snapshot (`tp_text_snap_dismissed_ts !== tp_text_snap_ts`), it surfaces a non-blocking banner at the top of the viewport: *"Snapshot from {ago} is longer than your current script."* with **Recover** and **Dismiss** buttons. Recover writes the snapshot back to `#prose`, invalidates layout, calls `save()`, and re-runs the multi pipeline if `multiOn`. Dismiss writes the current snapshot timestamp to `tp_text_snap_dismissed_ts` so the banner doesn't re-fire on every reload — but a fresh snapshot (different timestamp) re-prompts.

### 2. visibilitychange flush

`document.addEventListener('visibilitychange', …)` calls `flushPendingSave()` when `visibilityState === 'hidden'`. This catches mobile backgrounding (the most common save-loss path on phones) and tab-switching. The handler clears the pending `_proseSaveTimer` and calls `save()` immediately.

### 3. pagehide flush

`window.addEventListener('pagehide', flushPendingSave)`. Final save attempt before the page is unloaded — definitive close path. `pagehide` is more reliable than `unload` on modern browsers and is the recommended location for last-write-out.

### 4. beforeunload prompt

`window.addEventListener('beforeunload', …)` checks if `_proseSaveTimer` is pending and, if so, sets `e.returnValue = ''` to trigger the browser's native "Leave site?" dialog. The user can cancel the close. Belt-and-braces with the pagehide flush — the save will happen anyway, but this gives the user an explicit chance to abort.

This relies on `_proseSaveTimer` being a faithful "pending save" indicator: `saveProseSoon()` sets it, and the timeout's own callback nulls it back to `null` before calling `save()` (in addition to `flushPendingSave` clearing it on the manual flush path). Without that null-on-fire, the variable would hold a stale handle after the first edit and `beforeunload` would prompt on every page close, not just when there's actually a pending save.

### 5. Download script

`#btn-download-script` (in the Settings → Script tab) builds a `Blob` from `prose.innerText`, generates an object URL, programmatically clicks an `<a download>` to trigger the file save, and revokes the URL after a 1-second delay. Filename includes the date: `ultrascript-YYYY-MM-DD.txt`. Button label changes to "Downloaded!" for 1.8s, then reverts. Wrapped in try/catch so a Blob failure (very rare) flips the button to "Failed" instead of throwing.

### 6. Load from file

`#btn-import-script` (in the Settings → Script tab) opens the system file picker via a hidden `<input type="file" accept=".txt,.md,.json,text/plain,text/markdown,application/json">`. No confirmation step — the OS file picker itself (navigate + click Open) is the confirmation. A single click stops any active playback via `pauseOrCancel()` then opens the picker immediately.

**Format handling** (v1.1):

| Extension | Behavior |
|---|---|
| `.txt` | Loaded as plain text verbatim. |
| `.md` | Loaded as plain text verbatim. Markdown syntax is **not** rendered — the prompter doesn't parse Markdown, so headings / emphasis appear in the read. Users exporting from Notion / Bear / Obsidian / etc. expect this. |
| `.json` | Treated as a wrapper. If the parsed value is an object with a `script` or `text` key (string), that value is used. If it's an array of strings, the entries are joined with `\n\n`. Anything else is `JSON.stringify`-pretty-printed. Malformed JSON falls through to plain-text load — better than failing outright, the user can fix the file and reload. |

Format is detected from the filename extension (case-insensitive); content sniffing was rejected because hand-typed strings starting with `{` would otherwise be mis-classified.

After format normalization, the picker's `change` handler writes the result as a single text node into `#prose`, calls `normalizeProse()` to convert to `<div>` blocks, then resets the scroll state (`offset = 0`, `scrollEl.scrollTop = 0`), blurs `#prose`, and clears the document selection (`window.getSelection().removeAllRanges()`). The scroll reset ensures the imported script starts at the reading anchor; the blur + `removeAllRanges()` pair removes the contenteditable caret entirely — Chrome preserves an "inactive" caret in the element even after it loses focus, so both steps are needed. Then calls `save()` and re-runs the multi pipeline if `multiOn`. The input's `value` is reset to `''` after each pick so re-selecting the same file re-fires `change`.

---

## Timer

Counts up from `0:00` in `mm:ss` format using `setInterval` at 1-second intervals. Only increments when `running === true`. **Resets to `0:00` on pause** — the pause branch of the `#btn-go` click handler zeroes `secs`, calls `updTmr()`, and hides `#tmr` (`style.display = 'none'`). `doStart()` does not touch the timer; it just unhides `#tmr`. The model is: every read starts at 0:00, runs until pause, then the prompter returns to a clean rest state. There is no resume-from-mid-read path because pause is always a full stop.

`font-variant-numeric: tabular-nums` for stable width.

---

## Accessibility

- `#bar` has `role="toolbar" aria-label="Ultrascript controls"`.
- Icon-only and text-only buttons (`⏱`, `✕`, `System`, `Multi`) carry both `title` and `aria-label`.
- `.cpre` color and cue-color preset swatches are `<div>`s for styling reasons but are promoted at runtime to `tabindex="0"` `role="button"` and respond to Enter/Space.
- The Settings panel (`#info-panel`) implements a keyboard focus trap while open: `Tab` and `Shift+Tab` cycle through focusable elements inside the panel only. Implemented via a `keydown` listener added to the panel on open and removed on close (`_trapHandler` pattern).
- Focus restoration on close is **modality-conditional**: `closeInfoPanel(viaKeyboard)` accepts a boolean. Keyboard close (`Esc`) passes `true` and restores focus to `#btn-info` so screen-reader / keyboard-only users keep a sane landing point. Mouse close (`✕` click, overlay click) passes nothing and instead blurs whatever's currently focused inside the panel — restoring focus to a bar button after a mouse close lights up its `:focus-visible` ring on the next keyboard event and reads as a "stuck" selection to mouse users mixing modalities.
- The Multi-button auto-open path routes through `openInfoPanel()` rather than mutating the `.open` class directly, so the focus trap and initial-focus pass run regardless of how the panel was opened.
- `#tmr` carries `role="timer"` and `title="Elapsed time"`.
- A `VERSION` constant (`'1.1.0'`) is defined at the top of the script and written into `#version-label` in the panel footer on load. Update the constant when the version changes — it's the single source of truth.
- A `:focus-visible` outline (2px `var(--acc)`, 2px offset) covers every focusable control — `border:none` strips the UA default, so this brings keyboard focus back into view without affecting mouse clicks.
- `prefers-reduced-motion: reduce` disables transitions on `#bar`, `#scroll`, `#cue-overlay`, `#info-overlay`, and `#info-panel`. The acceleration curve itself is the product, so it's intentionally unaffected.

---

## Known limitations

- **WPM is approximate** — the engine now drives off WPM directly (no longer a unitless multiplier), and as of 1.0.1 the words-per-line factor is measured live from the rendered prose box and current font (`getWordsPerLine()`) rather than a fixed constant, so the displayed WPM is now accurate across desktop / tablet / phone-portrait / monospace fonts. The remaining variance comes from script vocabulary — the measurement assumes average English word length, so long-word scripts (legal, medical, technical) read slightly slower than displayed and short-word conversational scripts slightly faster. ±~5% variance is typical. The displayed WPM is a planning estimate; the engine's pixel rate is exact.
- **DOM scroll rendering** — `scrollTop` integer steps, not sub-pixel (addressed in Next.js version with canvas)
- **Single script** — no script library / multi-document workspace; one active script at a time. (File import / export *is* supported via the Guide tab's *Backup* section, so cross-device sync via any file-sync mechanism the user already has is covered.)
- **Mobile: virtual keyboard resize not handled** — when the virtual keyboard opens, the fixed viewport is not adjusted; this is the one remaining mobile gap
- **Dialogue: `However:`-style false positives** — a single Title-Case word followed by a colon and content (e.g. `However: this is fine`) will be treated as a speaker label. Accepted edge case for v1.
- **`normalizeProse()` is a no-op when block elements already exist** — covers Firefox's `<p>` insertion on Enter, content pasted from sources that produce `<p>` blocks, and any other already-structured prose. Flat text + `<br>` normalization (the Chrome contenteditable shape) only runs on genuinely unstructured prose. `parseDialogue()` and `renderDialogue()` walk both `<p>` and `<div>`; the `.sp-label` unwrap path is tag-agnostic, so both shapes are first-class.

---

## File stats

| | |
|---|---|
| Total lines | ~3860 |
| CSS | ~330 lines |
| HTML | ~340 lines |
| JavaScript | ~2580 lines |
| Dependencies | None |
| Network requests | None |
| File size | ~193KB unminified |
