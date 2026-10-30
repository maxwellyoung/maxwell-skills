---
name: app-bundle-size
description: "Shrink an iOS or Android app's download and install size without changing behaviour or appearance, for native iOS, native Android, React Native and Expo. Measure per-device thinned sizes, break down what's in the binary, remove what isn't used, turn on safe shrinking, and prove it with same-commit size reports, screenshot diffs and smoke flows on release builds. Use when asked to reduce app size, shrink the APK, AAB or IPA, cut download size, make the app smaller, trim the bundle, enable R8 or ProGuard, shrink resources, strip symbols, remove unused assets, or investigate why the app got bigger."
---

# App bundle size

Make an app smaller to download and install while it behaves and looks exactly the same. Measure what users actually download on each device class, find what the app really uses, cut to that, and prove the result against a baseline release build from the same commit.

## Rules

- **Behaviour and appearance are the contract.** Any visible change, crash, missing asset, missing string or changed behaviour is a regression unless the user approves it. If a saving needs a change (lossy image compression, dropping a language, dropping an old OS version), don't ship it by default. Report what it saves and what it changes, and let the user decide.
- **Measure the right number.** A universal APK or IPA is not what anyone downloads. Report per-device download and install sizes, and say which one each figure is.
- **Release builds only.** Debug builds include dev tooling, unminified code and unstripped symbols. Every number and every proof comes from a release configuration.
- **Follow the repo.** Use its package manager and lockfile, Gradle and Xcode setup, build scripts, CI and release process. Don't swap tools, upgrade the toolchain incidentally or regenerate a lockfile. On an Expo project with Continuous Native Generation (`android/` and `ios/` not committed), native changes go through `app.json`/`app.config.*` and config plugins, or they vanish on the next `npx expo prebuild`.
- **Crash reporting keeps working.** Stripping symbols from the shipped binary is fine; losing them is not. dSYMs, R8 mapping files, native debug symbols and JS source maps must still reach the crash reporter and store.
- **Authority.** Store submissions, TestFlight and Play uploads, `eas submit`, OTA update publishes (`eas update` or a self-hosted equivalent), pushes and merges need the user's explicit approval. Signing keys, keystore passwords, API keys and provisioning profiles are sensitive: never print, copy or commit them. Installing on a real phone or signing in may need the user. Use simulators and emulators for automated proof and list what stays unverified on real hardware.

## 1. Measure

Build the current commit as a release, keep the artefacts, and record this before changing anything.

**Android**

