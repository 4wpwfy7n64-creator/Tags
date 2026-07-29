# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Exhibit Tag Generator — a single-page, single-file web app for producing printable museum/collection exhibit tags (small adhesive tags, 6×4 index cards, and 3×2 print sheets), with PDF/PNG/DOCX export, CSV batch mode, QR codes, and local template/history persistence. There is no backend: everything runs client-side in the browser.

## Repository layout

- `index.html` — the entire application: markup, CSS, and JS all live in this one file. There is no other source code.
- `README.md` — just the project title.

There is no build system, package manager, bundler, linter, test suite, or CI configuration in this repo. It is meant to be opened directly (or served statically) as-is.

## Development workflow

- **Run it**: open `index.html` directly in a browser, or serve the directory with any static file server (e.g. `python3 -m http.server`) if `file://` CORS restrictions cause issues with the CDN scripts.
- **Edit it**: all changes are made in-place in `index.html` — CSS in the `<style>` block, markup in `<body>`, logic in the trailing `<script>` IIFE.
- **No build/lint/test commands exist.** Verify changes by loading the page in a browser and exercising the UI (fill the form, switch templates/views, export PDF/PNG/DOCX, parse a CSV batch).
- Third-party libraries are loaded from CDNs at the top of `<head>` (jsPDF, html2canvas, qrcode, PapaParse) — there are no local/vendored copies and no `node_modules`.

## Architecture

Everything lives inside one `(() => { ... })()` IIFE at the bottom of `index.html`, structured around a single mutable `state` object and a form-driven render loop:

- **`state`** tracks the current view (`tag` | `sheet` | `card`), selected template, bilingual/catalog-corner toggles, the uploaded photo (as a data URL), and `state.batch` (an array of parsed CSV records, or `null` for single-item mode).
- **`readForm()` / `writeForm(d)`** convert between the DOM form fields (defined in the `FIELDS` array) and plain data objects. History and saved-template entries are just these data objects persisted to `localStorage`.
- **Rendering** is DOM-based, not templated: `renderTag()`, `renderCard()`, and `renderSheet()` build tag/card/sheet elements node-by-node (async, because QR generation is async) and are also reused for PDF/PNG export by rendering off-screen and rasterizing with `html2canvas`. `render()` is the single dispatcher called after any form change; in batch mode it delegates to `renderBatchPreview()` instead.
- **Two independent input modes** share the same render/export pipeline: the single-item form (`.form-area`) and the CSV batch textarea (`.form-area.batch`), toggled via `body.batch-mode` and the tab buttons. `parseCsv()` (using PapaParse, with a comma→tab delimiter fallback) normalizes varied header names (e.g. `catalog`/`Catalog`/`id`) into the canonical field set.
- **Export paths** (`exportPdf(kind)`, `exportPng()`, `exportDocx()`) each re-render the relevant node fresh, capture it via `captureNode()` (off-screen DOM insertion + `html2canvas`), and emit a file via jsPDF, canvas `toDataURL`, or (for DOCX) a hand-built Word-compatible HTML blob saved with a `.doc` extension. The 6×4 print sheet additionally places 6 tags in a 3×2 grid with crop marks drawn directly via jsPDF primitives.
- **Persistence** is `localStorage` only, under two keys: `etg.history.v1` (last 5 generated items, deduped by catalog+name) and `etg.templates.v1` (named, user-saved field presets). There is no server-side storage.
- **Templates** (`TEMPLATES` object: `artifact`, `coin`, `sacred`, `militaria`, `custom`) currently only vary the default accent color and add a `tpl-*` CSS class for minor typographic tweaks (see the "Templates rendering modes" CSS section) — they don't change layout structure.
- Physical dimensions are authored directly in CSS using real units (`in`, `pt`) — e.g. `.tag-1625` is a 1.625"×1.625" tag, `.card-6x4` is a 6"×4" card — so on-screen preview, PNG capture, and PDF output all stay dimensionally consistent without separate print-layout code.

## Conventions

- Vanilla JS only (no framework, no modules/imports, no TypeScript, no JSX). Keep additions consistent with the existing `$`/`$$` query-selector helpers and direct DOM construction style already used throughout.
- User-supplied text that gets injected as HTML must go through the existing `esc()` helper to avoid breaking markup; text set via `.textContent` doesn't need it.
- Keep new CDN dependencies to a minimum — everything currently used is pinned to a specific version in the `<script src>` tags in `<head>`.
