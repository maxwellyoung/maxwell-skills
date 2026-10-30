---
name: font-subset
description: "Shrink a website's web fonts to what the site actually uses without changing how anything renders, prove it with before/after pixel diffs, and split each font into a preloaded core plus a unicode-range extended file. Use when asked to make web fonts load faster, subset or trim fonts, cut font bytes, optimise woff2 files, fix font preloads, or split fonts with unicode-range."
---

# Font subset

Make a site's web fonts smaller and faster to load while it keeps looking exactly the same. Measure first, cut each font down to what the site uses, split it into a core and an extended file, and prove the result with pixel diffs from the same content.

## Rules

- **Rendering is the contract.** Any visible change is a regression unless the user approves it. If an optimisation would change rendering, don't ship it by default. Report what it saves and what it changes, and keep the site looking the same.
- **Keep the loading behaviour.** Don't touch `font-display`, fallback fonts, or fallback metric overrides.
- **No build step.** Write a script that is run by hand, commit its output (the font files plus the CSS or data it generates), and say in the script header and the repo's docs when it needs rerunning.
- **Respect the licence.** If a font is OFL with a Reserved Font Name, a subset is a Modified Version, so rename it internally (name IDs 1, 3, 4, 6, 16, 17, 21, 25 as present). The CSS `font-family` stays the same, so the rename can't change rendering.
- **Check the generated files.** Font tools have quirks. Read back every output file and don't trust the command line you passed.

## 1. Measure the current state

Record this before changing anything, and keep it for the final report.

- **Per font file:** bytes on disk and over the wire, format, glyph count (`maxp.numGlyphs`), cmap coverage, `fvar` axes with min, default and max, GSUB and GPOS features per script and language, feature variations (`rvrn`), and hinting (`fpgm`, `prep`, `cvt `, glyph instructions, CFF hints, `gasp`). fontTools (`ttx -l`, a short Python reader) covers all of it.
- **Per page:** font bytes transferred (Playwright response sizes or the CDP `encodedDataLength`), which files load, and what `<link rel=preload as=font>` each page emits. Check that each preload URL exactly matches its `@font-face` `src` and carries `crossorigin`, otherwise the file downloads twice.
- **Licence:** read the OFL or licence file for each family and note any Reserved Font Name.

## 2. Find what the site actually uses

Collect this from the rendered site, not just the stylesheets. Inline component styles, CSS-in-JS, utility classes and runtime-rendered text all count. The most reliable source is a browser pass over every built page (or a representative set that covers every template and component). Walk the text nodes and `::before`/`::after` content, and record per element the computed `font-family` (the face that actually renders), `font-weight`, `font-style`, `font-stretch`, `font-variation-settings`, `font-feature-settings`, `font-variant-*`, `font-optical-sizing` with the `font-size`, `text-transform`, and `hyphens`.

From that, build per font file:

- **Characters.** Take the text as rendered, after `text-transform` (uppercase makes accented capitals, `ß` becomes `SS`). Include CSS `content`, list markers, `quotes` (curly quotes on `<q>`), `text-overflow: ellipsis` (U+2026), NBSP and soft hyphen, and `hyphens: auto`, which draws U+2010 when the font has it and U+002D when it doesn't. Dropping U+2010 changes rendering, so keep it in the file that renders hyphenated text. Collect characters per family: a mono font only needs what appears in code.
- **CJK.** If CJK text currently renders from system fonts (the webfont lacks those glyphs), leave it that way. If a webfont does draw some CJK today, removing it is a rendering change, so report it.
- **Axis values and weights.** Note every value each face is asked for: one-off inline overrides, `bolder`/`lighter` results, `<strong>` and `<b>` inside `<em>` (bold italic), headings, and `opsz` across every font size used when optical sizing is auto.
- **Features.** Note what's turned on: the browser defaults (`kern`, `liga`, `calt`, `clig`, `ccmp`, `locl`, `mark`, `mkmk`, `rlig`, `rvrn`, plus `curs` and `init`/`medi`/`fina` where scripts need them), plus anything enabled through `font-feature-settings` or `font-variant-*` (`tnum`, `ss01`, `zero`, `smcp`, and so on).

## 3. Cut each font down

