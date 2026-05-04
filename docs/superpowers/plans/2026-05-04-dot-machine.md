# DOT MACHINE Implementation Plan

> **For agentic workers:** Implement chunks in order. Each task is bite-sized (2–5 min). Manual browser verification replaces a test framework — see "Verification approach" below. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Build a single-file, vanilla JS halftone tool that turns images and videos into dotted renders, with brand-aware controls and multi-format export.

**Architecture:** One `index.html` file containing markup, CSS, and inline JS organized in clearly delimited sections. Canvas 2D for rendering. ES modules NOT used — keeps `file://` double-click working in every browser. Lazy-loaded `gif.js` is the only third-party dep.

**Tech Stack:** HTML / CSS / vanilla JS, Canvas 2D API, MediaRecorder API, Clipboard API, Geist fonts (Vercel CDN), `gif.js` (CDN, on demand).

**Spec:** [`docs/superpowers/specs/2026-05-04-dot-machine-design.md`](../specs/2026-05-04-dot-machine-design.md)

---

## File Structure

```
DOT MACHINE/
├── index.html         # everything — markup, CSS, inline JS in 8 sections
├── README.md          # one-paragraph usage notes + localhost tip
└── docs/
    └── superpowers/
        ├── specs/2026-05-04-dot-machine-design.md
        └── plans/2026-05-04-dot-machine.md (this file)
```

Inside `index.html`, the inline `<script>` is organized as:

```js
// ============================================================
// SECTION 1: STATE      — central reactive state object
// SECTION 2: DOM        — DOM refs, font loading
// SECTION 3: MEDIA      — file load, drag/drop, video lifecycle
// SECTION 4: RENDER     — halftone pipeline (luminance buffer → dots)
// SECTION 5: CONTROLS   — bind sliders/checkboxes/colors to state
// SECTION 6: EXPORT_IMG — PNG / JPEG / SVG / clipboard
// SECTION 7: EXPORT_VID — MediaRecorder (WebM/MP4) + GIF
// SECTION 8: BOOT       — wire everything, render initial empty state
// ============================================================
```

## Verification Approach

- **Manual:** open `index.html` (or `http://localhost:8080`) in Safari/Chrome, perform the described action, confirm the described outcome.
- **Asserts:** at the bottom of `<script>`, a `runDebugAsserts()` function with `console.assert(...)` calls runs only if `location.search.includes('debug')`. Used for pure helpers (luminance math, brightness/contrast curves, format probing logic).
- **Commit cadence:** one commit per task minimum. Squash later if needed.

---

## Chunk 1: Scaffold + Theme

### Task 1.1: HTML skeleton + Geist fonts

**Files:**
- Create: `index.html`

- [ ] **Step 1: Write the skeleton**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>DOT MACHINE</title>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link href="https://fonts.googleapis.com/css2?family=Geist:wght@400;500;600&family=Geist+Mono:wght@400;500&display=swap" rel="stylesheet" />
  <style>/* SECTION: base */</style>
</head>
<body>
  <main id="app">
    <aside id="rail"></aside>
    <section id="stage">
      <div id="empty">drop an image or video here</div>
      <canvas id="canvas" hidden></canvas>
    </section>
  </main>
  <script>/* sections 1-8 */</script>
</body>
</html>
```

- [ ] **Step 2: Verify**

Open `index.html` in Safari. Expect: blank page, "drop an image or video here" centered, no console errors. Geist font is loaded (inspect `font-family` on body via DevTools).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: scaffold index.html with Geist font wiring"
```

---

### Task 1.2: Base CSS — layout, theme, typography

**Files:**
- Modify: `index.html` (the `<style>` block)

- [ ] **Step 1: Add the base CSS**

```css
:root {
  --core: #CEADFF;
  --analog: #F6B9A2;
  --rush: #F2ADFF;
  --keyframe: #ADE7FF;
  --canvas: #FFF7FA;
  --gaffer: #1C0802;

  --bg: var(--canvas);
  --fg: var(--gaffer);
  --hairline: rgba(28, 8, 2, 0.08);
}

* { box-sizing: border-box; }
html, body { margin: 0; height: 100%; }

body {
  background: var(--bg);
  color: var(--fg);
  font-family: 'Geist', system-ui, sans-serif;
  font-size: 13px;
  -webkit-font-smoothing: antialiased;
}

#app {
  display: grid;
  grid-template-columns: 280px 1fr;
  height: 100vh;
  min-width: 720px;
}

#rail {
  border-right: 1px solid var(--hairline);
  padding: 24px 20px;
  overflow-y: auto;
}

#stage {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 32px;
  overflow: hidden;
}

#empty {
  font-family: 'Geist Mono', ui-monospace, monospace;
  color: rgba(28, 8, 2, 0.4);
  pointer-events: none;
}

#canvas {
  max-width: 100%;
  max-height: 100%;
  display: block;
}

.mono { font-family: 'Geist Mono', ui-monospace, monospace; }
```

- [ ] **Step 2: Verify**

Reload. Expect: `Canvas` background color (`#FFF7FA`), 280px left rail with hairline border, "drop here" message in Geist Mono, centered in the stage area.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: base CSS, brand color tokens, two-column layout"
```

---

### Task 1.3: Rail header

**Files:**
- Modify: `index.html` (rail markup, CSS for header + section dividers)

- [ ] **Step 1: Add rail header markup inside `#rail`**

```html
<header id="rail-head">
  <h1 class="title">DOT MACHINE</h1>
  <p class="subtitle mono">halftone tool</p>
</header>
```

- [ ] **Step 2: Add CSS**

```css
#rail-head { margin-bottom: 28px; }
.title { font-size: 14px; font-weight: 600; letter-spacing: 0.02em; margin: 0; }
.subtitle { font-size: 11px; color: rgba(28, 8, 2, 0.5); margin: 4px 0 0; }

.section { padding: 16px 0; border-top: 1px solid var(--hairline); }
.section:first-of-type { border-top: 0; padding-top: 0; }
.section-label { font-size: 10px; letter-spacing: 0.08em; text-transform: uppercase; color: rgba(28, 8, 2, 0.5); margin: 0 0 12px; }
```

