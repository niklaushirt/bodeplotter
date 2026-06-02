# Nick's Guitar Pickup Bode Plotter — Implementation Spec (Latest5)

Implement the app as one self-contained `index.html` file with inline HTML, CSS, and JavaScript. Use CDN scripts only for:

- Chart.js `https://cdn.jsdelivr.net/npm/chart.js@4.4.7/dist/chart.umd.min.js`
- Lucide icons `https://unpkg.com/lucide@latest/dist/umd/lucide.min.js`

Do not use a build step. Do not use Tailwind CDN. Run from localhost or a secure context so browser audio permissions work.

---

## Purpose

Build a dark-themed (with light/dark toggle) browser tool for rough guitar pickup frequency-response plotting. It generates a sine sweep through a selected soundcard output, measures a selected soundcard input, corrects measurements with calibration and a learned noise floor, optionally uses sine-only lock-in filtering, displays a smoothed Bode-style response graph, and manages six saved measurement slots.

---

## Theme And Colors

CSS variables on `:root` (dark mode defaults):

```css
--accent:   #004cfc
--accent-2: #ff9c00
--bg:       #0f1117
--surface:  #15181f
--surface2: #1e2230
--border:   #2a2f3f
--text:     #d4d8e8
--text-dim: #7a8099
--red:      #ff3b4f
--green:    #35d0a2
```

Light mode overrides on `body.light`:

```css
--bg:       #eef0f5
--surface:  #ffffff
--surface2: #e2e5ee
--border:   #c8cdde
--text:     #1a1d2e
--text-dim: #5a6080
```

`--accent` and `--accent-2` are identical in both modes.

Use `--accent-2` (`#ff9c00`) for:
- H1 title text
- Peak Response frequency and dB values
- START SWEEP button background
- Calibrating status text
- Help overlay links

Use `--red` for:
- Sweeping status text
- Stop button border/color while sweep is running
- Current frequency readout text during pre-scan calibration phase

---

## Header

Browser title and H1: `Nick's Guitar Pickup Bode Plotter`

- H1 color: `var(--accent-2)`, `font-size: 36px`, `font-weight: 700`

Subtitle (`font-size: 12px`, color `--text-dim`):
> Sweep a compensated sine signal through a pickup test rig, measure the soundcard input, and plot the normalized frequency response.

Header buttons row (`display: flex; gap: 6px; flex-wrap: wrap; align-items: center`):

| # | Label | Style | Icon |
|---|---|---|---|
| 1 | `Enable Audio` | `.btn-primary` (bg `#1a7a4a`, text `#fff`) | `power` |
| 2 | `Test Signal` | default `.btn`, disabled until audio enabled | `audio-lines` |
| 3 | `START SWEEP` | `.btn-orange` (bg `--accent-2`, text `#000`), disabled until audio enabled | `chart-spline` |
| 4 | *(stop icon)* | `.btn-stop`, disabled unless busy | `square` |
| 5 | `Export PNG` | default `.btn` | `download` |
| 6 | `Clear Latest` | default `.btn` | `eraser` |
| 7 | `Help` | default `.btn` | `circle-help` |
| 8 | *(theme toggle)* | default `.btn`, `margin-left: auto` (pushed far right), `title="Toggle light/dark mode"` | `sun` (dark) / `moon` (light) |

Stop button behavior:
- Disabled by default.
- Gets class `.sweeping` (red border + text color `--red`) only while a sweep is running.
- Enabled but not red during Test Signal.

When audio is enabled: button text becomes `Audio enabled`, button is disabled.

---

## Layout

### Grid structure

The app uses CSS grid-template-areas. The graph is **always full-width** on its own row. The settings/slots panels occupy the top row.

