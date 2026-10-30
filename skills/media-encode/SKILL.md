---
name: media-encode
description: "Re-encode a website's self-hosted video and audio so it is smaller and starts faster without visible or audible change, sized to how each file is actually displayed, with codec fallbacks, faststart, a hand-run committed ffmpeg script, and per-file VMAF/SSIM, frame-grab, colour-metadata and cross-browser playback proof. Use when asked to compress or optimise site video, shrink hero or background videos, make videos load or start faster, convert GIFs to video, add WebM/AV1/HEVC sources, fix slow video start or moov atom placement, cut audio file size, or reduce media bytes per page."
---

# Media encode

Make a site's video and audio smaller and faster to start while it looks, sounds and behaves the same. Measure first, find how each file is actually displayed, encode to that, and prove the result against the originals with objective metrics, frame grabs and real playback in each browser engine.

## Rules

- **Appearance, sound and behaviour are the contract.** Re-encoding is lossy, so "the same" means within a perceptual threshold the user approves before you encode (for example VMAF at the rendered size, minimum and mean, plus no visible difference in the frame grabs). Any visible quality drop, colour shift, audible change, or change to autoplay, looping, seeking, timing or layout is a regression unless the user approves it. Report what each change saves and what it changes.
- **Keep the playback behaviour.** Don't change `autoplay`, `muted`, `loop`, `playsinline`, `controls`, `preload`, `poster`, lazy-loading, `width`/`height` or the JavaScript that drives the element. Each one changes what the user sees or what the page downloads. Propose such changes separately.
- **Originals are sources.** Never overwrite them. Encode from the highest-quality original available, never from an already-compressed web copy if a master exists, and never re-encode an output.
- **No build step.** Write a script that is run by hand, commit it with its output, record the ffmpeg version and exact arguments, and say in the script header and the repo's docs when it needs rerunning.
- **Check the large-binary convention first.** Find how the repo stores media (Git LFS and `.gitattributes`, a CDN or object store, a `public/` folder, a media host) before committing outputs. If it is unclear, ask.
- **Read back every output.** ffmpeg accepts options it then ignores or overrides. Check every file with `ffprobe` and don't trust the command line you passed.
- **No external writes.** Don't upload to a CDN, object store or video host, deploy, or push without explicit approval.

## 1. Measure the current state

Record this before changing anything, and keep it for the final report.

- **Inventory** every media reference in the rendered site, not only the source tree: `<video>` and its `<source>` children, `<audio>`, `<track>`, `poster`, CSS or JS that sets `src` at runtime, background and hero loops, framework video components, and animated GIF, APNG or animated WebP that plays the role of video. Embedded hosted video (YouTube, Vimeo, Mux, Cloudflare Stream and similar `<iframe>` or player scripts) is out of scope for re-encoding; list it with its page weight and leave it alone.
- **Per file**, with `ffprobe -show_format -show_streams` (see `references/commands.md`): bytes, container, every stream and its codec, profile and level, `codec_tag_string`, `pix_fmt` and bit depth, width and height, `sample_aspect_ratio`, frame rate (`r_frame_rate` and `avg_frame_rate`; a mismatch means variable frame rate), frame count, duration, bitrate, keyframe interval, audio codec, channels, layout and sample rate, subtitle, data and chapter streams, colour metadata (`color_primaries`, `color_transfer`, `color_space`, `color_range`), HDR signs (`smpte2084` or `arib-std-b67` transfer, `bt2020` primaries, mastering display or Dolby Vision side data), rotation side data, and whether the `moov` atom comes before `mdat`.
- **Per page:** media bytes transferred in the first few seconds and after full playback (range requests return `206`, so sum them), which files load and when, the response `Content-Type`, whether the server answers `Range` with `206 Partial Content` and `Accept-Ranges: bytes`, cache headers, and time to first frame on a throttled connection.
- **Audio loudness:** integrated loudness and true peak per audio stream with the `ebur128` filter.

## 2. Find how each file is actually displayed

Collect this from the rendered site in a real browser, across the breakpoints the site uses, at DPR 1, 2 and 3.