- [ ] **Step 3: Verify**

Reload. Expect: title "DOT MACHINE" + "halftone tool" subtitle in mono.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: rail header"
```

---

## Chunk 2: Media Input + Lifecycle

### Task 2.1: Central state object (Section 1)

**Files:**
- Modify: `index.html` (`<script>` block)

- [ ] **Step 1: Add State section**

```js
// === SECTION 1: STATE ===
const state = {
  source: null,        // { kind: 'image'|'video', el: HTMLImageElement|HTMLVideoElement, width, height, durationOk }
  cellSize: 12,
  dotScale: 1.0,
  brightness: 0.0,
  contrast: 0.0,
  dotOpacity: 1.0,
  invert: false,
  dotColor: '#1C0802',
  bgColor: '#FFF7FA',
  dotTransparent: false,
  bgTransparent: false,
};

const listeners = new Set();
function setState(patch) {
  Object.assign(state, patch);
  listeners.forEach(fn => fn(state));
}
function onState(fn) { listeners.add(fn); fn(state); }
```

- [ ] **Step 2: Verify**

Reload, open DevTools, type `state` → expect the object. No errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: central state object with subscribe/setState"
```

---

### Task 2.2: File picker + drag-and-drop (Section 3, partial)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add hidden file input + drop binding**

In markup, add inside `#stage`:
```html
<input id="file" type="file" accept="image/*,video/*" hidden />
```

In script, add:
```js
// === SECTION 3: MEDIA ===
const fileInput = document.getElementById('file');
const stage = document.getElementById('stage');
const empty = document.getElementById('empty');

stage.addEventListener('click', () => { if (!state.source) fileInput.click(); });
fileInput.addEventListener('change', e => {
  const f = e.target.files?.[0];
  if (f) loadFile(f);
});

['dragenter', 'dragover'].forEach(evt =>
  stage.addEventListener(evt, e => { e.preventDefault(); stage.classList.add('drop-active'); })
);
['dragleave', 'drop'].forEach(evt =>
  stage.addEventListener(evt, e => { e.preventDefault(); stage.classList.remove('drop-active'); })
);
stage.addEventListener('drop', e => {
  const f = e.dataTransfer?.files?.[0];
  if (f) loadFile(f);
});

function loadFile(file) {
  console.log('loadFile', file.type, file.name);
  // Implemented in next tasks
}
```

Add CSS:
```css
#stage.drop-active { background: rgba(173, 231, 255, 0.15); }
```

- [ ] **Step 2: Verify**

Reload. Click empty stage → file picker opens. Drag a file over → background tint appears. Drop a file → console logs `loadFile image/jpeg foo.jpg`. No errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: file picker and drag-and-drop on stage"
```

---

### Task 2.3: Image loading via createImageBitmap

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Implement `loadFile` for images**

Replace the `loadFile` stub with:
```js
async function loadFile(file) {
  releaseSource();
  if (file.type.startsWith('image/')) {
    const bitmap = await createImageBitmap(file, { imageOrientation: 'from-image' });
    state.source = { kind: 'image', el: bitmap, width: bitmap.width, height: bitmap.height, durationOk: false };
    onSourceReady();
  } else if (file.type.startsWith('video/')) {
    await loadVideo(file);
  } else {
    alert('Unsupported file type');
  }
}

function releaseSource() {
  if (!state.source) return;
  if (state.source.kind === 'video') {
    state.source.el.pause();
    URL.revokeObjectURL(state.source.el.src);
  } else if (state.source.kind === 'image' && state.source.el.close) {
    state.source.el.close();
  }
  state.source = null;
}

async function loadVideo(file) {
  // Implemented in 2.4
}

function onSourceReady() {
  empty.hidden = true;
  document.getElementById('canvas').hidden = false;
  render(); // defined in chunk 3 stub
}

function render() {
  // Real implementation in chunk 3. Stub: just clear to bg.
  const c = document.getElementById('canvas');
  c.width = state.source.width;
  c.height = state.source.height;
  const ctx = c.getContext('2d');
  ctx.fillStyle = state.bgColor;
  ctx.fillRect(0, 0, c.width, c.height);
}
```

- [ ] **Step 2: Verify**

Drop a JPG. Expect: empty state hides, canvas appears filled with `Canvas` color, sized to the image. No errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: image loading via createImageBitmap, source lifecycle"
```

---

### Task 2.4: Video loading + first-frame draw

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Implement `loadVideo`**

```js
async function loadVideo(file) {
  const url = URL.createObjectURL(file);
  const v = document.createElement('video');
  v.src = url;
  v.muted = true;
  v.loop = true;
  v.playsInline = true;
  await new Promise((res, rej) => {
    v.addEventListener('loadeddata', res, { once: true });
    v.addEventListener('error', rej, { once: true });
  });
  state.source = {
    kind: 'video',
    el: v,
    width: v.videoWidth,
    height: v.videoHeight,
    durationOk: isFinite(v.duration) && v.duration > 0,
  };
  v.play().catch(() => { /* autoplay sometimes blocked; user can resume */ });
  onSourceReady();
  startVideoLoop();
}

let rafId = null;
function startVideoLoop() {
  cancelAnimationFrame(rafId);
  const tick = () => {
    if (state.source?.kind === 'video') {
      render();
      rafId = requestAnimationFrame(tick);
    }
  };
  rafId = requestAnimationFrame(tick);
}
```

- [ ] **Step 2: Verify**

