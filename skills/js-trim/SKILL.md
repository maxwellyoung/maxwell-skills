---
name: js-trim
description: "Ship less JavaScript per page without changing how the site behaves: measure bytes and main-thread cost per route, find what each page actually runs across real interaction flows, cut dead, duplicate and oversized code, and prove it with before/after e2e flows, screenshots and bundle diffs. Use when asked to reduce bundle size, trim or shrink JavaScript, cut JS bytes, speed up page load or hydration, fix Total Blocking Time, long tasks or INP, analyse a bundle, remove unused code or duplicate packages, fix tree-shaking, or code-split a page."
---

# JS trim

Make each page ship and run less JavaScript while it behaves exactly as before. Measure first, find what each route actually executes across real flows, remove what nothing needs, and prove the result with the same flows run before and after.

## Rules

- **Behaviour is the contract.** Any visible or behavioural change is a regression unless the user approves it: different loading states, timing of interactivity, error handling, supported browsers, logging, or appearance. If an optimisation would change behaviour, don't ship it by default. Report what it saves and what it changes, and keep the current behaviour.
- **Use the repo's tools.** Keep the existing bundler, framework config, minifier and lockfile. Don't swap bundlers or minifiers, add build plugins, or change minifier options without approval. Dependency additions, removals, upgrades, `overrides`/`resolutions` and dedupes go through the repo's package manager and are listed in the report.
- **Measure production builds.** Dev servers ship unminified, unsplit, instrumented code and skip prefetching. Every number comes from the production build served the way production serves it (same compression, same headers), or from production itself when read-only access is enough.
- **Bytes are not the goal on their own.** Moving code between chunks can lower one route and raise the cost of a navigation. Judge changes per route and per flow, compressed and parsed, plus main-thread time.
- **Check the output.** Bundlers and analysers have quirks: a `sideEffects` flag can silently drop CSS, an alias can be ignored, an analyser can report pre-minification sizes. Read the emitted chunks and the network log, not the config you wrote.
- **Authority.** No deploys, pushes, preview publishing, source-map uploads or other external writes without the user's explicit approval.

## 1. Measure the current state

Record this before changing anything, on a fixed commit, and keep it for the final report.

- **Per route, initial load:** every script request with transferred (compressed) bytes, decoded bytes, and whether it came from cache. Playwright `request.sizes()` gives the encoded body size; CDP `Network.loadingFinished` gives `encodedDataLength`. Note the `content-encoding` actually served: analyser gzip estimates don't match a CDN serving Brotli.
- **Per route, hints:** every `<link rel=preload as=script>`, `rel=modulepreload` and `rel=prefetch` in the HTML, plus anything the framework prefetches at runtime (Next.js `<Link>` prefetches in production, Vite injects modulepreload for dynamic-import dependencies). Preloaded but unused scripts still cost bandwidth.
- **Per flow, not just per page:** client-side navigations between routes, opening menus and dialogs, submitting forms, validation and server-error paths, sign-in and sign-out, and anything gated by role or flag. Record which chunks each step requests.
- **Main thread:** Total Blocking Time and long tasks from a Lighthouse run or a Playwright trace, with a fixed CPU throttle and network profile, median of at least five runs. For interaction cost, record long animation frames (`PerformanceObserver` with type `long-animation-frame`, Chromium only) and INP from the `web-vitals` attribution build (`web-vitals/attribution`, `onINP`) during each flow. If field data exists (CrUX, RUM), note it as the real baseline; lab TBT is only a proxy for INP.
- **Hydration and inline payload:** size of inline JSON and framework payloads in the HTML (`__NEXT_DATA__`, RSC flight data in `self.__next_f`, serialised island props), and time from first paint to interactive for the key controls.
- **Support contract:** the resolved browser list (`npx browserslist` in the project), the bundler target, polyfill setup, and any legacy or `nomodule` build.

## 2. Find what each route actually runs

Static analysis says what could run. Collect what does run, from the running production build.

- **Runtime coverage across flows.** Use Chrome DevTools Coverage or Playwright `page.coverage.startJSCoverage()` (Chromium only; it resets on navigation by default, so collect per page or per flow). Drive every flow from step 1 before stopping coverage. Map unused ranges back to modules with source maps.
- **Unused at load is not dead.** Code that didn't run in your session may be the error boundary, the retry path, the offline handler, keyboard and screen-reader handlers, a locale, an admin view, an A/B variant, a date edge case (leap years, time zones, month boundaries), print styles toggled by script, a browser-specific branch, or code inside a Web Worker, service worker or iframe that page coverage doesn't capture. Treat low coverage as a candidate for deferral, and only treat code as removable when nothing can reach it.
- **Data-driven usage.** Icon names, component types, locales and chart types often come from a CMS, API or user settings. Find every source of those names before narrowing a registry to the ones you saw.
- **Global registration.** Modules imported for their side effects (`customElements.define`, `Chart.register`, `dayjs.extend`, i18n resource bundles, analytics bootstraps, polyfills, CSS imports) look unused to coverage and to tree-shakers but are load-bearing.