**Default (≥ 940px):**
```css
.main-grid {
  display: grid;
  grid-template-columns: 280px 330px 1fr;
  grid-template-areas:
    "col1 col2 slots"
    "graph graph graph";
  padding: 12px;
  gap: 10px;
  align-items: start;
}
```
- `col1` (280px): Meters panel + Audio Routing panel (flex column, gap 10px)
- `col2` (330px): Sweep Settings panel + Graph Settings panel (flex column, gap 10px)
- `slots` (`1fr`, fills all remaining width): Measurement Slots panel
- `graph` (spans all 3 columns = full viewport width): Bode Plot

Grid area assignments:
```css
.col-left     { grid-area: col1; display: flex; flex-direction: column; gap: 10px; }
.col-settings { grid-area: col2; display: flex; flex-direction: column; gap: 10px; }
.col-graph    { grid-area: graph; }
.col-slots    { grid-area: slots; }
```

**Tablet portrait (≤ 939px):**
```css
grid-template-columns: 280px 1fr;
grid-template-areas:
  "col1 col2"
  "graph graph"
  "slots slots";
```
- col1 and col2 on top row
- graph full width below
- slots full width below graph

**Phone (≤ 639px):**
```css
grid-template-columns: 1fr;
grid-template-areas:
  "col1"
  "col2"
  "graph"
  "slots";
```

### Chart canvas

```css
.chart-wrap {
  position: relative;
  min-height: 520px;   /* all sizes ≥ 640px */
}
@media (max-width: 639px) {
  .chart-wrap { min-height: 380px; }
}
.chart-wrap canvas { width: 100% !important; height: 100% !important; }
```

---

## Panels

All panels:
```css
.panel {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 7px;
  padding: 12px;
}
```

Panel titles: `font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 0.07em; color: var(--text-dim); border-bottom: 1px solid var(--border); margin-bottom: 10px; padding-bottom: 6px`.

---

## Form Control Patterns

### `.form-row`
`display: flex; align-items: center; justify-content: space-between; margin-bottom: 8px; gap: 8px`

Used for number inputs and select dropdowns. Labels: `font-size: 12px; color: var(--text-dim); flex-shrink: 0`.

Number inputs: `background: var(--bg); border: 1px solid var(--border); border-radius: 4px; color: var(--text); font-size: 12px; padding: 4px 7px; width: 100px`.

### `.slider-grid`
All sliders (and the checkbox row) share a 3-column CSS grid for perfect vertical alignment:
```css
.slider-grid {
  display: grid;
  grid-template-columns: minmax(120px, auto) 1fr auto;
  align-items: center;
  gap: 6px 8px;
  margin-top: 6px;
}
```
Each row: `<label>` | `<input type="range">` | `<span class="range-val">`

`.range-val`: `font-size: 12px; color: var(--text); text-align: right; white-space: nowrap; min-width: 54px`

The Sine-only filter checkbox row goes in the same grid: label | checkbox (`justify-self: start; margin-top: 3px`) | empty `<span>`.

---

## Controls Reference

### Sweep Settings panel

| Control | Type | ID | Default | Range | Step | Unit display |
|---|---|---|---|---|---|---|
| Start frequency | number input | `startFreq` | 500 | 20–20000 | 1 | Hz |
| End frequency | number input | `endFreq` | 7000 | 20–20000 | 1 | Hz |
| Scan Durtation | **slider** | `scanDuration` | 10 | 1–60 | 1 | `10 s` |
| Readings/point | **slider** | `readingsPerPoint` | 15 | 1–50 | 1 | `15` |
| Input sensitivity | slider | `inputSensitivity` | 0 | -24–+24 | 0.5 | `0 dB` (show `+` for positive) |
| Output attenuation | slider | `outputAttenuation` | 6 | 0–6 | 0.1 | `6 dB/oct` |
| Sine-only filter | checkbox | `sineOnly` | **unchecked** | — | — | — |

> Keep the exact visible typo `Scan Durtation`.

### Graph Settings panel

