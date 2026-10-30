---
name: image-optimize
description: "Make a website's images smaller and faster to load without any visible or behavioural change: inventory every image the rendered site uses, size it to what each breakpoint and DPR actually displays, re-encode lossless first and lossy only within an approved perceptual threshold, and prove it with pixel diffs, perceptual metrics, and before/after LCP, CLS and bytes. Use when asked to optimise or compress images, shrink image bytes, convert to WebP or AVIF, add srcset/sizes or <picture>, fix oversized or responsive images, speed up LCP caused by an image, fix image layout shift, or audit OG/social images and favicons."
---

# Image optimise

Make a site's images smaller and faster while every page looks and behaves exactly the same. Measure first, find every image the rendered site uses and the size it is actually drawn at, cut each one to that, and prove the result against a baseline from the same content.

## Rules

- **Appearance is the contract.** Any visible change (colour, sharpness, crop, orientation, alpha edges, animation timing) is a regression unless the user approves it. If an optimisation would change output, report what it saves and what it changes, and keep the current result by default.
- **Loading behaviour is part of the contract.** Changing `loading`, `fetchpriority`, `decoding`, preloads, placeholders or the image's box changes when and how it appears. Preserve them, or propose the change with its measured effect and get approval.
- **Lossless first.** Lossy re-encoding is only allowed within a perceptual threshold the user has approved. A threshold you picked yourself is not approval.
- **Originals stay.** Keep the source images in the repo (or the existing asset store) as the input to the script. Never overwrite a source with its optimised output, and never re-encode an already lossy output as the next run's input.
- **No build step.** Write a script that is run by hand, commit its output, and say in the script header and the repo's docs when it needs rerunning. Don't add image processing to the build unless the user asks.
- **Don't double-process.** If a framework image component or image CDN already transforms images, measure what it actually serves before touching anything. Pre-compressing its input usually just stacks a second lossy generation on top.
- **Check the generated files.** Encoders have quirks and silent defaults (see step 4). Read back every output's dimensions, format, bit depth, colour profile, orientation, alpha and frame count rather than trusting the command line you passed.
- **No external writes.** No deploys, pushes, CDN config changes or cache purges without the user's explicit approval.

## 1. Measure the current state

Record this before changing anything, and keep it for the final report.

- **Per page**, in a real browser (Playwright or Chrome DevTools) at the site's real breakpoints: image bytes transferred (CDP `encodedDataLength` or Playwright response sizes), request count, format served (the response `Content-Type`, not the file extension), and LCP element, LCP time and CLS. Use a fixed network and CPU profile so before and after are comparable, and run several times; LCP is noisy.
- **Per image file:** bytes, format, pixel dimensions, bit depth, colour profile (embedded ICC name, or untagged), EXIF orientation, alpha, frame count and loop count for animations, and chroma subsampling for JPEGs. `magick identify -verbose`, `avifdec --info` and `webpmux -info` cover it.
- **Delivery:** `Cache-Control` and whether URLs are content-hashed, `Vary: Accept` or format negotiation, and whether an image CDN or framework loader rewrites URLs. A re-encoded file behind an unhashed URL with a long cache lifetime won't reach returning visitors.

## 2. Find every image the site uses

Collect this from the rendered site, not just the source tree. Unused files in `public/` cost nothing; images assembled at runtime cost a lot. Walk every built page (or a representative set covering every template and component) and record:

- `<img>` and `<picture>`: `currentSrc` (the candidate the browser actually chose), `srcset`, `sizes`, every `<source>` with its `type` and `media`, `loading`, `fetchpriority`, `decoding`, `width`/`height`, and the CSS `object-fit`, `object-position` and `aspect-ratio`.
- CSS `background-image`, `image-set()`, `mask-image`, `border-image`, `list-style-image` and `content: url()`, including `:hover`, `:focus` and other state rules and media-query variants that only apply at some breakpoints or in dark mode. Read the computed style on the element, then trigger the states.
- Images injected later: carousels, lightboxes, tabs, menus, lazy sections below the fold, client-rendered lists, and anything behind a click. Scroll the full page and open each interactive surface.
- Head and metadata images: `og:image`, `twitter:image`, favicons (`.ico` and PNG sizes), `apple-touch-icon`, `mask-icon`, web app manifest icons (including `maskable`), and `msapplication` tiles.
- Email templates, RSS/feed enclosures and CMS content, if the repo owns them.
- **SVG** is a different job (markup minification, not raster encoding). Inline SVG is out of scope (use the `svg-icon-optimize` skill); note any large SVGs and any SVGs that embed base64 rasters, and leave them unless the user widens the scope. Never rasterise an SVG to save bytes without approval.