## 3. Find the waste

Build with source maps into a scratch output so the shipped build is unchanged. Vite's `build.sourcemap: 'hidden'` emits maps without the comment that would publish them. Then look at composition per route chunk, not just the whole bundle.

- **Analysers.** `source-map-explorer` on emitted chunks (`--gzip`, `--json` for diffing; `--coverage` colours by a coverage export). Next.js 16.1+ with Turbopack: `next analyze`, with `--output` writing to `.next/diagnostics/analyze` for before/after comparison. Next.js with webpack: `@next/bundle-analyzer` behind `ANALYZE=true`. Vite and Rollup: `rollup-plugin-visualizer` (`gzipSize`, `brotliSize`, `sourcemap: true` for post-minification sizes, `template: 'raw-data'` for diffing). Plain webpack: `webpack --profile --json > stats.json` then `webpack-bundle-analyzer stats.json dist` (compare parsed and gzip, not stat). esbuild: its `metafile` output in the esbuild bundle analyser. Add an analyser as a temporary, uncommitted dependency unless the user wants it kept.
- **Duplicates.** Same package at several versions or paths (`npm ls <pkg> --all`, `pnpm why <pkg>`, `yarn why <pkg>`), and the same code bundled into several chunks. Two copies of React, a router, a state store or a styling runtime can also mean two contexts. Deduping can fix or break behaviour, so test the affected flows and list any version that moves.
- **Shared chunk bloat.** A module imported by the root layout or app shell ships on every route. Check which routes actually use each large module in the shared or framework chunk.
- **Server/client boundary.** In React Server Component frameworks, a `"use client"` near the top of a tree pulls the whole subtree, and everything it imports, into the client. In islands frameworks, check each hydrated component and its directive.
- **Dev-only code in production.** Development builds of libraries (look for dev warning strings in the bundle), loggers, mock service workers, devtools, story or fixture data, and `process.env.NODE_ENV` or `import.meta.env.DEV` branches the bundler isn't replacing.
- **Polyfills and transpilation.** Polyfills for features every target already has, `core-js` imported whole (`useBuiltIns: 'entry'` with a broad list), regenerator or class transforms for targets that don't need them, and helpers inlined per file instead of shared.
- **Dead feature flags.** Branches behind flags that are permanently on or off. Confirm with the flag's source of truth (remote config, environment, database) that no environment or cohort still uses the other branch.

## 4. Cut

Each change is small and separately revertible. Prefer changes that leave runtime semantics identical.

