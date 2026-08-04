# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page, client-side tool for batch-preparing product images (商品画像リサイズ). Users drag in images/PDFs, reorder them, mark exclusions, and export a ZIP of sequentially-named JPEGs. The UI is Japanese. All processing happens in the browser — there is no server, no backend, and no upload.

The entire application is `index.html`. There is no build step, no package manager, no test suite, and no lint config. Tracked files are just `index.html`, `.gitignore`, `.gitattributes`, and snapshots under `Backup/`.

**This repo is edited from more than one place** — much of the history is GitHub web-UI commits (hence the repeated "Update index.html" messages), and the local clone has been months stale before. It is also in production use by other people. Run `git fetch` and check `git rev-list --left-right --count main...origin/main` **before** reading or editing, or you will analyze and patch a version that no longer exists.

## Running and verifying

Dependencies load from jsDelivr CDN at pinned versions, and the app uses `<script type="module">` plus a WASM fetch — so `file://` will not work. Serve over HTTP:

```powershell
python -m http.server 8000    # then open http://localhost:8000/
```

Verification is manual: drop sample images/PDFs, watch the on-page `#log` panel, export a ZIP and inspect the JPEGs. Deployed via GitHub Pages from `main` (repo `r-tkbyc/ImageResizingTool`).

## Conventions

- **Bump the version badge on every substantive change.** `<div class="ver">Ver_x.y.z</div>` near the top of `<body>` is the only release marker; git history shows it incremented with essentially every functional commit.
- Comments and UI strings are Japanese; keep them that way.
- Snapshots are kept as whole-file copies named `bk<date>_index.html`. Two locations are in play: `Backup/` is **tracked**, while `Ignored files/` is gitignored and also holds the ICC profiles used for testing. Never commit `Ignored files/`.

## Dependencies

Two loading styles coexist and this matters when adding code:

- Classic globals via `<script src>`: **SortableJS** 1.15.2, **JSZip** 3.10.1 — available as `Sortable` / `JSZip`.
- ESM inside the single `<script type="module">`: **@imagemagick/magick-wasm** 0.0.37 (as `Magick`), **pdfjs-dist** 4.10.38. All application code lives at the top level of this one module — no exports, no separate files.

The magick WASM binary and the pdf.js worker are also pinned CDN URLs; if you bump a package version, bump its `magick.wasm` / `pdf.worker.mjs` URL to match.

## Architecture

### Page layout

The page itself never scrolls: `body` is `overflow:hidden` + column flex, `.gridWrap` is the only scrolling region, and `.log` is pinned to the bottom at a fixed 170px. The `min-height:0` on `.container` and `.gridWrap` is load-bearing — without it the inner scroll collapses. `.card` sets `height:auto` + `align-self:start` so tiles don't stretch to fill a row.

### State and ordering

`items` (array of `{id, file, kind, excluded, use400, warnings, info, thumbUrl?, thumbCanvas?}`) is the model; each item gets one `.card` element in `#grid`.

**DOM order is authoritative for sequencing.** SortableJS reorders the DOM directly; `updateItemOrderFromDOM()` then rebuilds `items` to match. Any code path that changes card order must call it. `refreshSeqBadges()` recomputes displayed sequence numbers and output filenames and toggles the export button.

### Numbering rules (deliberate, non-obvious)

- Output name is `${prefix}${pad(startNo + index, digits)}.jpg`, index being position in the list.
- **Excluded items are skipped but do not compact the numbering** — the sequence number they occupy is simply absent from the ZIP. This is an intentional operational rule (`連番は詰めない`), not a bug.
- `currentDigits()` pads to the width of the *last* number (`startNo + count - 1`), floored at 2. The count is the *total*, not the included count, again because exclusions don't renumber.

### Processing pipeline

Three item kinds converge on one renderer:

- `image` → `Magick.ImageMagick.read(bytes)`
- `pdf` → `rasterizePdfToCanvas()` renders **page 1 only** at ~2× the target size via pdf.js → `readFromCanvas`
- `dummy` → generated solid `#b0b8bb` canvas, sized to match the target box → `readFromCanvas`

All three call `renderToJpegBytesFromMagickImage(img, kind, use400)`, which does auto-orient → flatten alpha onto white → color-manage → resize → set density/quality → JPEG. Output constants live at the top of the module: `TARGET_W/H` 650×528, `W400/H400` 400×400, `OUT_DPI` 144, `OUT_QUALITY` 100.

Sizing: both modes are the same operation against a different box — *contain*-fit (never cropped, **upscaled** if smaller) then `extent()` centered with white padding. The per-card "400" checkbox picks the 400×400 box instead of 650×528; there is a single `extent()` call on the shared path.

`makeDummyExportCanvas(use400)` must generate at the target box's exact dimensions. If it emitted a fixed 650×528 while the 400 box was selected, the contain-fit would letterbox it and the dummy would stop being a flat color block.

### Color management

- CMYK/CMY input → `transformColorSpace(japanProfile, srgbProfile)`, requires both profiles loaded.
- RGB with embedded ICC → transform from embedded profile to sRGB.
- RGB without ICC → treated as sRGB (warned at tile level).

Profiles are supplied by the user through the two file inputs in the header, then cached in **IndexedDB** (db `imageResizingTool`, store `profiles`, keys `srgb`/`japan`) and auto-restored on the next load. The ICC files are deliberately *not* committed — they live untracked in `Ignored files/` — so IndexedDB is what spares the user from re-picking them every session.

`restoreProfilesFromDb()` must run **after** `initializeImageMagick()` resolves, since `new Magick.ColorProfile()` needs the WASM module. It never overwrites a profile the user already picked in this session. Missing profiles degrade to warnings rather than failures.

### magick-wasm gotchas

- `img.write()` hands the callback a buffer with a very short lifetime. Always go through `magickWriteCopyU8()`, which copies into a new `Uint8Array` and validates the JPEG SOI marker. Calling `write` directly produces intermittently corrupt output.
- Every `read`/`readFromCanvas` callback must `img.dispose()` in a `finally`.
- `autoOrient()` and `alpha()` are wrapped in bare `try/catch` because they throw for some inputs.

### TIFF and init timing

Browsers can't render TIFF in `<img>`, so thumbnails are generated through magick. If files are dropped before `initializeImageMagick()` resolves, a placeholder canvas is shown and `refreshTiffThumbnails()` retroactively replaces it after init completes. Anything else added that depends on magick must respect the `magickReady` flag the same way.

### Deletion and cleanup

Dragging a card onto `#trash` deletes it — handled both by a native `drop` listener and by a bounding-box check in Sortable's `onEnd` (a fallback for environments where the drop event doesn't fire). `thumbUrl` object URLs must be revoked on delete and on clear.