For each raster image, record the **rendered size**: the element's CSS box (`getBoundingClientRect`, minus any `object-fit` cropping) at each breakpoint, times each DPR you test (1, 2, and 3 for phones). That device-pixel width is the largest file the image needs. The intrinsic size of `currentSrc` tells you what is downloaded today; the gap between the two is usually the biggest saving.

## 3. Decide the target per image

- **Sizes.** Build `srcset` width candidates from the measured device-pixel widths, not a generic ladder, and never upscale beyond the source. Derive `sizes` from the measured CSS widths per breakpoint (`(min-width: 64rem) 40rem, 100vw` and so on); a missing `sizes` means `100vw`, which over-downloads for anything narrower than the viewport. Use `x` descriptors for fixed-size images. Don't mix `w` and `x` descriptors in one `srcset`. `sizes="auto"` is only valid with `loading="lazy"` and needs a fallback list after it.
- **Art direction.** If different breakpoints show different crops (`<source media>` or different `object-position`), the crops are part of the design. Keep each one; only resize within it.
- **Formats.** In `<picture>`, the browser takes the first `<source>` whose `type` it supports and whose `media` matches, so order is AVIF, then WebP, then the original format in `<img src>` as the fallback. Check AVIF and WebP support against the site's real audience before dropping the fallback. Keep the `<img>` element carrying `alt`, `width`, `height`, `loading` and classes exactly as before; `<source>` only swaps the URL.
- **Per-content choice.** Photos: AVIF or WebP lossy, or a better-encoded JPEG. Screenshots, UI, line art, text and flat colour: lossless WebP, lossless AVIF or an optimised PNG, often smaller and always exact; lossy on these smears edges and coloured text. Pick per image by comparing actual bytes at the approved threshold, not by format reputation.
- **Lossless.** Try lossless first for every image: PNG recompression, `jpegtran -optimize` (with `-copy` set deliberately, see step 4), lossless WebP (`cwebp -lossless`), lossless AVIF (`avifenc -l`). If the lossless output decodes to exactly the source pixels and is smaller, it needs no approval.
- **Lossy threshold.** Pixel diff at threshold 0 is the wrong test for a lossy re-encode; every pixel changes. Use a perceptual metric computed at the **rendered size and DPR**, not at source resolution: SSIMULACRA 2 (`ssimulacra2`), butteraugli (`butteraugli_main`) or DSSIM. Propose a threshold with examples (a crop of the worst-scoring image at 1:1), get the user to approve it, then encode each image at the smallest setting that stays inside it. Quality numbers are not comparable across encoders.
- **Social images** (`og:image`, `twitter:image`): crawlers have their own format support and size limits. Keep them in JPEG or PNG unless the platform's current docs say otherwise, keep the dimensions and aspect ratio (Facebook currently recommends at least 1200×630 near 1.91:1, with an 8 MB cap; check the other platforms' current docs), and keep `og:image:width`/`og:image:height` correct. Platforms cache by URL, so a changed image needs a new URL to show up.
- **Icons:** keep every declared size and the exact pixels at small sizes; a favicon is drawn at 16 or 32 px, where resampling artefacts show. Recompress losslessly only.

## 4. Edge cases that change output

