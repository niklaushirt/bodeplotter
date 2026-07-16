# Nick's Guitar Pickup Bode Plotter — Full Implementation Spec (Latest9999)

This document is self-contained: it describes the complete current state of the application so it can be re-implemented from scratch. It supersedes all previous briefs (baseline spec, LATEST100 layout changes, professional restyling, and all usability enhancements).

Implement the app as one self-contained `index.html` with inline HTML, CSS, and JavaScript. Use CDN scripts only for:

- Chart.js `https://cdn.jsdelivr.net/npm/chart.js@4.4.7/dist/chart.umd.min.js`
- Lucide icons `https://unpkg.com/lucide@latest/dist/umd/lucide.min.js`

No build step. No Tailwind. Must run from localhost or a secure context so browser audio permissions work.

**Assets:**

- `logo.png` next to `index.html` (source: `/Users/nhirt/Documents/NICI_GUITAR_LOGO.png`) — 779×501 RGBA PNG, transparent background, Keith-Haring-style guitar illustration in black.
- `app-icon-1024.png` — 1024×1024 RGB master icon with the simplified guitar mark on a charcoal-and-orange background.
- `apple-touch-icon.png` — 180×180 iPhone home-screen icon.
- `favicon-32x32.png` and `favicon-16x16.png` — browser favicon sizes.

The document head links both favicon PNGs, the 180px Apple touch icon, sets `theme-color` to `#0f1117`, and uses `Bode Plotter` as the Apple mobile web-app title.

---

## Purpose

Dark-themed (with light/dark toggle) browser tool for rough guitar pickup frequency-response plotting. It generates a sine sweep through a selected soundcard output, measures a selected soundcard input, corrects measurements with a learned noise floor, optionally uses sine-only lock-in filtering (hidden control), displays a smoothed Bode-style response graph with peak markers, manages six saved measurement slots, and persists all state in localStorage.

---

## Theme And Colors

CSS variables on `:root` (dark defaults):

```css
--accent:   #004cfc;
--accent-2: #ff9c00;
--bg:       #0f1117;
--surface:  #15181f;
--surface2: #1e2230;
--border:   #2a2f3f;
--text:     #d4d8e8;
--text-dim: #7a8099;
--red:      #ff3b4f;
--green:    #35d0a2;
/* professional-polish additions */
--shadow:     0 1px 2px rgba(0,0,0,0.35), 0 6px 20px rgba(0,0,0,0.22);
--shadow-sm:  0 1px 3px rgba(0,0,0,0.3);
--focus-ring: 0 0 0 3px rgba(0,76,252,0.28);
```

`body.light` overrides:

```css
--bg: #eef0f5;  --surface: #ffffff;  --surface2: #e2e5ee;
--border: #c8cdde;  --text: #1a1d2e;  --text-dim: #5a6080;
--shadow:    0 1px 2px rgba(20,25,60,0.06), 0 6px 20px rgba(20,25,60,0.08);
--shadow-sm: 0 1px 3px rgba(20,25,60,0.08);
```

`--accent` and `--accent-2` are identical in both modes.

Base styles: `* { box-sizing: border-box }`; body uses the system font stack (`-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif`), `-webkit-font-smoothing: antialiased`, `text-rendering: optimizeLegibility`. `::selection { background: rgba(0,76,252,0.35) }`. Custom webkit scrollbars: 10px, transparent track, `var(--surface2)` thumb (radius 5px, 2px `var(--bg)` border), `var(--border)` on hover.

---

## Welcome / Audio Permission Splash

On every page load, show a full-viewport welcome screen above the application (`z-index: 300`). It is a startup gate rather than a dismissible marketing modal: the user enters the workspace by enabling audio.

- Use the existing `logo.png` inside a circular orange-accented frame.
- Kicker: `PICKUP RESPONSE LAB`
- Heading: `Welcome to the Bode Plotter`
- Explain that enabling audio connects the soundcard and that audio stays on the device.
- Primary orange CTA: `Enable audio & enter`, wired to the same `enableAudio()` function as the header button.
- While permission is pending, disable both enable buttons and show `Connecting audio…` plus a live status message.
- On success, show a short `Audio ready` state, fade the splash away, then focus **Test Signal**.
- On failure, keep the splash visible, provide a permission-specific message when access was denied, and change the CTA to `Try enabling audio again`.
- Respect `prefers-reduced-motion` and collapse spacing/logo sizing on phones.

The splash uses orange accents (`--accent-2`) over the dark app background, with a thin orange top highlight and subtle radial glows. Keep the application shell inert and hidden from assistive technology until audio is ready. The header's **Enable Audio** button remains in place and becomes the disabled `Audio enabled` confirmation after entry.

---

## Header

Browser title and H1: `Nick's Guitar Pickup Bode Plotter`

```css
header { padding: 18px 16px 14px 16px; }
.header { position: relative; }           /* <header class="header"> */
header h1 { color: var(--accent-2); font-size: 36px; font-weight: 700; letter-spacing: -0.02em; margin: 0 0 4px 0; }
.subtitle { font-size: 12px; color: var(--text-dim); margin: 0 0 14px 0; letter-spacing: 0.01em; }
.header-btns { display: flex; gap: 7px; flex-wrap: wrap; align-items: center; }
.header-logo {
  position: absolute; top: 10px; right: 16px; height: 90px; width: auto;
  filter: invert(1); opacity: 0.85; pointer-events: none; user-select: none;
}
body.light .header-logo { filter: none; opacity: 0.9; }
```

First child inside `.header`: `<img class="header-logo" src="logo.png" alt="Logo">`

Subtitle text:
> Sweep a compensated sine signal through a pickup test rig, measure the soundcard input, and plot the normalized frequency response.

Header buttons row (in order):

