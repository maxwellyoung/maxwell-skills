---
name: third-party-scripts
description: "Cut the performance cost of a website's third-party scripts (analytics, tag managers, pixels, ads, A/B testing, session replay, chat, video, maps, social, reviews, payment and captcha embeds) while the data, consent and features they provide keep working, proven with before/after network logs, captured beacons, smoke tests, Web Vitals and screenshot diffs. Use when asked to speed up third-party scripts, audit tags or pixels, reduce tag manager bloat, defer or lazy-load analytics or chat widgets, fix INP or TBT caused by vendors, add facades for embeds, clean up GTM, or find which scripts slow the page down."
---

# Third-party scripts

Make a site's third-party scripts cheaper to load and run while every kept vendor still gets the same data, consent is still honoured, and every embed still works. Inventory from the network rather than the source, measure what each script costs, find out with the user what is still used, propose cuts and loading changes, and prove the result with captured beacons, smoke tests and before/after measurements under the same conditions.

## Rules

- **Behaviour is the contract.** Any visible change, behavioural change or change in what data is collected, when, or under which consent state is a regression unless the user approves it. If an optimisation would change any of these, don't ship it by default. Report what it saves and what it changes, and keep the current result.
- **Removing a vendor is not a performance decision.** Marketing, analytics, ads, support, legal and contracts often depend on a tag nobody in the codebase remembers. Inventory and propose; remove or delay a vendor only with the user's explicit approval for that vendor.
- **Stay local.** No deploys, pushes, tag manager publishes, container version changes or vendor dashboard edits without explicit approval. Tag manager changes go in a workspace or exported container file for the user to review; a published container is production.
- **Treat identifiers as sensitive.** Container IDs, measurement IDs, pixel IDs, API keys, site keys and captured payloads (which can carry user IDs, emails or hashed identifiers) stay out of commits, reports and logs. Redact them in anything you write down.
- **Consent first.** Never make a script load, set cookies or send requests earlier than it does today in any consent state. Privacy regressions are the worst kind here, and they are invisible in a performance trace.
- **Don't trust the vendor snippet or the docs.** Vendors change loaders and endpoints. Check what actually goes over the wire.

## 1. Inventory from the network

Source code lists only the direct tags. Tag managers, CDPs and scripts themselves inject more at runtime, so build the inventory from real page loads.

- **Load matrix.** Load every distinct template (home, listing, product or article, checkout or form, account, error page) in a clean profile under each condition that changes what loads: no consent decision, consent rejected, consent accepted, each partial category if the consent tool has them, logged out and logged in, and each region or locale with different rules (EU versus elsewhere, if the CMP or tag manager geo-targets). Include a scroll to the bottom, an idle wait (some tags fire on a timer or `requestIdleCallback`), and the key interactions (open chat, play video, submit a form, add to cart), because many tags fire only on those events.
- **Capture.** Record a HAR per cell, or use Playwright with request and response listeners. For each request note the initiator chain (the CDP `Network.requestWillBeSent` `initiator` field or the DevTools Initiator tab) so every script traces back to who loaded it: a direct tag, the tag manager, or another vendor script.
- **Look inside the tag managers.** For GTM, read the container (an export if the user can give you one, otherwise Preview mode or the loaded `gtm.js` content) and list every tag, its trigger and its consent settings. For a CDP such as Segment, list the enabled destinations that run in the browser (device mode) rather than server side. Note tags that fire on every page but are only needed on one.
- **Embeds.** List every iframe and widget: video, maps, social posts and buttons, chat and support, reviews, A/B testing and personalisation, session replay and heatmaps, ads, payment, captcha, and consent tools themselves. Note what the user sees, what it loads before any interaction, and its sub-requests.
- **Also catch:** service workers registered by vendors, `<link rel=preconnect|dns-prefetch|preload>` hints pointing at vendors (including stale ones for vendors no longer present), inline snippets that create queue stubs, `document.write`, and cookies and storage keys each vendor sets (DevTools Application panel, or the `Set-Cookie` headers and `document.cookie` after load).

## 2. Measure what each script costs

Record this before changing anything, and keep it for the final report. Use a fixed device profile and throttling (the same CPU slowdown and network preset every run), the same browser build, a cold cache, and enough runs to see the variance (five or more per cell; report the median).

