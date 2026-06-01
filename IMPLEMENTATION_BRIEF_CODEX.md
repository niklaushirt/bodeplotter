# Nick's Guitar Pickup Bode Plotter - Latest4 Implementation Spec

Implement the app as one self-contained `index.html` file with inline HTML, CSS, and JavaScript. Use CDN scripts only for:

- Chart.js `https://cdn.jsdelivr.net/npm/chart.js@4.4.7/dist/chart.umd.min.js`
- Lucide icons `https://unpkg.com/lucide@latest/dist/umd/lucide.min.js`

Do not use a build step. Do not use Tailwind CDN. Run from localhost or a secure context so browser audio permissions work.

## Purpose

Build a dark-themed browser tool for rough guitar pickup frequency-response plotting. It generates a sine sweep through a selected soundcard output, measures a selected soundcard input, corrects measurements with calibration and learned noise floor, optionally uses sine-only filtering, displays a smoothed Bode-style response graph, and manages six saved measurement slots.

## Theme And Colors

CSS variables:

- `--accent: #004cfc`
- `--accent-2: #ff9c00`

Use `--accent-2` for:

- H1 title text
- Peak Response value
- START SWEEP button background
- Calibrating status
- help links

Use red for:

- Sweeping status
- Stop button while sweep is running

## Header

Browser title and H1:

- `Nick's Guitar Pickup Bode Plotter`
- H1 color: `var(--accent-2)`

Subtitle:

- `Sweep a compensated sine signal through a pickup test rig, measure the soundcard input, and plot the normalized frequency response.`

Header buttons, in this order:

1. `Enable Audio`, green/primary style, Lucide `power`
2. `Test Signal`, Lucide `audio-lines`, disabled until audio is enabled
3. `START SWEEP`, orange style using `--accent-2`, Lucide `chart-spline`, disabled until audio is enabled
4. Stop icon button, Lucide `square`, immediately to the right of START SWEEP
5. `Export PNG`, Lucide `download`
6. `Clear Latest`, Lucide `eraser`
7. `Help`, Lucide `circle-help`

Stop button behavior:

- Disabled unless busy.
- Red only while a sweep is running.
- Not red during Test Signal.

## Layout

Use full browser width; no app max-width cap.

Desktop grid:

- `280px 330px minmax(0, 1fr) 300px`
- Column 1: `Meters`, then `Audio routing`
- Column 2: `Sweep settings`, then `Graph settings`
- Column 3: graph
- Column 4: `Measurement slots`

Responsive:

- `1121px` to `1500px`: `280px minmax(0, 1fr) 300px`; stack routing/settings in the left column, graph in middle, slots right.
- `721px` to `1120px`: two columns, graph spans full width.
- Under `720px`: one column.

Graph canvas:

- Desktop min height `860px`
- Mobile min height `520px`

## Defaults And Labels

- Start frequency: `500 Hz`
- End frequency: `7000 Hz`
- `Scan Durtation`: default `10 s`, range `1..60`
- Readings per point: default `15`, range `1..50`
- Input sensitivity: default `0 dB`, range `-24..+24`, step `0.5`
- Output attenuation: default `6 dB/oct`, range `0..6`, step `0.1`
- `Sine-only filter`: checked by default
- `Graph smoothness`: default `5`, range `1..10`
- `Graph offset`: default `4 dB/oct`, range `0..12`, step `0.1`

Keep the exact visible typo `Scan Durtation`.

## Audio Routing

Controls:

- Input device select
- Input channel:
  - `Channel 1 / left`, value `0`
  - `Channel 2 / right`, value `1`
  - `Mono mix`, value `mix`
- Output device select
- Output channel:
  - `Both channels`, value `both`
  - `Channel 1 / left`, value `0`
  - `Channel 2 / right`, value `1`

When enabling audio:

- Request audio with `echoCancellation: false`, `noiseSuppression: false`, `autoGainControl: false`, `channelCount: 2`.
- Enumerate devices after permission.
- Select the first real input and first real output device when available.
- If selecting the first real input requires changing from the temporary permission stream, stop the stream and request the selected device exactly.
- Build input graph, output routing, meters, and learn noise floor.
- If `setSinkId` is available, route output to the selected output device. Otherwise use system default and show a note.

## Web Audio Graphs

Input graph:

- `MediaStreamSource`
- `ChannelSplitter(2)`
- `GainNode`
- `AnalyserNode`
  - `fftSize = 8192`
  - `smoothingTimeConstant = 0.15`

