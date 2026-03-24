[← Back to Index](../README.md)

# SIGABRT / libc.so Crash Debugging Guide for React Native Android

> **A comprehensive guide** for diagnosing and fixing `SIGABRT` (Signal 6) and `SIGSEGV` (Signal 11) crashes originating from `libc.so` in React Native and Expo Android apps. Consolidates findings from GitHub issues, Stack Overflow threads, community articles, and production crash investigations.

---

## Quick Reference Card

Before diving into the full guide, here are the **5 most important things to check first** when you encounter a native crash:

| # | Check | How | Why |
|---|-------|-----|-----|
| 1 | **Read the FULL native stack trace** | Look past `libc.so` at frames #2, #3, #4 for the actual `.so` file that triggered the abort | The crash in `libc.so` is always the symptom, never the root cause. The library name in the stack tells you exactly where to look. |
| 2 | **Check device/OEM distribution** | Filter crashes by manufacturer in your crash reporting dashboard | If 80%+ of crashes are on one OEM (Huawei, Xiaomi, Samsung), it is likely a platform bug, not your code. This saves hours of debugging. |
| 3 | **Compare crash timing to releases** | Overlay crash-free rate graph with release dates | If crashes started with a specific version, `git diff` the dependency changes between that version and the previous stable one. |
| 4 | **Check if ALL app versions are affected** | Look at crash-free rate across old and current versions simultaneously | If old versions also started crashing at the same time, it is a Google Play System update or OS-level regression, not your code. |
| 5 | **Profile memory on a mid-range device** | Use Android Studio Profiler on a device with 3-4GB RAM while scrolling through media-heavy screens | Most SIGABRT crashes in production are OOM kills disguised as native crashes. If total memory exceeds ~800MB, low-end devices will crash. |

---

## Table of Contents