- **Tree-shaking blockers.** Prefer ESM entry points over CommonJS (`main`-only packages, `require` inside ESM code). Replace imports through barrel files and `export *` chains with direct paths where the barrel defeats shaking, or use the framework's support (Next.js `experimental.optimizePackageImports`, which already covers packages such as `lucide-react`, `date-fns` and `lodash-es` by default). Namespace imports (`import * as x`) shake only when every member access is static; `x[name]` keeps everything. `/*#__PURE__*/` on top-level calls that are genuinely pure can unblock removal.
- **`sideEffects`.** Setting `"sideEffects": false` in the project's own `package.json` tells the bundler whole modules can be skipped. List every side-effectful file instead (CSS imports as `"*.css"`, polyfills, registrations), or styles and registrations disappear in production only. Diff the emitted CSS and the registration calls after changing it.
- **Heavy libraries, same library.** Import per method (`lodash/debounce` or `lodash-es` instead of `import _ from 'lodash'`), import only the date-library locales the site can select (with webpack, Moment's locales are excluded with `IgnorePlugin({ resourceRegExp: /^\.\/locale$/, contextRegExp: /moment$/ })`, then the needed ones imported explicitly), import icons individually or through an explicit map that covers every data-driven name, and register only the chart and editor modules in use.
- **Swapping libraries is a behaviour change.** Moment to date-fns or Day.js, lodash to native, or one icon set to another changes parsing, rounding, time-zone handling, debounce edge cases (`leading`, `trailing`, `maxWait`), `_.get` path semantics, `isEmpty` rules, or glyph shapes. Propose it with the saving and the differences found; don't ship it by default.
- **Server/client boundary.** Push `"use client"` down to the interactive leaf, pass server-rendered children through client wrappers, and move pure render-time work (markdown, syntax highlighting, formatting) to the server when it needs no browser API or state. Props crossing the boundary must stay serialisable, and context providers must still wrap their consumers. Guard server-only modules with the `server-only` package where the repo uses it.
- **Dev-only code.** Make sure the production define/replacement for `process.env.NODE_ENV` is in place, gate devtools and mocks behind build-time checks, and remove debug-only imports. Dropping `console` calls through the minifier changes what monitoring and support can see, so ask first.
- **Polyfills and targets.** Remove a polyfill only if every browser in the resolved support list has the feature natively, and nothing else (a third-party script, an embedded widget) relies on the global. **The browserslist query and bundler target are a support contract:** raising them, or updating `caniuse-lite`/`update-browserslist-db` so a relative query like `defaults` or `> 0.5%` resolves differently, is a behaviour change needing approval. Vite's default `build.target` (`'baseline-widely-available'`, pinned per major) also moves on a Vite major upgrade.
- **Dead flags.** Remove the dead branch and the flag check together, once its source of truth confirms it. Keep any flag a remote system can still flip.
- **Hydration.** Trim serialised props to what the client reads, avoid passing whole records to client components, and stop hydrating components that have no interactivity. Watch for new hydration-mismatch warnings.
- **Minifier settings.** Leave them alone. Options like `keep_fnames`, `keep_classnames` or unsafe transforms exist because code may depend on `Function.name`, class names in serialisation or error reports, or exact numeric behaviour.

## 5. Split and defer (behaviour changes to flag)

Dynamic `import()`, `next/dynamic`, `React.lazy`, route-level splitting and deferred hydration (`client:idle`, `client:visible`, `client:media` in Astro) save initial bytes but change behaviour. For each one, record and report:

- **Loading states:** a spinner, skeleton or blank space now appears where the content was immediate. With `ssr: false`, the server HTML no longer contains it (bad for SEO, no-JS users and layout). In Next.js App Router, `ssr: false` only works inside Client Components.
- **Layout shift:** reserve the final size, and measure CLS before and after.
- **Waterfalls:** a lazy chunk that imports another lazy chunk serialises requests. Check the network log for chained requests and add a modulepreload or prefetch only where the bundler doesn't.
- **Lost or delayed interaction:** clicks, keystrokes and focus before the module or island loads may be dropped. Moving an island from `client:load` to `client:visible` changes when it becomes interactive.
- **Failure modes:** chunk loads fail offline, on flaky networks, and after a deploy when an open tab requests a chunk hash that no longer exists (`ChunkLoadError` and similar). Check there is an error boundary or retry, and that service-worker precaching (if any) includes the new chunks.
- **Accessibility:** focus management and announcements when lazy content mounts in dialogs and menus.

Prefer splitting code that only runs after an explicit user action (editor, chart, dialog contents, admin tools) and prefetching it on intent (hover, focus, idle) so the visible behaviour stays close to today's.

Third-party scripts (analytics, tag managers, chat, ads, embeds) are usually the largest main-thread cost but involve consent, contracts and data flows. Measure and report their share; hand the work to the `third-party-scripts` skill rather than changing them here.

## 6. Prove it

1. **Baseline from the same code paths.** Build the current commit and the changed commit with the same toolchain, environment variables, flags and data, and serve both the same way. Don't compare against production unless production runs the same commit and data.
2. **The repo's own checks:** test suite, type check, lint and build, all passing as before. A test that was skipped or flaky before stays noted, not hidden.
3. **Scripted e2e flows** (Playwright or the repo's runner) for every flow from step 1, run against both builds. Capture `console` errors and warnings, `pageerror`, `requestfailed`, and responses of 400 or above. Any new message or failed request is a regression until explained.
4. **Split-chunk failure checks:** for each new lazy chunk, run its flow with the network throttled (loading state visible and sized), offline after load (error path handled), and with the chunk request blocked.
5. **Screenshots of affected states:** pages and components touched, loading states, dialogs, menus, error states and any server/client boundary change, at fixed viewport and DPR with animations off. Diff them and classify every difference as an approved change, a regression to fix, or harmless rendering noise.
6. **Re-measure** bytes per route and flow (compressed and decoded), preloads and prefetches, TBT, long animation frames and INP under the same throttling and run count. Confirm moved code now loads only where it's needed and nothing new loads on routes that didn't need it.
7. **Bundle diff:** analyser output or module lists before and after. Every module that disappeared should be explained by a change you made; every module that appeared or grew is a regression until explained. Check the emitted CSS is unchanged unless the change intended otherwise.

Keep the measurement and flow scripts in a scratch location, or commit them if the user wants a repeatable harness.

## Report back

- Bytes before and after per route and per key flow (compressed and decoded), with the preloads and prefetches, and the main-thread numbers (TBT, long frames, INP) with run count and throttling.
- Each change, its saving, and why it leaves behaviour unchanged.
- e2e, console, network and screenshot results, with each difference classified.
- Changes that would save more but change behaviour (library swaps, raised targets, splits with new loading states, removed polyfills), with the saving and the exact behaviour change, awaiting approval.
- Dependency changes and any version that moved.
- Verification gaps: browsers not tested (coverage and long-animation-frame data are Chromium only), flows that need credentials or production data, field INP not yet observable.

**Later (proposals, not part of the default change):** a per-route size budget in CI using the repo's existing tooling, an import lint rule against whole-package imports of heavy libraries, differential serving of a modern build alongside a legacy one, and field monitoring of INP with attribution. Weigh each against maintenance cost before proposing it.