| Control | Type | ID | Default | Range | Step | Unit display |
|---|---|---|---|---|---|---|
| Graph smoothness | slider | `graphSmoothness` | 5 | 1–10 | 1 | `5` |
| Graph offset | slider | `graphOffset` | 0 | -6–+6 | 0.1 | `+0.0 dB/oct` (show `+` for positive) |

Trigger `updateChart()` on `input` events for: `graphSmoothness`, `graphOffset`, `startFreq` (also `change`), `endFreq` (also `change`).

---

## Meters Panel

Contains in order:

1. **Meter row** — two side-by-side blocks (Input / Output):
   - dBFS value (`-- dBFS` default), font-size 15px, tabular-nums
   - Color bar: 4px height, green (`--green`), turns red (`--red`) when level > −3 dBFS
   - Bar maps −60 dBFS → 0% to 0 dBFS → 100%

2. **Status row** — two blocks side by side:
   - `Current freq`: shows formatted Hz during sweep/test; shows red `Calibrating` text during pre-scan; `-- Hz` when idle
   - `Progress`: `0%`–`100%` during sweep

3. **Status bar** — single-line text. Colored `<span>` children: `.orange` (calibrating, noise floor status), `.red` (sweeping), `.green` (ready, complete). Initial text: `Audio not enabled`.

4. **Peak Response block** — label `Peak Response` on left; right side stacked:
   - **Top** (`.peak-value`): peak frequency — `font-size: 17px; font-weight: 800; color: var(--accent-2)`
   - **Bottom**: peak dB — `font-size: 13px; font-weight: 600; color: var(--accent-2); font-variant-numeric: tabular-nums`
   - Both show `--` / empty when no active source

---

## Audio Routing Panel

Controls (all in Column 1):
- `Input device` label + full-width `<select id="inputDevice">`
- `Input channel` select: `Channel 1 / left` (value `0`), `Channel 2 / right` (value `1`), `Mono mix` (value `mix`)
- Separator (`.sep`)
- `Output device` label + full-width `<select id="outputDevice">`
- `Output channel` select: `Both channels` (value `both`), `Channel 1 / left` (value `0`), `Channel 2 / right` (value `1`)
- Hidden note `id="sinkIdNote"` shown if `setSinkId` is unavailable

---

## Web Audio Graphs

### Enable Audio procedure

1. Request temp stream: `echoCancellation: false, noiseSuppression: false, autoGainControl: false, channelCount: 2`
2. Create `AudioContext`, `await audioCtx.resume()`
3. Enumerate devices; filter out `deviceId === 'default'` and `'communications'`
4. Populate input/output selects with real devices
5. Stop temp stream tracks
6. Re-request stream with `{ exact: firstInputDeviceId }`
7. `buildInputGraph()`, `buildOutputGraph()`, `startMeterLoop()`
8. If `setSinkId` available on output audio element, call it with selected output device; else show note
9. `learnNoiseFloor()`
10. Set status `Audio ready` (green)
11. Wire `change` listeners: `inputDevice → onInputDeviceChange`, `inputChannel → onInputChannelChange`, `outputDevice → onOutputDeviceChange`, `outputChannel → onOutputChannelChange`

### Input graph

```
MediaStreamSource(inputStream)
  → ChannelSplitter(2)
  → [channel routing] → GainNode (inputGainNode, gain = dbToGain(inputSensitivity))
  → AnalyserNode (fftSize=8192, smoothingTimeConstant=0.15)
```

Channel routing:
- `0` or `1`: `splitter.connect(inputGain, channelIndex)`
- `mix`: two GainNodes at 0.5, both splitter outputs connected, then → inputGain

### Output graph

```
OscillatorNode (sine) → GainNode (outGainNode)
  → AnalyserNode (fftSize=1024, smoothingTimeConstant=0.15)
  → ChannelMerger(2) → MediaStreamDestination
Audio() { autoplay=true, srcObject=destination.stream }
```

Output channel routing into merger:
- `both`: connect gain to channels 0 and 1
- `0` or `1`: connect to that channel only

### `stopOutput()`

