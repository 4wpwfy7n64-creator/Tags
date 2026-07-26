# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, client-only web app for designing and printing museum/exhibit item tags — small adhesive tags, 6×4" info cards, and 3×2 sheet layouts. Everything (markup, CSS, JS) lives in `index.html`. There is no build step, package manager, server, or test suite — open `index.html` directly in a browser (or serve the directory with any static file server) to run it.

## Development workflow

- Edit `index.html` directly; there's nothing to compile or bundle.
- To preview changes, open the file in a browser (e.g. `open index.html` or `python3 -m http.server` from the repo root and visit `localhost:8000`).
- There is no linter or test suite configured. Verify changes manually by exercising the UI: fill the single-item form, switch templates/views, try bilingual mode, upload a photo, and generate each export (PDF tag/sheet/card, PNG, DOCX). Also test the Batch (CSV) tab with the built-in sample data.
- All three external libraries are loaded from CDNs in `<head>` (jsPDF, html2canvas, QRCode, PapaParse) — there are no local dependencies to install.

## Architecture

The whole app is one IIFE in the `<script>` tag at the bottom of `index.html`, organized into a few clear sections (search for the `// ---` comment banners):

- **State**: a single `state` object (`view`, `template`, `bilingual`, `catalogCorner`, `photo`, `batch`) plus the DOM form fields themselves are the source of truth. `readForm()` / `writeForm()` convert between the two.
- **Rendering**: `renderTag`, `renderCard`, `renderSheet`, and `renderBatchSheet` each build a detached DOM subtree for one of the three physical layouts (`.tag-1625` = 1.625" square tag, `.card-6x4` = 6×4" info card, `.sheet-6x4` = 3×2 grid of tags on a 6×4" sheet). `render()` is the dispatcher called after every form change; it re-renders whichever stage is active (`stageSingle` or `stageBatch`, toggled via `body.batch-mode`).
- **Templates**: `TEMPLATES` maps a template id (`artifact`, `coin`, `sacred`, `militaria`, `custom`) to an accent color; the same id is added as a CSS class (`tpl-coin`, `tpl-militaria`, etc.) so template-specific styling lives entirely in CSS, not JS.
- **Export**: `exportPdf(kind)` and `exportPng()` render the relevant DOM node offscreen, rasterize it with `html2canvas` via `captureNode()`, and place the resulting image into a `jsPDF` document sized to match the physical layout (in inches, matching CSS `in` units used throughout). The 6×4 sheet export lays out 6 tags in a 3×2 grid with crop marks. `exportDocx()` takes a different path: it builds a standalone HTML string and saves it as a `.doc` blob (Word's HTML-import format), not a real DOCX.
- **Batch/CSV mode**: `parseCsv()` uses PapaParse to turn pasted CSV/TSV into `state.batch`, an array of record objects; batch mode renders/export one 3×2 sheet per 6 records. Column matching is case-insensitive with several header aliases (see `parseCsv()`).
- **Persistence**: `localStorage` under keys `etg.history.v1` (last 5 generated tags, deduped by catalog+name) and `etg.templates.v1` (user-named saved field presets). No server-side storage.

## Key conventions

- Physical dimensions are expressed in CSS inches (`in`) throughout, matching the real-world print sizes (1.625" tag, 6"×4" card/sheet) — keep new layout code in the same unit so on-screen preview, PDF export, and print-at-100% stay consistent.
- User-provided text is inserted via `esc()` (a manual HTML-entity escaper) whenever it's placed into `innerHTML`; always route new dynamic text through `esc()` rather than templating it in raw.
- Bilingual mode splits a tag/card into two mirrored columns (English + secondary-language `name2`); when adding new fields, check whether `renderTag`/`renderCard`'s bilingual branch also needs updating.
