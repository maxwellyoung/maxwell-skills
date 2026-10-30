---
name: svg-icon-optimize
description: "Shrink a website's SVG files and icon delivery to what the site actually uses without changing how anything looks or behaves, with a committed SVGO config and hand-run script, proven by per-icon pixel diffs in every size, DPR and colour state plus an accessibility-tree check. Use when asked to optimise or minify SVGs, run SVGO, shrink icons, cut icon or SVG bytes, remove unused icons, dedupe inlined SVGs, trim an icon package or icon font, build or fix an SVG sprite, or make icons load faster."
---

# SVG icon optimise

Make a site's SVGs and icons smaller and cheaper to deliver while every icon looks and behaves exactly as it does now. Measure first, find every way the site uses SVG and which icons it really renders, optimise each file with a configuration chosen for how that file is used, and prove the result with per-icon pixel diffs and an accessibility check from the same content.

## Rules

- **Appearance and behaviour are the contract.** Any change to pixels, layout, hit area, animation, theming, script hooks or the accessibility tree is a regression unless the user approves it. If an optimisation would change any of these, don't apply it by default. Report what it saves and what it changes, and keep the current result.
- **Configure per usage mode.** A file used in `<img>` or CSS needs different safety settings from markup inlined into HTML. Never run one aggressive config over everything.
- **No build step.** Write a script that is run by hand, commit its config and its output, and say in the script header and the repo's docs when it needs rerunning. Don't add SVGO, SVGR options or bundler plugins to the build unless the user asks.
- **Pin the tools.** Commit the SVGO config file (`svgo.config.mjs`) and pin the exact SVGO version. Plugin defaults have changed between major versions, and framework wrappers (SVGR and similar) apply their own defaults rather than `preset-default`.
- **Check the generated files.** SVGO bails out, skips or rewrites in ways the command line doesn't show. Read back every output and assert on it; don't trust the flags you passed.
- **Delete nothing public without approval.** Unused files under a public path may be linked from emails, CMS content, other sites or docs. Removing them from bundles is in scope; deleting served files is a proposal.
- **Authority.** No deploys, pushes, publishes or other external writes without the user's explicit approval.

## 1. Measure the current state

Record this before changing anything, and keep it for the final report.

- **Per SVG file:** bytes raw, gzip and brotli, element and path counts, `viewBox`, `width`/`height`, IDs defined and referenced, `<style>`, `<script>`, SMIL elements, `<title>`/`<desc>`, `role` and `aria-*`, and which editor exported it (metadata and namespaces hint at how much is removable).
- **Per page:** SVG bytes transferred as separate requests (Playwright response sizes or CDP `encodedDataLength`), inline SVG bytes inside the HTML, SVG data URIs inside CSS, and icon bytes inside JS bundles (bundle analyser or source map explorer). Measure inline bytes compressed as well as raw: repeated inline icons compress well, so raw counts overstate the saving from deduplication.
- **Duplication:** hash normalised markup and count how many times each icon is inlined per page and across pages, and how many near-identical copies exist under different names or colours.
- **Icon packages and fonts:** which icon packages ship to the client and how much of each (barrel imports and dynamic lookups such as `icons[name]` defeat tree shaking), and each icon font's size and glyph count.

## 2. Find every usage mode

Collect this from the rendered site, not just the source tree. A browser pass over every built page, or a representative set covering every template and component, is the reliable source. Record per icon which modes use it, because the safe optimisations differ:

- **Inline `<svg>`** in HTML or components. Shares one document with every other inline SVG, so IDs collide, page CSS can reach inner elements, and `currentColor` and custom properties inherit from the page.
- **`<img src>`, `<picture>`, `<object>`, `<embed>`, `<iframe>`.** A separate document: needs `xmlns`, gets no page CSS, inherits no `color`, runs no script in `<img>`. Its `width`/`height` attributes and `viewBox` set its intrinsic size, which drives layout when CSS doesn't size it.
- **CSS `background-image`, `mask-image`/`-webkit-mask-image`, `list-style-image`, `content: url()`, `border-image`, `cursor`.** Same document rules as `<img>`. Data URIs need `#` encoded as `%23` and quotes escaped; masks use only alpha, so colour changes there are invisible but shape changes are not.
- **`<use href>`** into an inline sprite, an external sprite file, or a `#fragment` inside the same file. The referenced IDs are a public API.
- **Fragment views and stacks:** `icons.svg#name` with `<view>` elements or `:target` styling. Those IDs are referenced from outside the file.
- **Icon fonts:** ligature or codepoint glyphs, usually in `::before`/`::after`.
- **Framework icon components and packages:** compiled to JS that emits markup at runtime; the SVG lives in the bundle, not on disk.
- **Platform icons:** `<link rel=icon type=image/svg+xml>` (often with an internal `@media (prefers-color-scheme)` rule), `mask-icon`, manifest icons, and any raster fallbacks next to them.

## 3. Find which icons are actually used

- **Rendered usage:** every inline `<svg>` and `<use>` target, every `<img>` and `<source>` with an SVG URL, and the computed `background-image`, `mask-image`, `list-style-image` and `content` of every element and pseudo-element.
- **State-only usage:** a single pass sees only the resting state. Search stylesheets for `url()` and icon-font codepoints under `:hover`, `:focus-visible`, `:active`, `:checked`, `[aria-expanded]`, `[disabled]`, `prefers-color-scheme`, `forced-colors`, `print` and breakpoints, and drive menus, modals, toasts, validation errors, empty states, loading spinners and the 404 page.
- **Dynamic names:** icons chosen from CMS data, config, translations or user content (`<Icon name={item.icon}>`) can't be found from rendered pages alone. Enumerate the data source, or keep the whole set and say why.
- **Non-site consumers:** emails, RSS, OG images, docs and other apps that load the same files. Treat them as users of those files.

## 4. Optimise each file

Start from SVGO `preset-default` and change it per usage mode. Verify plugin names and defaults against the pinned version's docs before relying on them; `references/svgo-plugins.md` lists the current preset with the risk of each plugin.