Drop an MP4. Expect: canvas sized to video dimensions, filled with `Canvas` color (still stub render). Console: no errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: video loading with looped muted playback and rAF tick"
```

---

## Chunk 3: Render Pipeline

### Task 3.1: Pure helpers — brightness/contrast curve

**Files:**
- Modify: `index.html` (Section 4)

- [ ] **Step 1: Add Section 4 with helpers**

```js
// === SECTION 4: RENDER ===

// Map [0,1] luminance through brightness (-1..1) and contrast (-1..1) curves.
// brightness shifts linearly; contrast pivots around 0.5.
function adjust(L, b, c) {
  L += b;                          // brightness shift
  L = (L - 0.5) * (1 + c) + 0.5;   // contrast around midpoint
  return Math.max(0, Math.min(1, L));
}

function rgbaToLuma(r, g, b) {
  // Rec. 709 coefficients
  return (0.2126 * r + 0.7152 * g + 0.0722 * b) / 255;
}
```

- [ ] **Step 2: Add debug asserts at bottom of script**

```js
function runDebugAsserts() {
  console.assert(adjust(0.5, 0, 0) === 0.5, 'identity midpoint');
  console.assert(adjust(0.5, 0.2, 0) > 0.5, 'brightness positive');
  console.assert(adjust(0.5, -0.2, 0) < 0.5, 'brightness negative');
  console.assert(adjust(0.0, 0, 1.0) === 0.0, 'contrast preserves 0');
  console.assert(adjust(1.0, 0, 1.0) === 1.0, 'contrast preserves 1');
  console.assert(adjust(0.5, 0, 1.0) === 0.5, 'contrast preserves midpoint');
  console.assert(adjust(2, 0, 0) === 1, 'clamp upper');
  console.assert(adjust(-2, 0, 0) === 0, 'clamp lower');
  console.assert(Math.abs(rgbaToLuma(255, 255, 255) - 1) < 1e-9, 'white luma');
  console.assert(rgbaToLuma(0, 0, 0) === 0, 'black luma');
  console.log('asserts: ok');
}
if (location.search.includes('debug')) runDebugAsserts();
```

- [ ] **Step 3: Verify**

Open `index.html?debug`. Expect: console shows `asserts: ok`, no failures.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: brightness/contrast/luma helpers + debug asserts"
```

---

### Task 3.2: Luminance buffer

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add buffer builder**

```js
const bufCanvas = document.createElement('canvas');
const bufCtx = bufCanvas.getContext('2d', { willReadFrequently: true });

function buildLumaBuffer(source, cols, rows) {
  bufCanvas.width = cols;
  bufCanvas.height = rows;
  bufCtx.clearRect(0, 0, cols, rows);
  bufCtx.drawImage(source, 0, 0, cols, rows);
  const { data } = bufCtx.getImageData(0, 0, cols, rows);
  const out = new Float32Array(cols * rows);
  for (let i = 0, j = 0; i < data.length; i += 4, j++) {
    out[j] = rgbaToLuma(data[i], data[i + 1], data[i + 2]);
  }
  return out;
}
```

- [ ] **Step 2: Verify (manual peek)**

Add temporary log inside `render()` calling `buildLumaBuffer(state.source.el, 10, 10)` and `console.log` first 5 values. Drop image → expect 5 numbers between 0 and 1.

Remove the temporary log before committing.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: luminance buffer via bilinear downscale"
```

---

### Task 3.3: Real render — draw dots from buffer

**Files:**
- Modify: `index.html` (replace stub `render`)

- [ ] **Step 1: Replace stub `render`**

```js
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

const PREVIEW_MAX = 1280; // longest side cap during preview/playback

function render() {
  const src = state.source;
  if (!src) return;

  // Compute preview size (capped)
  const scale = Math.min(1, PREVIEW_MAX / Math.max(src.width, src.height));
  const W = Math.round(src.width * scale);
  const H = Math.round(src.height * scale);

  if (canvas.width !== W) canvas.width = W;
  if (canvas.height !== H) canvas.height = H;

  const cs = state.cellSize;
  const cols = Math.max(1, Math.floor(W / cs));
  const rows = Math.max(1, Math.floor(H / cs));

  const luma = buildLumaBuffer(src.el, cols, rows);

  // Background
  if (state.bgTransparent) {
    ctx.clearRect(0, 0, W, H);
  } else {
    ctx.fillStyle = state.bgColor;
    ctx.fillRect(0, 0, W, H);
  }

  if (state.dotTransparent) {
    // Cut dots out of background → composite operation
    ctx.globalCompositeOperation = 'destination-out';
    ctx.fillStyle = '#000';
    ctx.globalAlpha = 1;
  } else {
    ctx.globalCompositeOperation = 'source-over';
    ctx.fillStyle = state.dotColor;
    ctx.globalAlpha = state.dotOpacity;
  }

  const halfCell = cs / 2;
  for (let j = 0; j < rows; j++) {
    for (let i = 0; i < cols; i++) {
      const L0 = luma[j * cols + i];
      const L = adjust(L0, state.brightness, state.contrast);
      const darkness = state.invert ? L : 1 - L;
      const r = darkness * halfCell * state.dotScale;
      if (r < 0.4) continue; // skip imperceptible dots
      ctx.beginPath();
      ctx.arc(i * cs + halfCell, j * cs + halfCell, r, 0, Math.PI * 2);
      ctx.fill();
    }
  }

  ctx.globalCompositeOperation = 'source-over';
  ctx.globalAlpha = 1;
}
```

- [ ] **Step 2: Verify**

Drop a portrait JPG. Expect: visible halftone, dark areas = bigger black dots, light areas = small dots, on `Canvas` background. Drop a video → motion preview is dotted.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: halftone render pipeline (dots from luma buffer)"
```

---

### Task 3.4: Re-render on state change

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Subscribe render to state**

At end of Section 4:
```js
onState(() => { if (state.source?.kind === 'image') render(); });
// for video, render() is called every frame by startVideoLoop
```

- [ ] **Step 2: Verify**

