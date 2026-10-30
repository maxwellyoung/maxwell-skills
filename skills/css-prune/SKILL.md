---
name: css-prune
description: "Remove CSS that no page uses and cut CSS bytes per page without changing how anything renders or behaves in any state, proven with runtime rule coverage across a page × state × viewport matrix, threshold-0 pixel diffs and computed-style diffs. Use when asked to remove unused or dead CSS, prune or purge stylesheets, cut CSS bytes or bundle size, reduce render-blocking CSS, clean up a Tailwind or utility-class build, dedupe CSS across bundles, audit CSS coverage, or make pages load less CSS."
---

# CSS prune

Make every page load less CSS while the site looks and behaves exactly the same in every state. Measure first, find which rules are actually used from the rendered site in every state that can occur, delete only what is proven dead, and prove the result with pixel and computed-style diffs from the same content.

## Rules

- **Rendering and behaviour are the contract.** Any visible or behavioural change, in any state, viewport, media mode or browser in the support range, is a regression unless the user approves it. If a cut would change output, don't ship it by default. Report what it saves and what it changes.
- **Unproven means kept.** A rule is deleted only when runtime coverage says no state used it and static search finds no reference to its selectors anywhere it could come from. Anything else stays and goes in the report as "kept, unproven".
- **Source edits, not new build steps.** Deleting dead rules from source files is a normal edit. Adding a purge step, a new PostCSS plugin, a minifier switch or a critical-CSS inliner to the build changes the pipeline, so propose it and wait for approval.
- **Browserslist is a contract.** Rules, prefixes and fallbacks for browsers still in the project's support range stay, even when Chromium never uses them.
- **Hand-run, committed, deterministic.** Coverage, diff and any generated-CSS scripts are run by hand, pin their tool versions, and produce the same output from the same input. Say in the script header and the repo's docs when to rerun them.
- **No deploys, pushes or external writes** without the user's explicit approval. A preview deploy for testing counts.
- **Check the outputs.** Coverage tools, purgers and minifiers have quirks. Read back what they produced instead of trusting the options you passed.

## 1. Measure the current state

Record this before changing anything, and keep it for the final report.

