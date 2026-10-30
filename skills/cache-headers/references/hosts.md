# Host and CDN caching behaviour

Last checked against vendor docs in October 2026. Defaults change; re-read the linked page before relying on any line, and confirm with real responses.

## Vercel

Docs: `vercel.com/docs/caching/cache-control-headers`, `vercel.com/docs/caching/cdn-cache`, `vercel.com/docs/skew-protection`.

- Default `Cache-Control: public, max-age=0, must-revalidate` (neither CDN nor browser caches).
- Function responses are CDN-cached only with `s-maxage` (optionally with `stale-while-revalidate`, `stale-if-error`) or a CDN-targeted header. Function headers override `vercel.json` / `next.config` headers for the same route.
- Precedence: `Vercel-CDN-Cache-Control` > `CDN-Cache-Control` > `Cache-Control`. `Vercel-CDN-Cache-Control` is consumed and not forwarded; `CDN-Cache-Control` and `Cache-Control` are forwarded.
- With only `Cache-Control`, Vercel strips `s-maxage` (and `stale-while-revalidate`) before the browser sees it. `stale-while-revalidate` and `stale-if-error` are consumed by the proxy. `proxy-revalidate` is not supported.
- Not cached: requests with `Authorization` or `Range`; responses with `Set-Cookie`, `private`, `no-cache`, `no-store`, `Vary: *`, or `Vary: Cookie`; statuses other than 200, 404, 410, 301, 302, 307, 308; bodies over the size limit.
- `Accept` and `Accept-Encoding` are in the cache key by default.
- Static files are cached for the deployment's lifetime; static files cannot bypass the cache. Hashed framework assets get `public, max-age=31536000, immutable`.
- `Pragma: no-cache` on the request (DevTools open) forces synchronous revalidation, so measure with DevTools cache settings in mind.
- Status header: `x-vercel-cache` (`HIT`, `MISS`, `STALE`, `REVALIDATED`, and others).
- Skew Protection (Pro/Enterprise; on by default for supported frameworks in newer projects) pins framework asset and data requests to the deployment that served the page, for a configurable maximum age. Full document navigations go to the latest deployment.

## Netlify

Docs: `docs.netlify.com/platform/caching/`.

- Browser default `Cache-Control: public, max-age=0, must-revalidate`; the CDN separately caches static assets for up to a year and invalidates on every deploy (per deploy context). Hashed assets therefore need an explicit long browser `max-age` in `_headers` or `netlify.toml` to avoid a 304 per asset per visit.
- Precedence: `Netlify-CDN-Cache-Control` > `CDN-Cache-Control` > `Cache-Control`; Netlify forwards `CDN-Cache-Control` and `Cache-Control`.
- `Netlify-Vary` keys on chosen query params, headers, language, country or specific cookie names. Standard `Vary` is honoured.
- Functions are not cached without cache headers; `durable` in `Netlify-CDN-Cache-Control` shares function responses across edge nodes.
- `private` keeps responses out of the shared cache. Basic auth on any page disables caching for the whole site.
- `Netlify-Cache-ID` opts an object out of deploy invalidation; do not use it for anything sensitive.
- Status header: `Cache-Status` (for example `"Netlify Edge"; hit`).

## Cloudflare (proxying an origin)

Docs: `developers.cloudflare.com/cache/concepts/default-cache-behavior/`, `.../cache-control/`, `.../cdn-cache-control/`, `.../how-to/edge-browser-cache-ttl/`.