| # | ID | Label | Style | Icon |
|---|---|---|---|---|
| 1 | `enableAudioBtn` | `Enable Audio` | `.btn-primary` | `power` |
| 2 | `testBtn` | `Test Signal` | `.btn`, disabled until audio enabled | `audio-lines` |
| 3 | `sweepBtn` | `START SWEEP` | `.btn-orange`, disabled until audio enabled | `chart-spline` |
| 4 | `stopBtn` | *(icon only)* | `.btn-stop`, disabled unless busy, `title="Stop"` | `square` |
| 5 | `exportBtn` | `Export PNG` | `.btn` | `download` |
| 6 | `exportCsvBtn` | `Export CSV` | `.btn` | `file-spreadsheet` |
| 7 | `clearBtn` | `Clear Latest` | `.btn` | `eraser` |
| 8 | `helpBtn` | `Help` | `.btn` | `circle-help` |
| 9 | `themeBtn` | *(icon only)* | `.btn`, `margin-left: auto`, `title="Toggle light/dark mode"` | `sun` (dark) / `moon` (light) |

When audio is enabled: button 1 text becomes `Audio enabled` and stays disabled; buttons 2 and 3 become enabled.

Stop button: disabled by default; gets class `.sweeping` (red border + text) only while a sweep runs; enabled but not red during Test Signal.

### Button styles

```css
.btn {
  display: inline-flex; align-items: center; justify-content: center; gap: 7px;
  background: var(--surface2); border: 1px solid var(--border); color: var(--text);
  font-size: 12px; font-weight: 600; letter-spacing: 0.01em;
  border-radius: 6px; padding: 0 14px; height: 32px; cursor: pointer;
  transition: filter .15s, border-color .15s, box-shadow .15s, transform .06s, opacity .15s;
  box-shadow: var(--shadow-sm);
}
.btn:hover:not(:disabled) { filter: brightness(1.12); border-color: var(--text-dim); }
.btn:active:not(:disabled) { transform: translateY(1px); filter: brightness(0.96); }
.btn:focus-visible { outline: none; box-shadow: var(--focus-ring); }
.btn:disabled { opacity: 0.4; cursor: not-allowed; box-shadow: none; }
.btn svg { width: 15px; height: 15px; }
.btn-primary { background: linear-gradient(180deg,#1f8f57,#17703f); border-color: #1a7a4a; color: #fff;
               box-shadow: inset 0 1px 0 rgba(255,255,255,.15), var(--shadow-sm); }
.btn-orange  { background: linear-gradient(180deg,#ffab2e,#f79300); border-color: #e68d00; color: #000;
               letter-spacing: .03em; box-shadow: inset 0 1px 0 rgba(255,255,255,.3), var(--shadow-sm); }
.btn-stop.sweeping { border-color: var(--red); color: var(--red); box-shadow: 0 0 0 3px rgba(255,59,79,.15); }
.btn-sm { padding: 0 10px; height: 24px; font-size: 11px; border-radius: 5px; background: var(--surface); box-shadow: none; }
.btn-sm.confirm { background: var(--red); border-color: var(--red); color: #fff; }
```

---

## Meters Bar (full-width strip between header and main grid)

Standalone horizontal strip, NOT inside `.main-grid`. `position: relative` (hosts the sweep progress bar).

```css
.meters-bar {
  position: relative; display: flex; align-items: stretch; gap: 0; padding: 10px 16px;
  background: var(--surface); border-top: 1px solid var(--border);
  border-bottom: 1px solid var(--border); box-shadow: var(--shadow-sm);
}
.meters-bar .meter-block, .meters-bar .status-block,
.meters-bar .status-bar, .meters-bar .peak-block { flex: 1; margin-bottom: 0; padding: 2px 16px; }
.meters-bar > div:first-child { padding-left: 0; }
.meters-bar > div + div { border-left: 1px solid var(--border); }   /* vertical separators */
.meters-bar .status-bar { display: flex; align-items: center; min-width: 120px; font-size: 15px; border-top: none; padding-top: 0; margin-bottom: 0; }
.meters-bar .peak-block { min-width: 120px; border-top: none; padding-top: 0; }
.meters-bar .meter-label, .meters-bar .status-label, .meters-bar .peak-label { font-size: 12px; letter-spacing: 0.08em; }
.meters-bar .meter-value, .meters-bar .status-value { font-size: 22px; line-height: 1.1; }
.meters-bar .peak-value { font-size: 22px; line-height: 1.1; }
.meters-bar #peakResponse { font-size: 18px !important; }
```

HTML (order matters):

```html
<div class="meters-bar">
  <div class="meter-block">
    <div class="meter-label">Input</div>
    <div class="meter-value" id="meterInVal">-- dBFS</div>
    <div class="meter-bar-bg"><div class="meter-bar" id="meterInBar"></div></div>
  </div>
  <div class="meter-block">
    <div class="meter-label">Output</div>
    <div class="meter-value" id="meterOutVal">-- dBFS</div>
    <div class="meter-bar-bg"><div class="meter-bar" id="meterOutBar"></div></div>
  </div>
  <div class="status-block">
    <div class="status-label">Current freq</div>
    <div class="status-value" id="currentFreq">-- Hz</div>
  </div>
  <div class="status-block">
    <div class="status-label">Progress</div>
    <div class="status-value" id="progress">0%</div>
  </div>
  <div class="status-bar" id="statusBar">Audio not enabled</div>
  <div class="peak-block">
    <div class="peak-label">Peak Response</div>
    <div style="text-align:right;">
      <div class="peak-value" id="peakFreq"></div>
      <div style="font-size:13px;font-weight:600;color:var(--accent-2);font-variant-numeric:tabular-nums;" id="peakResponse">--</div>
    </div>
  </div>
  <div id="sweepProgressBar"></div>
</div>
```

Component styles:

```css
.meter-label { font-size: 11px; color: var(--text-dim); margin-bottom: 2px; }
.meter-value { font-size: 15px; font-variant-numeric: tabular-nums; margin-bottom: 4px; }
.meter-bar-bg { height: 5px; background: var(--surface2); border-radius: 3px; overflow: hidden; box-shadow: inset 0 1px 2px rgba(0,0,0,.25); }
.meter-bar { height: 100%; width: 0%; background: linear-gradient(90deg,#27a37e,var(--green)); border-radius: 3px; transition: width .09s linear; }
.meter-bar.hot { background: linear-gradient(90deg,#ff6b3d,var(--red)); }
.status-label { font-size: 11px; color: var(--text-dim); margin-bottom: 2px; }
.status-value { font-size: 14px; font-variant-numeric: tabular-nums; }
.status-value.red { color: var(--red); }
.status-bar .orange { color: var(--accent-2); } .status-bar .red { color: var(--red); } .status-bar .green { color: var(--green); }
.peak-label { font-size: 12px; color: var(--text-dim); }
.peak-value { font-size: 17px; font-weight: 800; color: var(--accent-2); }
#sweepProgressBar {
  position: absolute; left: 0; bottom: 0; height: 3px; width: 0%;
  background: linear-gradient(90deg, var(--accent), #4ea1ff);
  opacity: 0; transition: width .15s linear, opacity .4s; pointer-events: none;
}
```

