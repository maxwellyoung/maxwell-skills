# Coverage harness notes

A minimal raw-CDP rule-usage tracker for Playwright (Chromium only), and the behaviour it showed when tested. Extend it into the committed `coverage` script; don't treat it as the script.

```js
import { chromium } from 'playwright';

const browser = await chromium.launch();
const context = await browser.newContext({ viewport: { width: 1280, height: 800 } });
const page = await context.newPage();
const cdp = await context.newCDPSession(page);

await cdp.send('DOM.enable');
await cdp.send('CSS.enable');
const sheets = new Map();
cdp.on('CSS.styleSheetAdded', ({ header }) => sheets.set(header.styleSheetId, header));

// Start before navigation: rules that matched earlier are never reported.
await cdp.send('CSS.startRuleUsageTracking');
await page.goto(url);

// Drive one state, then take its delta so each rule can be attributed to a state.
await page.hover('nav a');
const { coverage } = await cdp.send('CSS.takeCoverageDelta');

// Breadth for pseudo-classes without event handlers.
const { root } = await cdp.send('DOM.getDocument');
const { nodeId } = await cdp.send('DOM.querySelector', { nodeId: root.nodeId, selector: 'a' });
await cdp.send('CSS.forcePseudoState', { nodeId, forcedPseudoClasses: ['hover', 'focus-visible'] });
await page.evaluate(() => document.body.offsetHeight); // force a style recalc

const { ruleUsage } = await cdp.send('CSS.stopRuleUsageTracking');
for (const r of [...coverage, ...ruleUsage]) {
  const { text } = await cdp.send('CSS.getStyleSheetText', { styleSheetId: r.styleSheetId });
  // sheets.get(r.styleSheetId).sourceURL identifies the file; isInline and isConstructed flag the rest.
  record(sheets.get(r.styleSheetId)?.sourceURL, r.startOffset, r.endOffset, r.used, text);
}
```

Observed in Chromium (Playwright 1.64):

- Only rules that were used appear in the results. A rule that never matched is simply absent, so compute "unused" as every style rule in the parsed stylesheet minus the reported ranges.
- `@font-face`, `@keyframes`, `@property`, `@layer` statements and the contents of a false `@supports` block never appeared, used or not.
- `a:hover` appeared only after a real hover or `CSS.forcePseudoState`. `@media print` rules appeared only after `page.emulateMedia({ media: 'print' })` and a style recalc.
- `:has(input:checked)`, `details[open] …` and rules inside `@layer` blocks were reported once the state existed.
- `:root { --unused: 1px }` was reported as used: coverage is per rule, not per declaration.
- Starting Playwright's `page.coverage.startCSSCoverage()` and a separate CDP tracker on the same page left one of them with empty results.
- Playwright's coverage returned nothing for a `<style>` set through `page.setContent`; its docs say injected `<style>` tags without a `sourceURL` are excluded. Serve real pages, and measure CSS-in-JS from the DOM.

Parse each stylesheet's text with a real CSS parser (PostCSS, Lightning CSS, csstree) to list its rules and offsets. Map minified offsets back to source with the stylesheet's source map when one exists; otherwise report against the built file and find the source rule by selector.