- **`viewBox`: always keep it.** Without it the icon stops scaling to its CSS box. Current SVGO leaves `removeViewBox` out of `preset-default`; older versions and some wrappers still run it. Don't enable it, and assert `viewBox` survives.
- **`xmlns`: keep it for any file used outside inline HTML.** `removeXMLNS` makes the file unusable in `<img>`, CSS and as a standalone asset. Enable it only for markup that is guaranteed to be inlined.
- **`width`/`height`:** keep them where they set intrinsic size (`<img>` without CSS dimensions, CSS backgrounds with `auto` sizing). Removing them can change layout and cause layout shift.
- **IDs (`cleanupIds`, `prefixIds`).** `cleanupIds` removes IDs not referenced inside the file and minifies the rest. That deletes sprite `<symbol>` IDs, `<view>` IDs and anything targeted from page CSS (`url(#grad)`), JS (`getElementById`, `querySelector('#…')`), anchors, tests or analytics. Minified IDs from different files collide when several are inlined on one page, and the first definition in the document wins, so the second icon silently picks up another icon's gradient, clipPath, mask, filter, marker or pattern. For inlined SVGs, disable minification or run `prefixIds` with a stable, filename-derived prefix; a counter-based prefix is non-deterministic and breaks SSR hydration. `prefixIds` also prefixes class names by default (`prefixClassNames`), which breaks page CSS or JS that targets inner classes; turn that off when anything outside the file uses them. `cleanupIds` does nothing when the file has a `<style>` or `<script>` unless forced, so check whether it ran.
- **Paths (`convertPathData`, `mergePaths`, `convertShapeToPath`, `convertTransform`).** Rounding (`floatPrecision`, default 3 decimals in path data) is absolute, not relative: it is invisible on a 24-unit `viewBox` but visible on tiny coordinate systems (`objectBoundingBox` clipPaths, masks and gradients, `0 0 1 1` viewBoxes) and on icons drawn very large. Lower precision only after diffing at the largest rendered size. `mergePaths` refuses paths with different attributes, markers, clip-paths, masks or `url()` paints, but merging still changes behaviour that depends on separate elements: CSS or JS targeting one path, per-path hover, staggered animations, overlapping translucent paths, `fill-rule` interactions, and `pathLength`/`getTotalLength()` line-drawing effects. `convertShapeToPath` turns `<rect>` (without rounded corners), `<line>`, `<polyline>` and `<polygon>` into `<path>`, which breaks selectors and scripts that target the element type or its `x`, `width` or `points`. Baking transforms into path data changes how strokes scale; diff stroked icons, and any with `vector-effect: non-scaling-stroke`, at several sizes.
- **Paint and theming (`removeUnknownsAndDefaults`, `removeUselessStrokeAndFill`, `convertColors`, `moveElemsAttrsToGroup`, `collapseGroups`).** These reason from attributes and the file's own `<style>`, not from page CSS. An attribute that looks redundant can be the hook a theme overrides, the explicit value that stops an outer `fill` or `currentColor` from inheriting, or the value a `:hover` rule animates from. Moving or collapsing groups changes which element carries `fill`, `transform`, `opacity` or `class`, which breaks selectors such as `.icon > path`. Don't let `convertColors` turn colours into `currentColor` (or the reverse); that is a theming change.
- **Styles (`inlineStyles`, `minifyStyles`, `mergeStyles`).** Internal `<style>` may carry `:hover`, `@media (prefers-color-scheme)` (SVG favicons), `@keyframes` and selectors that JS toggles. Confirm each survives with its specificity and order. Moving a value from a presentation attribute into `style`, or the reverse, changes how page CSS overrides it: presentation attributes lose to any author CSS, inline `style` beats it.
- **Hidden content (`removeHiddenElems`, `removeEmptyContainers`, `removeUselessDefs`).** Elements at `opacity="0"`, `display="none"` or zero size may be shown later by CSS, SMIL, JS or a hover state. Invisible `<rect>`s that pad the bounding box (`fill="none"` frames) change `getBBox()`, hit area and `object-fit` in some setups. Keep anything a state, script or animation reaches.
- **Accessibility (`removeTitle`, `removeDesc`, `removeUnknownsAndDefaults`).** `<title>` and `<desc>` are accessible names and descriptions; current SVGO leaves `removeTitle` out of the preset and `removeDesc` only removes empty or editor-generated descriptions by default. `removeUnknownsAndDefaults` keeps `aria-*` and `data-*` but deletes any attribute it doesn't know for that element: `role` (unless `keepRoleAttr` is set, so `role="img"` disappears), `focusable="false"` (still needed to keep old Edge and IE from tabbing into SVGs) and framework or test hooks that aren't `data-*`. Set `keepRoleAttr`, and check the output for every attribute the source had that isn't presentational. Decorative icons (`aria-hidden="true"`) must stay decorative.
- **Animation and scripts.** SMIL timing references (`begin="a.end"`), `<animate href>`, CSS animations and JS that queries inner elements all depend on IDs, element types and structure. `removeScripts` (not in the preset) changes behaviour by definition.
- **Keep:** `preserveAspectRatio` values that aren't the default, `vector-effect`, `shape-rendering`, `paint-order`, `overflow`, `clip-rule`/`fill-rule`, legal comments (`<!--! … -->`, which `removeComments` keeps by default) and licence notices.
- **Exclude** files SVGO can't help or will harm: already-minified icon-package output, embedded rasters (`<image href="data:…">`; recompressing them is a separate, visible decision), and files with `<foreignObject>` or heavy scripting. Report them rather than forcing them through.

## 5. Optimise delivery