Input channel routing:

- Channel `0` or `1`: connect selected splitter output.
- `mix`: connect both splitter outputs to a gain node at `0.5`, then into input gain.

Output graph:

- `MediaStreamDestination`
- `Audio()` element with `autoplay = true` and `srcObject = destination.stream`
- `OscillatorNode` type `sine`
- `GainNode`
- `ChannelMerger(2)`
- Output analyser with `fftSize = 1024`

Output channel routing:

- `both`: connect to channels 0 and 1
- `0` or `1`: connect only to selected channel

Stopping output must:

- Stop oscillator if present.
- Disconnect oscillator, gain, merger, and analyser.
- Pause the output audio element.
- Clear signal state.

## Noise Floor

Learn noise floor after audio enable and after input channel changes.

Procedure:

- Do not learn during sweep.
- Stop active generated signal first.
- Status `Learning noise floor`.
- Wait about `120ms`.
- Take `14` RMS readings separated by about `25ms`.
- Average in power space: `sqrt(sum(rms*rms)/samples)`.
- Cache by key `{inputDevice || "default"}:{inputChannel}`.
- Status like `Noise floor -72.3 dBFS`.

Subtract noise in power space:

```js
Math.sqrt(Math.max(0, level * level - noise * noise))
```

For sine-only filter noise:

```js
detectorNoise = activeNoiseFloor * Math.sqrt(2 / sampleBuffer.length)
```

## Sweep

Constants:

```js
DEFAULT_START_FREQ = 500
DEFAULT_END_FREQ = 7000
PRESCAN_HZ = 300
POINT_COUNT = 128
FFT_SIZE = 8192
MIN_DB = -96
```

Actual scan starts `300 Hz` below selected start:

```js
scanStart = Math.max(20, start - 300)
```

Generate 128 logarithmic points from `scanStart` to selected end.

During the pre-start lead-in:

- Keep generating and measuring.
- Do not show frequencies below selected start frequency on the graph.
- Status `Calibrating`, colored orange.
- Current frequency readout must show `Calibrating` in red instead of the generated frequency.

At/above selected start:

- Status `Sweeping`, colored red.
- Current frequency readout switches back to the actual generated frequency.

Use `try/finally` so completed or aborted sweep always stops the generator and pauses audio output.

After completed sweep:

- Active Peak Response source becomes current measurement.
- Current frequency `-- Hz`
- Progress `100%`
- Final chart update.
- Status `Sweep complete`.

If aborted:

- Status `Stopped`.

## Output Attenuation

Output attenuation slider controls the complete per-octave slope:

- `0 dB/oct`: flat output, no attenuation
- `6 dB/oct`: voltage halves per octave above selected start

Formula:

```js
const octavesAboveStart = Math.max(0, Math.log2(frequency / selectedStartFrequency));
gain = 0.42 * dbToGain(-(attenuationPerOctave * octavesAboveStart));
```

Below selected start, clamp octaves to `0` so the pre-scan does not boost output.

## Test Signal

The `Test Signal` button toggles a 1000 Hz sine tone.

Start:

- Resume audio context.
- Generate 1000 Hz.
- Current frequency `1.00 kHz`.
- Button changes to pause icon + `Stop test`.
- Status `Playing test tone`.

Stop:

- Stop oscillator and pause audio output.
- Button returns to audio-lines icon + `Test Signal`.
- Status `Audio ready`.

## Measurement Reading

Each sweep point reading:

- Pull time-domain input samples.
- If `Sine-only filter` is checked, use lock-in projection at generated frequency:

```js
level = (2 / N) * Math.hypot(sinSum, cosSum) / Math.SQRT2
```

- Otherwise use RMS.
- Subtract learned noise floor.
- Convert to dB.
- Add input sensitivity.

Average repeated readings arithmetically in dB.

## Chart

Use Chart.js line chart with dynamic datasets.

Axis labels:

- X: `Generated frequency`
- Y: `Normalized measured Input (approx.)`

X-axis:

- Logarithmic.
- During sweep and for loaded/activated slots, minimum is selected start frequency, not pre-scan frequency.
- Maximum is the greater of selected end frequency and visible slot end frequencies.

Y-axis:

- dB ticks.
- Auto-fit all displayed datasets with margin `max(1.5, (max - min) * 0.12)`.
- After baseline normalization, divide all displayed y values by `3`.
- Peak Response reports this scaled graph value.