- **Per page:** every stylesheet requested, with transfer size (compressed, as served) and decoded size, from Playwright responses or Resource Timing (`transferSize`, `encodedBodySize`, `decodedBodySize`). Add inline `<style>` bytes in the HTML and CSS injected at runtime by CSS-in-JS. Note the compression the host actually serves; compare gzip or brotli bytes, not raw.
- **Render-blocking:** which stylesheets block first render (`<link rel=stylesheet>` in the head without a non-matching `media`, `@import` chains, which serialise). Chromium exposes `renderBlockingStatus` on resource timing entries. Note `<link rel=preload as=style>` and any `media=print` swap tricks already in use.
- **Per bundle:** rule count, selector count, bytes, and which pages load it. Look for the same rules shipped in several bundles (a component library imported by two entry points, a reset or Tailwind preflight included twice, global CSS imported from several layouts).
- **Pipeline:** how CSS is produced (plain files, Sass, PostCSS, Tailwind version and its content config, CSS modules, CSS-in-JS with or without static extraction, framework chunking), the minifier and its options, the browserslist query, and whether the framework inlines small stylesheets or splits CSS per route. Defaults differ (Vite keeps CSS imported by async chunks in separate files; Astro inlines stylesheets under `build.inlineStylesheets: 'auto'`'s 4 kB limit), and that decides what "per page" means and whether a byte saving shows up in HTML or in a CSS file.

## 2. Find what is actually used

Use both sources. Neither is enough alone.

- **Runtime coverage** sees the cascade as it really happened: rules that matched an element, in that state, in that browser. It misses every state you didn't drive, and it only runs in Chromium.
- **Static analysis** (grep, PurgeCSS-style extractors, the Tailwind scanner) sees every class name written as a complete string anywhere in source. It misses names built at runtime, classes in CMS or database content, and classes added by third-party scripts, and it cannot tell whether a matched rule is overridden.

### Runtime coverage

Run Chromium through Playwright over every built page, or a set that covers every template, component variant and content type. Use one of:

- `page.coverage.startCSSCoverage()` / `stopCSSCoverage()`, which returns each stylesheet's `url`, `text` and used `ranges`. It resets on navigation unless you pass `resetOnNavigation: false`, and it does not include dynamically injected `<style>` tags without a `sourceURL`, which is how many CSS-in-JS libraries inject.
- Raw CDP on `context.newCDPSession(page)`: `DOM.enable`, `CSS.enable`, `CSS.startRuleUsageTracking`, then `CSS.takeCoverageDelta` per state and `CSS.stopRuleUsageTracking` at the end. Each `RuleUsage` gives `styleSheetId`, `startOffset`, `endOffset` and `used`; map offsets back with `CSS.getStyleSheetText`, and track sheets with the `CSS.styleSheetAdded` event.

Behaviour to plan around (checked in Chromium; see `references/coverage-harness.md`):

- Start tracking **before** navigation. Rules that matched before tracking started are not reported.
- Don't run Playwright coverage and your own CDP tracker on the same page. They share the CSS agent and one steals the other's results.
- Only **style rules** are reported. `@font-face`, `@keyframes`, `@property`, `@counter-style`, `@page`, `@layer` statements and similar at-rules never show up, so coverage is not evidence that they are dead. DevTools' Coverage panel shows them as unused for the same reason.
- A rule counts as used only in a state you produced. `a:hover` is unused until something hovers a link; `@media print` rules are unused until you emulate print.
- A rule on `:root` or `body` is "used" even if half its custom properties are never read. Coverage is per rule, not per declaration.
- `CSS.forcePseudoState` (`hover`, `focus`, `focus-visible`, `active`, `visited` and so on) on a node makes its pseudo-class rules count. It is quick breadth, but it runs no event handlers, so JS-driven states still need real interaction.

### The state matrix

Coverage is per state, so drive every state that can occur, union the results per rule, and record which state first used each one. At minimum:

- **Pointer and keyboard:** hover, focus, `:focus-visible` (reach it with real `Tab` presses; programmatic `focus()` may not match), `:focus-within`, active, `:visited` (seed history first), `:checked`, `:indeterminate`, disabled, read-only, `:placeholder-shown`, `::selection` (select text), `:target` (navigate to a fragment).
- **Open things:** menus, dialogs with `::backdrop`, popovers, `<details open>`, tooltips, dropdowns, accordions, tabs, drawers, carousels at each slide, sticky headers after scroll, toasts and banners (cookie consent, announcements).
- **Data states:** form validation (`:invalid`, `:user-invalid`, server errors), empty, loading and skeleton, error, overflow (long names, many items), pagination ends, 404 and 500 pages, offline page.
- **Viewports:** every breakpoint in the CSS and one width either side of each, plus short heights where height media queries or `dvh` matter. Container queries respond to container size, not viewport, so check components in each container width they actually get.
- **Media modes:** `colorScheme: 'dark'` and `'light'`, `reducedMotion: 'reduce'`, `forcedColors: 'active'`, `contrast: 'more'` (Playwright `emulateMedia` or context options), `media: 'print'`, DPR 1 and 2 for resolution queries, `hasTouch` and `isMobile` for `(hover: none)` and `(pointer: coarse)`, `javaScriptEnabled: false` for `no-js` and `scripting` styles.
- **Locale and direction:** every shipped locale, real RTL pages (`dir=rtl`, `:dir()`, logical properties), `:lang()` rules, long-word languages.
- **Identity and flags:** logged out, logged in (Playwright `storageState`), each role or plan with its own UI, feature flags and A/B variants, first visit versus returning visitor.
- **Motion and time:** let animations and transitions run once, since `@starting-style`, `::view-transition-*` and transition end states only match mid-flow.

### Static references

For every rule that coverage never used, search for each class, id, attribute value and custom property name in:

- source (templates, components, scripts, server code, email templates, SVG), including names built from strings (`btn-${variant}`, `'is-' + state`, `classList.toggle`, `clsx` maps); search for the fragments, not just the full name;
- content: CMS exports, markdown and MDX (syntax-highlighting token classes such as `token`, `hljs-*` or Shiki's output only appear for languages used), rich-text fields, user-generated HTML;
- third-party widgets and libraries that add their own classes or read yours (embeds, consent managers, chat, payments, carousels, modal libraries, `html.js`/`no-js` toggles, `[data-state=open]` and `aria-expanded` attribute selectors from headless UI kits);
- other consumers of the same stylesheet: admin pages, docs, error pages, emails, embeddable widgets, server-rendered PDFs, iframes, Shadow DOM components that adopt it;
- JavaScript that reads CSS: `getComputedStyle(...).getPropertyValue('--x')`, `matchMedia`, `animationend` listeners that wait for a named `@keyframes`, `CSS.registerProperty`, `document.fonts` checks for a `@font-face` family.

Treat the union of runtime-used rules and statically referenced rules as live. Everything left is a candidate, and each candidate needs a reason in the report.

## 3. Cut

- **Delete whole dead rules and selector-list entries**, smallest first. Removing one selector from a list is safe only if the remaining list is still valid in every supported browser; one unknown selector in a list invalidates the whole rule, which is why `::-moz-selection` and `::selection` are written as separate rules. Don't merge them back.
- **At-rules:** delete `@keyframes` only when no `animation-name` in any rule or inline style, and no script, uses it; `@font-face` only when no `font-family` in any state resolves to it and nothing preloads it; `@property` only when nothing uses the property, because registration changes inheritance, initial value and whether it animates; custom properties only when no `var()`, inline style or script reads them.
- **Cascade traps.** Deleting a rule that never matches cannot change which rule wins. Deleting, moving or merging rules that do match can. Watch for:
  - declarations that look overridden in one state but win in another (a breakpoint, a pseudo-class, dark mode);
  - `@layer` order, which is fixed by the first appearance of each layer name; an empty-looking `@layer a, b;` statement or an early empty block sets the order for everything after it, so keep it;
  - source order between bundles, which can differ per route; deduplicating a rule into a shared bundle moves it, and can flip ties;
  - specificity changes from rewriting selectors (`:is()` takes its most specific argument, `:where()` adds none);
  - `@import` position, `@charset` and `@namespace` rules, and `!important` declarations.
- **Fallbacks and prefixes.** Keep `@supports not (...)` fallbacks, `-webkit-` and `-moz-` declarations and selectors, and old-syntax fallbacks needed by any browser in the browserslist range, even though Chromium coverage marks them unused. Hand-written prefixes that no supported browser needs may be deleted from source; check each against the target versions rather than assuming.
- **Utility frameworks.** In Tailwind, don't edit generated output; fix the input. In v4, detection is configured with `@source`, `@source not` and `@source inline()` (which brace-expands, for example `@source inline("{hover:,}bg-red-{100..900..100}")`). In v3 it is the `content` globs, `content.transform`/`content.extract`, and `safelist` with strings or `{ pattern, variants }`. Narrowing what gets scanned can drop classes that only appear in files outside the new globs, or in ignored ones (v4 skips `.gitignore`d files and `node_modules` by default), so check every class the old build emitted that the new one doesn't, one by one.
- **CSS modules:** unused exported classes are often dead, but composed classes (`composes:`), `:global` blocks and classes referenced through dynamic keys (`styles[variant]`) need the static search above.
- **CSS-in-JS:** runtime-injected styles may not show in coverage at all. Measure them from the DOM (`document.styleSheets`, `<style>` elements, constructed sheets) per state, and prune at the component, not the output.
- **Vendored CSS** (a UI kit or icon font stylesheet): if trimming it needs a tool, use a hand-run script with a pinned version and commit the trimmed output next to the original, which stays in the repo so the script can be rerun. PurgeCSS options worth knowing: `content`, `css`, `safelist` (array, or `standard`, `deep`, `greedy`, `keyframes`, `variables`), `blocklist`, `extractors`, `dynamicAttributes`, and `keyframes`, `fontFace`, `variables` (all off by default, which is the safe setting). Set `rejected: true` and read the list it removed.
- **Minifiers.** Don't switch minifier or change its options as part of this change. Minifiers differ in what they restructure: Lightning CSS merges only adjacent rules and adds or drops prefixes from its `targets` (`browserslistToTargets`), and has `unusedSymbols` for names you have proven unused; csso restructures by default (`restructure: true`) and its `forceMediaMerge` is documented as unsafe in general. If the user wants a minifier change, diff its output against the current one on its own and prove it separately.

## 4. Write the scripts

Commit them (for example under `scripts/css/`) with Playwright and any CSS tools pinned in the lockfile. They should:

- `coverage`: run the page × state matrix, union rule usage per stylesheet, and write a sorted, stable report: each rule's source location, used or not, the first state that used it, and the static references found. Same input, byte-identical report.
- `prove`: render the baseline and candidate across the matrix and write screenshots, diff images, computed-style diffs and a summary.
- any generator for vendored CSS: read the committed source and config, write the trimmed file deterministically, then re-read it and fail loudly if a safelisted selector, keyframe or font face is missing.

Put the rerun triggers in the header and the repo docs: new pages, templates, components or states; new CMS content types or markdown features; new third-party widgets; locale or RTL additions; a browserslist change; a Tailwind, framework or minifier upgrade.

## 5. Prove it

1. **Baseline from the same content.** Build the current CSS and the pruned CSS against identical content and code, for example by swapping only the stylesheets in a temporary copy. A diff against an older commit mixes content changes in.
2. **Screenshot matrix:** every page × every state in section 2 × every viewport and media mode, in Chromium, and in Firefox and WebKit for any page whose kept rules differ by engine (prefixes, `@supports` fallbacks). Fixed viewport and DPR, same browser builds before and after, fonts loaded (`document.fonts.ready`), caret hidden, network idle, time and random data frozen. Take full-page shots and element shots for open overlays.
3. **Animations.** Playwright's `animations: 'disabled'` makes screenshots stable but hides a deleted `@keyframes` or transition. Also capture, per state, `document.getAnimations()` (names, durations, keyframes via `effect.getKeyframes()`) and the computed `animation-*` and `transition-*` values, and diff them.
4. **Computed styles.** For key elements in each state (body, headings, links, buttons, inputs, overlays, each component root), diff the full `getComputedStyle()` result, plus `::before`, `::after`, `::marker` and `::placeholder`. This catches changes that are invisible in one screenshot: transitions, cursor, `pointer-events`, `user-select`, `scroll-*`, `will-change`, print-only values. `:visited` styles are hidden from `getComputedStyle` for privacy, so rely on screenshots there.
5. **Behaviour.** Run the existing test suite, then click through each interactive state once: focus rings visible, menus open and close, hit targets unchanged, nothing newly scrollable or clipped.
6. **Diff** screenshots with pixelmatch or ImageMagick at threshold 0. Report changed pixels and bounding boxes, save diff images, and classify every difference as a content change, a regression to fix, or harmless sub-pixel rounding (a few anti-aliased pixels, no layout, colour or shape change). Fix regressions and diff again.
7. **Re-measure** CSS bytes per page (transfer and decoded), render-blocking stylesheet count and bytes, and rule count per bundle. Rerun coverage on the result: the used set should be identical to before.

## Report back

- CSS bytes per page and per bundle before and after (compressed and decoded), render-blocking count and bytes, rules removed.
- The state matrix you drove, and any states or browsers you could not reach (logged-in roles without accounts, Windows forced colours, real devices).
- Pixel-diff and computed-style results per page, state and viewport, with each difference classified.
- Rules kept but unproven, with why (string-built class, CMS content you couldn't export, third-party widget, another browser), so the user can decide.
- Every place you deviated from the request, and what it cost (bytes kept, or a change awaiting approval).
- How to rerun the scripts, and when they need rerunning.

**Later (proposals, not part of the default change):** split route-specific CSS out of the global bundle, inline critical CSS and load the rest without blocking, add a purge or `unusedSymbols` step to the build with a reviewed safelist, or tighten the browserslist range to drop prefixes. Each changes the pipeline or the support contract, so weigh it against caching, flash-of-unstyled-content risk and maintenance before proposing it.