- **Rendered size:** the element's box per breakpoint, times DPR, gives the most pixels anyone sees. With `object-fit: cover` the visible part is cropped, but the crop moves with the viewport, so size to the largest box and never crop the file. Don't encode above the source resolution.
- **Attributes and state:** `autoplay`, `muted`, `loop`, `playsinline`, `controls`, `preload`, `poster`, `loading` or an IntersectionObserver lazy loader, `crossorigin`, `prefers-reduced-motion` handling, and any media fragment (`#t=`) in the URL.
- **Script use:** anything reading `currentTime`, `duration`, `timeupdate`, `requestVideoFrameCallback`, `seeked` or `ended`; scroll-scrubbed video; `canvas.drawImage` or WebGL textures from the video; captions toggled in JS. Scrubbing and timed effects depend on keyframe density, frame count and exact duration.
- **Sound:** whether audio can ever be heard. A muted autoplay loop with no controls and no script that sets `muted = false` never plays sound. If there are controls, or anything unmutes it, the audio stays.

## 3. Choose formats and the fallback order

- **H.264 in MP4 is the baseline** that every browser plays: High profile, 8-bit `yuv420p`, AAC-LC audio. Add more efficient formats only where they save enough to justify another file.
- **AV1** (MP4 or WebM) is the strongest saver. Chrome and Firefox decode it in software. Safari 17+ decodes it only on devices with a hardware decoder (iPhone 15 Pro, M3 Macs and later), so it needs a fallback.
- **VP9 in WebM** plays in Chrome and Firefox, and in recent Safari (check caniuse for the versions the site targets, and note Safari ignores WebM alpha). Treat it as the alpha-video format for Chrome and Firefox, or as a middle option when AV1 encoding is too slow.
- **HEVC** plays in Safari everywhere and elsewhere mostly where the hardware decodes it. Tag it `hvc1` in MP4 (`-tag:v hvc1`) for Safari. Use it mainly as Safari's alpha format.
- **Order `<source>` best first, H.264 last.** The browser takes the first source it thinks it can play, so give every source a `type` with a `codecs` parameter built from the encoded file (`video/mp4; codecs="av01.0.08M.08, mp4a.40.2"`, `video/mp4; codecs="avc1.640028, mp4a.40.2"`). Without `codecs`, Safari on a device with no AV1 decoder may pick an AV1 MP4 and fail. Derive profile and level from `ffprobe` (`avc1.PPCCLL` is hex profile, constraint flags and level x 10); don't copy strings from examples.
- **Alpha video** needs its own path: VP9 with `yuva420p` in WebM for Chrome and Firefox, and HEVC with alpha (`hevc_videotoolbox` with `-alpha_quality`, macOS only) for Safari, which ignores WebM alpha. H.264 and `yuv420p` have no alpha channel. Prove transparency over a contrasting background in each engine.
- **GIF to video** cuts bytes hugely but changes the element: `<img>` becomes `<video autoplay muted loop playsinline>`, so alt text, right-click save, reduced-motion handling, sizing CSS and lazy-loading all change. Treat it as a proposal unless the user asked for it, and keep the first-frame and loop timing identical (GIF frame delays are not a constant frame rate).
- **Audio formats:** AAC-LC in MP4 or M4A plays everywhere. Opus is more efficient and plays in WebM in every engine, but Safari's support for Opus in other containers depends on the OS version, so keep an AAC fallback. Don't transcode lossy to lossy unless the bitrate saving is large and the listening test passes.

## 4. Encode

Encode at the rendered size and the original frame rate, targeting quality, not a bitrate guess.