Meter bars map −60 dBFS → 0% to 0 dBFS → 100%; bar turns red (`.hot`) when level > −3 dBFS. Meter loop runs on a 100 ms `setInterval` reading the input analyser (Float32 time-domain RMS → dBFS) and, while an oscillator runs, the output analyser; otherwise output shows `-- dBFS` / 0%.

---

## Layout — plot on TOP, settings row BELOW

```css
/* Desktop ≥940px */
.main-grid {
  display: grid;
  grid-template-columns: 280px 330px 1fr;
  grid-template-areas:
    "graph graph graph"
    "col1 col2 slots";
  padding: 12px; gap: 10px; align-items: start;
}
/* Tablet ≤939px */
@media (max-width: 939px) {
  .main-grid { grid-template-columns: 280px 1fr;
    grid-template-areas: "graph graph" "col1 col2" "slots slots"; }
}
/* Phone ≤639px */
@media (max-width: 639px) {
  .main-grid { grid-template-columns: 1fr;
    grid-template-areas: "graph" "col1" "col2" "slots"; }
}
.col-left     { grid-area: col1; display: flex; flex-direction: column; gap: 10px; }  /* Audio Routing ONLY */
.col-settings { grid-area: col2; display: flex; flex-direction: column; gap: 10px; }  /* Sweep + Graph Settings */
.col-graph    { grid-area: graph; }
.col-slots    { grid-area: slots; }
```

### Chart area — fixed height + drag-resize handle

```css
.chart-wrap { position: relative; height: 364px; }
@media (max-width: 639px) { .chart-wrap { height: 265px; } }
.chart-wrap canvas { width: 100% !important; height: 100% !important; }

.chart-resize-handle {
  height: 7px; cursor: ns-resize; background: transparent;
  border-top: 2px solid var(--border); border-radius: 0 0 7px 7px; margin-top: 4px;
  display: flex; align-items: center; justify-content: center; transition: border-color .15s;
}
.chart-resize-handle:hover, .chart-resize-handle.dragging { border-color: var(--accent); }
.chart-resize-handle::after { content: ''; width: 36px; height: 3px; border-radius: 2px; background: var(--border); transition: background .15s; }
.chart-resize-handle:hover::after, .chart-resize-handle.dragging::after { background: var(--accent); }
```

HTML inside `.col-graph > .panel` (Bode Plot panel):

```html
<div class="chart-wrap" id="chartWrap">
  <canvas id="bodeChart"></canvas>
  <div class="chart-empty" id="chartEmpty">
    <i data-lucide="chart-spline"></i>
    <div class="chart-empty-title">No measurements yet</div>
    <div class="chart-empty-hint">Enable Audio, then press START SWEEP — or drop a saved .bode file anywhere on the page to load it into a slot.</div>
  </div>
</div>
<div class="chart-resize-handle" id="chartResizeHandle"></div>
```

Resize JS (IIFE, registered before init): on `mousedown` record startY/startH; on `mousemove` set `wrap.style.height = Math.max(120, startH + dy) + 'px'` and call `bodeChart.resize()`; on `mouseup` remove listeners, restore cursor/user-select, call `scheduleSave()`. Body cursor `ns-resize` and `user-select: none` while dragging; handle gets `.dragging`.

Empty-state CSS:

```css
.chart-empty {
  position: absolute; inset: 0; display: flex; flex-direction: column;
  align-items: center; justify-content: center; gap: 10px;
  color: var(--text-dim); pointer-events: none; text-align: center; padding: 0 20px;
}
.chart-empty svg { width: 44px; height: 44px; opacity: 0.45; }
.chart-empty-title { font-size: 14px; font-weight: 700; color: var(--text); }
.chart-empty-hint { font-size: 12px; max-width: 420px; line-height: 1.5; }
```

`updateChart()` toggles `#chartEmpty.hidden` — hidden whenever ≥1 dataset exists.

---

## Panels

```css
.panel { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; padding: 14px; box-shadow: var(--shadow); }
.panel-title {
  display: flex; align-items: center; gap: 8px;
  font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 0.09em;
  color: var(--text-dim); border-bottom: 1px solid var(--border); margin-bottom: 12px; padding-bottom: 8px;
}
.panel-title::before { content: ''; width: 3px; height: 11px; border-radius: 2px; background: var(--accent); flex-shrink: 0; }
```

Panels: Audio Routing (col1) · Sweep Settings + Graph Settings (col2) · Measurement Slots (slots) · Bode Plot (graph).

---

## Form Controls

### `.form-row`
`display:flex; align-items:center; justify-content:space-between; margin-bottom:8px; gap:8px`. Labels: `font-size:12px; color:var(--text-dim); flex-shrink:0`.

### Number inputs & selects

```css
input[type="number"] {
  background: var(--bg); border: 1px solid var(--border); border-radius: 6px; color: var(--text);
  font-size: 12px; padding: 5px 8px; width: 100px; text-align: right;
  font-variant-numeric: tabular-nums; transition: border-color .15s, box-shadow .15s;
}
select {
  background: var(--bg); border: 1px solid var(--border); border-radius: 6px; color: var(--text);
  font-size: 12px; padding: 5px 8px; width: 100%; margin-bottom: 8px; cursor: pointer;
  transition: border-color .15s, box-shadow .15s;
}
input[type="number"]:hover, select:hover { border-color: var(--text-dim); }
input[type="number"]:focus, select:focus { outline: none; border-color: var(--accent); box-shadow: var(--focus-ring); }
```

`.field-label`: `font-size:12px; color:var(--text-dim); display:block; margin-bottom:4px`. `.sep`: `border-top:1px solid var(--border); margin:10px 0`.

### `.slider-grid`