- Build the bundle the store gets (`bundleRelease`, or the project's EAS/CI profile). Generate an APK set with `bundletool build-apks --bundle=app.aab --output=app.apks` (add `--ks`/`--ks-key-alias` only with the user's keystore, or use the debug keystore for size work; signing doesn't change size materially).
- `bundletool get-size total --apks=app.apks --dimensions=ABI,SCREEN_DENSITY,LANGUAGE,SDK` gives min and max download sizes per configuration as CSV. For a specific device class, pass `--device-spec=spec.json`; partial specs are allowed, and `bundletool get-device-spec --output=spec.json` captures one from a connected emulator or device. Record at least a low-end and a high-end spec (for example `arm64-v8a` at 420 dpi and `armeabi-v7a` at 320 dpi) plus every ABI the app ships.
- Composition: Android Studio's APK Analyzer, or `apkanalyzer` from cmdline-tools (`apk download-size`, `apk file-size`, `files list`, `dex packages`, and `apk compare` for before/after). Break it down into dex, `lib/<abi>/*.so`, `res/`, `assets/` (the JS bundle lives here on React Native), `resources.arsc` and `META-INF`.
- If Play Console is available, its app size reports (download and on-device, per device) are the authority for shipped builds. Treat them as last-known, not current, for local changes.
- Install size differs from download size: native libraries stored uncompressed (the current default; legacy packaging compresses them) make the APK larger but avoid extracting a second copy on install, so flipping that setting can shrink the archive while growing the install; ART compilation adds on-device bytes not visible in the archive.

**iOS**

- Archive the release scheme (`xcodebuild archive`), then export with `xcodebuild -exportArchive` and an export options plist with `thinning` set to `<thin-for-all-variants>` (or a model identifier such as `iPhone15,2`) and a non-App Store `method` such as `release-testing`. The export writes `App Thinning Size Report.txt` with compressed (download) and uncompressed (install) sizes per variant. Thinned export needs a valid signing identity; if one isn't available locally, stop and ask the user rather than hunting for credentials.
- App Store Connect shows per-device file sizes for each processed build. Use it for the shipped baseline; it reflects Apple's encryption, which makes the executable compress poorly, so code size counts almost fully against download size.
- Composition: unzip the IPA and size `Payload/*.app` by component: the main executable, `Frameworks/` (embedded dynamic frameworks, and `libswift*.dylib` if the deployment target is old enough to need the Swift runtime embedded), `Assets.car` (`xcrun assetutil --info Assets.car` lists every rendition with scale, idiom and size), loose images and fonts, `*.lproj`, `PlugIns/` (app extensions and widgets), `Watch/`, and the JS bundle (`main.jsbundle`) and `assets/` folder on React Native.

**JavaScript (React Native and Expo)**

- Check the shipped bundle is Hermes bytecode, not plain JS (`file` on the extracted bundle, or its header), and that no `.map` file is inside the APK or IPA.
- Break the bundle down by module: `EXPO_ATLAS=true npx expo export` then `npx expo-atlas .expo/atlas.jsonl` on Expo, or `source-map-explorer` / `react-native-bundle-visualizer` against a source map from a `--no-bytecode` export or the release build's packager map. The Atlas file contains source and inlined `EXPO_PUBLIC_` values, so keep it out of the repo.
- If the app uses `expo-updates` or another OTA system, record the update payload size separately (bundle plus new assets). It is a different number from binary size and changes in different ways.

Keep one table: per device class, download and install size, plus the top 20 contributors by bytes.

## 2. Find what the app actually uses

Static search finds most references and misses the ones that matter. For every asset or module you plan to remove, look for:

- **Names built at runtime:** template strings and concatenation (`` `icon_${type}` ``), `require` inside maps keyed by server values, Android `Resources.getIdentifier`, iOS `UIImage(named:)` with a computed string, and `NSClassFromString`/`Class.forName`.
- **Remote and server-driven names:** remote config, feature flags, CMS payloads, A/B variants, server-driven UI, and push payloads that name an image, sound or screen. Ask the user or check the backend before treating a name as unused.
- **Entry points other than the main UI:** deep links and universal/app links, notification sounds and icons (Android small icons, iOS `.caf`/`.aiff` files named in payloads), widgets, App Clips, share and notification service extensions, watch apps, Siri intents, Live Activities, quick actions, launch screens and splash assets, adaptive and alternate app icons, and Android `xml/` resources referenced from the manifest.
- **Shared targets:** assets that a widget or extension reads from the main bundle or an app group, and resources pulled in by name from a native library.
- **Rare states:** onboarding, empty, error, offline, paywall, upgrade-required, maintenance, dark mode, RTL, large Dynamic Type and font scale, tablet layouts, and every supported locale.
- **Dependencies:** for each native module or package, find its call sites; for Expo, check which `expo-*` modules are autolinked but never imported. A package can be unused in JS but still required natively (config plugins, push, auth callbacks).

Record each candidate with its evidence (where it's referenced, or every place you looked and didn't find it).

## 3. Cut it down

Work in small, separately measured changes so each saving and each regression has a clear cause.

**Code shrinking (Android)**

- Turn on R8 for release: `isMinifyEnabled = true` and `isShrinkResources = true` with `proguard-android-optimize.txt` on AGP versions before 9.3, or `optimization { enable = true }` on AGP 9.3 and later (rules in `src/<variant>/keepRules/*.keep`). React Native templates expose `enableProguardInReleaseBuilds`; Expo uses `expo-build-properties` `android.enableMinifyInReleaseBuilds`, `enableShrinkResourcesInReleaseBuilds` and `extraProguardRules`. Check the current docs for the project's AGP and SDK version before writing the setting.
- Missing keep rules don't fail the build; they crash or misbehave at runtime, only on the code path that needs them. Audit: reflection and `Class.forName`, JSON models parsed by reflection (Gson, Moshi or Jackson without codegen), Kotlin serialization, `Parcelable`/`Serializable`, enums looked up by name, WebView `@JavascriptInterface` methods, JNI methods called from native code, classes named in XML or the manifest, React Native native modules and view managers (third-party libraries don't always ship consumer rules), Firebase and analytics SDKs, and payment SDKs.
- Obfuscation renames classes. If class names are persisted (serialised preferences, cached JSON keyed by class name, WorkManager input, saved state), an update over an older install can lose or corrupt data. Test an upgrade install, not just a fresh one.
- Read R8's outputs under `build/outputs/mapping/<variant>/` (`mapping.txt`, and `usage.txt` for what was removed). Add narrow `-keep` rules for what must survive; never use a blanket keep of the whole app package to make a crash go away.