```js
oscNode.stop(); oscNode.disconnect(); oscNode = null;
outGainNode.disconnect(); outGainNode = null;
outMerger.disconnect(); outMerger = null;
outAnalyser.disconnect();   // don't nullify — reused
outputAudio.pause();
```

### `setSweepFreq(freq, gainValue)`

Used during sweep — changes frequency in-place with no oscillator restart:
```js
if (!oscNode || !outGainNode) { startOscillator(freq, gainValue); return; }
oscNode.frequency.setValueAtTime(freq, audioCtx.currentTime);
outGainNode.gain.setValueAtTime(gainValue, audioCtx.currentTime);
```

### Device change handlers

- `onInputDeviceChange`: stop old stream tracks → get new stream with exact deviceId → `buildInputGraph()` → `learnNoiseFloor()`
- `onInputChannelChange`: `connectInputChannel(ch)` → `learnNoiseFloor()`
- `onOutputDeviceChange`: `outputAudio.setSinkId(devId)` if available
- `onOutputChannelChange`: if oscillator running, disconnect and reconnect `outGainNode → outMerger` for new channel

---

## Noise Floor

Learned after: audio enable, input device change, input channel change. Never during sweep.

```
1. stopOutput()
2. Status: "Learning noise floor"
3. await sleep(120)
4. Take 14 RMS readings, each separated by sleep(25)
5. noiseRms = sqrt(sum(rms²) / 14)   ← average in power space
6. Cache: noiseFloorCache[`${deviceId}:${channel}`] = noiseRms
7. Status: "Noise floor -XX.X dBFS"
```

Noise subtraction (power space): `sqrt(max(0, level² - noise²))`

For sine-only detector noise: `detNoise = noise * sqrt(2 / fftSize)`

---

## Sweep

### Constants

```js
DEFAULT_START_FREQ = 500
DEFAULT_END_FREQ   = 7000
PRESCAN_HZ         = 300
POINT_COUNT        = 128
FFT_SIZE           = 8192
MIN_DB             = -96
```

### Timing

```js
msPerPoint      = (duration * 1000) / POINT_COUNT
fftRefreshMs    = Math.round(FFT_SIZE / audioCtx.sampleRate * 1000)  // ≈186ms @ 44100Hz
settleMs        = Math.max(msPerPoint * 0.3, fftRefreshMs)           // must be ≥ one full buffer flush
remainingMs     = Math.max(0, msPerPoint - settleMs)
effectiveReadings = Math.max(1, Math.min(readings, Math.floor(remainingMs / 20) + 1))
readMs          = effectiveReadings > 1 ? Math.floor(remainingMs / effectiveReadings) : 5
```

**Critical**: `settleMs ≥ fftRefreshMs` guarantees the analyser's circular buffer contains only the new frequency when measuring. For short sweep durations the actual sweep will take longer than specified — this is correct.

### Procedure

1. Clear `chartData`, `chartPointCloud`
2. Stop test tone if running
3. Disable START SWEEP; enable Stop button with `.sweeping` class (red)
4. `scanStart = Math.max(20, start - PRESCAN_HZ)`
5. `points = logspace(scanStart, end, POINT_COUNT)` — 128 log-spaced frequencies
6. Compute timing (above)
7. `startOscillator(points[0], firstGain)` — start oscillator **once** before the loop
8. Loop over all 128 points:
   - Compute `gain = 0.42 * dbToGain(-(attenuation * Math.max(0, log2(freq / start))))`
   - `setSweepFreq(freq, gain)` — update in-place, no restart
   - If `freq < start`: status `Calibrating` (orange), currentFreq shows red text `Calibrating`
   - If `freq >= start`: status `Sweeping` (red), currentFreq shows formatted Hz
   - Update progress %
   - `await sleep(settleMs)` ← wait for full buffer flush
   - Take `effectiveReadings` measurements, spaced by `readMs`
   - Average dB values; push to `chartData` only if `freq >= start`
   - `updateChart()` for live preview