In DevTools, with an image loaded, run `setState({ cellSize: 30 })`. Expect: dots resize immediately. Run `setState({ invert: true })`. Expect: bright/dark inverts.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: image rerender on state change"
```

---

## Chunk 4: Controls Panel

### Task 4.1: Slider + numeric value pattern

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add CSS for control rows**

```css
.row { display: grid; grid-template-columns: 1fr auto; align-items: center; gap: 8px; margin-bottom: 12px; }
.row label { font-size: 11px; color: var(--fg); }
.row .val { font-size: 11px; color: rgba(28, 8, 2, 0.7); min-width: 56px; text-align: right; font-variant-numeric: tabular-nums; }
.row input[type='range'] { grid-column: 1 / -1; appearance: none; width: 100%; height: 2px; background: var(--hairline); border-radius: 1px; }
.row input[type='range']::-webkit-slider-thumb { appearance: none; width: 12px; height: 12px; border-radius: 50%; background: var(--gaffer); cursor: pointer; }
.row input[type='range']::-moz-range-thumb { width: 12px; height: 12px; border-radius: 50%; background: var(--gaffer); border: 0; cursor: pointer; }
```

- [ ] **Step 2: Add helper `mountSlider` in Section 5**

```js
// === SECTION 5: CONTROLS ===

function mountSlider({ container, key, label, min, max, step, format }) {
  const row = document.createElement('div');
  row.className = 'row';
  row.innerHTML = `
    <label>${label}</label>
    <span class="val mono"></span>
    <input type="range" min="${min}" max="${max}" step="${step}" value="${state[key]}" />
  `;
  const valEl = row.querySelector('.val');
  const input = row.querySelector('input');
  const display = () => { valEl.textContent = format(state[key]); };
  input.addEventListener('input', () => {
    setState({ [key]: parseFloat(input.value) });
  });
  onState(() => { input.value = state[key]; display(); });
  container.appendChild(row);
}
```

- [ ] **Step 3: Verify**

No visible change yet. No errors. (Used by next task.)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: slider control component"
```

---

### Task 4.2: Mount the six numeric controls

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add a controls section in markup inside `#rail`**

```html
<section class="section">
  <p class="section-label">geometry</p>
  <div id="ctrl-geom"></div>
</section>
<section class="section">
  <p class="section-label">tone</p>
  <div id="ctrl-tone"></div>
</section>
<section class="section">
  <p class="section-label">dots</p>
  <div id="ctrl-dots"></div>
</section>
```

- [ ] **Step 2: Mount sliders in BOOT (Section 8)**

```js
// === SECTION 8: BOOT ===
const fmtPx = v => `${Math.round(v)} px`;
const fmtFloat = v => `${v >= 0 ? '+' : ''}${v.toFixed(2)}`;
const fmtPct = v => `${Math.round(v * 100)}%`;

mountSlider({ container: document.getElementById('ctrl-geom'), key: 'cellSize', label: 'cell size', min: 4, max: 80, step: 1, format: fmtPx });
mountSlider({ container: document.getElementById('ctrl-geom'), key: 'dotScale', label: 'dot scale', min: 0, max: 1.5, step: 0.01, format: v => v.toFixed(2) });

mountSlider({ container: document.getElementById('ctrl-tone'), key: 'brightness', label: 'brightness', min: -1, max: 1, step: 0.01, format: fmtFloat });
mountSlider({ container: document.getElementById('ctrl-tone'), key: 'contrast', label: 'contrast', min: -1, max: 1, step: 0.01, format: fmtFloat });

mountSlider({ container: document.getElementById('ctrl-dots'), key: 'dotOpacity', label: 'opacity', min: 0, max: 1, step: 0.01, format: fmtPct });
```

- [ ] **Step 3: Verify**

Reload. Expect: 5 sliders in 3 sections with labeled values in mono. Drop an image. Move sliders → halftone updates live.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: mount cell/scale/brightness/contrast/opacity sliders"
```

---

### Task 4.3: Invert checkbox

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add helper and mount**

In Section 5:
```js
function mountCheckbox({ container, key, label }) {
  const row = document.createElement('label');
  row.className = 'row check';
  row.innerHTML = `<span>${label}</span><input type="checkbox" />`;
  const input = row.querySelector('input');
  input.checked = state[key];
  input.addEventListener('change', () => setState({ [key]: input.checked }));
  onState(() => { input.checked = state[key]; });
  container.appendChild(row);
}
```

CSS:
```css
.row.check { grid-template-columns: 1fr auto; cursor: pointer; }
.row.check input { accent-color: var(--gaffer); }
```

In BOOT, after the dots section sliders:
```js
mountCheckbox({ container: document.getElementById('ctrl-dots'), key: 'invert', label: 'invert' });
```

- [ ] **Step 2: Verify**

Tick "invert" with image loaded → light areas become big dots, dark areas small.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: invert toggle"
```

---

### Task 4.4: Color picker + brand swatches + transparent checkbox

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add markup for color sections**

```html
<section class="section">
  <p class="section-label">dot color</p>
  <div id="ctrl-dot-color"></div>
</section>
<section class="section">
  <p class="section-label">background</p>
  <div id="ctrl-bg-color"></div>
</section>
```

- [ ] **Step 2: Add helper `mountColor`**