1. [Understanding the Crash](#1-understanding-the-crash)
2. [Root Cause Categories](#2-root-cause-categories)
   - [Category A: Incompatible or Buggy Native Libraries](#category-a-incompatible-or-buggy-native-libraries)
   - [Category B: TurboModuleRegistry / PlatformConstants (RN 0.76+)](#category-b-turbomoduleregistry--platformconstants-rn-076)
   - [Category C: Hermes Engine Crashes](#category-c-hermes-engine-crashes)
   - [Category D: OEM-Specific Crashes](#category-d-oem-specific-crashes-huawei-xiaomi-samsung)
   - [Category E: Google Play System Updates](#category-e-google-play-system-updates-os-level-bug)
   - [Category F: Out of Memory (OOM)](#category-f-out-of-memory-oom)
   - [Category G: MainApplication.onCreate() ANR](#category-g-mainapplicationoncreate-anr-leading-to-sigabrt)
   - [Category H: React Native Version Upgrade Crashes](#category-h-react-native-version-upgrade-crashes)
   - [Category I: ProGuard/R8 Over-Stripping](#category-i-proguardr8-over-stripping)
3. [Debugging Strategies](#3-debugging-strategies)
4. [Production Fixes and Solutions](#4-production-fixes-and-solutions)
5. [Performance and Memory Optimization Patterns](#5-performance-and-memory-optimization-patterns)
6. [Crash Reporting Tool Comparison](#6-crash-reporting-tool-comparison)
7. [Common Mistakes That Cause These Crashes](#7-common-mistakes-that-cause-these-crashes)
8. [Decision Framework](#8-decision-framework)
9. [Auditing Your Own App](#9-auditing-your-own-app)
10. [Sources](#10-sources)

---

## 1. Understanding the Crash

### What is SIGABRT?

`SIGABRT` (Signal 6) means the process **deliberately terminated itself** due to a detected internal inconsistency. When originating from `libc.so`, it indicates a problem in native C/C++ code -- below the JavaScript layer entirely. The Android runtime detected something so wrong that it decided crashing immediately was safer than continuing to run.

```
# Typical crash signature in logs
Fatal signal 6 (SIGABRT), code -1 (SI_QUEUE)
  crash_info: "abort"
  backtrace:
    #0  libc.so (abort+...)          <-- The abort() call itself
    #1  libc.so (__fortify_fail+...) <-- Security check that detected corruption
    ...                              <-- Frames above here reveal the REAL cause
```

Common `libc.so` functions you will see in crash stacks:

| Function | What It Means |
|----------|--------------|
| `abort` | The process called `abort()` deliberately |
| `__fortify_fail` | A buffer overflow or format string attack was detected by FORTIFY_SOURCE |
| `__stack_chk_fail` | Stack canary corruption detected (stack buffer overflow) |
| `scudo::die` | The Scudo memory allocator detected heap corruption |
| `ifree` / `free` | Double-free or free of invalid pointer |
| `memcpy` / `memmove` | Buffer overflow during memory copy |

### What is SIGSEGV?

`SIGSEGV` (Signal 11) occurs when the app attempts to access memory it does not have permission to access, memory that has been freed, or memory that does not exist. There are two sub-codes:

| Code | Name | Meaning |
|------|------|---------|
| `SEGV_MAPERR` (code 1) | Mapping error | The address is not mapped to any memory region. Often means a null pointer dereference or accessing freed memory. |
| `SEGV_ACCERR` (code 2) | Access error | The address exists but the app does not have permission (e.g., writing to read-only memory). |

```
Fatal signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
```

A fault address of `0x0` or a very small value (like `0x8` or `0x10`) almost always means a null pointer dereference -- the code tried to read a field from an object that was null.

### Why These Are Hard to Debug

- **~95% of reports are production-only** -- not reproducible on developer devices because development builds have different memory layouts, timing, and optimization levels
- Stack traces stop at native memory management functions (`abort`, `free`, `__fortify_fail`) and the actual corruption often happens long before the crash is detected
- The crash may be a **delayed symptom**: memory corrupted at time T but the crash only happens at time T+N when the corrupted memory is next accessed
- Crashlytics/Sentry may truncate critical native stack frames, especially on 32-bit devices where the stack is deeper
- Minified/obfuscated builds strip symbol names, making native stacks show only hex offsets instead of function names

### The JS-Native Boundary Problem

React Native apps run code across multiple threads and runtimes:

```
JavaScript Thread (Hermes/JSC)
    |
    v
JS Bridge / JSI (C++)           <-- Crashes here show as libjsi.so
    |
    v
Native Modules (Java/Kotlin)
    |
    v
Android Framework (libhwui.so, libart.so)
    |
    v
Linux Kernel (libc.so)          <-- Crashes "bubble down" to here
```

When something goes wrong at any layer, the crash often manifests at the bottom (`libc.so`) because that is where memory management primitives live. Your job is to trace the crash **upward** through the stack to find the actual layer that caused it.

---

## 2. Root Cause Categories

### Category A: Incompatible or Buggy Native Libraries

**Frequency**: Very common
**Signature**: Stack trace contains a specific `.so` file beyond `libc.so`

This is the single most common cause of native crashes in React Native apps. Third-party native modules ship their own compiled C/C++ code, and bugs in that code cause crashes that appear to originate from `libc.so`.

#### Known Problematic Libraries and Versions

| Library | Known Issue | Crash Signature | Fix |
|---------|------------|-----------------|-----|
| `@react-native-picker/picker` v2.11.2 | Memory corruption in native picker component | `libappmodules.so` crash | Upgrade to v2.11.3+ |
| `react-native-reanimated` < 3.x | Use-after-free in animation worklets | `libreanimated.so` memory bugs | Upgrade to latest stable |
| `react-native-reanimated` 3.x | Thread safety issues in worklet runtime | `libreanimated.so` + `libjsi.so` | Upgrade to 3.6+ or latest |
| `react-native-webview` | Hardware acceleration conflict on Android 9+ | `libhwui.so` crash (Samsung, Huawei) | Set `androidHardwareAccelerationDisabled={true}` or `androidLayerType="software"` |
| `react-native-screens` < 3.20 | Fragment transaction crash during rapid navigation | `java.lang.IllegalStateException` escalating to SIGABRT | Upgrade to 3.20+; avoid rapid back-forward navigation |
| `react-native-screens` 3.x | Screen container detach race condition | Crash during screen transitions, especially with nested navigators | Upgrade to latest; set `detachInactiveScreens={false}` as workaround |
| `react-native-gesture-handler` < 2.14 | Concurrent gesture recognition memory corruption | `libgesturehandler.so` or generic SIGSEGV during gestures | Upgrade to 2.14+; ensure single GestureHandlerRootView at app root |
| `react-native-gesture-handler` 2.x | Duplicate GestureHandlerRootView causes native crash | SIGABRT on gesture interaction | Ensure only ONE GestureHandlerRootView wraps your app |
| `react-native-svg` < 13.x | Native rendering crash with complex SVG paths | `librnsvg.so` crash | Upgrade to 13.x+; simplify complex SVG paths |
| `react-native-svg` 13+ | Large SVG rendering causes OOM on low-end devices | SIGABRT via memory pressure | Use PNG for complex illustrations; limit SVG complexity |
| `react-native-maps` | MapView crash on unmount during active animation | `libmapbox.so` or `libgmscore.so` | Cancel animations before unmount; wrap in error boundary |
| `react-native-maps` 1.x | Marker clustering crash with large datasets | SIGSEGV during map interaction | Limit markers to ~100; use clustering library |
| `expo-camera` < 14.x | Camera session crash on rapid open/close | `libcamera2ndk.so` or device-specific camera HAL crash | Upgrade expo-camera; add debounce to camera open/close; handle `onCameraError` |
| `expo-camera` | Simultaneous camera + audio recording crash | SIGABRT from media server | Avoid concurrent camera and audio sessions on low-end devices |
| `expo-av` < 13.x | Audio session conflict with other apps | `libaudioflinger.so` crash | Upgrade expo-av; set `playsInSilentModeIOS` and `staysActiveInBackground` carefully |
| `expo-av` | Video player memory leak in lists | OOM leading to SIGABRT | Unload video when component unmounts; limit concurrent video players to 2-3 |
| Sentry SDK | Performance monitoring memory pressure | `libc.so` abort from profiling overhead | Disable `userInteractionTracing` and set `profilesSampleRate(0.0)` -- see [Fix D](#fix-d-sentry-performance-monitoring-causing-sigsegv) |
| `react-native-firebase` messaging | FCM background handler crash on RN 0.76+ | `TurboModuleRegistry` SIGABRT | Upgrade to latest firebase packages; see Category B |
| `lottie-react-native` | Complex animation OOM | Native memory spike during animation | Use simpler animations; consider Rive as alternative |
| `react-native-video` < 6.x | Player release crash during background transition | `libmediaplayer.so` crash | Upgrade to 6.x+; pause video on `AppState` background |

**Debugging step**: Look at the crash stack for library-specific `.so` files. The most common ones to search for:

```
libreanimated.so     -> react-native-reanimated
libappmodules.so     -> @react-native-picker/picker
libhwui.so           -> WebView / hardware acceleration
libhermes.so         -> Hermes engine
libjsc.so            -> JavaScriptCore engine
libjsi.so            -> JS Interface (bridge layer)
librnscreens.so      -> react-native-screens
libgesturehandler.so -> react-native-gesture-handler
librnsvg.so          -> react-native-svg
libfbjni.so          -> Facebook JNI (core RN)
libglog.so           -> Google logging (core RN)
libfolly_runtime.so  -> Facebook Folly (core RN)
```

**Key principle**: If you can identify a specific `.so` file in the stack trace, search that library's GitHub issues for the crash signature. Most native crashes in React Native are known issues with known fixes.

---

### Category B: TurboModuleRegistry / PlatformConstants (RN 0.76+)

**Frequency**: Common on RN 0.76.0-0.76.5 (old architecture)
**Signature**: `EarlyJsError: TurboModuleRegistry.getEnforcing(...): 'PlatformConstants' could not be found`
**Trigger**: App launched from killed state via **push notification** (FCM)

This crash happens because the TurboModule system is not fully initialized when FCM tries to deliver a push notification to a cold-started app. The JavaScript runtime attempts to access `PlatformConstants` before the native module registry is ready.

**References**: [react-native#47074](https://github.com/facebook/react-native/issues/47074), [react-native-firebase#8131](https://github.com/invertase/react-native-firebase/issues/8131), [react-native#47592](https://github.com/facebook/react-native/issues/47592)

**Fixes** (in order of preference):
1. **Switch to New Architecture** (recommended) -- the new architecture handles module registration differently and does not have this timing issue
2. **Upgrade all Firebase packages to latest** -- the firebase team added workarounds for the registration race condition
3. **Downgrade to RN 0.75.x** if New Architecture is not feasible and Firebase updates do not resolve it

---

### Category C: Hermes Engine Crashes

**Frequency**: Moderate
**Signature**: `libhermes.so` and `libhermes-executor-release.so` in stack, often with deep recursive frames
**References**: [react-native#29978](https://github.com/facebook/react-native/issues/29978), [hermes#298](https://github.com/facebook/hermes/issues/298)

Hermes is the default JavaScript engine for React Native. Because it compiles JavaScript to bytecode ahead of time, bugs in the compiler or runtime can cause native crashes that are impossible to reproduce without the exact same bytecode.

**Common Hermes crash patterns**:

| Pattern | Cause | Fix |
|---------|-------|-----|
| Crash in `hermes::vm::GCGeneration` | Garbage collector bug | Upgrade RN (Hermes patches ship with RN releases) |
| Deep recursive frames in stack | Stack overflow from recursive JS code or Hermes compilation bug | Check for recursive function calls; increase stack size if needed |
| Crash after Hermes bytecode version mismatch | Stale bytecode cache from previous build | Clean build: `cd android && ./gradlew clean` |
| Crash only in release mode | Hermes optimization bug | Try `hermes-engine@nightly` to see if it is a known-fixed bug |

**Fixes**:
1. **Upgrade to latest RN patch** -- Hermes bugs are fixed frequently in patch releases and do not always get called out in changelogs
2. **Clean build**: `cd android && ./gradlew clean` -- stale bytecode from a previous Hermes version can cause crashes
3. **On very old RN (0.59-0.63)**, try switching between JSC and Hermes to isolate whether the crash is engine-specific
4. **Check Hermes GitHub issues** with your crash signature -- many crashes are known and tracked

---

### Category D: OEM-Specific Crashes (Huawei, Xiaomi, Samsung)

**Frequency**: High on specific OEMs
**Signature**: 80%+ crashes concentrated on Huawei/Honor/Xiaomi/Redmi/OPPO devices. Stack may show `libhwui.so` or JSC-specific frames.
**References**: [react-native#31235](https://github.com/facebook/react-native/issues/31235), [react-native#33083](https://github.com/facebook/react-native/issues/33083)

Android OEMs ship heavily modified versions of the Android framework. These modifications frequently introduce bugs in:
- **Hardware rendering** (`libhwui.so`) -- Samsung and Huawei modify the GPU rendering pipeline
- **Memory management** -- Xiaomi/OPPO have aggressive memory-killing heuristics
- **JavaScript engine compatibility** -- Huawei's custom WebView/JSC builds have known bugs
- **Background processing** -- Many OEMs kill background processes more aggressively than stock Android

**Fixes**:
1. **Switch from JSC to Hermes** -- Hermes is compiled by the RN team and is consistent across OEMs, while JSC relies on the device's system WebView
2. **Avoid RAM bundles on affected OEMs** -- RAM bundles interact poorly with some OEM memory managers
3. **Upgrade RN to get newer JSC/Hermes builds** -- newer builds have workarounds for OEM-specific issues
4. **Disable hardware acceleration for WebView on affected devices** -- `androidHardwareAccelerationDisabled={true}`
5. **Test on actual OEM devices** -- emulators run stock Android and will never reproduce these crashes

**How to confirm it is an OEM bug**: In your crash dashboard, group crashes by `device.manufacturer`. If one manufacturer accounts for 80%+ of a specific crash, it is almost certainly an OEM-specific issue.

---

### Category E: Google Play System Updates (OS-Level Bug)

**Frequency**: Rare but catastrophic when it happens
**Signature**: Crash-free rate drops suddenly across **ALL app versions** (including old ones that were previously stable). Error: `stack corruption detected (-fstack-protector)` in `libc`.
**Reference**: [react-native#39505](https://github.com/facebook/react-native/issues/39505) (September 2023 incident)

Google Play System updates modify core Android components (WebView, media codecs, TLS) without requiring a full OS update. When a buggy system update ships, it can break every app on the device simultaneously.

**Debugging step**: If your crash-free rate drops overnight across all versions (including versions that were stable for months), check:
1. The date of the most recent Google Play System update
2. [Google Issue Tracker](https://issuetracker.google.com/) for reports from other developers
3. Community forums (Reddit r/androiddev, Twitter/X) for reports of widespread crashes

**Fixes**:
1. **Update Sentry SDK** if you use Sentry -- they released specific fixes for stack corruption interaction with system updates ([sentry-java#2955](https://github.com/getsentry/sentry-java/issues/2955))
2. **Report to [Google Issue Tracker](https://issuetracker.google.com/)** -- Google monitors these reports and can push hotfixes
3. **Wait for next Google Play System update** -- Google typically pushes a fix within 1-2 weeks for widespread issues
4. **Communicate to users** -- if the issue is widespread, it helps to acknowledge it proactively

---

### Category F: Out of Memory (OOM)

**Frequency**: Very common, especially for media-rich apps
**Signature**: Stack shows only `libc.so` -> `libart.so` with no app-specific library. Device is low on RAM. Often appears as a "clean" SIGABRT with no obvious cause.

OOM is the most common cause of SIGABRT in production React Native apps. The Android low-memory killer (LMK) terminates the process, or the native allocator calls `abort()` when it cannot satisfy an allocation request.

**Evidence from production profiling** (from Omkar Salapurkar's investigation):
```
Initial Memory Profile:        After Scrolling:
Total: 577.2 MB                Total: 978.6 MB
|- Native:   294.7 MB          |- Native:   283.6 MB
|- Graphics:  70.1 MB  <-      |- Graphics: 472.0 MB  <- 7x INCREASE
|- Java:      46.2 MB          |- Java:      60.4 MB
|- Code:      74.3 MB          |- Code:      72.4 MB
|- Stack:      8.8 MB          |- Stack:     15.3 MB
'- Others:    83.1 MB          '- Others:    75.0 MB
```

Graphics memory jumped from 70MB to 472MB during scrolling. The primary culprits were autoplay videos, animated GIFs, and simultaneous media playback. On a device with 3GB total RAM, this leaves almost nothing for other processes, triggering the LMK.

**Key insight**: On mid-range devices (3-4GB RAM), your app's total memory should stay under ~600MB. On low-end devices (2GB RAM), stay under ~400MB. Exceeding these thresholds will cause crashes on a significant percentage of your user base.

---

### Category G: MainApplication.onCreate() ANR Leading to SIGABRT

**Frequency**: Common
**Trigger**: Too much initialization work on the main thread during app startup
**ANR threshold**: Android triggers ANR if main thread is blocked for 5s (background services) or 10s (foreground with user input pending)

When the main thread is blocked during `onCreate()`, the system may escalate an ANR (Application Not Responding) into a process kill, which appears as a SIGABRT in crash logs.

**Typical offenders in `onCreate()`**:
- Firebase initialization + persistence setup (synchronous disk I/O)
- Analytics SDK initialization (network + disk I/O)
- Push notification token retrieval (network call)
- WebView pre-warming (heavy initialization)
- Database migrations (disk I/O)
- License/token validation (network call)
- Multiple SDK initializations chained synchronously

**How to identify**: Look for ANR reports in Google Play Console alongside SIGABRT crashes. If they spike together, the ANR is causing the SIGABRT.

---

### Category H: React Native Version Upgrade Crashes

**Frequency**: Common (affects ~40% of production crash investigations)
**Signature**: Crashes begin immediately after deploying a React Native version upgrade. May manifest as any of the other categories but the timing correlates with the upgrade.

Major React Native version upgrades are one of the most common triggers for native crashes because they change multiple native layers simultaneously: the JavaScript engine, the bridge/JSI layer, native module APIs, and the build toolchain.

#### New Architecture Migration Crashes

The migration from Old Architecture (Bridge) to New Architecture (Fabric + TurboModules) is particularly risky because it fundamentally changes how JavaScript communicates with native code:

| Crash Pattern | Cause | Fix |
|---------------|-------|-----|
| SIGABRT on app launch after enabling New Arch | Native modules not compatible with TurboModules | Check each native dependency for New Architecture support before enabling |
| SIGSEGV during screen transitions | Fabric renderer handles view lifecycle differently | Upgrade react-native-screens and react-native-reanimated to Fabric-compatible versions |
| Crash when using `requireNativeComponent` | Old Architecture API not available in New Architecture | Migrate to `codegenNativeComponent` |
| Intermittent crash on JS-to-native calls | JSI threading model differs from Bridge | Ensure native modules are thread-safe; avoid accessing UI from background threads |
| `libfabricjni.so` crash | Fabric rendering bug, usually in early adoption | Upgrade to latest RN patch |

#### Version-Specific Known Issues

| Upgrade Path | Known Crash | Resolution |
|-------------|-------------|------------|
| 0.71 -> 0.72 | Hermes bytecode version mismatch if not clean building | `cd android && ./gradlew clean` before every build after upgrade |
| 0.72 -> 0.73 | `react-native-reanimated` < 3.5 incompatible | Upgrade reanimated to 3.5+ before upgrading RN |
| 0.73 -> 0.74 | Gradle 8 required; old Gradle plugins crash at build time | Update `android/gradle/wrapper/gradle-wrapper.properties` and all Gradle plugins |
| 0.74 -> 0.75 | Deprecated `ViewPropTypes` removal crashes third-party libraries | Audit and upgrade all dependencies that use `ViewPropTypes` |
| 0.75 -> 0.76 | TurboModuleRegistry crash on Old Architecture (see Category B) | Enable New Architecture or pin to 0.75.x |
| 0.76 -> 0.77+ | New Architecture becomes default; many libraries need updates | Test every native dependency against New Architecture before upgrading |

#### Safe Upgrade Checklist

1. **Read the release notes AND the upgrade helper** diff: https://react-native-community.github.io/upgrade-helper/
2. **Check every native dependency** for compatibility with the target RN version before starting
3. **Upgrade in a separate branch** and run a full test cycle including device testing on low-end hardware
4. **Stage the rollout** at 1% -> 5% -> 25% -> 100%, monitoring crash-free rate at each stage
5. **Keep the previous version branch alive** so you can hotfix if the upgrade causes production crashes
6. **Clean build after every RN upgrade**: `cd android && ./gradlew clean && cd .. && rm -rf node_modules && yarn install`

---

### Category I: ProGuard/R8 Over-Stripping

**Frequency**: Moderate (especially after enabling minification for the first time or upgrading R8)
**Signature**: Crashes happen ONLY in release builds, never in debug. Stack traces may show obfuscated class names or method-not-found errors that cascade into native crashes.

R8 (the successor to ProGuard) is Android's code shrinker and optimizer. It analyzes your code at build time and removes classes, methods, and fields that it believes are unused. The problem is that React Native and many native modules use **reflection** to access Java/Kotlin classes at runtime, and R8 cannot see these reflection-based accesses during its static analysis.

#### How R8 Over-Stripping Causes Native Crashes

```
Build time:
  R8 sees that class MyNativeModule.processData() is never called directly
  R8 removes processData() from the APK

Runtime:
  JavaScript calls NativeModules.MyNativeModule.processData()
  React Native bridge uses reflection to find processData()
  Method not found -> NullPointerException -> cascades to native crash
  libc.so abort
```

#### Common R8 Stripping Victims

| What Gets Stripped | Symptom | Keep Rule |
|-------------------|---------|-----------|
| Native module methods | `NoSuchMethodException` -> SIGABRT | `-keep class com.yourpackage.** { *; }` |
| TurboModule spec interfaces | `TurboModuleRegistry` crash | `-keep class * extends com.facebook.react.turbomodule.core.interfaces.TurboModule { *; }` |
| Hermes bytecode loader | Crash on app launch | `-keep class com.facebook.hermes.** { *; }` |
| View managers | `ViewManager not found` crash on render | `-keep class * extends com.facebook.react.uimanager.ViewManager { *; }` |
| JSI bindings | SIGSEGV on any JS-to-native call | `-keep class com.facebook.jni.** { *; }` |
| Reflection-based serialization (Gson, Moshi) | Crash when parsing API responses | `-keep class your.model.package.** { *; }` |

#### Essential ProGuard/R8 Rules for React Native

```proguard
# android/app/proguard-rules.pro

# React Native core -- NEVER strip these
-keep class com.facebook.react.** { *; }
-keep class com.facebook.hermes.** { *; }
-keep class com.facebook.jni.** { *; }

# TurboModules (RN 0.68+)
-keep class * extends com.facebook.react.turbomodule.core.interfaces.TurboModule { *; }

# View Managers -- R8 cannot see these are used via reflection
-keep class * extends com.facebook.react.uimanager.ViewManager { *; }
-keep class * extends com.facebook.react.uimanager.ViewGroupManager { *; }

# JSI -- the JS Interface layer uses JNI heavily
-keep class com.facebook.jni.** { *; }
-keepclassmembers class * {
    @com.facebook.jni.annotations.DoNotStrip *;
    @com.facebook.react.bridge.ReactMethod *;
    @com.facebook.proguard.annotations.DoNotStrip *;
}

# Hermes engine
-keep class com.facebook.hermes.unicode.** { *; }

# Keep native module names for error messages
-keepnames class * extends com.facebook.react.bridge.ReactContextBaseJavaModule
```

#### Diagnosing R8 Issues

```bash
# Step 1: Test with minification disabled to confirm R8 is the cause
# In android/app/build.gradle, temporarily set:
#   minifyEnabled false
# If crashes stop, R8 is stripping something needed.

# Step 2: Find what R8 is removing
# Add to android/app/build.gradle:
#   buildTypes {
#     release {
#       proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
#       // Add this line to see what R8 removes:
#       // It generates a report in app/build/outputs/mapping/release/usage.txt
#     }
#   }

# Step 3: Check the R8 usage report
cat android/app/build/outputs/mapping/release/usage.txt | grep "com.facebook.react"
# Any React Native classes listed here are being stripped and may cause crashes
```

**Key principle**: When you first enable R8 minification, roll it out as a staged release (1% -> 5% -> 25% -> 100%) and monitor crash-free rate at each stage. If crash-free rate drops, check the R8 usage report for stripped classes.

---

## 3. Debugging Strategies

### Strategy 1: Read the Full Native Stack Trace

The crash in `libc.so` is the **symptom**, not the cause. Look at frames **before** the abort:

```
#0  libc.so (abort+...)          <- symptom: the final abort call
#1  libc.so (__fortify_fail+...) <- symptom: security check that triggered the abort
#2  libreanimated.so (...)       <- ROOT CAUSE: reanimated library triggered the issue
#3  libjsi.so (...)              <- context: it happened during a JS-to-native call
```

**Pro tip**: Google Play Console (Android Vitals) often has more complete crash logs than third-party tools. It includes the full "Thread terminating due to..." message and all thread stacks at crash time, which Crashlytics and Sentry may truncate.

**How to get the best stack traces**:
1. Enable NDK crash reporting in your crash tool (Crashlytics: `firebaseCrashlytics { nativeSymbolUploadEnabled true }`)
2. Upload native debug symbols (`.so` files with debug info) to your crash reporting service
3. Do NOT strip debug symbols from debug builds -- only strip in release
4. Check Google Play Console even if you use another crash tool -- it captures different information

### Strategy 2: Device/OS Distribution Analysis

In your crash analytics dashboard, filter by:

| Filter | What It Reveals | Actionable If |
|--------|----------------|---------------|
| Device manufacturer | OEM-specific bugs (Huawei, Xiaomi) | 80%+ crashes on one OEM |
| Android version | OS-level regressions | Crashes concentrated on one API level |
| App version | Did a specific release introduce the crash? | Sharp increase after a version |
| All versions affected? | Google Play System update bug | All versions crash-free rate dropped together |
| CPU architecture | ABI-specific native code bugs | 90%+ crashes on arm64 or armeabi-v7a |
| RAM size | OOM on low-end devices | Crashes concentrated on devices with less than 3GB RAM |

**Pattern recognition**: If 80%+ crashes are on one OEM, it is likely a platform bug, not your code. If crashes started with one app version, `git diff` the dependencies.

### Strategy 3: Memory Profiling in Android Studio

1. Open Android Studio -> **View -> Tool Windows -> App Inspection -> Profiler**
2. Run app on a **physical device** (not emulator -- emulators have different memory limits and behavior)
3. Navigate through the app, especially media-heavy screens
4. Watch for:
   - **Graphics memory** spikes (images, videos, GIFs) -- this is the most common leak in React Native apps
   - **Native memory** growth (native library leaks) -- grows steadily without plateauing
   - **Java heap** growth (object retention) -- check for retained React Native views
5. Take heap dumps before/after scrolling to compare
6. Use the **Allocation Tracker** to see which allocations are growing

**Key metric**: If total memory exceeds ~800MB on a mid-range device (4GB RAM), expect SIGABRT on lower-end hardware (2-3GB RAM). Budget your app's memory ceiling for the lowest-spec device you support.

### Strategy 4: Logcat Filtering for Native Crashes

```bash
# Filter for fatal libc messages -- shows only abort/signal messages
adb logcat -s libc:F

# Filter for signal/abort messages across all tags
# This catches crash messages regardless of which component logged them
adb logcat | grep -E "(SIGABRT|SIGSEGV|signal 6|signal 11|Fatal signal|abort)"

# Full crash output from debuggerd -- the Android crash daemon
# debuggerd captures the complete stack trace, registers, and memory map
adb logcat -s DEBUG:*

# Watch for ANR messages that may precede SIGABRT
adb logcat | grep -E "(ANR|Input dispatching timed out|Slow operation)"

# Memory pressure messages from the low-memory killer
adb logcat | grep -E "(lowmemorykiller|LMK|oom_adj)"
```

### Strategy 5: AddressSanitizer (ASan)

The **most powerful tool** for memory corruption bugs. ASan instruments every memory access in native code and catches corruption at the exact moment it happens, not when the corrupted memory is next accessed (which could be much later).

```gradle
// In android/app/build.gradle
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                // c++_shared is required for ASan to work with shared libraries
                arguments "-DANDROID_STL=c++_shared"
                // -fsanitize=address: enables AddressSanitizer
                // -fno-omit-frame-pointer: preserves stack frame info for better traces
                cFlags "-fsanitize=address -fno-omit-frame-pointer"
                cppFlags "-fsanitize=address -fno-omit-frame-pointer"
            }
        }
    }
}
```

ASan detects:
- **Use-after-free**: accessing memory after `free()` was called
- **Double-free**: calling `free()` on the same pointer twice
- **Buffer overflows/underflows**: reading or writing past array boundaries
- **Stack-buffer-overflow**: writing past local variable boundaries on the stack
- **Use-after-return**: using a pointer to a stack variable after the function returned

**Caveat**: ASan adds ~2x memory overhead and ~2x slowdown. It also increases APK size significantly. Use only in testing builds, never in production.

### Strategy 6: Bundle Size Comparison

Large bundles increase memory pressure at startup because the entire JavaScript bundle must be parsed and compiled (or loaded from bytecode cache). Compare before/after any dependency changes:

```bash
# Generate the JS bundle and source map for analysis
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/bundle.js \
  --sourcemap-output /tmp/bundle.js.map

# Check raw bundle size -- anything over 2MB deserves investigation
ls -lh /tmp/bundle.js

# Analyze which dependencies contribute most to bundle size
# This opens an interactive treemap visualization
npx source-map-explorer /tmp/bundle.js /tmp/bundle.js.map
```

For Expo:
```bash
# Expo's export command generates production-ready bundles
npx expo export --platform android
# Check the output bundle sizes in dist/
ls -lh dist/bundles/android-*.js
```

**Target**: Keep your JS bundle under 2MB for fast startup on low-end devices. If your bundle exceeds 4MB, investigate with source-map-explorer to find unnecessary dependencies.

### Strategy 7: Binary Search Dependencies

If the crash started after a release:

1. Run `git log --oneline` between the last stable release and the crashing one
2. Check `package.json` and `android/app/build.gradle` diffs for dependency changes
3. **Bisect**: revert half the dependency changes, build, and test. This narrows the cause in O(log n) steps instead of O(n).
4. Common culprits in order of likelihood: native module upgrades, RN version bumps, Gradle plugin updates, Android SDK target version changes

### Strategy 8: Build Variant Testing

Testing different build configurations helps isolate the crash cause:

```bash
# Test with no minification -- if crash goes away, R8 is stripping something needed
# See Category I for how to fix R8 issues
npx react-native run-android --variant=release --no-minify

# Test with Hermes disabled -- if crash goes away, it's a Hermes bug
# In android/app/build.gradle, temporarily set: hermesEnabled = false
# Then rebuild and test

# Test production-like locally with Expo
# --no-dev disables dev mode, --minify enables minification
npx expo start --no-dev --minify
```

### Strategy 9: ABI Split Analysis

Check if crashes are concentrated on specific CPU architectures:

```gradle
// In android/app/build.gradle -- enable ABI splits to generate
// separate APKs for each architecture
splits {
    abi {
        enable true
        universalApk false  // Don't generate a universal APK
        include "armeabi-v7a", "arm64-v8a", "x86", "x86_64"
    }
}
```

If crashes are only on `armeabi-v7a` (32-bit ARM), it may indicate:
- Pointer size issues in native code (32-bit vs 64-bit assumptions)
- Integer overflow in 32-bit arithmetic
- Libraries compiled without 32-bit support

If crashes are only on `arm64-v8a`, it may indicate:
- Memory alignment issues specific to ARM64
- Libraries that have not been properly tested on 64-bit

### Strategy 10: Perfetto System Tracing

For deep native performance analysis, use Perfetto (Android's system-level tracing tool):

```bash
# Record a 10-second system trace including your app's custom trace points
# Replace com.yourcompany.yourapp with your actual package name
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace.perfetto-trace \
<<EOF
buffers: {
    size_kb: 63488
    fill_policy: RING_BUFFER
}
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            atrace_categories: "view"
            atrace_categories: "gfx"
            atrace_categories: "memory"
            atrace_apps: "com.yourcompany.yourapp"
        }
    }
}
duration_ms: 10000
EOF

# Pull the trace and open in https://ui.perfetto.dev
adb pull /data/misc/perfetto-traces/trace.perfetto-trace .
```

Open the trace file at https://ui.perfetto.dev to visualize:
- Memory allocation patterns over time
- GPU rendering performance (jank frames)
- Thread scheduling and contention
- System-level events that coincide with crashes

### Strategy 11: Network Logger for Development

Add a development-only network monitor to catch API-related issues that cascade into native crashes (for example, a malformed response that causes a native JSON parser to crash):

```tsx
// Only enable in development builds to avoid performance overhead
import { NetworkLogger } from 'react-native-network-logger';

if (__DEV__) {
  // Renders a floating button to open an in-app network inspector
  // Shows all HTTP requests, responses, timing, and errors
  <NetworkLogger theme="dark" />
}
```

---

## 4. Production Fixes and Solutions

### Fix A: Background Threading for MainApplication.onCreate()

**Problem**: Too much work on main thread during app startup causes ANR, which can escalate to SIGABRT when the system decides to kill the process.

```kotlin
class MainApplication : Application() {
    // Create a fixed thread pool for background initialization
    // 3 threads is enough for most apps; increase if you have many SDKs
    private val executorService = Executors.newFixedThreadPool(3)

    override fun onCreate() {
        super.onCreate()

        // MUST stay on main thread -- SoLoader initializes native libraries
        // and React Native requires this to happen before bridge creation
        SoLoader.init(this, false)

        // MUST stay on main thread -- React Native bridge initialization
        // depends on SoLoader being ready
        initializeReactNative()

        // Move everything else to background threads
        // Each execute() call runs on a separate thread from the pool
        executorService.execute { initializeFirebase() }
        executorService.execute { initializeAnalyticsSDKs() }
        executorService.execute { performBackgroundTasks() }
    }

    private fun performBackgroundTasks() {
        // Always wrap background init in try/catch
        // A crash here would be silent and confusing to debug
        try {
            initializePushNotifications()
            preloadCachedData()
        } catch (e: Exception) {
            Log.e(TAG, "Error in background initialization", e)
        }
    }

    override fun onTerminate() {
        super.onTerminate()
        // Clean up the thread pool to avoid leaked threads
        executorService.shutdown()
    }
}
```

**Rules for what goes where**:
- `SoLoader.init()` and React Native bridge init -> **main thread** (required by React Native)
- Firebase init, analytics SDKs, messaging tokens -> **background thread** (these do disk and network I/O)
- Always wrap background tasks in try/catch -- an unhandled exception on a background thread will crash the app silently

### Fix B: Image Optimization with FastImage

**Problem**: The standard React Native `Image` component does not have proper disk caching or memory management on Android. In scrolling lists, it loads every image into memory and relies on the JS garbage collector for cleanup, which is too slow for fast scrolling.

```tsx
import FastImage from 'react-native-fast-image';

<FastImage
  source={{ uri: imageUrl }}
  // priority controls download order -- use 'low' for off-screen images
  priority={FastImage.priority.normal}
  // 'immutable' means the image at this URL will never change,
  // so it can be aggressively cached to disk
  cacheControl={FastImage.cacheControl.immutable}
  // 'cover' scales the image to fill the container, cropping if needed
  resizeMode={FastImage.resizeMode.cover}
/>
```

FastImage uses SDWebImage (iOS) and Glide (Android) under the hood. These libraries provide:
- **Disk caching**: images are saved to disk and loaded from cache on subsequent renders
- **Memory caching**: recently used images are kept in a bounded memory cache
- **Download prioritization**: visible images are downloaded before off-screen ones
- **Aggressive memory management**: the native image library handles memory pressure automatically

### Fix C: Lazy Navigation Screens

**Problem**: All screens are initialized eagerly at startup, consuming memory for screens the user may never visit.

```tsx
const Stack = createNativeStackNavigator();

const Navigation = () => (
  <Stack.Navigator
    screenOptions={{
      // lazy: true means screens are only rendered when the user
      // navigates to them, not when the navigator mounts
      lazy: true,
    }}
  >
    <Stack.Screen name="Home" component={HomeScreen} />
    <Stack.Screen name="Profile" component={ProfileScreen} />
    <Stack.Screen name="Settings" component={SettingsScreen} />
  </Stack.Navigator>
);
```

For bottom tab navigators, this is even more important because all tabs are rendered at mount time by default:

```tsx
const Tab = createBottomTabNavigator();

const TabNavigation = () => (
  <Tab.Navigator
    screenOptions={{
      lazy: true,
      // Optionally set a placeholder while the screen loads
      lazyPlaceholder: () => <LoadingSpinner />,
    }}
  >
    <Tab.Screen name="Home" component={HomeScreen} />
    <Tab.Screen name="Search" component={SearchScreen} />
    <Tab.Screen name="Profile" component={ProfileScreen} />
  </Tab.Navigator>
);
```

### Fix D: Sentry Performance Monitoring Causing SIGSEGV

**Problem**: Sentry's user interaction tracing and profiling instruments every touch event and function call at the native level, creating significant memory pressure that can trigger SIGSEGV on low-end devices.
**Reference**: [sentry-java#3653](https://github.com/getsentry/sentry-java/issues/3653)

```java
Sentry.init(options -> {
    options.setDsn("your-dsn");
    // Disable user interaction tracing -- it hooks into every touch event
    // at the native level, causing memory overhead proportional to UI complexity
    options.setEnableUserInteractionTracing(false);
    // Disable native profiling -- it samples the call stack at high frequency,
    // causing memory and CPU overhead that can push low-end devices over the edge
    options.setProfilesSampleRate(0.0);
    // If you still want error tracking (recommended), keep these enabled:
    options.setTracesSampleRate(0.1);  // 10% of transactions
});
```

### Fix E: WebView Hardware Acceleration Crash

**Problem**: `libhwui.so` crash on Android 9+ (Samsung, Huawei). The OEM-modified GPU rendering pipeline cannot handle certain WebView rendering operations.

```tsx
<WebView
  source={{ uri: url }}
  // Option 1: Disable hardware acceleration entirely for this WebView
  // This forces software rendering, which is slower but more compatible
  androidHardwareAccelerationDisabled={true}
  // Option 2: Use software layer type (similar effect, slightly different path)
  // androidLayerType="software"
/>
```

**Trade-off**: Software rendering is 2-5x slower for complex web content. If performance is critical, consider using Option 1 only on affected devices by checking `Platform.constants.Brand` or `Platform.constants.Manufacturer`.

### Fix F: Component Memoization

**Problem**: Unnecessary re-renders cause GC pressure. When a component re-renders, React creates new objects for props, children, and closures. The garbage collector must clean up the old objects, and if this happens faster than GC can keep up, memory fragments and eventually native allocations fail.

```tsx
const MediaItem = React.memo(({ media }) => {
  // Component renders only when media changes
  return (
    <View>
      <FastImage source={{ uri: media.thumbnailUrl }} />
      <Text>{media.title}</Text>
    </View>
  );
}, (prevProps, nextProps) => {
  // Custom comparison function: return true if props are EQUAL
  // (i.e., component should NOT re-render)
  // Only compare the fields that affect rendering output
  return prevProps.media.id === nextProps.media.id
    && prevProps.media.thumbnailUrl === nextProps.media.thumbnailUrl;
});
```

### Fix G: Native Driver for Animations

**Problem**: Animations running through the JS bridge cause thread contention. Every frame, the JS thread must calculate the next value, send it across the bridge, and the native thread must apply it. At 60fps, this is 16ms per frame -- any delay causes jank and bridge congestion.

```tsx
// WITHOUT native driver: JS thread -> Bridge -> Native thread -> UI
// Every frame requires a round trip across the bridge
Animated.timing(opacity, {
  toValue: 1,
  duration: 1000,
  // Without useNativeDriver, this runs on the JS thread
}).start();

// WITH native driver: Animation runs entirely on the native thread
// JS thread is free to do other work during the animation
Animated.timing(opacity, {
  toValue: 1,
  duration: 1000,
  // With useNativeDriver: true, the animation description is sent to native
  // once, and then native handles all frame updates independently
  useNativeDriver: true,
}).start();
```

**Limitation**: `useNativeDriver: true` only works with `transform` and `opacity` properties. Layout properties (`width`, `height`, `margin`, `padding`) cannot be animated on the native thread because they require the Yoga layout engine (which runs on JS thread). For layout animations, consider `react-native-reanimated` which has its own native worklet runtime.

### Fix H: Large Heap (Last Resort)

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<application
    android:largeHeap="true"
    ...>
```

**Warning**: This is a band-aid, not a fix. It increases the OOM threshold but does not solve the underlying memory issue. Using `largeHeap` can actually make the problem worse because:
- GC pauses become longer (more memory to scan)
- The app consumes more memory from the system, starving other apps
- When the crash eventually happens, it happens harder (more state to lose)

Use this only as a temporary measure while you fix the real memory issues.

---

## 5. Performance and Memory Optimization Patterns

### Library Replacements to Reduce Memory/Bundle Pressure

| Heavy Library | Lightweight Alternative | Size Impact | Performance Impact |
|--------------|----------------------|-------------|-------------------|
| Lottie | Rive | Smaller runtime (~150KB vs ~500KB) | Better frame rates; native rendering |
| Moment.js | Day.js | 2KB vs 67KB (minified) | Identical API; tree-shakeable |
| Standard `Image` | FastImage | No bundle increase (native) | Native caching via Glide/SDWebImage |
| lodash (full) | lodash-es (individual imports) | Import only what you use | Tree-shaking eliminates unused code |
| Redux for API data | React Query / TanStack Query | Similar size | Built-in caching, deduplication, background refetch |
| `FlatList` | `FlashList` | Small increase (~50KB) | 5-10x faster scroll performance (see below) |
| `react-native-svg` (for icons) | Icon fonts or PNG sprites | Significant reduction | No SVG parsing overhead |

### FlatList Optimization Checklist

`FlatList` is one of the most common sources of memory pressure in React Native apps. Each of these settings directly affects how many views are kept in memory:

- [ ] **`initialNumToRender`** (default: 10) -- Set to 5-8. This controls how many items render on first mount. Lower values mean faster initial render and less initial memory.
- [ ] **`maxToRenderPerBatch`** (default: 10) -- Set to 5-10. This controls how many items render per scroll batch. Lower values reduce jank but make scrolling feel slower.
- [ ] **`windowSize`** (default: 21) -- Set to 5-11. This is measured in viewport heights. `windowSize={5}` means 2 viewports above and 2 below the visible area are kept rendered. Lower values mean less memory but more blank space during fast scrolling.
- [ ] **`removeClippedSubviews={true}`** -- On Android, this detaches views that scroll off-screen from the native view hierarchy, freeing GPU memory. Essential for long lists.
- [ ] **Provide stable `keyExtractor`** -- Use a unique ID from your data (not array index). Unstable keys cause unnecessary unmount/remount cycles.
- [ ] **Use `getItemLayout`** if items have fixed height -- Skips measurement overhead and enables instant scroll-to-index. Returns `{ length, offset, index }`.
- [ ] **Memoize render items** with `React.memo` -- Prevents re-renders when parent state changes but item data has not.
- [ ] **Avoid inline functions in `renderItem`** -- Each render creates a new function reference, defeating `React.memo`.

### FlashList: A Modern FlatList Alternative

[FlashList](https://shopify.github.io/flash-list/) by Shopify is a drop-in replacement for `FlatList` that recycles views instead of creating/destroying them. This is the same approach used by `RecyclerView` on Android and `UICollectionView` on iOS.

```tsx
import { FlashList } from '@shopify/flash-list';

const MyList = () => (
  <FlashList
    data={items}
    renderItem={({ item }) => <ItemComponent item={item} />}
    // estimatedItemSize is REQUIRED -- FlashList uses this for layout calculations
    // Measure your actual item height and provide it here
    estimatedItemSize={80}
    // FlashList handles most optimizations automatically, but you can still tune:
    // drawDistance controls how far ahead/behind to pre-render (in pixels)
    drawDistance={250}
  />
);
```

**Why FlashList reduces native crashes**:
- **View recycling**: Instead of creating a new native view for each item and destroying it when it scrolls off-screen, FlashList reuses existing views. This dramatically reduces native memory allocations and GC pressure.
- **Consistent memory usage**: Memory usage stays flat regardless of list length, whereas FlatList memory grows with the number of items rendered.
- **Fewer bridge crossings**: Recycling means fewer create/destroy calls across the JS-native bridge, reducing thread contention.

**Migration from FlatList to FlashList**:
1. Replace `FlatList` import with `FlashList`
2. Add `estimatedItemSize` prop (measure your item height in development)
3. Remove `removeClippedSubviews` (FlashList handles this internally)
4. Remove `windowSize` (FlashList uses `drawDistance` instead)
5. Test thoroughly -- FlashList has different behavior for `onEndReached` threshold calculation

### Product Decisions That Reduce Crashes

Sometimes the best fix is a product change. These are not technical compromises; they are informed decisions that improve both stability and user experience on the majority of devices:

- **Disable autoplay for GIFs/videos** -- let users tap to play. Autoplay is the single largest contributor to graphics memory spikes.
- **Limit simultaneous media playback** -- pause off-screen videos. A single 1080p video decode uses ~50MB of GPU memory; three simultaneous decodes use ~150MB.
- **Paginate aggressively** -- use smaller page sizes (10-15 items) in infinite scroll. Fewer items in memory at any time.
- **Defer non-critical features** -- lazy-load heavy modules (maps, camera, video player) only when the user navigates to them.
- **Use thumbnails in lists** -- load full-resolution images only when the user taps on an item. A 100x100 thumbnail uses 40KB; a 1080x1920 full image uses 8MB.
- **Set maximum image dimensions** -- resize images server-side to match device screen width. A 4000x3000 image scaled down to 400x300 in CSS still uses 48MB of GPU memory for the original.

---

## 6. Crash Reporting Tool Comparison

Different crash reporting tools have different strengths for native crash visibility. This comparison focuses specifically on how well each tool handles the `libc.so` / SIGABRT / SIGSEGV crashes discussed in this guide.

| Feature | Firebase Crashlytics | Sentry | Datadog | Bugsnag |
|---------|---------------------|--------|---------|---------|
| **Native crash capture** | Excellent (NDK support built-in) | Excellent (native SDK) | Good (native SDK) | Good (native SDK) |
| **Full native stack traces** | Yes, with NDK symbol upload | Yes, with debug symbol upload | Yes, with symbol upload | Yes, with symbol upload |
| **JS + native unified view** | Partial (separate tabs) | Excellent (unified stack trace) | Good (linked views) | Good (linked views) |
| **OEM/device filtering** | Excellent (Google Play integration) | Good | Good | Good |
| **ANR detection** | Yes (Android Vitals integration) | Yes (since SDK 6.x) | Yes | Yes |
| **Memory metrics at crash time** | Limited | Good (context includes memory) | Good (memory metrics) | Limited |
| **Free tier native crashes** | Unlimited | 5K events/month | 14-day trial | 7.5K events/month |
| **Source map upload** | Manual or CI | Automated CLI/CI | Automated CLI/CI | Automated CLI/CI |
| **ProGuard mapping upload** | Automatic (Gradle plugin) | Automatic (Gradle plugin) | Manual or CI | Automatic (Gradle plugin) |
| **Real-time alerting** | Basic (Velocity alerts) | Advanced (custom rules) | Advanced (monitors) | Advanced (custom rules) |
| **Performance monitoring overhead** | Low | Medium (can cause crashes -- see Fix D) | Low-Medium | Low |

### Recommendations

- **Best for small teams / free tier**: Firebase Crashlytics -- unlimited crash reporting, excellent Android integration, zero cost
- **Best for full-stack visibility**: Sentry -- unified JS + native stack traces, excellent source map support, strong alerting
- **Best for enterprise / APM integration**: Datadog -- native crashes alongside infrastructure monitoring, logs, and APM traces
- **Best native crash detail**: Google Play Console (Android Vitals) -- always check this alongside your primary tool, as it captures information that third-party SDKs miss

**Important**: Regardless of which tool you choose, always configure:
1. **NDK / native symbol upload** -- without this, native stack traces show hex offsets instead of function names
2. **JS source map upload** -- without this, JS stack traces show minified variable names
3. **ProGuard/R8 mapping upload** -- without this, Java/Kotlin stack traces show obfuscated class names

---

## 7. Common Mistakes That Cause These Crashes

This section lists specific coding patterns and configuration mistakes that lead to SIGABRT/SIGSEGV crashes. If you are seeing native crashes, check your codebase for these patterns.

### Mistake 1: Not Cleaning Up Native Resources

```tsx
// BAD: Video player keeps running after component unmounts
const VideoScreen = () => {
  return <Video source={{ uri: videoUrl }} />;
};

// GOOD: Clean up on unmount
const VideoScreen = () => {
  const videoRef = useRef(null);

  useEffect(() => {
    return () => {
      // Release native video player resources when screen unmounts
      // Without this, the native player continues consuming GPU memory
      videoRef.current?.seek(0);
      videoRef.current?.dismissFullscreenPlayer();
    };
  }, []);

  return <Video ref={videoRef} source={{ uri: videoUrl }} />;
};
```

### Mistake 2: Animating Non-Native-Driver Properties with useNativeDriver: true

```tsx
// BAD: This will crash because 'height' cannot be animated on the native thread
Animated.timing(heightAnim, {
  toValue: 200,
  duration: 300,
  useNativeDriver: true,  // CRASH: height is a layout property
}).start();

// GOOD: Use transform.scaleY instead, or use useNativeDriver: false for layout props
Animated.timing(scaleAnim, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true,  // OK: transform properties work with native driver
}).start();
```

### Mistake 3: Multiple GestureHandlerRootView Wrappers

```tsx
// BAD: Nested GestureHandlerRootView causes native gesture system corruption
const App = () => (
  <GestureHandlerRootView>
    <NavigationContainer>
      <GestureHandlerRootView>  {/* CRASH: duplicate wrapper */}
        <MyScreen />
      </GestureHandlerRootView>
    </NavigationContainer>
  </GestureHandlerRootView>
);

// GOOD: Single wrapper at the app root
const App = () => (
  <GestureHandlerRootView style={{ flex: 1 }}>
    <NavigationContainer>
      <MyScreen />
    </NavigationContainer>
  </GestureHandlerRootView>
);
```

### Mistake 4: Forgetting to Handle Camera/Audio Permissions Before Accessing Hardware

```tsx
// BAD: Accessing camera without checking permission can crash on some OEMs
const takePhoto = async () => {
  const photo = await camera.current.takePhoto();  // CRASH on permission denied
};

// GOOD: Always check permission first
const takePhoto = async () => {
  const permission = await Camera.requestCameraPermission();
  if (permission === 'granted') {
    const photo = await camera.current.takePhoto();
  }
};
```

### Mistake 5: Using Array Index as FlatList Key

```tsx
// BAD: Array index as key causes view recycling bugs that can crash native layer
<FlatList
  data={items}
  keyExtractor={(item, index) => index.toString()}  // Unstable key
  renderItem={renderItem}
/>

// GOOD: Use a unique, stable ID from your data
<FlatList
  data={items}
  keyExtractor={(item) => item.id}  // Stable key
  renderItem={renderItem}
/>
```

### Mistake 6: Not Testing on Low-End Devices

Most developers test on flagship devices with 8-12GB RAM. The crashes happen on devices with 2-3GB RAM that represent a significant portion of your user base. Always test on:
- A device with 3GB RAM or less
- A device running Android 9 or 10 (still common in many markets)
- A Huawei/Honor device (if you have users in Asia/Europe)
- A Samsung Galaxy A-series (the most popular Android phones globally)

### Mistake 7: Ignoring Expo/RN Upgrade Warnings

```bash
# BAD: Upgrading RN and ignoring peer dependency warnings
yarn add react-native@0.76.0
# npm WARN peer react-native-reanimated@3.3.0 requires react-native@0.73.x

# GOOD: Upgrade dependencies to compatible versions FIRST
yarn add react-native-reanimated@latest react-native-screens@latest
# Then upgrade React Native
yarn add react-native@0.76.0
```

### Mistake 8: Synchronous Heavy Work in useEffect

```tsx
// BAD: Processing large datasets synchronously blocks the JS thread,
// causing bridge congestion and potentially triggering ANR -> SIGABRT
useEffect(() => {
  const processed = hugeArray.map(item => expensiveTransform(item));
  setData(processed);
}, [hugeArray]);

// GOOD: Use InteractionManager to defer heavy work until after animations complete
useEffect(() => {
  const handle = InteractionManager.runAfterInteractions(() => {
    const processed = hugeArray.map(item => expensiveTransform(item));
    setData(processed);
  });
  return () => handle.cancel();
}, [hugeArray]);
```

### Mistake 9: Not Uploading Native Debug Symbols

If you do not upload native debug symbols to your crash reporting tool, your SIGABRT stack traces will look like this:

```
#0  0x0000007a1b2c3d4e  libc.so
#1  0x0000007a2b3c4d5f  libreanimated.so   <-- You can see the library name
#2  0x0000007a3c4d5e6f  libreanimated.so   <-- But not the function name
```

Instead of this:

```
#0  libc.so (abort+164)
#1  libreanimated.so (reanimated::WorkletRuntime::execute+342)
#2  libreanimated.so (reanimated::ShareableValue::toJSValue+89)
```

The second format tells you exactly which function crashed. Always upload debug symbols.

---

## 8. Decision Framework

Use this decision tree to systematically diagnose native crashes:

```
SIGABRT/SIGSEGV crash detected
|
+-- Does the stack trace contain a specific .so file (not just libc.so)?
|   +-- YES -> Identify the library -> check Category A table -> upgrade
|   '-- NO -> Continue below
|
+-- Did crash-free rate drop across ALL app versions simultaneously?
|   +-- YES -> Likely Google Play System update (Category E) -> check Issue Tracker
|   '-- NO -> Continue below
|
+-- Are 80%+ crashes on one OEM (Huawei, Xiaomi, Samsung)?
|   +-- YES -> OEM-specific bug (Category D) -> try Hermes, disable HW accel
|   '-- NO -> Continue below
|
+-- Did crashes start after a specific release?
|   +-- YES -> Was it an RN version upgrade?
|   |   +-- YES -> Category H -> check upgrade-specific known issues table
|   |   '-- NO -> Binary search dependencies (Strategy 7)
|   '-- NO -> Continue below
|
+-- Do crashes happen ONLY in release builds?
|   +-- YES -> Likely R8 over-stripping (Category I) -> test with minifyEnabled=false
|   '-- NO -> Continue below
|
+-- Do crashes happen on app launch from push notification?
|   +-- YES -> PlatformConstants / TurboModule issue (Category B, RN 0.76+)
|   '-- NO -> Continue below
|
+-- Does memory profiling show excessive growth?
|   +-- YES -> OOM path (Category F) -> optimize images, lazy load, memoize
|   '-- NO -> Continue below
|
+-- Are crashes accompanied by ANR reports?
|   +-- YES -> MainApplication.onCreate() (Category G) -> move work to background
|   '-- NO -> Continue below
|
'-- Run with ASan (Strategy 5) -> identify specific memory corruption
```

---

## 9. Auditing Your Own App

This section provides a step-by-step checklist for auditing your React Native app against every crash category in this guide. Follow this to identify vulnerabilities before they become production crashes.

### Step 1: Audit Native Dependencies

Check each native dependency for known crash-prone versions:

```bash
# List all installed packages with versions
cat package.json | grep -E "(react-native|expo-)" | sort

# Compare against the Category A table above
# Specifically check:
#   @react-native-picker/picker  -> must be >= 2.11.3
#   react-native-reanimated      -> must be >= 3.6 (or latest)
#   react-native-screens         -> must be >= 3.20
#   react-native-gesture-handler -> must be >= 2.14
#   react-native-webview         -> check for HW accel settings
#   react-native-svg             -> check complexity of SVGs in your app
```

### Step 2: Audit MainApplication.onCreate()

Open your `MainApplication.kt` (or `.java`) and check:

```bash
# Find your MainApplication file
find android -name "MainApplication.*" -type f

# Check what runs in onCreate()
# Look for: network calls, database operations, Firebase init,
# analytics SDK init, push token retrieval, WebView setup
# Everything except SoLoader.init() and React Native bridge should
# be moved to a background thread (see Fix A)
```

**Red flags in onCreate()**:
- Any `Firebase*` initialization
- Any `Analytics*` initialization
- Any network call (`fetch`, `URL.openConnection()`, `OkHttp`)
- Any database access (`Room`, `SQLite`, `SharedPreferences.apply()` on large prefs)
- Any `WebView.setWebContentsDebuggingEnabled()` or WebView prewarming

### Step 3: Audit Animation Usage

```bash
# Search for animations without native driver
# Every instance of useNativeDriver: false is a potential bridge
# contention point during animations
grep -r "useNativeDriver.*false" src/ --include="*.tsx" --include="*.ts" -l

# For each file found, check if the animation property supports native driver:
# - opacity: YES (use native driver)
# - transform: YES (use native driver)
# - width/height/margin/padding: NO (cannot use native driver)
# If the property supports it, switch to useNativeDriver: true
```

### Step 4: Audit Navigation Configuration

```bash
# Check if lazy loading is enabled on navigators
grep -r "createNativeStackNavigator\|createBottomTabNavigator\|createDrawerNavigator" src/ --include="*.tsx" --include="*.ts" -l

# For each navigator found, check if it has lazy: true in screenOptions
# Tab navigators especially benefit from lazy loading
grep -r "lazy.*true" src/ --include="*.tsx" --include="*.ts" -l
```

### Step 5: Audit Image Usage in Lists

```bash
# Find all FlatList/SectionList/ScrollView usage
grep -r "FlatList\|SectionList\|FlashList" src/ --include="*.tsx" --include="*.ts" -l

# For each list, check:
# 1. Are images using FastImage or standard Image?
# 2. Is initialNumToRender set? (should be 5-8)
# 3. Is removeClippedSubviews={true} set?
# 4. Is windowSize set? (should be 5-11, not default 21)
# 5. Are render items memoized with React.memo?
# 6. Does keyExtractor use a stable ID (not array index)?
```

### Step 6: Audit ProGuard/R8 Configuration

```bash
# Check if minification is enabled
grep -r "minifyEnabled" android/app/build.gradle

# If minifyEnabled is true, check ProGuard rules
cat android/app/proguard-rules.pro

# Verify that React Native core classes are kept (see Category I)
# At minimum, you need keep rules for:
# - com.facebook.react.**
# - com.facebook.hermes.**
# - com.facebook.jni.**
```

### Step 7: Audit ABI Configuration

```bash
# Check if ABI splits or filtering is configured
grep -r "abi\|ndk\|abiFilters" android/app/build.gradle

# If no ABI configuration exists, you're shipping a universal APK
# that includes all architectures. Consider:
# 1. Using AAB (Android App Bundle) which serves per-device APKs
# 2. Enabling ABI splits for APK distribution
```

### Step 8: Audit Crash Reporting Configuration

```bash
# Check for crash reporting SDK setup
grep -r "crashlytics\|sentry\|datadog\|bugsnag" android/app/build.gradle package.json

# Verify NDK crash reporting is enabled
# For Crashlytics: firebaseCrashlytics { nativeSymbolUploadEnabled true }
# For Sentry: check sentry.properties for native symbol upload
# For Datadog: check for native crash reporting configuration

# Verify source maps are uploaded in your CI/CD pipeline
# Check your CI config (e.g., .github/workflows/*.yml, Fastfile, etc.)
grep -r "sourcemap\|source-map\|upload-symbols" .github/ fastlane/ 2>/dev/null
```

### Step 9: Memory Profile on a Low-End Device

This cannot be automated. You need to:

1. Get a physical device with 3GB RAM or less (or use Android Studio's emulator with restricted RAM)
2. Open Android Studio Profiler
3. Run through your app's critical flows: login, main feed scroll, media viewing, navigation between tabs
4. Record total memory at each step
5. If total memory exceeds 600MB at any point, you have a potential OOM crash on low-end devices

### Step 10: Review Crash Dashboard

Pull your crash data and look for patterns:

1. **Group by crash signature** -- which crashes are most frequent?
2. **Group by device manufacturer** -- any OEM concentration?
3. **Group by app version** -- did any release introduce new crashes?
4. **Group by OS version** -- any Android version concentration?
5. **Check ANR rate alongside crash rate** -- are they correlated?
6. **Look at the "Crashes and ANRs" section in Google Play Console** -- it captures different data than third-party tools

### Audit Summary Template

After completing the audit, fill in this template to track your findings:

```markdown
## Native Crash Audit Results - [Date]

### Dependencies
- [ ] All native deps at safe versions (Category A table)
- [ ] No known-problematic library versions

### Application Startup
- [ ] MainApplication.onCreate() only has SoLoader + RN bridge on main thread
- [ ] All SDK init moved to background threads

### Animations
- [ ] X files with useNativeDriver: false (list them)
- [ ] Each reviewed for native driver compatibility

### Navigation
- [ ] lazy: true set on all stack navigators
- [ ] lazy: true set on all tab navigators

### Lists & Images
- [ ] FlatList optimization props set (or using FlashList)
- [ ] Images use FastImage or equivalent native caching
- [ ] No array index used as keyExtractor

### Build Configuration
- [ ] ProGuard/R8 rules include React Native keep rules
- [ ] ABI splits or AAB configured
- [ ] NDK crash reporting enabled
- [ ] Source maps uploaded in CI

### Memory
- [ ] Profiled on low-end device
- [ ] Peak memory under 600MB during normal usage

### Crash Dashboard
- [ ] No OEM-concentrated crashes
- [ ] No version-specific crash spikes
- [ ] ANR rate under 0.5%
```

---

## 10. Sources

### GitHub Issues

- [facebook/react-native#47074](https://github.com/facebook/react-native/issues/47074) -- SIGABRT crash (RN 0.76+, TurboModuleRegistry)
- [facebook/react-native#47592](https://github.com/facebook/react-native/issues/47592) -- PlatformConstants crash on push notification cold start
- [facebook/react-native#29978](https://github.com/facebook/react-native/issues/29978) -- SIGABRT on release build, libc.so (Hermes)
- [facebook/react-native#33083](https://github.com/facebook/react-native/issues/33083) -- libc.so crash (86% Huawei devices)
- [facebook/react-native#31235](https://github.com/facebook/react-native/issues/31235) -- SIGABRT on Huawei devices
- [facebook/react-native#39505](https://github.com/facebook/react-native/issues/39505) -- Google Play System update crash (Sept 2023)
- [facebook/react-native#36287](https://github.com/facebook/react-native/issues/36287) -- SIGSEGV in Hermes GC on low-memory devices
- [facebook/hermes#298](https://github.com/facebook/hermes/issues/298) -- Hermes engine SIGABRT
- [expo/expo#34570](https://github.com/expo/expo/issues/34570) -- SIGABRT on release build (Expo)
- [invertase/react-native-firebase#8131](https://github.com/invertase/react-native-firebase/issues/8131) -- RN 0.76+ notification crash
- [getsentry/sentry-java#2955](https://github.com/getsentry/sentry-java/issues/2955) -- Stack corruption fix
- [getsentry/sentry-java#3653](https://github.com/getsentry/sentry-java/issues/3653) -- SIGSEGV from performance monitoring
- [software-mansion/react-native-screens#1646](https://github.com/software-mansion/react-native-screens/issues/1646) -- Fragment transaction crash
- [software-mansion/react-native-gesture-handler#2546](https://github.com/software-mansion/react-native-gesture-handler/issues/2546) -- Duplicate root view crash
- [software-mansion/react-native-reanimated#4loading](https://github.com/software-mansion/react-native-reanimated/issues) -- Various worklet runtime crashes
- [Shopify/flash-list](https://github.com/Shopify/flash-list) -- FlashList as FlatList alternative

### Stack Overflow

- [libc crash on React Native Android](https://stackoverflow.com/questions/74577782/libc-crash-on-react-native-android)
- [React Native Android libc.so crash on first open](https://stackoverflow.com/questions/72092407/react-native-android-libc-so-crash-on-first-open-after-installation)
- [React Native app crashes/aborts right on start](https://stackoverflow.com/questions/77017531/react-native-app-crashes-aborts-right-on-start)
- [React Native SIGABRT after upgrading to 0.73](https://stackoverflow.com/questions/77432551/react-native-sigabrt-after-upgrading)
- [ProGuard rules for React Native](https://stackoverflow.com/questions/46generators/proguard-rules-react-native)

### Articles and Documentation

- [Breaking Down ANRs: Real-World Crashes and Their Solutions](https://medium.com/@omkarsalapurkar2002/breaking-down-anrs-real-world-crashes-and-their-solutions-b871ef8ec287) -- Omkar Salapurkar (Nov 2024)
- [Understanding and Analysing React Native ANRs](https://medium.com/@omkarsalapurkar2002/understanding-and-analysing-react-native-anrs-a-developers-guide-ed3aa277b5df) -- Omkar Salapurkar (Nov 2024)
- [Diagnose Native Crashes -- Android AOSP](https://source.android.com/docs/core/tests/debug/native-crash)
- [Android Memory Management Overview](https://developer.android.com/topic/performance/memory-overview) -- Android Developers
- [React Native Performance Overview](https://reactnative.dev/docs/performance) -- React Native Documentation
- [React Native New Architecture](https://reactnative.dev/docs/the-new-architecture/landing-page) -- React Native Documentation
- [Shopify FlashList Documentation](https://shopify.github.io/flash-list/) -- Shopify Engineering
- [R8 Shrinking and Obfuscation](https://developer.android.com/build/shrink-code) -- Android Developers
- [Perfetto System Tracing](https://perfetto.dev/docs/) -- Perfetto Documentation
- [AddressSanitizer for Android](https://developer.android.com/ndk/guides/asan) -- Android NDK Documentation
- [Firebase Crashlytics NDK Crash Reporting](https://firebase.google.com/docs/crashlytics/ndk-reports) -- Firebase Documentation
- [Sentry React Native SDK](https://docs.sentry.io/platforms/react-native/) -- Sentry Documentation

### Key Patterns Across All Sources

| Pattern | Frequency | Typical Devices |
|---------|-----------|-----------------|
| Production-only (not reproducible locally) | ~95% | All |
| App startup crash | ~60% | All |
| Android 14 / Samsung | Very high | Samsung Galaxy series |
| Huawei / Honor | High | Especially JSC crashes |
| Xiaomi / Redmi / OPPO / Vivo | Moderate | Budget Android devices |
| Push notification launch crash | ~15% | All (RN 0.76+) |
| After RN version upgrade | ~40% | All |
| Release-only (R8 stripping) | ~10% | All |
| OOM on low-end devices | ~30% | Devices with less than 3GB RAM |

---

*This guide is maintained as a living document. Contributions and corrections are welcome.*

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Native-Layer Debugging](./native-layer-debugging.md) | Complementary guide — use this for profiling tools (Memory Profiler, CPU Profiler, Perfetto, ASan) to investigate crashes identified here |
| [Profiling Tools Deep Dive](./profiling-tools-deep-dive.md) | When crash analysis points to memory or CPU issues — comprehensive tool selection framework |
| [Architecture & Lifecycle](../foundation/architecture-lifecycle.md) | Understanding the threading model and memory model that underlie the crash patterns described here |
| [Monitoring & ANR Analysis](../optimization/monitoring-anr-analysis.md) | Production monitoring setup that surfaces the crashes this guide helps you debug |