- **Colour profiles.** Stripping an ICC profile makes the browser assume sRGB, so a Display P3 or Adobe RGB image visibly shifts. Converting P3 to sRGB clips wide-gamut colours. Keep the profile, or convert deliberately and diff on a wide-gamut display. Defaults differ: `cwebp` drops ICC unless given `-metadata icc`; `avifenc` keeps it; `jpegtran` keeps only comments by default (`-copy all` keeps ICC and EXIF, `-copy icc` keeps ICC only); sharp converts to sRGB and strips all metadata unless told to keep it (`keepIccProfile`, `withMetadata`, `keepMetadata`).
- **EXIF orientation.** Many cameras store pixels unrotated with an Orientation tag. Stripping EXIF without applying the rotation turns the image sideways. `cwebp` (default `-metadata none`), `jpegtran` (default `-copy comments`), `magick -strip` and sharp (unless `autoOrient()` is called) all drop the tag without rotating; `avifenc` converts it to an `irot`/`imir` transform. Bake the rotation into the pixels first (`magick -auto-orient`, sharp `autoOrient()`), then strip, then check the output's dimensions are swapped where expected.
- **Bit depth.** A 16-bit source PNG won't round-trip through 8-bit WebP or AVIF, so "lossless" isn't pixel-identical to the source. Compare against the 8-bit image the browser actually draws, and note it.
- **Transparency.** Keep alpha where any pixel is not fully opaque, and check edges against every background the image sits on (including dark mode). Premultiplication, alpha quantisation (`cwebp -alpha_q`, `avifenc --qalpha`) and palette reduction (`pngquant`, which is lossy and falls under the approved threshold) can create halos or banding in soft shadows. `cwebp` rewrites RGB under fully transparent pixels unless `-exact` is passed; harmless on screen but it shows up in an RGBA pixel diff, and it matters if CSS ever uses the image as a mask or blends it unusually.
- **Animation.** `cwebp` reads only the first frame of an animated PNG or WebP; use `gif2webp` (lossless by default) or `img2webp` for animated WebP. Preserve frame count, per-frame durations, loop count and disposal. Compare frame by frame and total duration. Next.js serves animated sources in their original format. Converting a GIF to `<video>` changes behaviour (autoplay rules, controls, reduced-motion handling) and needs approval.
- **Chroma subsampling.** Lossy WebP is always 4:2:0; `-sharp_yuv` reduces, but doesn't remove, the colour bleed on thin coloured lines and text. AVIF can be 4:4:4 (`avifenc -y 444`; `avifenc` and sharp default to 4:4:4 for PNG input, JPEG input keeps its own subsampling under `avifenc -y auto`). sharp's JPEG output defaults to 4:2:0. Use 4:4:4 or lossless for screenshots, UI and anything with small coloured text.
- **Progressive rendering.** A progressive JPEG refines from a coarse preview while loading; a baseline one paints top to bottom. Keep whichever the source was (`jpegtran -progressive` writes progressive). AVIF doesn't refine progressively unless encoded that way (`avifenc --progressive`), and WebP doesn't either. If the current image visibly builds up during a slow LCP, say that the new format changes the loading experience.
- **Placeholders.** Blur-up placeholders, `background-color` boxes and dominant-colour fills are generated from the image; regenerate them from the same source so the swap stays seamless.

## 5. Loading behaviour and layout

- **LCP image:** must not be lazy-loaded. Record its current `loading`, `fetchpriority` and any preload (`<link rel=preload as=image>` with `imagesrcset`/`imagesizes` that exactly match the element, or the browser fetches twice). Next.js `next/image` lazy-loads by default and, from Next.js 16, uses `preload` instead of the deprecated `priority`. If the LCP image is lazy or late today, fixing it is a behaviour change worth proposing with measured LCP, not something to slip in.
- **Layout shift:** every image needs a reserved box (`width` and `height` attributes, or CSS `aspect-ratio`). If the new file's aspect ratio differs by even a pixel from the old one, the box or the crop changes. Keep exact aspect ratios when resizing; round heights from widths consistently.
- **Frameworks and CDNs.** `next/image`, Nuxt Image, Astro `<Image>`, Gatsby, Cloudinary, Imgix, Vercel or Netlify image transforms and Cloudflare Images already resize and pick formats. Measure what they serve (format, dimensions, bytes, quality) before changing anything. The fix there is usually configuration (correct `sizes`, quality, formats list; in Next.js the `qualities` allowlist and `formats`, which defaults to WebP only), not pre-processing, and changing CDN or framework config needs the user's approval. Don't feed them pre-compressed lossy sources.