- **Inlined duplicates.** When one page inlines the same icon many times, define it once (a hidden inline sprite of `<symbol>`s) and reference it with `<use href="#id">`. This moves the icon into a shadow tree: page CSS can no longer select inner elements, only inherited properties (`fill`, `stroke`, `color`) and custom properties pass in, and per-path hover or theming stops working. Apply it only to icons whose styling survives that; otherwise report the saving.
- **External sprite files** are cached across pages but must be same-origin (cross-origin `<use>` doesn't load), and a page then downloads every icon in the sprite. Weigh that against inline bytes per page before proposing it.
- **Icon packages.** Switch to per-icon imports where the package supports them, and replace dynamic name lookups with an explicit map of the icons in use. Confirm the emitted markup is unchanged.
- **Icon fonts.** Subsetting the font to the glyphs in use keeps rendering identical (use the `font-subset` skill). Converting the font to SVG changes baseline alignment, hinting, sub-pixel positioning, text decoration and selection behaviour, and how it reacts to `font-size`, `color` and forced colours. That conversion is a proposal needing the user's approval.
- **Raster fallbacks** (PNG for old clients, emails, platform icons) stay as they are.

## 6. Write the script

Commit one script (for example `scripts/svg/optimise.mjs`) and `svgo.config.mjs`, with SVGO pinned to an exact version through the repo's package manager, touching only that entry in the lockfile. It should:

- read the source SVGs from a committed source directory and a committed per-file config: usage mode, prefix, exclusions and per-icon overrides;
- write the optimised files and any generated sprite or icon map deterministically, so rerunning on the same input gives byte-identical output, and passing `path` so `prefixIds` derives its prefix from the filename;
- re-read every output and fail loudly if it doesn't parse as XML, lost its `viewBox`, lost `xmlns` while used outside inline HTML, lost a `<title>`, `role` or `aria-*`, has an `href="#…"` or `url(#…)` that doesn't resolve, or defines an ID that collides with another SVG inlined on the same page; then print bytes before and after per file.

Put the rerun triggers in the header and the repo docs: new or changed icons, new usage modes (an inline icon now used in `<img>`, or the reverse), new pages that inline several icons together, an SVGO upgrade, or a change to theming or animation that targets inner elements.

## 7. Prove it

1. **Baseline from the same content.** Swap only the SVG files, sprite and icon imports in a temporary copy of the current content. A diff against an older commit mixes content changes in with icon changes.
2. **Icon harness plus real pages.** Render every icon in every usage mode it has, at every size the site renders it, at DPR 1, 2 and 3. Then shoot the real pages and components that use them, so collisions, inheritance and layout show up in context.
3. **States.** Default, hover, focus-visible, active, selected, disabled and error; light and dark (`emulateMedia({ colorScheme })`); `forcedColors: 'active'`; reduced motion; print where icons appear in print. Freeze animations at fixed times (`pauseAnimations()` and `setCurrentTime()` for SMIL, `document.getAnimations()` for CSS) and shoot several frames.
4. **Diff** with pixelmatch or ImageMagick at threshold 0, using the same browser build for before and after, and repeat in WebKit and Firefox if their users matter, since engines rasterise differently. Report changed pixels and bounding boxes, save diff images, and classify each difference as a content change, a regression to fix, or harmless sub-pixel rounding (a few anti-aliased edge pixels, no shape, position or colour change). Fix regressions and diff again.
5. **Behaviour and accessibility.** Compare the accessibility tree before and after (Playwright `ariaSnapshot()`), including accessible names of icon-only buttons and links. Check that element bounding boxes, hit areas, scripted lookups and line-drawing lengths are unchanged where code depends on them.
6. **Re-measure** SVG bytes per file and per page (requests, inline HTML, CSS data URIs, JS bundles), raw and compressed, and confirm nothing new downloads where it didn't before.

## Report back

- Bytes before and after per file and per page, raw and compressed, split by usage mode, with bundle deltas for icon packages.
- Pixel-diff results per icon, size, DPR, state and engine, with each difference classified, and the accessibility-tree comparison.
- Icons found unused, and which were removed from bundles versus left in place pending approval.
- Every place you deviated from the request or held back an optimisation, and what it would save and change.
- How to rerun the script, and when it needs rerunning.
- Verification gaps, such as engines or states you didn't shoot, or dynamic icon names you couldn't enumerate.

**Later (proposals, not part of the default change):** a shared external sprite, converting an icon font to SVG, converting `<img>` icons to inline markup for `currentColor` theming, deleting unused public files, enabling `removeTitle`, `removeXMLNS` or lower precision for files that provably allow it, and wiring the optimisation into the build. Weigh each against caching, styling hooks and accessibility before proposing it.