9. `finally`: `stopOutput()`, re-enable buttons, clear `.sweeping`

After completed sweep: `currentFreq = '-- Hz'`, `progress = '100%'`, `activePeakSource = 'current'`, status `Sweep complete` (green).

After abort: status `Stopped`.

### Output attenuation formula

```js
octavesAboveStart = Math.max(0, Math.log2(freq / selectedStart))
gain = 0.42 * dbToGain(-(attenuationPerOctave * octavesAboveStart))
```

Pre-scan frequencies clamp octaves to 0 (no boost below start).

### Measurement reading

```js
async function measureLevel(freq, sineOnly, noise) {
  analyserNode.getFloatTimeDomainData(buf);  // Float32Array of fftSize

  if (sineOnly) {
    const w = (2π * freq) / audioCtx.sampleRate;
    sinSum = Σ buf[i] * sin(w*i)
    cosSum = Σ buf[i] * cos(w*i)
    detNoise = noise * sqrt(2 / N)
    lvl = (2/N) * hypot(sinSum, cosSum) / √2
    lvl = sqrt(max(0, lvl² - detNoise²))
  } else {
    lvl = rms(buf)
    lvl = sqrt(max(0, lvl² - noise²))
  }
  return lvl;
}
```

Convert to dB per reading: `gainToDb(Math.max(1e-10, lvl)) + inputSensitivity`

Average `effectiveReadings` dB values arithmetically.

---

## Test Signal

Toggles a 1000 Hz sine tone.

**Start**: `audioCtx.resume()` → `startOscillator(1000, 0.42)` → `isTestTone = true` → currentFreq `1.00 kHz` → status `Playing test tone` → Stop button enabled (not red) → button shows `pause` icon + `Stop test`

**Stop**: `stopOutput()` → `isTestTone = false` → currentFreq `-- Hz` → status `Audio ready` (green) → Stop button disabled → button shows `audio-lines` icon + `Test Signal`

---

## Chart (Chart.js)

`type: 'line'`, `animation: false`, `responsive: true`, `maintainAspectRatio: false`, `parsing: false` per dataset.

### X-axis

```js
type: 'logarithmic',
title: { display: true, text: 'Generated frequency', color: '#7a8099' },
grid: { color: '#2a2f3f' },
ticks: { color: '#7a8099', maxTicksLimit: 10 },
min: selectedStartFrequency,
max: Math.max(selectedEnd, maxVisibleSlotEnd)
```

### Y-axis

```js
title: { display: true, text: 'Normalized measured Input (approx.)', color: '#7a8099' },
grid: { color: '#2a2f3f' },
ticks: { color: '#7a8099', callback: v => `${v.toFixed(1)} dB` }
```

Y auto-fit: `margin = Math.max(1.5, (max - min) * 0.12)` applied above and below all visible data.

### Legend

```js
legend: {
  display: true,
  position: 'top',
  labels: {
    color: '#d4d8e8',
    boxWidth: 14, boxHeight: 2,
    padding: 12,
    font: { size: 11 },
    filter: item => item.text !== 'Current'  // hides current measurement from legend
  }
}
```

### Tooltip

```js
tooltip: {
  mode: 'index',
  intersect: false,
  backgroundColor: '#1e2230',
  borderColor: '#2a2f3f',
  borderWidth: 1,
  titleColor: '#ff9c00',
  bodyColor: '#d4d8e8',
  callbacks: {
    title: items => fmtHz(items[0].parsed.x),
    label: item  => ` ${item.dataset.label || 'Current'}: ${item.parsed.y.toFixed(2)} dB`
  }
}
```

### Data processing (`processData(rawPoints, startHz)`)

Applied fresh on every `updateChart()` call. Never stored — slots always store raw `{x, rawDb}` points.