```css
.slider-grid { display: grid; grid-template-columns: minmax(120px,auto) 1fr auto; align-items: center; gap: 6px 8px; margin-top: 6px; }
.slider-grid label { font-size: 12px; color: var(--text-dim); }
.slider-grid input[type="range"] { width: 100%; margin: 0; }
.slider-grid input[type="checkbox"] { justify-self: start; margin-top: 3px; accent-color: var(--accent); width: 15px; height: 15px; cursor: pointer; }
.range-val { font-size: 12px; color: var(--text); text-align: right; white-space: nowrap; min-width: 54px; font-variant-numeric: tabular-nums; }
```

### Custom range sliders (all range inputs)

```css
input[type="range"] { -webkit-appearance: none; appearance: none; height: 4px; border-radius: 2px; background: var(--surface2); outline: none; cursor: pointer; }
input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none; appearance: none; width: 14px; height: 14px; border-radius: 50%;
  background: #fff; border: 2px solid var(--accent); box-shadow: 0 1px 4px rgba(0,0,0,.35); transition: transform .1s;
}
input[type="range"]::-webkit-slider-thumb:hover { transform: scale(1.2); }
input[type="range"]::-moz-range-thumb { width: 10px; height: 10px; border-radius: 50%; background: #fff; border: 2px solid var(--accent); box-shadow: 0 1px 4px rgba(0,0,0,.35); }
input[type="range"]:focus-visible { box-shadow: var(--focus-ring); }
```

Filled-track painting (JS): `paintRange(el)` sets `el.style.background = linear-gradient(to right, var(--accent) pct%, var(--surface2) pct%)` where `pct = (value-min)/(max-min)*100`. Applied to all range inputs at init and on their `input` events, and re-applied by `refreshSliderDisplays()` after state restore.

---

## Controls Reference

### Sweep Settings panel

| Control | Type | ID | Default | Range | Step | Display | Visible? |
|---|---|---|---|---|---|---|---|
| Start frequency | number | `startFreq` | 500 | 20–20000 | 1 | — | yes |
| End frequency | number | `endFreq` | 7000 | 20–20000 | 1 | — | yes |
| Scan Durtation | slider | `scanDuration` | 10 | 1–60 | 1 | `10 s` | yes |
| Readings/point | slider | `readingsPerPoint` | 15 | 1–50 | 1 | `15` | yes |
| Input sensitivity | slider | `inputSensitivity` | 0 | −24–+24 | 0.5 | `0 dB` (+ for positive) | **HIDDEN** |
| Output attenuation | slider | `outputAttenuation` | 6 | 0–6 | 0.1 | `6 dB/oct` | **HIDDEN** |
| Sine-only filter | checkbox | `sineOnly` | unchecked | — | — | — | **HIDDEN** |

> Keep the exact visible typo **`Scan Durtation`**.

**Hidden controls**: the three hidden rows keep their full markup (label + input + `.range-val` span) inside the slider grid, but every element carries `class="hidden"` (`.hidden { display: none !important; }`). They stay in the DOM so their default values still drive the sweep/measurement code and persistence.

### Graph Settings panel

| Control | Type | ID | Default | Range | Step | Display |
|---|---|---|---|---|---|---|
| Graph smoothness | slider | `graphSmoothness` | 5 | 1–10 | 1 | `5` |
| Graph tilt | slider | `graphOffset` | 0 | −6–+6 | 0.1 | `+0.0 dB/oct` (+ for positive) |

> Visible label is **“Graph tilt”** but the element ID stays `graphOffset`.

`updateChart()` fires on `input` for `graphSmoothness`/`graphOffset` and on both `input` and `change` for `startFreq`/`endFreq`.

---

## Audio Routing Panel (col1, the only panel there)

- `Input device` label + full-width `<select id="inputDevice">`
- `Input channel` select: `Channel 1 / left` (`0`), `Channel 2 / right` (`1`), `Mono mix` (`mix`)
- `.sep`
- `Output device` label + full-width `<select id="outputDevice">`
- `Output channel` select: `Both channels` (`both`), `Channel 1 / left` (`0`), `Channel 2 / right` (`1`)
- Hidden note `id="sinkIdNote"` (class `note`, orange text) shown if `setSinkId` unavailable

---

## Web Audio

### Enable Audio procedure

1. `getUserMedia` temp stream: `echoCancellation:false, noiseSuppression:false, autoGainControl:false, channelCount:2`
2. Create `AudioContext`, `await resume()`
3. Enumerate devices; filter out `deviceId === 'default'` and `'communications'`; populate both selects
4. Stop temp stream tracks
5. **Restore saved devices**: if `savedDevices.input`/`.output` (from localStorage) still exist among enumerated devices, pre-select them
6. Re-request stream with `deviceId: { exact: selectedInputId }` (selected value, else first device)
7. `buildInputGraph()`, `buildOutputGraph()`, `startMeterLoop()`
8. `setSinkId(outputDeviceEl.value)` on the output `Audio` element if supported, else reveal `#sinkIdNote`
9. `await learnNoiseFloor()`
10. Status `Audio ready` (green); Enable button → `Audio enabled` (disabled); enable Test Signal + START SWEEP
11. Wire `change` listeners: input device / input channel / output device / output channel handlers
12. On any error: status `Audio error: <msg>` (red), re-enable the Enable button

### Input graph

```
MediaStreamSource(inputStream) → ChannelSplitter(2)
  → [channel routing] → GainNode (gain = dbToGain(inputSensitivity))
  → AnalyserNode (fftSize = 8192, smoothingTimeConstant = 0.15)
```

Channel routing: `0`/`1` → `splitter.connect(inputGain, ch)`; `mix` → two 0.5-gain nodes from splitter outputs 0 and 1 both into inputGain. `connectInputChannel` first calls `splitter.disconnect()`.

### Output graph

Persistent parts built once (`buildOutputGraph`): output `AnalyserNode` (fftSize 1024, smoothing 0.15), `MediaStreamDestination`, and `Audio` element (`autoplay = true`, `srcObject = dest.stream`).

Per-tone parts (`startOscillator(freq, gainValue)`): sine `OscillatorNode` → `GainNode` → outAnalyser → `ChannelMerger(2)` → dest. Channel routing into merger: `both` → analyser to merger inputs 0 and 1; `0`/`1` → that input only. Then `osc.start()` and `outputAudio.play()`.