- **Quality targeting:** use CRF (x264 `-crf`, x265 `-crf`, SVT-AV1 `-crf`, VP9 `-crf` with `-b:v 0`) and search for the highest CRF that still meets the approved metric on the worst segment, not the average. Use two-pass (`-pass 1`/`-pass 2`) only when a byte budget is fixed. Short loops and dark, grainy or gradient-heavy footage need lower CRF than talking heads.
- **Scaling:** scale once, with an explicit filter (`scale=-2:H:flags=lanczos`), so both dimensions stay even. `yuv420p` needs even dimensions, and ffmpeg can silently drop an odd row or column when it isn't told how (seen in 9.0.1). Keep the display aspect ratio, check `sample_aspect_ratio` (non-square pixels must be resolved to square or kept tagged), and keep the `width`/`height` attributes' ratio, or the layout shifts.
- **Frame rate:** keep it. Dropping 60 to 30 or converting variable to constant frame rate changes motion and can shift frames against audio or script timing. If a change is worth proposing, show it as a separate option with its own diff. Check the frame count and duration match the original.
- **Keyframes:** a keyframe interval of about 2 seconds (`-g`, `-keyint_min`, `-sc_threshold 0` for x264) keeps seeking responsive. Scroll-scrubbed video needs far denser keyframes (often every frame or every few frames); match or beat the original's density. A loop must start on a keyframe, and the last frame must lead back to the first as it did before.
- **Colour:** keep the source's colour description and set it explicitly. Tag primaries, transfer and matrix with the `setparams` filter or the encoder's own params (`-x264-params colorprim=...:transfer=...:colormatrix=...`); the generic output options `-color_primaries` and `-color_trc` can be silently not written for untagged input (seen in ffmpeg 9.0.1 with libx264). Untagged output is guessed differently per browser, and a BT.601 versus BT.709 mismatch shifts hues. Keep limited ("tv") range unless the source is full range, and convert range explicitly (`scale=in_range=...:out_range=...`) instead of letting it pass through mislabelled. Output `yuv420p` 8-bit for H.264 compatibility; 4:2:2, 4:4:4 and 10-bit H.264 do not play reliably in browsers.
- **HDR:** don't tone-map to SDR by default. It changes how the video looks, and good tone mapping needs `zscale` or `libplacebo`, which many ffmpeg builds lack (`ffmpeg -filters`). Either keep HDR (10-bit AV1, VP9 profile 2 or HEVC Main 10, with transfer, primaries and mastering metadata carried over) with an SDR H.264 fallback the user approves, or ask.
- **Rotation:** phone footage often carries rotation side data. ffmpeg auto-rotates on decode by default, so the output is upright with swapped dimensions and no side data. Confirm the displayed orientation and size match the original in every browser.
- **Container:** MP4 gets `-movflags +faststart` so the `moov` atom is at the front and playback starts before the whole file arrives. WebM gets `-cues_to_front 1` for the same reason. Map streams explicitly (`-map 0:v:0 -map 0:a:0?`); ffmpeg's default selection keeps one stream per type and silently drops extra audio, subtitle and data streams. Strip encoder metadata with `-map_metadata -1` and `-fflags +bitexact` only after checking nothing needs it (chapters need `-map_chapters 0`).
- **Audio:** drop the audio track (`-an`) only from video that can never be heard (step 2); it also removes the browser's "has audio" state, which some autoplay policies and UIs read. Otherwise keep channel count and layout (stereo to mono collapses the image and can cancel out-of-phase sound), keep the sample rate unless resampling is required (Opus is always 48 kHz internally), and never normalise or change loudness (`loudnorm`, `volume`) without approval. AAC adds priming samples, so check that audio still lines up with video and that seamless audio loops still loop without a gap.
- **Captions and chapters:** external `<track>` WebVTT files are untouched by re-encoding, but check their cue timings still match if duration or frame rate changed at all. Embedded subtitle or chapter streams survive only if mapped.
- **Posters:** the poster shows before the first frame and whenever autoplay is refused. It must match the first frame of the new encode (same crop, colour and frame) or the switch flashes. Extract it from the encode at time 0, size it to the rendered box like any image, and count its bytes, since it may be the page's largest image.

## 5. Write the script

Commit one script (for example `scripts/media/encode.sh`) and a committed config listing, per source: output names, target heights, codecs, CRF per codec, keyframe interval, colour tags, audio handling and poster time. It should:

- check `ffmpeg -version` against the recorded version and fail if a required encoder or filter (`libx264`, `libsvtav1`, `libvpx-vp9`, `libvmaf`) is missing;
- encode from the originals with exact arguments, writing to a temporary name and moving into place only on success;
- re-probe every output and fail loudly if dimensions, frame rate, frame count, duration (within one frame), colour tags, pixel format, stream set, rotation or `moov` position don't match the config;
- print bytes per output and the `codecs` string to paste into `type`.

x264 and SVT-AV1 gave byte-identical output across two runs on the same machine and build; a different build, CPU or thread count may not. Record output checksums, and when a rerun differs, accept it only if the metrics still pass.

Put the rerun triggers in the header and the repo docs: a new or replaced source file, a change to the rendered size or breakpoints, a new codec or browser target, an ffmpeg or encoder upgrade, or a change to the approved threshold.

## 6. Prove it

1. **Metrics at the rendered size.** For each output, scale both the original and the encode to the largest rendered size with the same scaler, align timestamps, and run `libvmaf` (encode first, original second) and `ssim`. Report mean and minimum VMAF and the lowest-scoring timestamps; a good mean can hide a bad scene. Use `vmaf_4k_v0.6.1` for 4K renders. Give both inputs identical colour tags first (the same `setparams` on each, set to what the source really is): when tags differ, ffmpeg auto-inserts a range or matrix conversion on one input and the score is meaningless (an untagged original against a BT.709-tagged encode scored VMAF 54 instead of 97 in testing). A score far below expectations usually means misaligned frames or mismatched tags, not a bad encode.
2. **Frame grabs** at fixed timestamps from original and encode, side by side at the rendered size: the first frame, the last frame, the darkest scene (`blackdetect` or the VMAF low points), smooth gradients and skies (banding), fine texture and grain, fast motion, and text or logos. Look at them, then name any visible difference.
3. **Metadata check:** `ffprobe` every output and diff against the original for colour tags, range, pixel format, dimensions, aspect ratio, frame rate, frame count, duration, rotation, streams and chapters. Check `ebur128` loudness matches within 0.5 LU where audio was re-encoded, and listen to the start, the end and the loop point.
4. **Playback in Safari, Chrome and Firefox** (WebKit, Chromium and Gecko; iOS Safari on a device where the site targets phones): which `<source>` each browser chose (`currentSrc`), autoplay starts muted, inline on iOS, the loop seam compared with the original, seeking to several points, scrubbing if the page scrubs, captions, alpha over the real background, and colour next to the original in the same browser. Test Safari on hardware without an AV1 decoder if AV1 is offered.
5. **Server:** `curl -r 0-99` against each media URL returns `206` and 100 bytes, with the right `Content-Type`. Safari needs range support to play at all.
6. **Re-measure** bytes per page (first few seconds and full playback) and time to first frame (`loadeddata` or the first `requestVideoFrameCallback`, from navigation start) on the same throttled profile as the baseline.

Classify every difference as one of: a regression to fix, a below-threshold encoding difference (name where it shows), or an approved change. Fix regressions and prove again.

## Report back

- Bytes before and after, per file and per page (first seconds and full playback), with posters, and time to first frame before and after.
- Per file: chosen formats and `<source>` order, rendered size it was encoded for, CRF, and VMAF mean and minimum and SSIM at that size.
- Frame-grab and playback results per browser, with each difference classified.
- Every deviation from the request, and what it cost (bytes kept, or a change awaiting approval), including dropped audio tracks and anything left as a proposal.
- How to rerun the script, when it needs rerunning, and where the outputs are stored.
- Verification gaps, such as browsers or devices not tested, or HDR displays not checked.

**Later (proposals, not part of the default change):** HLS or DASH adaptive streaming for long-form video (segmenting, a player for non-Safari browsers, and hosting all change the setup); `<source media>` breakpoint-specific files for heroes whose rendered size differs a lot between phone and desktop (check current browser support first); lazy-loading or `preload="none"` for below-the-fold video; frame-rate reduction for slow ambient loops; and GIF-to-video conversion where it wasn't requested. Each changes behaviour or infrastructure, so weigh it and ask.