1. Filter: keep only `p.x >= startHz`
2. Apply graph offset: `y = rawDb + graphOffsetDbPerOctave * log2(x / startHz)`
3. Gaussian smooth (in linear index space):
   ```js
   radius = Math.max(1, Math.round(smoothness * 1.5))
   sigma  = Math.max(1, radius / 2)
   weight[d] = exp(-(d² / (2 * sigma²)))
   ```
4. Normalize to first point and divide by 3: `y = (y - y[0]) / 3`

### Datasets

- **Current**: `label: 'Current'`, `borderColor: '#ff3b4f'`, `borderWidth: 2`, `pointRadius: 0`, `tension: 0.3`
- **Slots**: `label: slot.name`, `borderColor: SLOT_COLORS[i]`, same styling

`updateChart()` always ends with `updatePeakResponse()`.

---

## Peak Response

Displayed in Meters panel. Stacked right-aligned, both lines orange (`--accent-2`):

- **Line 1 — frequency** (top): `fmtHz(pts[peakIdx].x)` — `.peak-value` class: `font-size: 17px; font-weight: 800; color: var(--accent-2)`
- **Line 2 — dB** (bottom): `fmtDb(pts[peakIdx].y)` — inline style: `font-size: 13px; font-weight: 600; color: var(--accent-2); font-variant-numeric: tabular-nums`

Peak = point with maximum y in the processed/normalized curve.

Active source priority:
- `'current'` — uses `chartData` (set when sweep completes)
- `0..5` (number) — uses that slot (set when slot is stored, loaded, or show checkbox turned on)
- `null` / invisible / empty — show `--` / empty string

---

## Light/Dark Toggle

Button at far right of header (`margin-left: auto`). Shows `sun` icon in dark mode, `moon` in light mode.

On click:
1. `document.body.classList.toggle('light')`
2. Swap icon, call `renderIcons()`
3. Update Chart.js runtime colors:
   ```js
   gridColor  = isLight ? '#d0d4e0' : '#2a2f3f'
   labelColor = isLight ? '#5a6080' : '#7a8099'
   // apply to: x/y grid.color, x/y title.color, x/y ticks.color
   // legend.labels.color: isLight ? '#1a1d2e' : '#d4d8e8'
   // tooltip.backgroundColor: isLight ? '#ffffff' : '#1e2230'
   // tooltip.borderColor: isLight ? '#c8cdde' : '#2a2f3f'
   // tooltip.bodyColor: isLight ? '#1a1d2e' : '#d4d8e8'
   // tooltip.titleColor stays '#ff9c00'
   bodeChart.update()
   ```

---

## Measurement Slots

Six slots. The `#slotList` container is a **2-column grid**:

```css
#slotList {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 7px;
}
```

Slots 1–3 appear in the left column, slots 4–6 in the right column (natural DOM order).

### Colors

| Slot | Color |
|---|---|
| 1 | `#35d0a2` |
| 2 | `#4ea1ff` |
| 3 | `#ff9b45` |
| 4 | `#ffffff` |
| 5 | `#b76cff` |
| 6 | `#00d8ff` |

### Each slot contains

- Colored dot (10×10px circle, `border-radius: 50%`)
- Editable name `<input class="slot-name">` (transparent bg, underline border, `data-slot="i"`)
- Show/hide `<input type="checkbox">` (accent-colored to slot color, `data-slot-show="i"`)
- Action buttons: `Store`, `Save`, `<label>Load<input type="file" accept=".txt,.json"></label>`
- Metadata `<div class="slot-meta">`: `N pts · startHz–endHz` or `Empty`

### Store
Copies `chartData` → `slot.data`, sets `slot.start/end` from current UI, marks visible, sets `activePeakSource = i`, calls `updateChart()`.

### Save
Downloads `.txt` with pretty-printed JSON:
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
Parse JSON file, validate `{ x, rawDb }` numeric points, restore name/start/end, mark visible, `activePeakSource = i`, update slot name input, `updateChart()`.

### Show/hide checkbox
On check: `slot.visible = true`, `activePeakSource = i`, `updateChart()`.
On uncheck: `slot.visible = false`, `updateChart()`.

