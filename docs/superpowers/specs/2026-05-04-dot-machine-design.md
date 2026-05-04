# DOT MACHINE — Design Spec

**Date:** 2026-05-04
**Project:** LA BOULE / DOT MACHINE
**Status:** Draft for review

---

## 1. Context

LA BOULE is a creative/production project (the parent folder contains brand decks: Big Picture, Look & feel, Wording, Valeurs, etc.). DOT MACHINE is a small, single-purpose web tool delivered to the client: it transforms an input image or video into a halftone-style "dotted" rendering, with live controls, then lets the user copy or download the result in several formats. It must feel like a single, focused instrument — not an app.

## 2. Goals

- One-page tool, no build step, runs by opening `index.html`.
- 100% client-side: source media never leaves the browser.
- Live preview at interactive frame rates for both images and videos.
- Brand-aware: brand colors pre-populated as one-click swatches.
- Minimalist UI in Geist (sans for labels, mono for numerical values).

## 3. Non-Goals

- Multi-color halftone (CMYK, riso/sérigraphie multi-passes).
- Alternative dot shapes (square, diamond, line).
- Grid rotation / angled screens.
- Audio handling on video files.
- Server-side processing, accounts, or persistence across sessions.
- Mobile-first design (desktop-only is acceptable; the client uses a Mac).

## 4. User Flows

### 4.1 Image flow
1. User drops an image (or clicks to pick) → preview renders immediately with default settings.
2. User adjusts sliders / colors → preview updates live.
3. User clicks **Copy** (PNG to clipboard) or **Download** → format menu shows: PNG, JPEG, SVG.

