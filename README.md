# DOT MACHINE

A single-file halftone tool for LA BOULE. Drop an image or video, dial the dots, export.

## Use

Double-click `index.html` to open it. Drop a file in the stage (or click to pick one).

For full functionality (Copy to clipboard) during local development, serve over `localhost`:

```bash
python3 -m http.server 8080
```

Then open `http://localhost:8080`.

## Controls

- **Cell size / Dot scale** — geometry of the dot grid.
- **Brightness / Contrast** — tonal mapping before halftoning.
- **Opacity / Invert** — dot-level adjustments.
- **Dot color / Background** — pick from brand swatches (Core, Analog, Rush, Keyframe, Canvas, Gaffer) or any hex. Tick "transparent" to export a layer with alpha.

## Export

| Input | Formats |
|---|---|
| Image | PNG, JPEG, SVG |
| Video | WebM (always), MP4 (Safari and recent Chrome on macOS), GIF |

JPEG, MP4 and WebM don't carry alpha — they are disabled when "transparent" is on. GIF supports binary (1-bit) transparency. Image export runs at the source's native resolution, capped at 4096px on the longest side. Video export captures the preview canvas (max 1280px), and the recorded duration matches the source.

The **Copy** button puts a PNG of the current preview on the clipboard. If the browser refuses (e.g. on `file://`), it falls back to a download.

## Tech

Vanilla HTML/CSS/JS, Canvas 2D, MediaRecorder, Clipboard API. No build, no node_modules. The only third-party dependency is `gif.js` (lazy-loaded from jsDelivr the first time you export a GIF). Geist and Geist Mono served from Google Fonts.