### `stopOutput()`

Stop/disconnect/nullify osc, gain, merger; `outAnalyser.disconnect()` but do **not** nullify (reused); `outputAudio.pause()`. All guarded with try/catch.

### `setSweepFreq(freq, gainValue)` — no oscillator restart

```js
if (!oscNode || !outGainNode) { startOscillator(freq, gainValue); return; }
oscNode.frequency.setValueAtTime(freq, audioCtx.currentTime);
outGainNode.gain.setValueAtTime(gainValue, audioCtx.currentTime);
```

### Device change handlers

- input device: stop old tracks → new stream (exact id) → `buildInputGraph()` → `learnNoiseFloor()`
- input channel: `connectInputChannel(ch)` → `learnNoiseFloor()`
- output device: `outputAudio.setSinkId(id)` if available
- output channel: if oscillator running, disconnect analyser→merger and reconnect for the new channel

The input sensitivity slider's `input` listener also live-updates `inputGainNode.gain` (control is hidden but functional).

---

## Noise Floor

Learned after audio enable, input device change, input channel change. Never during a sweep (guard `if (isSweeping) return`).

```
stopOutput() → status "Learning noise floor" (orange) → sleep(120)
→ 14 × { read Float32 time-domain buffer, rms², sleep(25) }
→ noiseRms = sqrt(Σrms²/14)                        (power-space average)
→ noiseFloorCache[`${deviceId}:${channel}`] = noiseRms
→ status `Noise floor -XX.X dBFS` (orange)
```

Noise subtraction (power space): `sqrt(max(0, level² − noise²))`. Sine-only detector noise: `detNoise = noise * sqrt(2 / fftSize)`.

---

## Sweep

### Constants

```js
DEFAULT_START_FREQ = 500;  DEFAULT_END_FREQ = 7000;
PRESCAN_HZ = 300;  POINT_COUNT = 128;  FFT_SIZE = 8192;  MIN_DB = -96;
```

### Timing

```js
msPerPoint   = duration*1000 / POINT_COUNT;
fftRefreshMs = Math.round(FFT_SIZE / audioCtx.sampleRate * 1000);   // ≈186 ms @44.1k
settleMs     = Math.max(msPerPoint * 0.3, fftRefreshMs);            // ≥ one full buffer flush
remainingMs  = Math.max(0, msPerPoint - settleMs);
effectiveReadings = Math.max(1, Math.min(readings, Math.floor(remainingMs/20) + 1));
readMs       = effectiveReadings > 1 ? Math.floor(remainingMs/effectiveReadings) : 5;
```

### Procedure

1. Clear `chartData`/`chartPointCloud`; stop test tone if running
2. Disable START SWEEP + Test Signal; enable Stop with `.sweeping`
3. `scanStart = max(20, start − 300)`; `points = logspace(scanStart, end, 128)`
4. Show `#sweepProgressBar` (opacity 1, width 0%)
5. `startOscillator(points[0], gain₀)` — oscillator started **once**
6. Loop over the 128 points (break on `sweepAbort`, checked before and after settle):
   - `gain = 0.42 * dbToGain(−(atten * max(0, log2(freq/start))))` (pre-scan clamps octaves to 0)
   - `setSweepFreq(freq, gain)`
   - `freq < start`: status `Calibrating` (orange), `#currentFreq` shows red `Calibrating`
   - `freq ≥ start`: status `Sweeping` (red) — or `Sweeping — input clipping!` if clipping was seen; `#currentFreq` shows `fmtHz(freq)`
   - progress % → `#progress` text AND `#sweepProgressBar` width
   - `await sleep(settleMs)`
   - Take `effectiveReadings` readings spaced `readMs`; each: `measureLevel(...)` → `db = gainToDb(max(1e-10,lvl)) + inputSensitivity`; accumulate; push to `chartPointCloud`; if the reading's raw buffer level (`lastRawDb`) > −3 dBFS set `sweepClipped = true`
   - `avgDb = max(MIN_DB, mean)`; if `freq ≥ start` push `{x:freq, rawDb:avgDb}` to `chartData` and `updateChart()`
7. Completed: `currentFreq='-- Hz'`, progress 100%, `activePeakSource='current'`, status `Sweep complete` (green) — or, if clipped, status `Sweep complete — input clipped, reduce level` (orange) + orange toast `Input exceeded −3 dBFS during the sweep — results may be distorted`; then `updateChart()` + `scheduleSave()`
8. Aborted: status `Stopped` (no color)
9. `finally`: `stopOutput()`, re-enable buttons, remove `.sweeping`, clear red class, fade progress bar (opacity 0, width reset to 0 after 450 ms if not sweeping)

### Measurement reading

`measureLevel(freq, sineOnly, noise)` reads a Float32 time-domain buffer of `fftSize` samples and first records `lastRawDb = gainToDb(rms(buf))` (global, used for clip detection). Then:

- **sineOnly** (lock-in): `w = 2π·freq/sampleRate`; `sinSum = Σ buf[i]·sin(wi)`, `cosSum = Σ buf[i]·cos(wi)`; `lvl = (2/N)·hypot(sinSum,cosSum)/√2`; subtract `detNoise = noise·√(2/N)` in power space.
- **RMS** (default): `lvl = rms(buf)`; subtract `noise` in power space.

---

## Test Signal

Toggles a 1 kHz sine at gain 0.42.

**Start**: resume ctx → `startOscillator(1000, 0.42)` → currentFreq `1.00 kHz` → status `Playing test tone` → Stop enabled (not red) → button becomes `pause` icon + `Stop test`. Then a **1.5 s sanity check**: if still playing and input RMS dB < noiseFloorDb + 6, set status `No input signal detected — check routing and cables` (orange) and show orange toast `Test tone is playing but the input is silent — check routing`.

**Stop**: `stopOutput()` → currentFreq `-- Hz` → status `Audio ready` (green) → Stop disabled → button back to `audio-lines` + `Test Signal`.

---

## Chart (Chart.js)

