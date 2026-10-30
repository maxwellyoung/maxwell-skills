# SVGO plugin risks

Checked against SVGO v4 docs and source (v4.1.0 was the latest release when written). Re-check against the version you pin: run `svgo --show-plugins` and read the docs for that version before relying on any default below.

## v3 to v4 changes that matter here

- `removeViewBox` and `removeTitle` are no longer in `preset-default`. Configs copied from v3 that disable them are now no-ops; configs that enable them opt back in to breakage.
- `removeScriptElement` is now `removeScripts` and also strips event handler attributes and `javascript:` URIs.
- Only named exports (`import { optimize } from 'svgo'`); the CLI reads `svgo.config.mjs` by default, or `--config <file>`.
- Wrappers (SVGR, bundler plugins) may run a different plugin set from `preset-default`.

## `preset-default`, in order, with what to check

| Plugin | Risk to check |
| --- | --- |
| `removeDoctype`, `removeXMLProcInst` | Safe for web use. Diff any file that declares entities in its doctype. |
| `removeComments` | Keeps `<!--! … -->` legal comments by default (`preservePatterns`). Plain licence comments without `!` are removed. |
| `removeMetadata`, `removeEditorsNSData` | Licence or attribution in `<metadata>` disappears; keep it if the licence requires it. |
| `cleanupAttrs` | Whitespace only. |
| `mergeStyles`, `inlineStyles`, `minifyStyles` | Rewrite internal `<style>`. Check `:hover`, media queries (SVG favicon dark mode), `@keyframes`, specificity and order, and classes JS toggles. |
| `cleanupIds` | Removes IDs not referenced inside the file (sprite `<symbol>`, `<view>`, IDs used by page CSS or JS) and minifies the rest, which collides across inlined SVGs. Skips the file when it has `<style>` or `<script>` unless `force`. Params: `remove`, `minify`, `preserve`, `preservePrefixes`, `force`. |
| `removeUselessDefs` | Drops `<defs>` children without an ID. Check anything referenced from outside. |
| `cleanupNumericValues` | Rounds numbers and removes `px`. Check tiny coordinate systems. |
| `convertColors` | Shortens colour syntax. Its `currentColor` param (off by default) is a theming change; leave it off. |
| `removeUnknownsAndDefaults` | Deletes attributes it doesn't know for the element (`role` unless `keepRoleAttr`, `focusable`, non-`data-*` hooks) and default or inherited-duplicate values, judged from the file's own CSS only. Keeps `aria-*`, `data-*`, `xmlns`. Skips default and override removal on elements with an `id`. |
| `removeNonInheritableGroupAttrs` | Removes presentation attributes from groups that can't inherit them. Check selectors that read them. |
| `removeUselessStrokeAndFill` | Removes stroke or fill it judges invisible from the file alone. Bails out when the file has `<style>` or `<script>`. `removeNone` (off) deletes whole elements; leave it off. |
| `cleanupEnableBackground` | Legacy attribute; safe unless a filter depends on it. |
| `removeHiddenElems` | Removes `opacity` 0, `display: none`, hidden and zero-size elements. Check anything a state, animation or script reveals. |
| `removeEmptyText` | Safe unless script fills the text later. |
| `convertShapeToPath` | Converts `rect` (without `rx`/`ry`), `line`, `polyline`, `polygon` to `path`; `circle`/`ellipse` only with `convertArcs`. Breaks selectors and scripts on element type or geometry attributes. |
| `convertEllipseToCircle` | Element type changes; same selector check. |
| `moveElemsAttrsToGroup`, `moveGroupAttrsToElems`, `collapseGroups` | Move `fill`, `transform`, `class` and others between elements and remove groups. Check child selectors, JS hooks and animation targets. |
| `convertPathData` | Rewrites path data; `floatPrecision` 3 and `transformPrecision` 5 by default; `applyTransforms` and `applyTransformsStroked` bake transforms in. Check large renders, `objectBoundingBox` units, stroked icons and path-length animations. |
| `convertTransform` | Rewrites or collapses transforms. Check transform-origin-dependent CSS animation. |
| `removeEmptyAttrs`, `removeEmptyContainers` | An empty group may be a script or animation target. |
| `removeUnusedNS` | Safe; `xmlns` on the root is not removed here. |
| `mergePaths` | Skips paths with differing attributes, markers, clip-path, mask or `url()` paint, and intersecting paths unless `force`. Still breaks per-path styling, hover, staggered animation and path-length effects. |
| `sortAttrs`, `sortDefsChildren` | Order only. |
| `removeDesc` | By default removes only empty or editor-generated `<desc>`; `removeAny` removes all. Leave `removeAny` off. |

## Not in the preset, and what enabling them costs

- `removeViewBox`: stops scaling. Don't enable.
- `removeTitle`: removes the accessible name. Don't enable for meaningful icons.
- `removeXMLNS`: breaks `<img>`, CSS and standalone use. Inline-only markup.
- `removeDimensions`: removes `width`/`height`; changes intrinsic size in `<img>` and CSS.
- `prefixIds`: prefixes IDs, classes, `href="#…"`, `url(#…)`, `<style>` selectors and SMIL `begin`/`end` references. Defaults: `delim` `__`, `prefixIds` true, `prefixClassNames` true; with no `prefix` it uses the file basename when `path` is passed. Turn `prefixClassNames` off when outside CSS or JS targets inner classes.
- `removeScripts`: removes behaviour by definition.

## Starter config

Adjust per usage mode; this is a safe starting point for files that are both inlined and used as `<img>`.

```js
// svgo.config.mjs — pinned with the exact SVGO version in package.json
export default {
  multipass: true,
  js2svg: { pretty: false },
  plugins: [
    {
      name: 'preset-default',
      params: {
        overrides: {
          // IDs are prefixed below instead of minified, so inlined icons don't collide.
          cleanupIds: { minify: false },
          // Keep role="img"; check the output for other non-presentational attributes.
          removeUnknownsAndDefaults: { keepRoleAttr: true },
        },
      },
    },
    // Prefix derives from the filename when the script passes `path`.
    { name: 'prefixIds', params: { prefixClassNames: false } },
  ],
};
```

`cleanupIds` with `remove` left on still deletes IDs referenced only from outside the file. For sprites, `<view>` stacks and IDs used by page CSS or JS, list them in `preserve` or `preservePrefixes`, or set `cleanupIds: false`.