**Resource shrinking (Android)**

- `shrinkResources` keeps any resource whose name looks reachable from string constants, unless strict mode is on. For names built at runtime, keep them explicitly with `tools:keep` in a keep file such as `res/raw/keep.xml`; use `tools:discard` only for proven-dead resources.
- Language splits in an app bundle are automatic: users get only their device languages. If the app has an in-app language picker, that drops strings for other languages unless language splitting is disabled or the app requests them. Check before and after.
- `resourceConfigurations` (`resConfigs`) removes locales that libraries ship but the app doesn't support. That changes what library UI shows in those locales, so treat it as a change for the user to approve, and list the locales.
- Native libraries: the app bundle already splits by ABI. Restricting ABIs (`abiFilters`, `reactNativeArchitectures`, Expo `buildArchs`) drops devices or emulators; never remove `arm64-v8a`, and only drop `armeabi-v7a` or `x86_64` with the user's approval. Duplicated `.so` files across libraries resolved with `pickFirst` can silently pick an incompatible version; check what was picked.

**iOS**

- Release settings: `DEAD_CODE_STRIPPING = YES`, `STRIP_INSTALLED_PRODUCT = YES`, `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym`, and consider `SWIFT_OPTIMIZATION_LEVEL = -Osize` (it changes codegen, so it needs the full proof pass). Bitcode is gone from current Xcode; remove stale `ENABLE_BITCODE` settings rather than relying on them.
- Move loose images into an asset catalog so the App Store slices by scale and idiom; loose `@1x/@2x/@3x` PNGs ship every scale to every device. React Native images required from JS ship as loose files on iOS, so this mostly applies to native images.
- Drop `@1x` only if no supported device uses it. Check iPad and Mac Catalyst targets before assuming it's unused.
- Dynamic frameworks each carry their own overhead; `use_frameworks!` or many embedded frameworks cost size and launch time. Switching linkage is a proposal, not a default edit, because it can change initialisation order.
- Assets duplicated between the app and its extensions or watch app are shipped once per bundle. Note them; consolidating is a proposal.

**Assets and media (both platforms)**

- Lossless recompression (oxipng, zopflipng, lossless WebP on Android) changes no pixels and is fine by default. Lossy compression, resizing, format changes that alter colour or alpha, and HEIC or AVIF need approval. Check the build doesn't undo it: Xcode and AGP PNG crunching can re-encode files (Expo exposes `enablePngCrunchInReleaseBuilds`).
- Raster images shipped at a larger pixel size than any device ever renders can be resized without visible change; prove it with the screenshot diff at the largest supported scale.
- Simple icons can become vectors (Android vector drawables, iOS PDF/SVG in asset catalogs, `react-native-svg` on RN). Compare rendering exactly; vector and raster anti-aliasing differ, and `react-native-svg` adds its own native code, so it only pays when it replaces enough PNGs.
- Lottie files with embedded base64 images are often the biggest single assets; extract or optimise the images, or use the dotLottie format if the player version supports it, and diff the animation frames.
- Fonts: drop unused weights and families after checking every style, including native screens and extensions. Subsetting app fonts has the same edge cases as web fonts (see the font-subset skill).
- Video and audio: check bitrate and resolution against how they're displayed. Re-encoding is lossy, so it needs approval.

**JavaScript (React Native and Expo)**