`type:'line'`, `animation:false`, `responsive:true`, `maintainAspectRatio:false`, `parsing:false` per dataset, `interaction: {mode:'index', intersect:false}`. Chart registered with the custom `peakMarkerPlugin` (via the chart's `plugins: [peakMarkerPlugin]` array).

- **X**: logarithmic; title `Generated frequency`; grid `#2a2f3f`; ticks `#7a8099`, `maxTicksLimit:10`; `min = startFreq`, `max = max(endFreq, max visible slot end)`
- **Y**: title `Normalized measured Input (approx.)`; ticks callback `` v => `${v.toFixed(1)} dB` ``; auto-fit min/max over all visible processed points with `margin = max(1.5, (max−min)*0.12)`; min/max `undefined` when no data
- **Legend**: top; `boxWidth:14, boxHeight:2, padding:12, font 11`; color `#d4d8e8`; `filter: item => item.text !== 'Current'`
- **Tooltip**: index/no-intersect; bg `#1e2230`, border `#2a2f3f`, title `#ff9c00`, body `#d4d8e8`; title `fmtHz(x)`, label `` ` ${label||'Current'}: ${y.toFixed(2)} dB` ``

### Peak marker plugin

`afterDatasetsDraw`: for each dataset with ≥3 points, find the max-y index, get the element's pixel position, skip if outside `chartArea` horizontally, then draw a filled 3.5px-radius circle in `ds.borderColor` plus a centered `600 10px` system-font label `fmtHz(peak.x)` at `y−7` (clamped below `chartArea.top+12`).

### Data processing — `processData(rawPoints, startHz)`

Applied fresh on every `updateChart()`; never stored.

1. Filter `p.x >= startHz`
2. Tilt: `y = rawDb + graphOffset * log2(x/startHz)`
3. Gaussian smooth in index space: `radius = max(1, round(smoothness*1.5))`, `sigma = max(1, radius/2)`, weights `exp(−d²/(2σ²))`, edge-normalized
4. Normalize to first point and divide by 3: `y = (y − y₀)/3`

### Datasets

- **Current**: label `Current`, borderColor `#ff3b4f`, borderWidth 2, pointRadius 0, tension 0.3
- **Slots** (visible + non-empty): label = slot name, borderColor `slotColor(i)`, same styling; processed with the slot's own stored `start`

`updateChart()` ends with `updatePeakResponse()`; it also toggles `#chartEmpty`.

---

## Peak Response

`activePeakSource`: `'current'` | slot index `0..5` | `null`. Peak = max y of the processed curve. Display: `#peakFreq` = `fmtHz(x)`, `#peakResponse` = `fmtDb(y)`. When no valid source: `#peakFreq` = `''`, `#peakResponse` = `'--'`.

---

## Light/Dark Toggle

`applyTheme(isLight)` (also used at startup for restored theme):

1. `body.classList.toggle('light', isLight)`; theme button icon `moon` (light) / `sun` (dark); `renderIcons()`
2. Chart runtime colors: grid `#d0d4e0`/`#2a2f3f`; axis titles+ticks `#5a6080`/`#7a8099`; legend labels `#1a1d2e`/`#d4d8e8`; tooltip bg `#ffffff`/`#1e2230`, border `#c8cdde`/`#2a2f3f`, body `#1a1d2e`/`#d4d8e8`; tooltip title stays `#ff9c00`
3. `refreshSlotDots()` and `updateChart()` (rebuilds datasets so theme-dependent slot colors apply)

`toggleTheme()` = `applyTheme(!isLight)` + `scheduleSave()`.

---

## Measurement Slots

`#slotList` — 2 columns, **column-flow** so slots 1–3 fill the left column and 4–6 the right:

```css
#slotList { display: grid; grid-template-columns: 1fr 1fr; grid-template-rows: repeat(3,auto); grid-auto-flow: column; gap: 7px; }
```

### Colors

`SLOT_COLORS = ['#35d0a2','#4ea1ff','#ff9b45','#ffffff','#b76cff','#00d8ff']`

`slotColor(i)`: returns `SLOT_COLORS[i]`, **except** `#ffffff` maps to `#4a5165` in light mode (slot 4 visibility fix). Used for datasets, dots, and PNG-export legend. `refreshSlotDots()` re-applies background+color to every `.slot-dot` on theme change.

### Slot card

```css
.slot { background: var(--surface2); border: 1px solid var(--border); border-radius: 8px; padding: 10px; transition: border-color .15s, box-shadow .15s; }
.slot:hover { border-color: var(--text-dim); box-shadow: var(--shadow-sm); }
.slot.dragover { border-color: var(--accent); box-shadow: var(--focus-ring); }
.slot-head { display: flex; align-items: center; gap: 8px; margin-bottom: 8px; }
.slot-dot { width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; box-shadow: 0 0 6px currentColor; border: 1px solid rgba(127,127,127,.4); }
.slot-name { flex: 1; min-width: 0; background: transparent; border: none; border-bottom: 1px solid var(--border);
             color: var(--text); font-size: 12px; font-weight: 600; padding: 2px 0; outline: none; transition: border-color .15s; }
.slot-name:focus { border-bottom-color: var(--accent); }
.slot input[type="checkbox"] { width: 15px; height: 15px; cursor: pointer; }
.slot-actions { display: flex; gap: 5px; margin-bottom: 5px; flex-wrap: wrap; }
.slot-meta { font-size: 11px; color: var(--text-dim); }
```

Each card (built by `buildSlotUI()`): colored dot (inline `background` and `color` = `slotColor(i)`), editable `.slot-name` input (`data-slot="i"`), show checkbox (`data-slot-show="i"`, inline `accent-color` = slot color, **pre-checked when `slot.visible`** — needed for state restore), buttons `Store` (`data-slot-store`), `Save` (`data-slot-save`), `Load` (label wrapping a hidden `<input type="file" accept=".bode,.txt,.json" data-slot-load="i">`), and meta `id="slotMeta{i}"` showing `N pts · start–end Hz` or `Empty`.

Each card is also a **drop target**: `dragover` → preventDefault + `.dragover`; `dragleave` removes it; `drop` → preventDefault + stopPropagation, load first dropped file via `loadSlotFromFile(i, file)`.

### Store — with overwrite protection

`storeSlot(i, btn)`:
- No `chartData` → orange toast `No measurement to store — run a sweep first`
- Slot already has data and button not armed → arm: `.confirm` class, text `Sure?`, auto-revert after 2 s. Second click within 2 s proceeds.
- Proceed: copy `chartData` → `slot.data` (raw `{x, rawDb}`), `slot.start/end` from UI, `visible = true`, `activePeakSource = i`, refresh slot UI, `updateChart()`, green toast `Stored measurement in "name"`, `scheduleSave()`

### Save

Downloads pretty-printed JSON as **`{sanitized-name}.bode`** (sanitize: `replace(/[^\w\- ]+/g,'_')`), then green toast `Saved "file.bode"`. Empty slot → orange toast. Format:

```json
{
  "type": "guitar-pickup-bode-measurement",
  "version": 1,
  "name": "Slot name",
  "start": 500,
  "end": 7000,
  "points": [{ "x": 500, "rawDb": -42.1 }]
}
```

### Load

`loadSlotFromFile(i, file)` (used by both the file input and drag & drop): parse JSON; validate `points` is a non-empty array of finite-number `{x, rawDb}`; restore data/name/start/end (fall back to first/last point x); `visible = true`; `activePeakSource = i`; refresh UI; `updateChart()`; green toast `Loaded "name" into slot N`; `scheduleSave()`. Invalid → red toast `Could not load "file" — not a valid measurement`. The file-input wrapper clears `input.value` afterwards.

### Show/hide checkbox

Check: `visible = true`, `activePeakSource = i`, `updateChart()`. Uncheck: `visible = false`, `updateChart()`.

### Page-level drag & drop

`document` `dragover` → preventDefault. `drop` → preventDefault; first dropped file goes into the **first empty slot**; if none, orange toast `All slots are full — drop onto a specific slot to replace it`.

---

## Clear Latest

`chartData=[]`, `chartPointCloud=[]`, currentFreq `-- Hz` (red class removed), progress `0%`, `#sweepProgressBar` width 0, `activePeakSource=null` if it was `'current'`, toast `Latest measurement cleared`, `updateChart()`, `scheduleSave()`.

---

## Export PNG (composed)

`exportVisibleCurves()` returns `[{name, color, data}]`: `Current` (`#ff3b4f`) if `chartData` non-empty, then visible non-empty slots with `slotColor(i)`.

`exportPng()`:
- `bg = isLight ? '#ffffff' : '#15181f'`; `dim`/`fg` per theme; `headerH = 150`
- Canvas: `src.width*2 × (src.height*2 + headerH)`, filled with bg
- Title `Nick's Guitar Pickup Bode Plotter` — `700 38px` system font, `#ff9c00`, at (40, 58)
- Date `new Date().toLocaleString()` — `400 20px`, dim, at (40, 92)
- Legend row at y≈128: for each curve, 26×7 color swatch, then name in `600 20px` fg; advance `x += 34 + textWidth + 28`
- `drawImage(chartCanvas, 0, headerH, w, src.height*2)`
- Download `bode-plot-{ISO timestamp with : and . replaced by -}.png`

## Export CSV

Header `source,frequency_hz,raw_db`; one row per raw point of every visible curve (name CSV-quoted, freq `toFixed(2)`, rawDb `toFixed(3)`). No data → orange toast. Download `bode-data-{timestamp}.csv`; green toast `Exported N curve(s) as CSV`.

---

## Toasts

Container `<div id="toasts"></div>` before the script.

```css
#toasts { position: fixed; right: 16px; bottom: 16px; display: flex; flex-direction: column; gap: 8px; z-index: 200; }
.toast { background: var(--surface2); border: 1px solid var(--border); border-left: 3px solid var(--accent);
         border-radius: 8px; padding: 10px 14px; font-size: 12px; font-weight: 600; color: var(--text);
         box-shadow: var(--shadow); max-width: 320px; animation: toast-in .2s ease; transition: opacity .3s, transform .3s; }
.toast.green { border-left-color: var(--green); } .toast.orange { border-left-color: var(--accent-2); } .toast.red { border-left-color: var(--red); }
.toast.out { opacity: 0; transform: translateX(12px); }
@keyframes toast-in { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: none; } }
```

`showToast(msg, type)` appends, waits 2800 ms, adds `.out`, removes after 320 ms. Toasts are for transient events; the status bar keeps persistent state (Audio ready / Sweeping / Noise floor …).

---

## Keyboard Shortcuts

Single `keydown` listener. `Escape` always closes Help. Everything else is ignored when the event target is an INPUT / SELECT / TEXTAREA / BUTTON / contentEditable, or when meta/ctrl/alt is held.

| Key | Action |
|---|---|
| `Space` | preventDefault; abort sweep if sweeping, else `startSweep()` if audio enabled and button not disabled |
| `T` | toggle test tone |
| `1`–`6` | toggle that slot's visibility (no-op if slot empty); syncs its checkbox; sets `activePeakSource` when showing; `updateChart()` + `scheduleSave()` |
| `D` | toggle theme |
| `H` / `?` | open Help |
| `Esc` | close Help |

---

## Help Overlay

Modal `#helpOverlay`, backdrop `rgba(0,0,0,0.6)` + `backdrop-filter: blur(5px)`. Box: surface, radius 12px, padding 24, max-width 560px, max-height 85vh, shadow `0 24px 64px rgba(0,0,0,.45)`, pop-in animation (fade + translateY(10px) scale(.98), 0.18 s). Closes via Close button, backdrop click (`e.target === overlay`), or Escape.

Steps (ordered list):
1. Place your pickup in the device.
2. Connect to your soundcard.
3. Enable Audio.
4. Select the audio input and output of your soundcard.
5. Launch the Test Signal tone to check for input signal.
6. Start the Sweep.
7. Adjust Smoothness and Measurement offset to taste.

Then a **Keyboard shortcuts** block using `<kbd>` chips (`Space` start/stop sweep · `T` test tone · `1`–`6` toggle slots · `D` light/dark · `H` help · `Esc` close). Kbd style: surface2 bg, border with 2px bottom, radius 4, `1px 6px`, 11px monospace.

Footer (12px, dim, links in `--accent-2`):
- `Build your own coil driver https://www.youtube.com/watch?v=SCeFtqdcS1Y`
- `This tool is for playing around and gives rough approximations. If you are serious about plotting your pickups, get the Bode Plotter at https://www.axetech.com`
- `©2026 Built with ❤️ by Niklaus Hirt`

---

## Persistence (localStorage)

Key: `bodePlotter_v1`. Best-effort (all try/catch).

**Saved shape:**

```js
{
  theme: 'light' | 'dark',
  settings: { startFreq, endFreq, scanDuration, readingsPerPoint,
              inputSensitivity, outputAttenuation, sineOnly,
              graphSmoothness, graphOffset, inputChannel, outputChannel },
  devices: { input: deviceId, output: deviceId },
  chartHeight: '412px' | null,          // chartWrap inline height, if dragged
  chartData: [{x, rawDb}, …],           // latest measurement
  activePeakSource: 'current' | 0..5 | null,
  slots: [{ name, data, start, end, visible } × 6]
}
```

**Saving**: `scheduleSave()` debounces `saveState()` by 300 ms. Triggers: document-level `input` and `change` listeners (covers every control), `beforeunload` → immediate `saveState()`, plus explicit calls after: sweep completion, store/load slot, clear latest, theme toggle, chart-resize mouseup, keyboard slot toggle.

**Loading** (`loadState()`, called first in init): restore setting values (only when present), checkbox state, channel selects; stash `devices` into `savedDevices` for Enable Audio; filter-validate `chartData` and each slot's `data` with `isRawPoint` (`x`/`rawDb` finite numbers); a slot is only `visible` if it has data; restore `chartWrap` inline height; add `light` class to body (chart colors fixed later by `applyTheme`).

---

## Utility Functions

```js
const dbToGain = db => Math.pow(10, db/20);
const gainToDb = g  => 20 * Math.log10(Math.max(1e-10, g));
const rms   = buf => Math.sqrt(buf.reduce((s,v) => s+v*v, 0) / buf.length);
const fmtHz = hz => hz >= 1000 ? `${(hz/1000).toFixed(2)} kHz` : `${Math.round(hz)} Hz`;
const fmtDb = db => `${db >= 0 ? '+' : ''}${db.toFixed(1)} dB`;
const sleep = ms => new Promise(r => setTimeout(r, ms));
function logspace(start, end, n) { /* start * (end/start)^(i/(n-1)) for i in 0..n-1 */ }
const isRawPoint = p => p && typeof p.x === 'number' && typeof p.rawDb === 'number' && isFinite(p.x) && isFinite(p.rawDb);
function fmtSigned(v, decimals) { /* '+' prefix when > 0; optional toFixed */ }
```

`setStatus(text, cls)` replaces `#statusBar` content with one `<span>` of class `orange|red|green|''`. `renderIcons()` = `lucide.createIcons()`.

`refreshSliderDisplays()` sets all six `.range-val` texts from current values and repaints all range tracks — called at init after `loadState()`.

---

## DOM Order (body)

```
<header class="header">  (logo img, h1, subtitle, button row)
<div class="meters-bar"> (…, #sweepProgressBar)
<div class="main-grid">
  <div class="col-left">      Audio Routing panel
  <div class="col-settings">  Sweep Settings + Graph Settings panels
  <div class="col-slots">     Measurement Slots panel (#slotList)
  <div class="col-graph">     Bode Plot panel (chart wrap + empty state + resize handle)
</div>
<div class="overlay hidden" id="helpOverlay"> …
<div id="toasts"></div>
<script> …
```

On screen, grid rows render: **Bode Plot (top) → Audio Routing / Settings / Slots (bottom)**.

---

## Initialisation order (bottom of script — order is critical)

```js
// listeners: all header buttons (incl. exportCsvBtn → exportCsv), help overlay,
// keyboard shortcuts, theme button, wireSliders()
// chart resize handle IIFE (mouseup → scheduleSave)
// paintRange definition + initial wiring of all range inputs
// refreshSliderDisplays definition
// page-level dragover/drop handlers
// document-level input/change → scheduleSave; beforeunload → saveState

loadState();                 // 1. restore persisted state (may add body.light)
refreshSliderDisplays();     // 2. sync labels + track fills to restored values
initChart();                 // 3. create chart (dark defaults) with peakMarkerPlugin
buildSlotUI();               // 4. slot cards reflect restored names/visibility/meta
renderIcons();               // 5. lucide
if (body has 'light') applyTheme(true);   // 6a. fixes chart colors + dots + redraws
else updateChart();                        // 6b. renders restored measurements
```

---

## Key Implementation Notes

1. **Never restart the oscillator between sweep points** — `setValueAtTime` only; a restart puts a click/silence gap into the analyser's circular buffer and contaminates the next measurement.
2. **`settleMs ≥ one full FFT buffer refresh`** (≈186 ms @ 44.1 kHz). Short sweep durations therefore take longer than requested — intentional.
3. **Lock-in magnitude is phase-independent** (`hypot(sinSum, cosSum)`), no phase sync needed.
4. **Y values are divided by 3** after baseline normalization; peak response uses post-division numbers.
5. **Slots store raw `{x, rawDb}` only**; tilt/smoothing/normalization/÷3 are recomputed on every `updateChart()`.
6. **`Current` is excluded from the legend** via the label filter but appears in tooltips.
7. **Slot grid flows column-wise** (`grid-auto-flow: column` + 3 explicit rows) so slots 1–3 are the left column — a plain 2-column grid would interleave them.
8. **Hidden controls** (input sensitivity, output attenuation, sine-only) must remain in the DOM with their defaults; all sweep code reads them normally.
9. **`.bode` is the canonical measurement extension**; `.txt`/`.json` remain accepted on load for backward compatibility.
10. **Toasts = transient events, status bar = persistent state.** Don't route persistent states through toasts or events through the status bar.
11. **Slot 4 (`#ffffff`) maps to `#4a5165` in light mode** everywhere a slot color is drawn (curve, dot, export legend) via `slotColor(i)`.
12. **Restored visible slots need their checkbox `checked` in the generated slot markup**, otherwise the UI desyncs from restored state.
13. **Meter loop** (100 ms interval) must handle a missing/idle output analyser gracefully (`-- dBFS`, 0% bar).
14. Optional dev convenience: `.claude/launch.json` starting `python3 -m http.server 8642` for a localhost preview.