### 4.2 Video flow
1. User drops a video → first frame renders, then playback starts (looped, muted).
2. User adjusts controls — every rendered frame uses current settings.
3. User clicks **Download** → format menu shows: MP4/WebM (whichever the browser's `MediaRecorder` supports best), GIF. Recording captures one full loop of the video at current settings; a small progress indicator shows during encode.

### 4.3 Transparency flow
- "Transparent" checkbox next to each color picker. When ticked, that layer renders with alpha 0.
- Format menu disables JPEG, MP4, and WebM when any transparency is active (these formats can't reliably carry alpha — JPEG has none, and WebM/VP9 alpha support across players is too patchy to ship).
- GIF stays available with transparency on, but its alpha is **binary** (1-bit). At export time, dot alpha is thresholded at 0.5 — semi-transparent dots become fully opaque or fully cut out. A small note next to the GIF option states this.
- Both transparent at once is disallowed: when the user ticks one, the other checkbox is disabled and a one-line hint appears beneath it ("can't make both transparent — output would be empty").

## 5. Layout & Visual Direction

**Layout:** desktop-only. Fixed left rail (~280px) with controls, flexible right area for the canvas preview. Below ~720px wide the layout simply lets things wrap — no dedicated mobile design.

**Style:**
- Background: brand `Canvas` `#FFF7FA`
- Text: brand `Gaffer` `#1C0802`
- No cards, no borders, no drop shadows, no decorative icons.
- Sliders are native `<input type="range">`, restyled minimally (thin track, small thumb).
- Section dividers are thin 1px lines in `Gaffer` at 8% opacity.
- Color pickers are small color swatches (24px circles) opening the native `<input type="color">`.

**Typography:**
- `Geist Sans` for labels, headings, buttons.
- `Geist Mono` for all numerical values displayed beside sliders (e.g. `12 px`, `+0.34`).
- Loaded from a CDN (Vercel Geist) with `font-display: swap`.

**Drop zone:** the entire preview area accepts drag-and-drop. Empty state shows a centered message in Geist Mono: "drop an image or video here".

## 6. Halftone Algorithm

For each frame:

1. The source media is drawn into an offscreen `<canvas>` at its native size (capped at e.g. 4096px on the longest side to keep memory predictable).
2. Brightness/contrast adjustments are applied to the source pixels in a single pass.
3. The source is downscaled in one `drawImage` call into a luminance buffer canvas of `(width / cellSize) × (height / cellSize)` pixels — one pixel per cell. The browser's bilinear filter gives an accurate per-cell average for free, and avoids aliasing on moving video. We then `getImageData` once and read luminance per cell.
4. For each cell `(i, j)`, luminance L (0–1) is the corresponding pixel in the buffer.
5. Map L to dot radius:
   - `darkness = invert ? L : (1 - L)`
   - `radius = darkness * (cellSize / 2) * dotScale`
6. Draw a filled circle at the cell center with `dotColor` and `dotOpacity`.

**Background**: the visible canvas is cleared to `backgroundColor` (or transparent) before dots are drawn.

**Live performance budget:**
- Canvas 2D handles 1080p at typical cell sizes (≥6 px) at 30+ fps on modern hardware.
- For heavy cases (4K video, very small cells), the preview canvas is sized to 1280px on the longest side. Image export always re-renders at the source's native resolution. Video export keeps the preview canvas size — boosting it during recording would cause a visible stutter and mismatch with what the user just dialed in. The export resolution is shown next to the Download button so there's no surprise.

## 7. Controls Reference

| Control | Type | Range | Default |
|---|---|---|---|
| Cell size | range | 4–80 px | 12 |
| Dot scale | range | 0.0–1.5 | 1.0 |
| Brightness | range | -1.0 to +1.0 | 0.0 |
| Contrast | range | -1.0 to +1.0 | 0.0 |
| Dot opacity | range | 0.0–1.0 | 1.0 |
| Invert | checkbox | — | off |
| Dot color | color + swatches + transparent checkbox | hex | `#1C0802` (Gaffer) |
| Background color | color + swatches + transparent checkbox | hex | `#FFF7FA` (Canvas) |

All numeric values display in Geist Mono next to their slider, formatted with sign where relevant (`+0.34`, `-0.20`, `12 px`).

A small "Reset" text button restores defaults without clearing the loaded media.

## 8. Brand Palette Integration

Six pre-populated swatches under each color picker:

| Name | Hex |
|---|---|
| Core | `#CEADFF` |
| Analog | `#F6B9A2` |
| Rush | `#F2ADFF` |
| Keyframe | `#ADE7FF` |
| Canvas | `#FFF7FA` |
| Gaffer | `#1C0802` |

Swatches are 18px circles in a tight row. Hover shows the name in Geist Mono. Click sets the corresponding color picker to that hex. The native picker remains available for any custom hex.

## 9. Export

### 9.1 Image input
- **PNG**: `canvas.toBlob('image/png')`. Honors transparency.
- **JPEG**: `canvas.toBlob('image/jpeg', 0.92)`. Disabled if any transparency is on.
- **SVG**: a `<svg>` document is generated by walking the cell grid and emitting one `<circle>` per non-zero dot. The background is either a `<rect>` (opaque mode) or omitted (transparent mode). Defs are minimal; `cx/cy/r` use integer pixel values to keep the file small.
- **Copy to clipboard**: PNG only. Requires a secure context — when the page is opened over `file://`, the Clipboard API may refuse the write. In that case the button surfaces "Clipboard unavailable — use Download" rather than failing silently. Hosting over `https://` (or `localhost`) makes Copy work everywhere.

### 9.2 Video input
- **WebM / MP4**: `canvas.captureStream(fps)` piped into a `MediaRecorder`. WebM (`video/webm;codecs=vp9`) is the default — the most widely supported format in MediaRecorder across browsers in 2026. MP4 (`video/mp4;codecs=avc1`) is only offered if `MediaRecorder.isTypeSupported()` returns true (currently reliable on Safari, partial on Chrome macOS, never on Firefox). The format menu only lists what the current browser actually supports.
- Recording flow: pressing export disables the controls, seeks the video to `0`, plays it through once muted, and stops the recorder on the `ended` event. A small progress indicator shows elapsed time / total duration; a Cancel button stops and discards.
- **Pre-flight check**: video export is gated on `isFinite(video.duration) && video.duration > 0`. Streamed sources without a known duration show a tooltip "duration unknown — can't export" instead.
- **GIF**: `gif.js` (lazy-loaded the first time the user picks GIF). Frame rate capped at 15 fps. Palette quantized to the dot + background colors (and transparent if enabled) — keeps file small and on-brand. With transparency on, dot alpha is thresholded to 1-bit at export.
- WebM and MP4 are disabled when any transparency is on. GIF stays available with the binary-alpha caveat noted above.

## 10. Edge Cases & Constraints

- **Huge media**: source images > 4096px on the longest side are downscaled before processing. The user is not warned; the result still looks right.
- **EXIF orientation**: respected via `createImageBitmap(blob, { imageOrientation: 'from-image' })`.
- **Video codec failure**: if `MediaRecorder` can't produce MP4 nor WebM, the export button surfaces an error and falls back to GIF.
- **Browser without clipboard image support**: Copy button shows "Copied as PNG ✓" only if the clipboard write actually succeeds; otherwise it falls back to triggering a download.
- **First frame race on video**: we wait for `loadeddata` before rendering to avoid black initial frames.
- **Hot reload of media**: dropping a new file while a video is playing stops the previous video, releases the object URL, and resets the canvas.
- **Streaming / unknown-duration videos**: video export is unavailable; the source still previews fine.
- **Very short videos (< 0.5s)**: still export, but GIF/WebM/MP4 may produce a single-frame file — acceptable.

## 11. Tech Stack

- HTML + CSS + vanilla JavaScript, no framework, no build.
- One file: `index.html` (with `<style>` and `<script>` inline) — easy to email, host, or paste into a static bucket.
- One optional dependency, lazy-loaded only when needed: `gif.js` from a CDN.
- Geist Sans + Geist Mono from a font CDN (Vercel) with `<link rel="preconnect">`.

## 12. File Structure

```
DOT MACHINE/
├── index.html                 # the whole tool
├── README.md                  # one-paragraph usage notes
└── docs/superpowers/specs/
    └── 2026-05-04-dot-machine-design.md
```

That's it. No package.json, no node_modules, no config.

## 13. Acceptance Criteria

- Opening `index.html` in Safari and Chrome on macOS shows the empty state.
- Dropping a JPG/PNG renders the halftone with default Canvas/Gaffer colors at interactive speed.
- Dropping a 1080p MP4 plays it back as halftone at ≥24 fps.
- All eight controls update the preview live with no stutter.
- Each brand swatch sets the correct color when clicked.
- The "transparent dots" checkbox produces a PNG with alpha cutouts.
- The "transparent background" checkbox produces a PNG with only the dots visible.
- PNG, JPEG, SVG export all download a valid file; SVG opens cleanly in a vector tool.
- MP4/WebM and GIF export produce valid playable files of the loaded video.
- The UI uses Geist Sans for labels and Geist Mono for numbers, with the brand color palette throughout.