- **Network, per vendor:** script and sub-request bytes (transfer and decoded), request count, distinct origins, and whether each origin needs its own DNS, TCP and TLS setup. Check whether a vendor's origin is already warmed by a `preconnect`, and whether any `preconnect` is wasted (connections opened that are never used, or opened before consent allows the vendor).
- **Main thread, per vendor:** script evaluation, compile and task time, and long tasks over 50 ms attributed to it. The DevTools Performance panel can dim third parties and has a 1st/3rd party table in the Summary tab with transfer size and main-thread time per entity; Lighthouse reports a third-party breakdown as well. Attribution by origin breaks when a vendor is self-hosted or proxied through the site's own domain, so map those by URL path.
- **Web Vitals:** LCP (does a vendor delay the LCP resource, compete for bandwidth, or hide content as anti-flicker does), INP (vendor event listeners on clicks, keypresses and scroll; session replay and heatmap tools often add input handlers), CLS (late-injected banners, chat bubbles, ad slots and review widgets without reserved space), and TBT in the lab. Use field data (CrUX or the site's RUM) if it exists; lab numbers show the cause, field numbers show whether it matters.
- **Other costs:** `unload` handlers added by vendors (they can make a page ineligible for the back/forward cache), polling timers, `MutationObserver`s over the whole document, and memory growth on long sessions.

## 3. Find out who owns each script and whether it is used

Build one table, a row per vendor: what it is, which pages and conditions load it, cost from step 2, cookies set, who owns it, what it is for, and whether its output is still used. Most of the last three need the user, so ask rather than guess.

- **Duplicates.** Two analytics tools measuring the same thing, the same pixel loaded directly and through the tag manager (double-counted conversions), GA4 configured twice, a CDP destination that duplicates a direct tag.
- **Probably dead.** A/B tests with no live experiment, pixels for ad accounts no longer spending, heatmap or replay tools nobody has logged into, chat widgets for a support tool that changed, tags for retired campaigns, legacy Universal Analytics snippets. Treat these as candidates for the user to confirm, never as facts.
- **Over-scoped.** A checkout or support script loading on every page, a map library on pages without a map, session replay recording at 100% when a sample would do (a data change, so it needs approval).

## 4. Propose changes

For each change, state the saving, what it changes, and the trap below that applies. Order proposals by saving per unit of risk. Group them as: safe (no change to data, consent or UX), needs confirmation (removal or scoping of a vendor), and changes behaviour (needs explicit approval).

- **Remove or scope** (approval per vendor): delete dead tags, fix duplicates so each event fires once, restrict page-specific tags to their pages through triggers or route-level loading. Removing a duplicate changes reported numbers; tell the user which metric will drop and why.
- **`async` and `defer`.** `async` scripts run in no guaranteed order; `defer` scripts run in document order after parsing; neither attribute affects inline scripts, and scripts inserted by other scripts are async by default. Many vendor snippets are an inline stub (a queue such as `window.dataLayer = window.dataLayer || []`, or a function that buffers calls until the loader arrives) followed by the loader. The stub must run before anything that pushes to it, and code that calls the vendor's real API directly (not through the queue) must run after the loader. Never reassign `window.dataLayer` after the tag manager loads; it overwrites queued values. Data pushed before the container snippet (page type, user state, consent defaults) must stay above it.
- **Consent ordering.** In Google consent mode, `gtag('consent', 'default', ...)` must run before any `config` or `event` command, on every page, or the defaults don't apply. Consent mode v2 adds `ad_user_data` and `ad_personalization` alongside `ad_storage` and `analytics_storage`; `wait_for_update` gives an async CMP time to send its update; region-specific defaults can differ by visitor location. In GTM, CMP and consent-default tags belong on the Consent Initialization trigger, which fires before all others. Moving, deferring or bundling the CMP or the consent defaults can break this order. Other consent frameworks (IAB TCF `__tcfapi`, CMP-specific APIs, script blocking via `type="text/plain"` that the CMP rewrites) have the same shape: the stub and the consent signal come first. Diff consent behaviour per state, not only performance.
- **A/B testing and personalisation.** Client-side testing tools that change the page usually load synchronously, or hide the page with an anti-flicker snippet until they apply variants. Deferring them causes a visible flash of the original content, and wrong experiment exposure data. Cheaper options that keep behaviour: lower the anti-flicker timeout only with approval, scope the tool to pages with live experiments, or (as a proposal) move testing server side or to the edge.
- **Facades for heavy embeds.** Replace a video player, map or chat widget with a static lookalike that loads the real embed on interaction (optionally preconnecting on hover or focus). This changes first-interaction UX: the first click loads rather than plays, autoplay may not work, chat proactive messages and unread badges won't show until load, video and chat analytics (impressions, view starts) change, and the facade must keep keyboard access, focus and an accessible name. Always flag a facade as a behaviour change.
- **Lazy-loading on interaction or visibility.** Load below-the-fold embeds with `loading="lazy"` on iframes or an `IntersectionObserver`, and load widgets on first interaction or idle. Traps: tags that must see the page view (load them late and you lose bounces), events that happen before the script loads (queue them, don't drop them), and scroll-depth or engagement tools that need to start early.
- **Resource hints and priority.** Add `preconnect` only for origins used early on most loads and allowed under the current consent state, keep it to a few origins, and remove hints for vendors that are gone. `fetchpriority="low"` on non-critical external vendor scripts can stop them competing with the LCP resource.
- **Self-hosting or proxying vendor scripts** (proposal): it saves a connection, but you stop getting vendor fixes and security updates, versioned or signed loaders can break, it may breach the vendor's terms, and the vendor may still fetch its own sub-resources. Proxying collection endpoints through the site's domain to dodge blockers is a privacy and consent decision, not a performance one.
- **Workers (Partytown and similar)** (proposal): moves scripts off the main thread, but DOM access becomes throttled and asynchronous, `event.preventDefault()` from the script has no effect, exit events (`pagehide`, `visibilitychange`, `beforeunload`) are unreliable so engagement or exit beacons can be lost, many vendors need a reverse proxy for CORS, and UI-heavy scripts (chat, consent banners, A/B testing) are poor fits. Only analytics-style scripts that mostly send data are candidates, and each needs its beacons verified.
- **CSP and SRI.** Any change of script origin, proxy or new inline stub needs the Content Security Policy updated, or the script silently fails to load (check the console and any CSP report endpoint). With `'strict-dynamic'`, scripts loaded by a trusted script inherit trust and host allowlists are ignored. Add SRI only to pinned, versioned files: a vendor loader that changes in place will fail its hash and stop loading, and a cross-origin `integrity` check needs the `crossorigin` attribute and CORS headers from the vendor.

## 5. Implement

Change one vendor or one mechanism at a time so each difference can be traced. Keep tag manager edits in an unpublished workspace or an exported container diff, and code edits behind the repo's normal review. Record for each change what you expect to stay the same (events, payload fields, cookies, timing relative to consent) so step 6 can check it.

## 6. Prove it

1. **Same baseline.** Compare against the current code and the current container version, under the same load matrix, throttling, browser build and test data. Don't compare against an older commit or a different container version.
2. **Network diff per cell.** For every page and consent state, diff the before and after request lists: added, removed and reordered requests, and changes in origins and cookies. No vendor may make a request or set a cookie in a consent state where it didn't before.
3. **Beacons and payloads.** For each kept vendor, capture its outgoing collection requests (`fetch`, XHR, `sendBeacon`, image pixels) in the browser and compare count, endpoint, event names and payload shape (parameter names and which are present, not values like timestamps or random IDs) for the page view and each key interaction, including events sent on page exit. Check consent signals carried in the payload too. Confirmation in the vendor's dashboard or debug view needs the user's access; list it as pending until they check.
4. **Feature smoke tests.** Every embed still works end to end: video plays (with sound and controls, and autoplay where it did before), map pans, chat opens and sends, payment and captcha complete in their test mode, reviews and social content render, the consent banner shows, records a choice and the choice persists.
5. **Web Vitals and main thread.** LCP, INP (scripted interactions), CLS and TBT, plus per-vendor main-thread time and long tasks, before and after, same conditions, medians with spread. A gain inside the run-to-run variance is not a gain.
6. **Screenshots.** Deterministic screenshots at fixed viewports and moments (first paint, after load, after consent choice) to catch flicker from A/B testing, layout shifts from late widgets, and facade appearance. A screen recording of the load shows anti-flicker and flash-of-original problems that a single screenshot misses.
7. **Classify every difference** as an intended saving, an approved behaviour change, a regression to fix, or noise, and re-run after fixes.

## Report back

- The vendor table: what each is, owner, where it loads, cost before and after, and the decision (kept, scoped, deferred, removed, or awaiting the user).
- Bytes, requests, origins, main-thread time and Web Vitals before and after, per template and consent state, with the run conditions.
- Beacon and payload comparison per kept vendor, with dashboard confirmation listed as pending until the user checks.
- Every behaviour, data or consent change, approved or awaiting approval, and every deviation from the request with its cost.
- CSP or SRI changes, and anything that needs a tag manager publish or dashboard change the user must do.
- Verification gaps: regions, consent states, devices or logged-in flows not tested, field data not yet available.

**Later (proposals, not part of the default change):** server-side tagging or a server-side CDP to move collection off the browser, server-side or edge A/B testing to remove anti-flicker, moving analytics-style tags to a worker, and a performance budget for third-party bytes and main-thread time checked in CI. Each changes data flow or ownership, so weigh it with the vendor owners before proposing it.