Current measurement display:

- Current/live measurement average line is red `#ff3b4f`.
- Current point cloud is hidden by default while measuring.

Stored slot display:

- Visible slots render as smoothed average curves in their assigned colors.

Data processing:

1. Apply graph offset:

```js
rawDb += graphOffsetDbPerOctave * Math.log2(point.x / selectedStartFrequency)
```

2. Smooth with Gaussian weighted average:

```js
radius = Math.max(1, Math.round(smoothness * 1.5))
sigma = Math.max(1, radius / 2)
weight = Math.exp(-(offset * offset) / (2 * sigma * sigma))
```

3. Hide data below selected start frequency.
4. Normalize so `0 dB` is the first plotted point at or above selected start frequency.
5. Divide normalized y values by `3`.

## Peak Response

Peak Response value is orange.

Calculate Peak Response from active measurement source:

- Completed sweep: current measurement
- Loaded slot: loaded slot
- Slot show checkbox turned on: that slot

If active source is hidden or empty, show `--`.

Peak is calculated from the displayed normalized/scaled average curve.

## Measurement Slots

There are six slots. Each has:

- Editable name
- Show/hide checkbox
- `Store`
- `Save`
- `Load`
- Metadata line

Colors:

- Slot 1: `#35d0a2`
- Slot 2: `#4ea1ff`
- Slot 3: `#ff9b45`
- Slot 4: `#ffffff`
- Slot 5: `#b76cff`
- Slot 6: `#00d8ff`

Store:

- Copies current `chartData` into the slot.
- Stores selected start/end metadata.
- Marks slot visible.

Save:

- Download as `.txt` with pretty-printed JSON:

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

Load:

- Parse JSON text file.
- Validate numeric `{ x, rawDb }` points.
- Restore name/start/end.
- Mark slot visible.
- Set active Peak Response source to this slot.
- Update graph.

## Clear Latest

The `Clear Latest` button:

- Clears only current/latest measurement data.
- Leaves all six slots untouched.
- Sets final-average-only display so graph starts at selected start frequency.
- Resets current frequency to `-- Hz`.
- Resets progress to `0%`.
- Clears current active peak source if it was current measurement.
- Redraws graph.
- Status `Latest measurement cleared`.

## Export PNG

The `Export PNG` button:

- Creates a new canvas at `2x` chart canvas size.
- Fills background `#15181f`.
- Draws the chart canvas.
- Downloads as `bode-plot-{timestamp}.png`.

## Help Overlay

Help button opens a modal overlay. It closes via Close button, backdrop click, or Escape.

Help content:

1. Place your pickup in the device.
2. Connect to your soundcard.
3. Enable Audio.
4. Select the audio input and output of your soundcard.
5. Launch the Test Signal tone to check for input signal.
6. Start the Sweep.
7. Adjust Smoothness and Measurement offset to taste.

Bottom links/text:

- `Build your own coil driver https://www.youtube.com/watch?v=SCeFtqdcS1Y`
- `This tool is for playing around and gives rough approximations. If you are serious about plotting your pickups, get the Bode Plotter at https://www.axetech.com`
- `©2026 Built with ❤️ by Niklaus Hirt`

## Verification Checklist

- Inline JavaScript parses.
- Page loads from local HTTP server without console errors.
- H1 reads `Nick's Guitar Pickup Bode Plotter`, color `#ff9c00`.
- CSS variables are `--accent: #004cfc` and `--accent-2: #ff9c00`.
- Default end frequency is `7000`.
- Visible labels include `Enable Audio`, `Test Signal`, `START SWEEP`, `Clear Latest`, `Sine-only filter`, `Scan Durtation`, `Graph smoothness`, `Graph offset`.
- START SWEEP is orange and Stop is immediately to its right.
- Stop button turns red only while a sweep is running.
- During pre-start calibration, Current frequency readout shows red `Calibrating`.
- Help overlay contains all seven steps, two links, and copyright line.
- Calibrating status is orange; Sweeping status is red.
- Current points are hidden by default while measuring.
- Y-axis label reads `Normalized measured Input (approx.)`.
- Y plotted/reported values are divided by `3`.
- Six slots render, store/save/load/show/hide work.
- Loaded/activated slots clamp graph x minimum to selected start frequency.
- Clear Latest removes only current measurement, not slots, and graph starts at selected start frequency afterward.
- Stopping Test Signal or sweep stops audio output.