---

## Clear Latest

```
chartData = []; chartPointCloud = [];
currentFreq → '-- Hz'; progress → '0%'
if activePeakSource === 'current': activePeakSource = null
status: 'Latest measurement cleared'
updateChart()
```

---

## Export PNG

```js
const canvas = document.createElement('canvas');
canvas.width  = bodeChartCanvas.width  * 2;
canvas.height = bodeChartCanvas.height * 2;
ctx.fillStyle = '#15181f';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.drawImage(bodeChartCanvas, 0, 0, canvas.width, canvas.height);
// download as bode-plot-{ISO-timestamp}.png
```

---

## Help Overlay

Modal with semi-transparent backdrop (`rgba(0,0,0,0.7)`). Opens via Help button; closes via Close button, backdrop click, or Escape key.

Steps:
1. Place your pickup in the device.
2. Connect to your soundcard.
3. Enable Audio.
4. Select the audio input and output of your soundcard.
5. Launch the Test Signal tone to check for input signal.
6. Start the Sweep.
7. Adjust Smoothness and Measurement offset to taste.

Footer:
- `Build your own coil driver https://www.youtube.com/watch?v=SCeFtqdcS1Y`
- `This tool is for playing around and gives rough approximations. If you are serious about plotting your pickups, get the Bode Plotter at https://www.axetech.com`
- `©2026 Built with ❤️ by Niklaus Hirt`

---

## Utility Functions

```js
const dbToGain = db => Math.pow(10, db / 20);
const gainToDb = g  => 20 * Math.log10(Math.max(1e-10, g));
const rms = buf => Math.sqrt(buf.reduce((s,v) => s + v*v, 0) / buf.length);
const fmtHz = hz => hz >= 1000 ? `${(hz/1000).toFixed(2)} kHz` : `${Math.round(hz)} Hz`;
const fmtDb = db => `${db >= 0 ? '+' : ''}${db.toFixed(1)} dB`;
const sleep = ms => new Promise(r => setTimeout(r, ms));
function logspace(start, end, n) { /* n log-spaced values from start to end */ }
```

---

## Initialisation (bottom of `<script>`)

```js
initChart();
buildSlotUI();
renderIcons();
```

---

## Key Implementation Notes

1. **Never restart the oscillator between sweep points.** Call `setSweepFreq()` to update `frequency` and `gain` via `setValueAtTime`. Restarting creates a click/silence gap in the analyser's circular buffer that contaminates the next measurement.

2. **`settleMs` must equal at least one full FFT buffer refresh** (`FFT_SIZE / sampleRate * 1000` ≈ 186 ms at 44100 Hz). For short sweep durations (e.g. 10 s), the actual sweep takes longer than specified — this is intentional and correct.

3. **Lock-in magnitude is phase-independent**: `hypot(sinSum, cosSum)` gives the amplitude regardless of the unknown phase offset in the buffer. No phase synchronisation needed.

4. **Y values are divided by 3** after baseline normalisation. All peak response values use the post-division numbers.

5. **`sineOnly` is read from the checkbox at sweep start** — default unchecked (RMS mode). The lock-in detector is the optional mode.

6. **Slot data stores raw `{x, rawDb}` points only.** Processing (offset, smoothing, normalisation, ÷3) is applied fresh every `updateChart()`. Slots are never stored as processed values.

7. **The "Current" dataset is excluded from the chart legend** via the `filter` callback, but it does appear in tooltips with the label `Current`.

8. **Slots are displayed in a 2-column grid** inside the Measurement Slots panel: slots 1–3 in the left column, 4–6 in the right — using `grid-template-columns: 1fr 1fr` on `#slotList`.

9. **The Measurement Slots panel (`col-slots`) takes `1fr`** of the remaining width after the two fixed settings columns (280px + 330px). The graph row (`graph graph graph`) spans all three grid columns, giving it true full-viewport width.