- Caches by file extension, not MIME type. HTML and JSON are not cached by default; `robots.txt` is.
- Respects origin headers unless a cache rule overrides them. Does not cache `private`, `no-store`, `no-cache` or `max-age=0` by default. Without headers, caches 200/206/301 for 120 minutes, 302/303 for 20 minutes, 404/410 for 3 minutes.
- `Set-Cookie`: with origin cache control on (always on for Free, Pro, Business), responses with `Set-Cookie` are not cached. With it off, content may be cached with `Set-Cookie` stripped.
- `Vary`: ignored by default except `Accept-Encoding`, a configured cache-rule Vary, or Vary for images.
- `CDN-Cache-Control` and `Cloudflare-CDN-Cache-Control` are supported; if both are present only the latter is used, and it is not forwarded. When either is present, `Cache-Control` passes through unchanged and does not drive Cloudflare's cache.
- `stale-while-revalidate` is asynchronous (status `UPDATING`). Cloudflare's docs say not to combine it with `s-maxage`.
- `immutable` does not change Cloudflare's own caching; it is passed to clients with origin cache control on.
- Browser Cache TTL (default 4 hours) raises an origin `max-age` shorter than the setting and adds one where the origin sends none, unless set to respect existing headers. Check it does not lengthen HTML or unhashed assets.
- "Cache everything" style rules cache HTML regardless of extension; check them for personalised pages.
- Status header: `cf-cache-status` (`HIT`, `MISS`, `EXPIRED`, `UPDATING`, `BYPASS`, `DYNAMIC`, and others).

## nginx

Docs: `nginx.org/en/docs/http/ngx_http_headers_module.html`, `.../ngx_http_proxy_module.html`.

- `add_header` and `expires` apply only to 200, 201, 204, 206, 301, 302, 303, 304, 307, 308 unless `always` is given. A long-lived header without `always` therefore does not reach 404s.
- `add_header` at one level replaces, not merges, all `add_header` inherited from parent levels. Adding one header in a `location` silently drops the server-level ones. nginx 1.29.3+ has `add_header_inherit merge`.
- `proxy_cache` does not cache responses with `Set-Cookie` or `Vary: *`; other `Vary` values are keyed. `X-Accel-Expires` takes priority over `Cache-Control`. `proxy_ignore_headers` can disable these protections; treat it as a red flag.
- `stale-while-revalidate` needs `proxy_cache_use_stale updating` and `proxy_cache_background_update on`.
- `try_files $uri /index.html` returns the SPA shell with 200 for missing assets; pair it with a separate asset `location` that returns 404.

## S3 and CloudFront

Docs: `docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html`.

- S3 sends whatever `Cache-Control` is stored in each object's metadata, set at upload. A sync that does not set it per path leaves objects with no header, so browsers use heuristics. Upload hashed and unhashed paths with different metadata, and remember that changing metadata on existing objects means rewriting them.
- CloudFront without a cache policy defaults to a 24-hour TTL when the origin sends no `max-age`. With a cache policy, the edge TTL is clamped between the minimum and maximum TTL; `s-maxage` drives the edge and `max-age` the browser.
- With minimum TTL above 0, CloudFront caches `private`, `no-store` and `no-cache` responses for the minimum TTL. Keep minimum TTL at 0 on any behaviour that can serve personalised content.
- Supports `stale-while-revalidate` and `stale-if-error`, capped by the maximum TTL.
- Viewer `Cache-Control` / `Pragma` request headers cannot force an origin fetch.
- Check which query strings, headers and cookies the cache policy puts in the key, especially for `?v=` cache busting and for anything that varies by cookie.
- Status header: `x-cache` (`Hit from cloudfront`, `Miss from cloudfront`, `RefreshHit from cloudfront`).

## Browsers

- `immutable`: avoids revalidation on reload where implemented. Chrome does not implement it but does not revalidate subresources on reload (MDN caching guide).
- `stale-while-revalidate`: Chrome 75+, Firefox 68+, Safari 14+ (caniuse).
- Reload sends `Cache-Control: max-age=0` with validators, so reload measurements show revalidation cost, not repeat-navigation cost.
- bfcache and `no-store`: historically excluded; Chrome admits `no-store` pages in limited conditions (`developer.chrome.com/docs/web-platform/bfcache-ccns`). Check other browsers at the time of the change.
- Service worker `updateViaCache` defaults to `'imports'`: the main script is fetched from the network on update checks, imported scripts use the HTTP cache.
- `Clear-Site-Data: "cache"` clears the browser cache for the origin only, never intermediate caches.