- **Characters:** keep the used set plus all of Latin-1 Supplement (U+00A0–00FF) and Latin Extended-A (U+0100–017F). Make sure GSUB closure keeps glyphs reachable only through kept features (ligatures, alternates).
- **Axes:** limit each axis to the range actually used, with `fontTools.varLib.instancer` (`wght=400:700`). Pin an axis to one value only if no style anywhere asks for a different one. If the original default falls outside the new range, the default moves. Diff that case closely.
- **Faces:** if trimming a range means a requested weight or style no longer matches the face (bold inside italics, for instance), the browser will synthesise it. That's a regression, so widen the range back out. Update the `@font-face` `font-weight` and `font-style` descriptors only when doing so leaves face matching unchanged.
- **Features:** drop the ones nobody turns on. Keep the browser defaults the font actually has, and keep `rvrn` and the feature variations on variable fonts.
- **Hinting:** drop it (`--no-hinting`, `--desubroutinize` for CFF). Hinting only affects rendering on rasterisers that read it (Windows GDI/DirectWrite at small sizes). A macOS or headless Linux diff can't prove parity there, so report it as unverified on Windows unless you actually checked there.
- **Output:** woff2. Keep the source fonts in the repo so the script can be rerun.

## 4. Split into core and extended

- **Core** holds ASCII plus the punctuation most pages use: curly quotes, en and em dashes, ellipsis, NBSP, U+2010 if hyphenation is on, and any other character that shows up on most pages. Preload it on every page that uses that family.
- **Extended** holds everything else that was kept. Declare it as a second `@font-face` with the **same family, weight and style** and a `unicode-range`, and don't preload it. It downloads only when a page renders one of its characters.
- **`unicode-range`** comes from each output file's cmap. List only characters the file actually contains, compressed into ranges. Never use a broad block range.
- **Fallback face trap.** If the site generates a metric-matched fallback face (next/font, fontaine, capsize and similar), the extended face must not sit after it in the stack, for example as a separate family name. If it does, the fallback draws the extended characters before the real font gets the chance. Keep core and extended in the same family so both come ahead of the fallback.
- **Shaping across the split.** The browser shapes each font separately, so kerning, contextual alternates and mark attachment between a core glyph and an extended glyph are lost (`Té`, `Vá`, a combining mark on an ASCII base). Look for kern pairs and contexts that cross the split in the content. Fix it by moving those characters into core, or report the saving against the visible change and let the user decide.

## 5. Write the script

Commit one script (for example `scripts/fonts/subset.py`) with the fontTools version pinned (`uv run --with 'fonttools[woff]==X'`, or a requirements file). It should:

- read the source fonts and a committed config: per-family character set, axis ranges, features, core set, and RFN rename;
- write the woff2 files and the generated `@font-face` CSS or `unicode-range` data, deterministically, so rerunning on the same input gives a byte-identical result;
- re-read each output and print its glyph count, axes, features, hinting status, cmap and size, failing loudly if a configured character, axis value or feature is missing.

Put the rerun triggers in the header and the repo docs: content using characters outside the kept set (anything beyond the core set already falls through to extended), new weights, axis values or OpenType features in CSS or components, a source font upgrade, or a change to the core/extended split.

## 6. Prove it

1. **Baseline from the same content.** Render the current content with the old fonts and CSS, for example by swapping only the font files and `@font-face` CSS in a temporary copy. A diff against an older commit mixes content changes in with font changes.
2. **Deterministic screenshots** in Playwright: fixed viewport, DPR 1 and 2, animations and carets off, wait for `document.fonts.ready` and confirm every expected face reports `loaded`, then take a full-page shot. Use the same browser build for before and after.
3. **Pages:** code blocks, italics with bold inside, accented characters (including the cross-split pairs), each component with its own font settings, hyphenated text, CJK, and every distinct template.
4. **Diff** with pixelmatch or ImageMagick at threshold 0. Report changed pixels and bounding boxes, save diff images, and explain every difference as one of: a content change, a regression to fix, or harmless sub-pixel rounding (a few anti-aliased pixels, no glyph shape or advance change). Fix regressions and diff again.
5. **Re-measure** font bytes per page and the preloads. Confirm the extended files load only on pages that contain their characters, and that pages without them never request them.

## Report back

- Bytes before and after, per font file and per page (core, extended, total), with the preloads.
- Pixel-diff results per page and DPR, with each difference classified.
- Every place you deviated from the request, and what it cost (bytes kept, or a rendering change awaiting approval).
- How to rerun the script, and when it needs rerunning.
- Verification gaps, such as hinting on Windows.

**Later (proposals, not part of the default change):** embed the core font in the global stylesheet as a data URI to save a request, and precache the extended files after idle (`requestIdleCallback` with `FontFace.load()` or `rel=prefetch`). Weigh each against cache reuse and stylesheet size before proposing it.