- Hermes on, source maps generated for upload but never bundled.
- Remove unused dependencies and their native code; replace heavy libraries (whole-library `lodash`, `moment` with all locales, full icon fonts when a few glyphs are used) only where the replacement behaves identically, including dates, time zones and number formatting.
- On Expo, remove unused `expo-*` packages from `package.json`, or exclude them from autolinking via `expo.autolinking.exclude`, then run the project's checks (`npx expo-doctor` if used) and a clean prebuild.
- `expo-updates` embeds an update in the binary: check which assets it bundles (`updates.assetPatternsToBeBundled`, or `assetBundlePatterns` on older SDKs) so nothing needed offline on first launch is left to download.
- Removing a native module changes the native runtime. Bump `runtimeVersion` (or whatever the project uses) so already-published OTA updates that call the removed module can't reach the new binary.
- Re-encoding an asset changes its hash, so the next OTA update makes every client download it again. Report the one-off OTA cost alongside the binary saving.

**Symbols**

- iOS: keep generating dSYMs, confirm the crash reporter's upload step still runs for the release configuration, and keep `uploadSymbols` for App Store exports.
- Android: Play receives R8 mappings with the bundle. Set `ndk.debugSymbolLevel` (`SYMBOL_TABLE` or `FULL`) on the release build type so native symbols ship with the bundle, and confirm third-party crash reporters (Crashlytics, Sentry and others) upload mappings and native symbols themselves.
- React Native with Hermes: the crash reporter needs the composed Hermes source map, not just the packager map. Confirm its build step produces and uploads it.

## 4. Prove it

1. **Baseline from the same commit.** Build the baseline release from the commit you started on and the candidate from your change, with the same toolchain, flags and signing approach. A store build from weeks ago mixes in unrelated changes.
2. **Sizes.** Rerun every measurement from step 1 for both builds: `bundletool get-size total` per device spec and dimension, `apkanalyzer apk compare`, the App Thinning Size Report per variant, and the JS bundle and OTA payload sizes.
3. **Contents diff.** List files in both archives and explain every removed or changed entry. Anything removed that you didn't intend to remove is a finding.
4. **Screenshot diffs** of every screen and state from step 2 on a simulator and an emulator, driven by the project's device and e2e tooling. Make them deterministic: fixed device model and OS, fixed locale, time zone and clock, status bar overridden, animations off, the same test account and seeded data, light and dark, largest font scale, RTL where supported. Diff at threshold 0, save diff images, and classify every difference as content, regression, or harmless rendering noise (with the evidence).
5. **Smoke flows on the release build** covering every path minification or removal could break: cold start and warm start, every deep link and universal/app link, push notification received in foreground, background and killed state (including tap-through and custom sounds), every serialised model round-trip (network parsing, local cache, saved state), WebView bridges, payments in the sandbox, sign-in and sign-out, widgets and extensions, offline launch, and an upgrade install over the baseline build with existing data. Watch logcat and the device log for `ClassNotFoundException`, `NoSuchMethodError`, missing resource and missing image warnings, and JS red boxes and errors.
6. **Crash reporting.** Trigger a test crash in the release build (native and JS where relevant) and confirm the reporter shows a symbolicated stack trace. If the upload needs credentials you don't have, say so and leave it unverified.
7. **Fix and repeat.** For each regression, add the narrow keep rule or restore the asset, rebuild and rerun the affected proof steps.

## Report back

- Download and install size before and after, per device class and platform, with the source of each number (bundletool, thinning report, store console) and whether it's measured or last-known.
- Each change with its saving and what it changes, including changes held back for approval.
- Screenshot diff results per screen, state and device, with every difference classified.
- Smoke flow results, the keep rules added and why, and the crash reporting check.
- OTA effects: runtime version changes and one-off re-download cost.
- What wasn't verified: real hardware, older OS versions and low-end devices, store-processed sizes, payments in production, and any flow you couldn't drive.

**Later (proposals, not part of the default change):** on-demand resources or Play Asset Delivery and Play Feature Delivery for rarely used content, streaming large video or downloading media after install, dropping old OS versions or ABIs, static linkage of dynamic frameworks, sharing assets between the app and its extensions, and a CI size budget that fails a build when per-device download size grows past a threshold. Weigh each against complexity, offline behaviour and the users it would affect before proposing it.