## 6. Write the script

Commit one script (for example `scripts/images/optimise.mjs` or `.py`) with every tool version pinned (`package.json` with exact versions for sharp, or recorded `cwebp -version` and `avifenc --version` checked at start). It should:

- read the source images and a committed config: per image, target formats, widths, encoder settings (lossless or the approved lossy setting), colour handling, and metadata to keep;
- write outputs with content-hashed or versioned names, deterministically, so rerunning on the same input gives byte-identical files (fix thread counts where an encoder's output depends on them, and check by running twice);
- re-read each output and print its dimensions, format, bit depth, ICC profile, orientation, alpha, frame count and bytes, failing loudly if one doesn't match the config, if it is larger than its source (keep the source then), or if a lossy output falls outside the approved threshold at rendered size;
- emit the `srcset`/`sizes` data or markup the templates consume, rather than hand-copied strings.

Put the rerun triggers in the header and the repo docs: a new or replaced source image, a layout change that alters an image's rendered width at any breakpoint, a new breakpoint, an approved threshold change, or an encoder upgrade.

## 7. Prove it

1. **Baseline from the same content.** Render the current content with the old images and markup, for example by swapping only image files and image markup in a temporary copy. A diff against an older commit mixes content changes in with image changes.
2. **Deterministic screenshots** in Playwright: each real breakpoint at DPR 1, 2 and 3, animations paused, wait until every `<img>` reports `complete` with `naturalWidth > 0` and background images have loaded, scroll the whole page so lazy images load, then take full-page shots. Same browser build before and after. Use a wide-gamut colour profile for one pass if any image carries a P3 profile.
3. **Pages:** every template, each component that shows images, art-directed images at each breakpoint, transparent images on every background they appear on (including dark mode), animations, and pages whose LCP is an image.
4. **Diff.** Where outputs should be identical (lossless recompression, format swaps that decode to the same pixels), pixelmatch or `magick compare -metric AE` at threshold 0. Where encoding is lossy, the perceptual metric per image at rendered size, against the approved threshold, plus a side-by-side review of the worst-scoring crops. Normalise inputs to plain 8-bit sRGB PNG before metrics (some decoders' PNGs don't load in `ssimulacra2`), and run an identical pair through each metric first to learn its scale and direction; ImageMagick 7's `SSIM` and `DSSIM` metrics both report a dissimilarity (0 for identical). Classify every difference as a content change, a regression to fix, or harmless rounding (resampling noise of a level or two, no shift in edges, colour or crop). Fix regressions and diff again.
5. **Re-measure** image bytes and requests per page and per breakpoint, which candidate `currentSrc` picks at each DPR, format served, LCP and CLS with the same throttling, and that no image is fetched twice (preload mismatch, or `<picture>` plus a CSS background of the same asset).

## Report back

- Bytes before and after, per image and per page at each breakpoint and DPR, with LCP and CLS before and after.
- Diff results: pixel diffs for lossless cases, perceptual scores against the approved threshold for lossy ones, each difference classified.
- Every deviation from the request and what it cost (bytes kept because of a profile, alpha, animation or social-platform constraint), and every behaviour change awaiting approval (loading attributes, preloads, GIF to video, CDN config).
- How to rerun the script, and when it needs rerunning.
- Verification gaps: browsers or devices not checked, wide-gamut displays not checked, social previews not refreshed through each platform's debugger.

**Later (proposals, not part of the default change):** move image transforms to the framework's or CDN's on-demand pipeline, add JPEG XL as a first `<source>` where the audience supports it, give the LCP image `fetchpriority="high"` or a matching preload, replace large animated GIFs with muted looping video, and add `decoding` or `content-visibility` hints. Measure each against its behaviour change before proposing it.
