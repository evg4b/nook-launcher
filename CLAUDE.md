# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Nook Launcher: an Android TV / Google TV home screen (fork of Couchy Launcher, GPLv3). It is a single module (`app`) in Kotlin + Jetpack Compose for TV (`androidx.tv:tv-material`), with min SDK 21 and target 34. It ships on F-Droid, so dependencies must stay FOSS: no Play Services and no trackers.

## Build & run

You need JDK 17 and Android SDK 34. There is no `local.properties` in the repo, so export the paths yourself (these are the paths on this machine):

```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@17 ANDROID_HOME=~/Library/Android/sdk
./gradlew assembleDebug
./gradlew assembleDebug assembleRelease lintRelease   # what CI runs on every PR
./gradlew installRelease -PlocalSign                  # release signed with the debug key, installable locally
```

- **Release signing:** without `-PlocalSign` the release APK is unsigned. That is the F-Droid variant. CI signs tagged builds (`v*`) using `KEYSTORE_*` env vars.
- **No tests:** there are no unit or instrumentation tests. Verify changes by running the app on a TV or emulator.
- **Running on the TV:** after `adb connect <tv-ip>:5555`, launch with `adb shell am start -n com.evg4b.nooklauncher/com.conreo.couchytv.MainActivity`.

## Package names: two different things

- **`applicationId`** is `com.evg4b.nooklauncher`.
- **`namespace` and every Kotlin package** stay `com.conreo.couchytv`.

So the activity component is `com.evg4b.nooklauncher/com.conreo.couchytv.MainActivity`. The short form `.MainActivity` does not resolve. The R8 keep rules in `app/proguard-rules.pro` and the F-Droid recipe (`fdroid/com.conreo.couchytv.yml`) still use the old names.

## Architecture

**Single activity, single state holder.** `MainActivity` hosts `LauncherApp` in `ui/LauncherScreen.kt`, and nearly all screen state lives in that one composable:
- menus, move mode, settings visibility;
- wallpaper and clock;
- the dock's expanded state.

`LauncherApp` renders one of three layouts from `config.layout`:
- `LAYOUT_CAROUSEL`: a `LazyRow` per section;
- `LAYOUT_GRID`: `GridSection`;
- `LAYOUT_DOCK` (the default): `DockArea`, a single glass row that expands to a full grid on Down.

**Config (`data/Config.kt`).** `LauncherConfig` is one `@Serializable` data class, stored as a single JSON string in Preferences DataStore. `ConfigStore` keeps a process-wide in-memory `MutableStateFlow` as the source of truth:
- `update {}` changes memory first, so the UI updates on the next frame, then writes to disk.
- The write always saves the latest in-memory value.

New fields need defaults, because decoding uses `ignoreUnknownKeys` and old JSON must keep loading. The UI language is also mirrored to plain SharedPreferences, because `attachBaseContext` has to read it synchronously. A language change calls `recreate()`.

**App scan (`data/AppRepository.kt`).** `scan()` queries LEANBACK_LAUNCHER and LAUNCHER activities and loads labels, banners and icons in parallel on a capped dispatcher.
- Banners and icons are cached as lossless WebP in `filesDir/iconcache`, keyed by package, `lastUpdateTime` and pixel size.
- Unchanged packages reuse the previous `AppEntry` instance. `AppEntry` is `@Immutable`, which lets Compose skip those cards.
- `memoryCache` makes a warm relaunch instant.
- `MainActivity`'s package broadcast receiver bumps `rescanTick` to trigger a rescan.
- `categorize()` maps apps to sections. The sources, in order: explicit `config.sections`, then a known-package or `ApplicationInfo.category` guess, then the fallback section. `__all__` is the auto-filled "All apps" section.

**Input model (remote-first).** Every control must be reachable with the D-pad, and text input only happens in a dedicated dialog.
- **Long press:** `AppCard` detects it itself from OK key auto-repeats (≥450 ms), then suppresses the click that the release would otherwise fire.
- **Move mode:** `LauncherApp`'s `onPreviewKeyEvent` owns every key while move mode is on. It ends on key-*up*, so the release can't fall through and launch the focused card.
- **MENU** always opens settings.

**Wallpaper.** Priority order: built-in aerials, then local video, then photo, then the preset gradient.
- **Video** uses ExoPlayer on a `SurfaceView`. The window background is cleared only in video mode. The player is released on `ON_STOP`.
- **Photo, preset and scrim** are drawn together inside one `graphicsLayer(CompositingStrategy.Offscreen)` so HWUI caches them.

## Performance constraints (measured on a weak TCL TV: RTD2851M, 4× A55, 2 GB)

- **Bottleneck:** frames there are GPU-bound, not UI-thread-bound. Any extra full-screen draw pushes focus animations from 60 to 30 fps.
- **Static full-screen content** must go inside the cached backdrop layer.
- **Shapes on tv-material `Surface`:** never pass a path-based (`Outline.Generic`) shape. `Surface` clips, glows (`setShadowLayer` + `drawPath`) and borders the path again on every frame. Use `RoundedCornerShape`.
- **Measure on release builds only.** Debug Compose is several times slower.
- **Measuring tools:**
  - frame times: `adb shell dumpsys gfxinfo <pkg> reset`, then `adb shell dumpsys gfxinfo <pkg> framestats`;
  - baseline profile: `app/src/main/baseline-prof.txt` is hand-written with wildcard rules and applied by profileinstaller.

Before benchmarking a fresh install, apply the profile and compile:

```bash
adb shell am broadcast -a androidx.profileinstaller.action.INSTALL_PROFILE -n com.evg4b.nooklauncher/androidx.profileinstaller.ProfileInstallReceiver
adb shell cmd package compile -m speed-profile -f com.evg4b.nooklauncher
```

Without this step the code runs interpreted and the numbers are misleading.

## Testing on a device with adb

- **Long press:** `input keyevent --longpress` cannot open the long-press menu, because it sends no real repeats. Use the `hid` tool with a virtual USB keyboard and hold Enter for about 1 s.
- **Reading saved config:** `run-as <pkg>` works only on debug builds. The config file is `files/datastore/launcher.preferences_pb`.

## Strings and translations

- **No hardcoded user-facing text.** Add it to `res/values/strings.xml` and reference it with `stringResource(...)`.
- **Locales:** 17 translations ship (`ar de es fr hi in it ja ko nl pl pt ru th tr vi zh`). New strings should also go into `values-fr` and `values-zh` at least.