In Section 5:
```js
const PALETTE = [
  { name: 'Core',     hex: '#CEADFF' },
  { name: 'Analog',   hex: '#F6B9A2' },
  { name: 'Rush',     hex: '#F2ADFF' },
  { name: 'Keyframe', hex: '#ADE7FF' },
  { name: 'Canvas',   hex: '#FFF7FA' },
  { name: 'Gaffer',   hex: '#1C0802' },
];

function mountColor({ container, colorKey, transparentKey, otherTransparentKey }) {
  const wrap = document.createElement('div');
  wrap.className = 'color-block';
  wrap.innerHTML = `
    <div class="color-row">
      <input type="color" value="${state[colorKey]}" />
      <span class="hex mono"></span>
    </div>
    <div class="swatches"></div>
    <label class="row check trans"><span>transparent</span><input type="checkbox" /></label>
    <p class="hint mono" hidden>can't make both transparent</p>
  `;
  const picker = wrap.querySelector('input[type="color"]');
  const hexEl = wrap.querySelector('.hex');
  const swatchesEl = wrap.querySelector('.swatches');
  const trans = wrap.querySelector('.trans input');
  const hint = wrap.querySelector('.hint');

  PALETTE.forEach(({ name, hex }) => {
    const sw = document.createElement('button');
    sw.className = 'sw';
    sw.style.background = hex;
    sw.title = name;
    sw.addEventListener('click', () => setState({ [colorKey]: hex }));
    swatchesEl.appendChild(sw);
  });

  picker.addEventListener('input', () => setState({ [colorKey]: picker.value }));
  trans.addEventListener('change', () => setState({ [transparentKey]: trans.checked }));

  onState(() => {
    picker.value = state[colorKey];
    hexEl.textContent = state[colorKey].toUpperCase();
    trans.checked = state[transparentKey];
    const otherOn = state[otherTransparentKey];
    trans.disabled = otherOn && !state[transparentKey];
    hint.hidden = !trans.disabled;
  });

  container.appendChild(wrap);
}
```

- [ ] **Step 3: CSS for color block**

```css
.color-block { display: flex; flex-direction: column; gap: 10px; }
.color-row { display: flex; align-items: center; gap: 10px; }
.color-row input[type='color'] { -webkit-appearance: none; appearance: none; width: 28px; height: 28px; border: 1px solid var(--hairline); border-radius: 50%; padding: 0; background: none; cursor: pointer; }
.color-row input[type='color']::-webkit-color-swatch { border: none; border-radius: 50%; }
.color-row input[type='color']::-moz-color-swatch { border: none; border-radius: 50%; }
.hex { font-size: 11px; color: rgba(28, 8, 2, 0.7); }
.swatches { display: flex; gap: 6px; }
.sw { width: 18px; height: 18px; border-radius: 50%; border: 1px solid var(--hairline); padding: 0; cursor: pointer; }
.sw:hover { transform: scale(1.1); }
.hint { font-size: 10px; color: rgba(28, 8, 2, 0.5); margin: 0; }
```

- [ ] **Step 4: Mount in BOOT**

```js
mountColor({
  container: document.getElementById('ctrl-dot-color'),
  colorKey: 'dotColor', transparentKey: 'dotTransparent', otherTransparentKey: 'bgTransparent',
});
mountColor({
  container: document.getElementById('ctrl-bg-color'),
  colorKey: 'bgColor', transparentKey: 'bgTransparent', otherTransparentKey: 'dotTransparent',
});
```

- [ ] **Step 5: Verify**

Reload. Expect: two color blocks (Dot color, Background) with picker + 6 swatches + transparent checkbox each. Click `Core` swatch under Dot color → dots turn pastel purple. Tick "transparent" under dot → dots disappear (cut out from bg). Tick the other "transparent" → it's disabled with a hint underneath.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: color pickers, brand swatches, transparent toggles"
```

---

### Task 4.5: Reset button

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add reset markup at top of rail (after head)**

```html
<button id="reset" class="text-btn">reset values</button>
```

CSS:
```css
.text-btn { background: none; border: 0; padding: 0; color: rgba(28, 8, 2, 0.5); font-family: 'Geist Mono', ui-monospace, monospace; font-size: 11px; cursor: pointer; margin-bottom: 16px; }
.text-btn:hover { color: var(--fg); text-decoration: underline; }
```

- [ ] **Step 2: Wire reset**

```js
const DEFAULTS = { cellSize: 12, dotScale: 1.0, brightness: 0, contrast: 0, dotOpacity: 1.0, invert: false, dotColor: '#1C0802', bgColor: '#FFF7FA', dotTransparent: false, bgTransparent: false };
document.getElementById('reset').addEventListener('click', () => setState({ ...DEFAULTS }));
```

- [ ] **Step 3: Verify**

Move some sliders, click reset. Expect: all values revert to defaults, image redraws.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: reset button"
```

---

## Chunk 5: Image Export

### Task 5.1: Action bar (Copy / Download)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add action bar at bottom of rail**

```html
<section class="section actions">
  <button id="copy-btn" class="btn">Copy</button>
  <div id="dl-wrap" class="dl">
    <button id="dl-btn" class="btn primary">Download</button>
    <div id="dl-menu" class="dl-menu" hidden></div>
  </div>
  <p id="export-info" class="hint mono"></p>
</section>
```

CSS:
```css
.section.actions { position: sticky; bottom: 0; background: var(--bg); padding-top: 16px; }
.btn { display: block; width: 100%; padding: 8px 12px; border: 1px solid var(--hairline); background: var(--bg); color: var(--fg); font-family: inherit; font-size: 12px; cursor: pointer; margin-bottom: 8px; }
.btn:hover { background: rgba(28, 8, 2, 0.04); }
.btn.primary { background: var(--gaffer); color: var(--canvas); border-color: var(--gaffer); }
.btn.primary:hover { opacity: 0.9; }
.btn:disabled { opacity: 0.4; cursor: not-allowed; }
.dl { position: relative; }
.dl-menu { position: absolute; bottom: calc(100% + 4px); left: 0; right: 0; background: var(--bg); border: 1px solid var(--hairline); }
.dl-menu button { display: block; width: 100%; padding: 8px 12px; border: 0; background: none; text-align: left; cursor: pointer; font-family: inherit; font-size: 12px; }
.dl-menu button:hover:not(:disabled) { background: rgba(28, 8, 2, 0.06); }
.dl-menu button:disabled { opacity: 0.4; cursor: not-allowed; }
```

- [ ] **Step 2: Add Section 6 with menu logic**

