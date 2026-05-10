# Ultrascript

**v1.1.0**

A professional single-file teleprompter that runs entirely in your browser. No server, no install, no accounts.

Open the HTML file, paste your script, hit START.

**[heypaul.xyz](https://heypaul.xyz)**

---

## Why

Most teleprompter apps are either native binaries with subscriptions or web apps that send your script to someone else's server. This is neither. It's a single HTML file (~193 KB) you can fork, host on a USB stick, or open directly from disk. Your script and settings stay on your device.

The reading surface is also the editing surface — there is no setup screen, no modes to switch between, no launch step. You paste or type directly into the text, adjust settings in the **Settings** panel (bottom-right of the bar), and hit START on the bar itself.

---

## Quick start

1. Download `ultrascript.html`.
2. Double-click it, or open in any modern browser.
3. Click anywhere on the screen and paste your script.
4. Hit `START`.

That's it.

---

## Features

- **One-click profiles.** Nine named setups grounded in industry conventions — **Newsroom** (Autocue/QTV broadcast), **High Contrast** (Autocue's documented yellow-on-black for difficult readers), **Studio Voiceover**, **Creator** (podcast/YouTube), **Vlog** (phone-to-face recording), **Educator** (lecture/explainer pace), **Conference** (keynote/TED), **Speed Read** (fast scrub for memorization), and **Terminal** (a CRT-phosphor aesthetic). Pick one to load font, size, line height, WPM, theme, reading-line position, and cue indicator together. The picker switches to **Custom** the moment you tweak anything.
- **Lean control bar.** WPM lives on the bar; everything else (Size, Line, Theme, Color, Cue, Reading line, Reset, Countdown, Multi-speaker tools, file load/save) is one click away in **Settings**. The bar carries WPM, START, Multi, and Settings — and an elapsed-time readout that appears next to START during playback and tucks away at rest. The wordmark in the top-left fades out the moment you start reading, so the surface is uncluttered while you're recording.
- **Time-domain acceleration curve.** Asymmetric piecewise smoothstep that mimics how the eye actually reads — fast ease-in (40%), flat peak at full speed (25%), slower ease-out (35%) with a built-in micro-pause at line end. Calibrated so the average velocity equals exactly the speed you set.
- **Reading line at 33%.** Default sits at the documented Autocue/QTV broadcast standard — used by BBC, CNN, Sky, and most major newsrooms. Toggle to 38% Eye-rest in Settings → Preferences for slow / deliberate reads. The first word lands on the line, no warm-up scroll.
- **Click-to-type values.** Click any number to type an exact value. WPM (in the bar) accepts 40–500 (slider visually pegs at 300; click-to-type runs the full range). Font size and line height (in Settings → Preferences) also editable by typing.
- **WPM as the primary unit.** The bar reads in words-per-minute — the universal teleprompter unit. Bump font size or line height, switch to a monospace font, rotate from desktop to phone portrait — the displayed WPM stays put. The engine measures actual words-per-line live from the rendered prose box and recomputes its underlying rate so the read pace remains accurate to within ±~5% across every device and font.
- **Manual scroll resume.** Two-finger scroll any time; auto-scroll picks back up at the correct sub-line phase after ~1 second of inactivity, no ease-in restart.
- **Soft skip-to-next-speaker.** `Tab` or `→` eases the prompt onto the next speaker label with a cubic ease-out (220–420ms scaled by distance) instead of snapping. Honors `prefers-reduced-motion`.
- **Mobile and tablet support.** Touch scroll with auto-resume, larger tap targets, and a bottom-sheet Settings panel on narrow screens.
- **Multi-speaker mode.** Click Multi in the bar to parse speaker labels and assign each a distinct colour. **Compact spacing** (default on) collapses blank lines between speaker turns so raw scripts look tight without reformatting — toggle off in the Multi-speaker tab if blank lines are intentional pauses. Supports three formats: `NAME: text` (inline colon), `NAME:` (standalone label), and `[Name] text` (bracket). **Multilingual** — Greek (`ΑΛΕΞ:`), Cyrillic (`АЛЕКС:`), Arabic, Hebrew, CJK, Hangul, and other non-Latin scripts parse the same way Latin scripts do. Mid-paragraph speakers are detected too — pasted scripts that pack multiple turns into a single paragraph are split automatically. ✕ a speaker row to exclude false positives. Colour customizations survive Re-parse, Multi off→on toggle, and reload. Colours survive the browser's contenteditable sanitizer. Copy AI prompt button generates a reformatting instruction (with explicit "no two speakers in the same paragraph" guidance) you can paste into any AI. Toggle off to strip colours non-destructively.
- **Cue indicator.** Optional reading-position marker — line, band, left-bar, or fade styles. Color auto-pairs with the active scheme.
- **Live editing.** Click into the text mid-scroll to fix a word.
- **Auto-saved + recoverable.** Script + every setting persisted to `localStorage`, debounced so long scripts don't stutter. A second snapshot slot is written every 5 minutes — if the primary save ever comes back blank on reload, a *Recover snapshot from N min ago?* banner offers it back. Pending edits are flushed when you close or background the tab. **Load from file** in Settings → Script accepts `.txt`, `.md`, and `.json` (script keys, text keys, or arrays of strings); **Download script** writes a dated `.txt` for off-device backup.
- **Auto-hiding bar.** Bar fades out 3 s after START, wakes on any input.
- **Themes.** Auto / Light / Dark in Settings → Preferences, with custom HEX or RGB overrides.
- **Countdown.** Optional 3-2-1 ring before scroll starts. Toggle in Settings → Preferences.
- **Accessible.** Toolbar role on the control bar, ARIA labels on all icon-only buttons, `aria-pressed` on the Countdown toggle, keyboard-navigable color swatches, `:focus-visible` outlines, `prefers-reduced-motion` respect.

---

## Keyboard shortcuts

| Key | Action |
|---|---|
| `Space` | Start / Pause |
| `W` or `↑` | WPM +5 |
| `S` or `↓` | WPM −5 |
| `Tab` or `→` | Skip to next speaker (Multi mode) |
| `Esc` | Close info panel → blur prose → cancel click-to-type (in that priority order) |
| `Enter` | Commit click-to-type value |

---

## Defaults

| Setting | Default | Notes |
|---|---|---|
| WPM | 160 | Centre of the news-anchor range (150–175) |
| Font | Georgia, 1.75rem | Designed for screen legibility |
| Line height | 3.0 | Generous by web standards, correct for teleprompter use |
| Theme | Auto | Follows OS preference |
| Reading line | 33% (Broadcast) | Autocue/QTV standard. Switch to 38% (Eye-rest) for slow reads |
| Cue indicator | Off | Opt-in, lives in Settings → Preferences |
| Countdown | Off | Toggle in Settings → Preferences |

Pace reference:

| WPM | Use case |
|---|---|
| 100–130 | Conference / keynote — slow, deliberate |
| 130–150 | Podcast / creator — conversational |
| **150–175** | **News anchor / monologue (default 160)** |
| 175+ | Fast read — brisk, narrative, sermon-pace |
| 250+ | Speed Read — scrub for memorization or structure review |

---

## Tech

- Single HTML file, ~193 KB unminified.
- Vanilla ES5-compatible JS, `'use strict'`. No build step, no dependencies, no network requests.
- System fonts only — works fully offline.
- Persists to `localStorage`. Nothing leaves your device.
- Compatible with mobile browsers (Chrome/Safari on iOS and Android); touch scroll supported.
- Latest two versions of Chrome / Safari / Firefox / Edge on desktop and mobile.

---

## Repo layout

```
ultrascript/
├── ultrascript.html              The whole app
├── ultrascript-html-spec.md      Technical spec — architecture, math, state model, invariants
└── README.md                     This file
```

---

## Customizing

The relevant places to edit:

- **Acceleration curve** — `bezierMultiplier(t)` in the `<script>` block. Constants are picked so the curve is C0-continuous and the time-weighted average equals exactly 1.0; if you change them, re-derive both.
- **Calibration** — `SPEED_PX_PER_MS_UNIT` (px per ms per engine speed unit) is the global pace knob; bumping it makes every WPM read faster across the board. `getWordsPerLine()` measures actual word capacity live from the rendered prose box and current font, so the WPM↔speed conversion stays accurate across viewport sizes and font choices automatically. `WORDS_PER_LINE_AVG = 6.4` is now the early-load fallback only.
- **Defaults** — `var wpm = 160`, the HTML slider `value="160"`, and the `setWPM(160)` calls in the load fresh-install branch and the Reset handler. Font / line defaults follow the same pattern. See the spec's "Defaults must agree across all sources of truth" note.
- **Color presets** — the `.cpre` swatches in Settings → Preferences.
- **CSS variables** — `:root` and `html[data-theme="light"]` blocks.

See [`ultrascript-html-spec.md`](./ultrascript-html-spec.md) for the full architecture and the invariants you need to preserve when editing.

---

## Known limitations

- WPM is approximate. The engine measures actual words-per-line live from the rendered prose box and current font, so accuracy holds across desktop / tablet / phone-portrait / monospace. Remaining variance comes from script vocabulary — long-word scripts (legal, medical, technical) read slightly slower than displayed; short-word conversational scripts slightly faster. Typically ±~5%. The displayed WPM is a planning estimate; the engine's pixel rate is exact.
- Single script — no script library / multi-document workspace. (File import and export are supported via Settings → Script, so cross-device sync via any file-sync mechanism is covered.)
- Mobile: virtual keyboard resize is not handled — this is the one remaining mobile gap.
- Timer resets on pause (not on START) — every read starts from 0:00.
- Dialogue: single Title-Case words like `However:` can be detected as speaker labels by the parser. The Copy AI prompt button now includes an explicit rule (#7) telling the AI to neutralise these at source-format time by replacing the colon with an em-dash, which avoids the false-positive without changing the parser's grammar.

---

## License

MIT.

---

Built by [heypaul.xyz](https://heypaul.xyz).