```js
// === SECTION 6: EXPORT_IMG ===
const dlBtn = document.getElementById('dl-btn');
const dlMenu = document.getElementById('dl-menu');
const copyBtn = document.getElementById('copy-btn');
const exportInfo = document.getElementById('export-info');

dlBtn.addEventListener('click', () => {
  if (!state.source) return;
  buildDlMenu();
  dlMenu.hidden = !dlMenu.hidden;
});

document.addEventListener('click', e => {
  if (!e.target.closest('#dl-wrap')) dlMenu.hidden = true;
});

function buildDlMenu() {
  dlMenu.innerHTML = '';
  if (state.source.kind === 'image') {
    addOption('PNG', () => exportImage('png'));
    addOption('JPEG', () => exportImage('jpeg'), state.dotTransparent || state.bgTransparent);
    addOption('SVG', () => exportImage('svg'));
  } else {
    // wired in chunk 6
  }
}

function addOption(label, fn, disabled = false) {
  const b = document.createElement('button');
  b.textContent = label;
  if (disabled) b.disabled = true;
  b.addEventListener('click', () => { dlMenu.hidden = true; fn(); });
  dlMenu.appendChild(b);
}

async function exportImage(fmt) {
  console.log('export', fmt);
  // Implemented in next tasks
}
```

- [ ] **Step 3: Verify**

Drop an image, click Download → menu shows PNG / JPEG / SVG. Tick "transparent" on dots → JPEG becomes disabled. Click outside menu → it closes. Click PNG → console logs `export png`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: action bar with download menu and disabled-state logic"
```

---

### Task 5.2: PNG / JPEG export

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Implement bitmap export with full-res re-render**

```js
function renderToOffscreen() {
  // Re-render to a fresh offscreen canvas at the source's native resolution.
  const src = state.source;
  const W = src.width, H = src.height;
  const off = document.createElement('canvas');
  off.width = W; off.height = H;
  const octx = off.getContext('2d');

  const cs = state.cellSize;
  const cols = Math.max(1, Math.floor(W / cs));
  const rows = Math.max(1, Math.floor(H / cs));
  const luma = buildLumaBuffer(src.el, cols, rows);

  if (state.bgTransparent) {
    octx.clearRect(0, 0, W, H);
  } else {
    octx.fillStyle = state.bgColor;
    octx.fillRect(0, 0, W, H);
  }

  if (state.dotTransparent) {
    octx.globalCompositeOperation = 'destination-out';
    octx.fillStyle = '#000';
    octx.globalAlpha = 1;
  } else {
    octx.fillStyle = state.dotColor;
    octx.globalAlpha = state.dotOpacity;
  }

  const halfCell = cs / 2;
  for (let j = 0; j < rows; j++) {
    for (let i = 0; i < cols; i++) {
      const L = adjust(luma[j * cols + i], state.brightness, state.contrast);
      const darkness = state.invert ? L : 1 - L;
      const r = darkness * halfCell * state.dotScale;
      if (r < 0.4) continue;
      octx.beginPath();
      octx.arc(i * cs + halfCell, j * cs + halfCell, r, 0, Math.PI * 2);
      octx.fill();
    }
  }
  return off;
}

async function exportImage(fmt) {
  if (fmt === 'png' || fmt === 'jpeg') {
    const off = renderToOffscreen();
    const mime = fmt === 'png' ? 'image/png' : 'image/jpeg';
    const quality = fmt === 'jpeg' ? 0.92 : undefined;
    off.toBlob(blob => downloadBlob(blob, `dot-machine.${fmt === 'jpeg' ? 'jpg' : 'png'}`), mime, quality);
  } else if (fmt === 'svg') {
    exportSvg(); // task 5.3
  }
}

function downloadBlob(blob, filename) {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = filename; a.click();
  setTimeout(() => URL.revokeObjectURL(url), 1000);
}
```

- [ ] **Step 2: Verify**

Drop an image, Download → PNG. Open the downloaded file. Expect: same halftone, at native resolution. Try JPEG. Try with transparent dots → PNG has cutouts.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: PNG and JPEG export at native resolution"
```

---

### Task 5.3: SVG export

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Implement SVG generator**

```js
function exportSvg() {
  const src = state.source;
  const W = src.width, H = src.height;
  const cs = state.cellSize;
  const cols = Math.max(1, Math.floor(W / cs));
  const rows = Math.max(1, Math.floor(H / cs));
  const luma = buildLumaBuffer(src.el, cols, rows);

  const halfCell = cs / 2;
  const dotFill = state.dotTransparent ? 'none' : state.dotColor;
  const dotOp = state.dotTransparent ? 0 : state.dotOpacity;
  const bgRect = state.bgTransparent
    ? ''
    : `<rect width="${W}" height="${H}" fill="${state.bgColor}"/>`;

  const circles = [];
  for (let j = 0; j < rows; j++) {
    for (let i = 0; i < cols; i++) {
      const L = adjust(luma[j * cols + i], state.brightness, state.contrast);
      const darkness = state.invert ? L : 1 - L;
      const r = darkness * halfCell * state.dotScale;
      if (r < 0.4) continue;
      circles.push(`<circle cx="${i * cs + halfCell}" cy="${j * cs + halfCell}" r="${r.toFixed(2)}"/>`);
    }
  }

  // For "transparent dots" we need cutouts — handled via a mask:
  let body;
  if (state.dotTransparent && !state.bgTransparent) {
    body = `
      <defs>
        <mask id="m"><rect width="${W}" height="${H}" fill="white"/>
          <g fill="black">${circles.join('')}</g>
        </mask>
      </defs>
      <rect width="${W}" height="${H}" fill="${state.bgColor}" mask="url(#m)"/>`;
  } else {
    body = `${bgRect}<g fill="${dotFill}" fill-opacity="${dotOp}">${circles.join('')}</g>`;
  }

  const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${W}" height="${H}" viewBox="0 0 ${W} ${H}">${body}</svg>`;
  const blob = new Blob([svg], { type: 'image/svg+xml' });
  downloadBlob(blob, 'dot-machine.svg');
}
```

- [ ] **Step 2: Verify**

Drop an image. Download → SVG. Open in browser or Illustrator/Figma. Expect: same halftone, scalable to any size, file size reasonable (1–3 MB for 1080p with cellSize=12).

Test with transparent dots → SVG shows bg with circular holes.
Test with transparent bg → SVG shows just dots, no rect.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: SVG export with transparent-dots masking"
```

---

### Task 5.4: Copy to clipboard

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Implement clipboard write**

```js
copyBtn.addEventListener('click', async () => {
  if (!state.source) return;
  copyBtn.disabled = true;
  copyBtn.textContent = 'Copying…';
  try {
    if (state.source.kind === 'image' || state.source.kind === 'video') {
      const off = renderToOffscreen();
      const blob = await new Promise(r => off.toBlob(r, 'image/png'));
      await navigator.clipboard.write([new ClipboardItem({ 'image/png': blob })]);
      copyBtn.textContent = 'Copied ✓';
    }
  } catch (e) {
    console.error(e);
    copyBtn.textContent = 'Clipboard unavailable';
  }
  setTimeout(() => { copyBtn.disabled = false; copyBtn.textContent = 'Copy'; }, 1500);
});
```

- [ ] **Step 2: Verify (over localhost)**

Run `python3 -m http.server 8080`, open `http://localhost:8080`. Drop an image, click Copy → button reads "Copied ✓". Paste in Preview / Figma → image is there.

In `file://` mode, click Copy → button reads "Clipboard unavailable" (expected on file://).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: Copy to clipboard with graceful fallback"
```

---

## Chunk 6: Video Export

### Task 6.1: Format probing

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add capability probe in Section 7**

```js
// === SECTION 7: EXPORT_VID ===

const VID_FORMATS = (() => {
  const out = [];
  if (typeof MediaRecorder !== 'undefined') {
    if (MediaRecorder.isTypeSupported('video/webm;codecs=vp9')) {
      out.push({ label: 'WebM', mime: 'video/webm;codecs=vp9', ext: 'webm' });
    } else if (MediaRecorder.isTypeSupported('video/webm')) {
      out.push({ label: 'WebM', mime: 'video/webm', ext: 'webm' });
    }
    if (MediaRecorder.isTypeSupported('video/mp4;codecs=avc1')) {
      out.push({ label: 'MP4', mime: 'video/mp4;codecs=avc1', ext: 'mp4' });
    }
  }
  out.push({ label: 'GIF', mime: 'image/gif', ext: 'gif' });
  return out;
})();
```

- [ ] **Step 2: Add debug assert**

In `runDebugAsserts`:
```js
console.assert(VID_FORMATS.some(f => f.label === 'GIF'), 'GIF always present');
console.log('vid formats:', VID_FORMATS.map(f => f.label).join(', '));
```

- [ ] **Step 3: Verify**

Open `?debug` → console shows the list (Safari: WebM + MP4 + GIF; Chrome: WebM + GIF; Firefox: WebM + GIF).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: video format capability probing"
```

---

### Task 6.2: WebM/MP4 recording

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Wire video options in `buildDlMenu`**

Replace the `else` branch in `buildDlMenu`:
```js
} else {
  const transOn = state.dotTransparent || state.bgTransparent;
  const src = state.source;
  const canRecord = src.durationOk;
  VID_FORMATS.forEach(f => {
    const isBitmapVideo = f.label !== 'GIF';
    const disabled = (isBitmapVideo && transOn) || !canRecord;
    addOption(f.label, () => exportVideo(f), disabled);
  });
}
```

- [ ] **Step 2: Implement bitmap-video export**

```js
async function exportVideo(format) {
  if (format.label === 'GIF') return exportGif(); // task 6.4
  await recordCanvasStream(format);
}

async function recordCanvasStream(format) {
  const v = state.source.el;
  const fps = 30;
  const stream = canvas.captureStream(fps);
  const recorder = new MediaRecorder(stream, { mimeType: format.mime });
  const chunks = [];
  recorder.ondataavailable = e => { if (e.data.size) chunks.push(e.data); };

  showProgress('Recording…');
  v.pause();
  v.currentTime = 0;
  await new Promise(r => v.addEventListener('seeked', r, { once: true }));

  const onEnded = () => {
    recorder.stop();
    v.removeEventListener('ended', onEnded);
  };
  v.addEventListener('ended', onEnded);

  recorder.onstop = () => {
    hideProgress();
    const blob = new Blob(chunks, { type: format.mime });
    downloadBlob(blob, `dot-machine.${format.ext}`);
    v.loop = true;
    v.play();
  };

  v.loop = false;
  recorder.start();
  v.play();
}
```

- [ ] **Step 3: Add minimal progress overlay**

Markup (in `#stage`):
```html
<div id="progress" hidden><span class="mono">recording…</span><button id="cancel-rec">cancel</button></div>
```

CSS:
```css
#progress { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; gap: 16px; background: rgba(255, 247, 250, 0.9); }
#progress button { font-family: inherit; font-size: 12px; }
```

JS helpers:
```js
function showProgress(text) {
  const p = document.getElementById('progress');
  p.querySelector('span').textContent = text;
  p.hidden = false;
}
function hideProgress() { document.getElementById('progress').hidden = true; }
```

- [ ] **Step 4: Verify**

Drop a 5-second MP4. Download → WebM. File downloads after the video plays through once. Open the file → halftone version of the original video. No audio (expected). Try MP4 if Safari.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: video export via canvas.captureStream + MediaRecorder"
```

---

### Task 6.3: Cancel recording

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Wire cancel button**

```js
let activeRecorder = null;
// Inside recordCanvasStream, after creating recorder:
activeRecorder = recorder;
document.getElementById('cancel-rec').onclick = () => {
  if (activeRecorder && activeRecorder.state !== 'inactive') {
    activeRecorder.ondataavailable = null;
    activeRecorder.onstop = () => { hideProgress(); v.loop = true; v.play(); };
    activeRecorder.stop();
  }
};
// At the end of onstop, set activeRecorder = null;
```

- [ ] **Step 2: Verify**

Start a recording, click Cancel mid-flight → progress hides, no file downloads, video resumes looping.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: cancel video recording"
```

---

### Task 6.4: GIF export via gif.js

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add lazy loader for gif.js**

```js
let gifJsLoaded = null;
function loadGifJs() {
  if (gifJsLoaded) return gifJsLoaded;
  gifJsLoaded = new Promise((res, rej) => {
    const s = document.createElement('script');
    s.src = 'https://cdn.jsdelivr.net/npm/gif.js.optimized@1.0.1/dist/gif.js';
    s.onload = res;
    s.onerror = rej;
    document.head.appendChild(s);
  });
  return gifJsLoaded;
}
```

- [ ] **Step 2: Implement `exportGif`**

```js
async function exportGif() {
  await loadGifJs();
  const v = state.source.el;
  showProgress('Encoding GIF… 0%');

  const gif = new GIF({
    workers: 2,
    quality: 10,
    workerScript: 'https://cdn.jsdelivr.net/npm/gif.js.optimized@1.0.1/dist/gif.worker.js',
    width: canvas.width,
    height: canvas.height,
    transparent: (state.dotTransparent || state.bgTransparent) ? 0x000000 : null,
  });

  const fps = 15;
  const frameDelay = 1000 / fps;
  v.pause();
  v.currentTime = 0;
  await new Promise(r => v.addEventListener('seeked', r, { once: true }));

  // For binary alpha, threshold dot opacity at 0.5 by snapping it for the duration of the encode.
  const restoreOpacity = state.dotOpacity;
  const useThreshold = state.dotTransparent || state.bgTransparent;
  if (useThreshold) state.dotOpacity = state.dotOpacity > 0.5 ? 1 : 0;

  return new Promise(resolve => {
    const captureFrame = () => {
      render();
      gif.addFrame(canvas, { copy: true, delay: frameDelay });
      if (v.currentTime + frameDelay / 1000 >= v.duration) {
        gif.on('progress', p => showProgress(`Encoding GIF… ${Math.round(p * 100)}%`));
        gif.on('finished', blob => {
          hideProgress();
          if (useThreshold) state.dotOpacity = restoreOpacity;
          downloadBlob(blob, 'dot-machine.gif');
          v.loop = true;
          v.play();
          resolve();
        });
        gif.render();
      } else {
        v.currentTime += frameDelay / 1000;
        v.addEventListener('seeked', captureFrame, { once: true });
      }
    };
    captureFrame();
  });
}
```

- [ ] **Step 3: Verify**

Drop a short MP4 (3–5 sec). Download → GIF. Wait through encode (progress updates). Open the GIF → animated halftone, looping.

Try with transparent bg → GIF has transparent background.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: animated GIF export via gif.js"
```

---

## Chunk 7: Polish

### Task 7.1: Export resolution display

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Update `#export-info` from state**

```js
function updateExportInfo() {
  const src = state.source;
  if (!src) { exportInfo.textContent = ''; return; }
  if (src.kind === 'image') {
    exportInfo.textContent = `${src.width} × ${src.height}`;
  } else {
    const previewW = canvas.width, previewH = canvas.height;
    const tag = (previewW === src.width) ? '' : ' (preview)';
    exportInfo.textContent = `${previewW} × ${previewH}${tag}`;
  }
}
onState(updateExportInfo);
```

Add a hook in `render()` to call `updateExportInfo()` after sizing.

- [ ] **Step 2: Verify**

Drop a 4K image → bottom of rail shows `3840 × 2160`. Drop a 4K video → shows `1280 × 720 (preview)`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: show export resolution in action bar"
```

---

### Task 7.2: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

```markdown
# DOT MACHINE

A single-file halftone tool for LA BOULE. Drop an image or video, dial the dots, export.

## Use

Double-click `index.html` and drop a file in the stage.

For full functionality (Copy to clipboard) during local development, serve over localhost:

\`\`\`bash
python3 -m http.server 8080
\`\`\`

Then open http://localhost:8080.

## Controls

- **Cell size / Dot scale** — geometry of the dot grid.
- **Brightness / Contrast** — tonal mapping before halftoning.
- **Opacity / Invert** — dot-level adjustments.
- **Dot color / Background** — pick from brand swatches or any hex. Tick "transparent" to export a layer with alpha (PNG, SVG, GIF only).

## Export

Image input → PNG, JPEG (no alpha), SVG.
Video input → WebM, MP4 (Safari), GIF.

The Copy button puts a PNG of the current preview on the clipboard.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: README with usage and localhost tip"
```

---

### Task 7.3: Final manual QA pass

- [ ] Drop a JPG → halftone renders, all controls work, all swatches work, all 3 image formats download cleanly.
- [ ] Drop an MP4 → halftone plays back, WebM/GIF export work, MP4 if Safari.
- [ ] Transparent dots + PNG export → cutouts present.
- [ ] Transparent bg + PNG export → only dots visible.
- [ ] Both transparent toggles can't both be on.
- [ ] Reset restores defaults and re-renders.
- [ ] Drop a new file while one is loaded → previous one is replaced cleanly.
- [ ] `?debug` shows console asserts pass.

- [ ] **Final commit**

```bash
git commit --allow-empty -m "chore: end of v1 implementation"
```

---

## Done

After Chunk 7, the tool is feature-complete per the spec. To deploy: drop `index.html` onto Vercel, Netlify, GitHub Pages, or any static host. The folder doesn't need a build.
