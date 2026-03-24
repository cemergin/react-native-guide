[← Back to Index](../README.md)

# Native-Layer Debugging & Optimization Guide for React Native / Expo

> **Purpose**: Go deeper than the JavaScript/Expo framework level. This guide covers tools and techniques in Android Studio, Xcode, and cross-platform native profilers for diagnosing crashes, memory leaks, CPU bottlenecks, and startup performance at the native layer.

---

<details>
<summary><strong>TL;DR</strong></summary>

- Android: Memory Profiler + CPU Profiler (Android Studio), Perfetto for system traces, heapprofd for native heap
- iOS: Instruments (Time Profiler, Leaks, Allocations), Memory Graph Debugger for retain cycles
- Perfetto is best for cold start analysis — shows every thread and process during startup
- ASan/TSan/UBSan catch memory corruption and threading bugs in native code
- Flipper is dead (deprecated RN 0.73+) — use React DevTools + platform-native tools

</details>

## Table of Contents

1. [Getting Started](#1-getting-started)
2. [Why Go Native?](#2-why-go-native)
3. [Android Studio Debugging](#3-android-studio-debugging)
4. [Xcode Debugging (iOS)](#4-xcode-debugging-ios)
5. [Cross-Platform Native Techniques](#5-cross-platform-native-techniques)
6. [App Optimization at Native Level](#6-app-optimization-at-native-level)
7. [Troubleshooting Common Issues](#7-troubleshooting-common-issues)
8. [Tool Summary Matrix](#8-tool-summary-matrix)
9. [Recommended Reading](#9-recommended-reading)

---

## 1. Getting Started

Before diving into native-layer debugging, you need to have the right environment set up. Native profiling tools are powerful but demand specific prerequisites that differ from typical JavaScript development.

### 1.1 Prerequisites

| Requirement | Android | iOS |
|------------|---------|-----|
| **IDE** | Android Studio (Ladybug 2024.2+ recommended) | Xcode 15+ |
| **Device** | Physical device or emulator with Google APIs | Physical device (many Instruments templates require hardware) |
| **SDK** | Android SDK + NDK installed via SDK Manager | Xcode Command Line Tools installed |
| **Build Type** | Debug builds for most profiling; Release for accurate perf data | Debug for memory tools; Release (via `Cmd+I`) for CPU profiling |

### 1.2 Android Studio Setup

1. **Install Android Studio** from [developer.android.com/studio](https://developer.android.com/studio)
2. **Install the NDK**: Open SDK Manager (`Tools > SDK Manager > SDK Tools` tab) and check "NDK (Side by side)" and "CMake"
3. **Enable USB Debugging** on your device: `Settings > Developer Options > USB Debugging`
4. **Verify the connection**:
   ```bash
   adb devices
   # Should list your device
   ```
5. **Open the native project**: From your React Native project root:
   ```bash
   open android -a "Android Studio"
   ```
   Wait for Gradle sync to complete before attempting to profile.

### 1.3 Xcode Setup

1. **Install Xcode** from the Mac App Store
2. **Install Command Line Tools**: `xcode-select --install`
3. **Register your device**: Connect via USB, trust the computer on the device, then verify in `Window > Devices and Simulators`
4. **Open the native project**:
   ```bash
   open ios/*.xcworkspace
   ```
   Always open the `.xcworkspace` (not `.xcodeproj`) to ensure CocoaPods dependencies are included.

### 1.4 Debug vs Release Builds

Understanding when to use each build type is critical for accurate profiling:

| Scenario | Build Type | Why |
|----------|-----------|-----|
| Memory leak detection | Debug | Full symbol info, sanitizers available |
| CPU profiling | Release | Debug builds include extra checks that distort timings |
| Crash symbolication | Release + symbols | Matches production crash conditions |
| Layout debugging | Debug | Layout Inspector needs debuggable app |
| Startup optimization | Release | JIT, ProGuard, and bytecode compilation only active in release |

**For Android release profiling**, add this to your `AndroidManifest.xml`:
```xml
<application android:debuggable="false" ...>
    <!-- Allows profiling release builds without debuggable flag -->
    <profileable android:shell="true" />
</application>
```

**For iOS release profiling**, use Xcode's `Product > Profile` (Cmd+I) which automatically builds a Release configuration optimized for Instruments.

### 1.5 Expo Managed Workflow

If you use Expo's managed workflow, you need to "prebuild" to access native projects:

```bash
# Generate native directories
npx expo prebuild

# Verbose mode to debug config plugin issues
EXPO_DEBUG=1 npx expo prebuild
```

Once prebuilt, you have full access to all Android Studio and Xcode debugging tools described in this guide. Note that `npx expo prebuild` is non-destructive if native directories already exist — it will apply config plugins on top of existing code.

---

## 2. Why Go Native?

React Native runs JavaScript on a native runtime (Hermes or JSC). When crashes happen in `libc.so`, `libhermes.so`, or `libhwui.so`, JS-level tools like React DevTools, Flipper (deprecated), or console.log are **blind** to the problem. You need native tooling.

### When JS-Level Tools Are Insufficient

| Symptom | JS Tools See | Native Tools Reveal |
|---------|-------------|-------------------|
| SIGABRT / SIGSEGV crash | Nothing (app just dies) | Exact memory address, corrupted pointer, offending `.so` |
| Gradual memory growth | "RAM usage is high" | Which native allocations (Fresco bitmaps, Hermes heap, JNI refs) are leaking |
| Jank during scroll | "JS FPS dropped" | Whether it's the UI thread, Render thread, or GPU pipeline stalling |
| Slow cold start | "App took 3s to load" | Which native init step (SoLoader, Hermes bundle parse, native module init) is the bottleneck |
| Battery drain | Nothing | Which threads are CPU-hot, which syscalls are frequent |

The key insight is that React Native is a **multi-layer system**: JavaScript, the bridge/JSI layer, and the native platform. A problem at any layer can manifest as symptoms at another layer. Native tools let you see the full picture.

---

## 3. Android Studio Debugging

### 3.1 Memory Profiler — Native Heap Analysis

**When to use**: Investigating OOM, SIGABRT, or steadily growing memory.

**Why this matters**: React Native apps on Android have a complex memory landscape. The JavaScript heap (managed by Hermes) coexists with the Java/Kotlin heap (managed by ART), native C++ allocations (from the RN runtime, image libraries, etc.), and GPU memory (for textures and framebuffers). An OOM can originate from any of these pools, and the Android Memory Profiler is the only tool that shows them all simultaneously.

**Step-by-step walkthrough**:

1. **Open the Profiler**: In Android Studio, go to `View > Tool Windows > Profiler`. If the Profiler tab is already visible at the bottom, click it.
2. **Start a profiling session**: Click the `+` button in the Profiler window. Select your connected device and your running app process. If your app is not listed, ensure it is built with `debuggable=true` or has `<profileable android:shell="true" />`.
3. **Navigate to Memory**: The Profiler shows a timeline with CPU, Memory, Network, and Energy rows. Click on the **Memory** row to expand the detailed memory view.
4. **Choose the recording configuration**: Click the dropdown near the record button. Select **"Profiler: Run app as debuggable (complete data)"** for the most comprehensive capture. This mode enables full allocation tracking but adds overhead — do not use it for CPU timing benchmarks.
5. **Read the memory timeline**: The timeline shows a stacked area chart:
   - **Blue (Java)** — objects on the ART heap
   - **Yellow (Native)** — C/C++ allocations via malloc/new
   - **Green (Graphics)** — GPU buffers and texture memory
   - **Purple (Code)** — memory used by loaded .so libraries and DEX files
   - **Gray (Stack/Others)** — thread stacks and miscellaneous

**Memory breakdown structure**:
```
Total Memory Breakdown:
├─ Native    → C/C++ allocations (Hermes heap, Fresco, native modules)
├─ Graphics  → GPU textures, bitmaps being rendered
├─ Java      → JVM heap (React Native bridge objects, JNI refs)
├─ Code      → Loaded .so libraries, DEX code
├─ Stack     → Thread stacks
└─ Others    → Miscellaneous
```

**Red flags**:
- **Graphics > 300MB** — images/videos not being recycled (see Fresco/Glide cache config)
- **Native growing without plateau** — native memory leak (likely in a native module or image library)
- **Java heap fragmented** — excessive JNI local references not being cleaned up

#### Interpreting the Allocation Tracking Timeline

When you enable allocation tracking (by clicking the "Record" button in the Memory profiler):

1. **Allocation count vs. size**: A high allocation count with small sizes indicates churn (many short-lived objects). This causes GC pressure. A low count with large sizes suggests bitmap or buffer allocations.
2. **GC events**: Vertical dashed lines on the timeline represent garbage collection events. Frequent GC events during scroll or animation correlate with jank — the GC pauses all threads briefly.
3. **Allocation site attribution**: Click any point on the timeline, then use the "Arrange by callstack" mode to see which functions are allocating. Sort by "Shallow Size" to find the largest individual allocations, or by "Retained Size" to find the allocations holding the most downstream memory.

#### Identifying Fresco Bitmap Cache Issues

Fresco (the default image library used by React Native on Android) manages its own bitmap cache, which operates in native memory (not the Java heap). This means Java heap dumps will not show Fresco bitmaps — you must look at the **Native** memory category.

**Signs of Fresco cache issues**:
- Native memory grows proportionally with the number of images displayed
- Native memory does not decrease after navigating away from image-heavy screens
- The app crashes with SIGABRT and the tombstone references `libhwui.so` or Fresco-related frames

**Diagnosing**:
1. In the Memory Profiler, filter native allocations by the calling library. Look for allocations from `libfbjni.so` or `libimagepipeline.so`.
2. Use `adb shell dumpsys meminfo <your-package-name>` to see a summary that includes "Graphics" memory — Fresco bitmaps rendered via hardware acceleration will appear here.
3. Check your Fresco configuration: if `setDownsampleEnabled(true)` is not set, Fresco may be decoding images at their full resolution regardless of the ImageView size.

#### Detecting JNI Reference Leaks

JNI (Java Native Interface) references allow native code to hold references to Java objects. If native code creates a local reference and never deletes it (or creates a global reference and never releases it), the referenced Java object cannot be garbage collected.

**Symptoms**:
- Java heap grows steadily even though Java-side profiling shows no obvious leaks
- `JNI ERROR: local reference table overflow` in logcat (hard limit of 512 local refs per frame)
- Crash with `java.lang.OutOfMemoryError` despite apparently low Java heap usage

**Detection**:
1. Enable JNI reference checking: `adb shell setprop debug.checkjni 1` and restart the app
2. Watch logcat for JNI warnings: `adb logcat | grep -i "JNI"`
3. In Android Studio's Memory Profiler, take heap dumps before and after an action. Filter by "JNI Global References" in the heap dump viewer to see which objects are held by native code.

**Heap dump comparison technique**:
1. Take Heap Dump A (before the action)
2. Perform the suspect action (scroll, navigate, etc.)
3. Take Heap Dump B (after)
4. Compare: sort by "Retained Size" descending to identify what grew

### 3.2 CPU Profiler — Finding Native Hotspots

**When to use**: Identifying which native functions consume CPU during jank or ANR.

**Why this matters**: In React Native, jank can originate from any of several threads. The JS thread might be computing an expensive layout, the native modules thread might be doing synchronous I/O, or the UI thread might be blocked on a synchronous bridge call. The CPU Profiler lets you see exactly which thread is busy and which function is responsible.

**Steps**:
1. Profiler > CPU section
2. Choose **"Find CPU Hotspots (Java/Kotlin Method Recording)"** for native method profiling
3. Choose **"System Trace Recording"** for thread-level timing across all threads

**Critical threads to monitor in React Native**:
| Thread Name | Purpose | If Blocked... |
|------------|---------|---------------|
| `main` (UI thread) | UI rendering, touch dispatch | ANR, frozen UI |
| `mqt_js` or `mqt_v_js` | JavaScript execution (Hermes/JSC) | JS callbacks delayed |
| `mqt_native_modules` | Native module calls | Native API calls hang |
| `RenderThread` | GPU command submission (Android 5.0+) | Visual jank, dropped frames |
| `create_react_co` | RN context initialization | Slow cold start |

### 3.3 Perfetto — Deep System Tracing

**When to use**: Analyzing app startup performance, thread scheduling, or system-level bottlenecks.

**Why this matters**: Perfetto captures OS-level scheduling events, showing not just what your app is doing but what the entire system is doing. This is invaluable for diagnosing startup issues where your app is being preempted by other processes, or for understanding why a thread is blocked (waiting for I/O, waiting for a lock, waiting to be scheduled by the kernel).

**Setup**:
```xml
<!-- Add to AndroidManifest.xml for release profiling -->
<profileable android:shell="true" />
```

**Capture a trace**:
```bash
# Automated trace capture
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace.pftrace \
  <<EOF
buffers: { size_kb: 63488 fill_policy: RING_BUFFER }
data_sources: { config { name: "linux.process_stats" } }
data_sources: { config { name: "linux.ftrace" target_buffer: 0
  ftrace_config {
    ftrace_events: "sched/sched_switch"
    ftrace_events: "power/suspend_resume"
    atrace_categories: "am" atrace_categories: "wm"
    atrace_categories: "view" atrace_categories: "gfx"
    atrace_apps: "com.yourcompany.yourapp"
  }
} }
duration_ms: 10000
EOF

# Pull and open in Perfetto UI
adb pull /data/misc/perfetto-traces/trace.pftrace .
# Open at https://ui.perfetto.dev
```

**What to look for in startup trace**:
- Main thread: `bindApplication` > `onCreate` > first `Choreographer#doFrame`
- `create_react_co`: How long RN context init takes
- `mqt_js`: When JS bundle parsing starts/ends
- Gap between `loadReactNative()` and first JS frame = native overhead

### 3.4 Heapprofd — Native Heap Profiling

**When to use**: Tracking C/C++ malloc/free to find native memory leaks. Requires Android 10+.

**Why this matters**: The Android Studio Memory Profiler shows you *that* native memory is growing, but heapprofd tells you *exactly which C/C++ function* allocated it. This is essential for diagnosing leaks in native modules, the Hermes engine, or image processing libraries where the allocation happens far below the Java layer.

**Steps**:
1. Open [Perfetto UI](https://ui.perfetto.dev) > Record new trace
2. Enable **Memory probe > Native Heap Profiling**
3. Set target app: `com.yourcompany.yourapp`
4. Sampling interval: 4096 bytes (default)
5. Visualize as flamegraphs — each diamond = allocation snapshot

This hooks into `malloc`/`free` and C++ `new`/`delete`, showing exactly which native code allocates memory.

### 3.5 ndk-stack — Symbolicate Native Crash Stacks

**When to use**: Converting raw memory addresses from crash logs into source file + line numbers.

**Why this matters**: When a native crash occurs, logcat shows a stack trace with raw memory addresses like `#00 pc 0x0012a4f0 /data/app/.../lib/arm64/libhermes.so`. Without symbolication, these addresses are meaningless. `ndk-stack` maps them back to source file names and line numbers using the debug symbols from your build.

```bash
# From logcat
adb logcat | $ANDROID_HOME/ndk/<version>/ndk-stack -sym ./android/app/build/intermediates/merged_native_libs/release/out/lib/arm64-v8a

# From a tombstone file
$ANDROID_HOME/ndk/<version>/ndk-stack \
  -sym ./path_to_symbols/arm64-v8a \
  < tombstone_00.txt
```

**Where to find symbols**:
- Debug builds: `android/app/build/intermediates/merged_native_libs/`
- EAS builds: download from Expo dashboard or configure symbol upload in `eas.json`

### 3.6 Layout Inspector

**When to use**: Debugging view hierarchy issues, overdraw, or unexpected layout behavior at the native level.

**Steps**: Running Devices window > Toggle Layout Inspector

Shows the actual native view tree (not the React component tree) — useful for spotting excessive nesting that causes render overhead. In React Native, the virtual DOM translates into real native views, and sometimes the nesting goes deeper than expected (wrapper views for overflow handling, accessibility, etc.).

### 3.7 Logcat Filters for Native Crashes

```bash
# Fatal libc messages
adb logcat -s libc:F

# All crash-related signals
adb logcat | grep -E "(SIGABRT|SIGSEGV|Fatal signal|signal 6|signal 11|abort|tombstone)"

# Full debuggerd output
adb logcat -s DEBUG:*

# React Native specific
adb logcat "*:S" ReactNative:V ReactNativeJS:V
```

### 3.8 StrictMode for Native Development

**When to use**: Detecting performance anti-patterns during development — disk reads/writes on the main thread, network operations on the main thread, and resource leaks.

**Why this matters**: StrictMode is a developer tool that raises violations when your app does something slow on the main thread. In React Native, native modules often perform I/O synchronously without the developer realizing it. StrictMode surfaces these issues early, before they cause ANRs in production.

#### Enabling StrictMode

Add this to your `MainApplication.java` or `MainApplication.kt` (inside `onCreate`):

```java
// Java
import android.os.StrictMode;

@Override
public void onCreate() {
    super.onCreate();

    if (BuildConfig.DEBUG) {
        StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
            .detectDiskReads()       // Disk reads on main thread
            .detectDiskWrites()      // Disk writes on main thread
            .detectNetwork()         // Network on main thread
            .detectCustomSlowCalls() // Custom slow calls
            .penaltyLog()            // Log to logcat
            .build());

        StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
            .detectLeakedClosableObjects()  // Unclosed streams, cursors, etc.
            .detectLeakedSqlLiteObjects()   // Unclosed SQLite objects
            .detectActivityLeaks()          // Activities not garbage collected
            .penaltyLog()
            .build());
    }
}
```

```kotlin
// Kotlin
import android.os.StrictMode

override fun onCreate() {
    super.onCreate()

    if (BuildConfig.DEBUG) {
        StrictMode.setThreadPolicy(
            StrictMode.ThreadPolicy.Builder()
                .detectDiskReads()
                .detectDiskWrites()
                .detectNetwork()
                .detectCustomSlowCalls()
                .penaltyLog()
                .build()
        )

        StrictMode.setVmPolicy(
            StrictMode.VmPolicy.Builder()
                .detectLeakedClosableObjects()
                .detectLeakedSqlLiteObjects()
                .detectActivityLeaks()
                .penaltyLog()
                .build()
        )
    }
}
```

#### Penalty Configurations

StrictMode supports several penalty options — choose based on your workflow:

| Penalty | Effect | Best For |
|---------|--------|----------|
| `penaltyLog()` | Logs violation to logcat | Day-to-day development |
| `penaltyDeath()` | Crashes the app on violation | Strict CI enforcement |
| `penaltyDeathOnNetwork()` | Crashes only on network violations | Preventing the most severe anti-pattern |
| `penaltyDropBox()` | Records violation in DropBoxManager | Automated collection |
| `penaltyFlashScreen()` | Flashes screen red on violation | Visual feedback during manual testing |
| `penaltyDialog()` | Shows dialog on violation | Alerting non-developer testers |

#### Detecting Disk I/O on Main Thread

Filter logcat for StrictMode violations:
```bash
adb logcat | grep -E "StrictMode"
```

A typical violation looks like:
```
D/StrictMode: StrictMode policy violation: android.os.StrictMode$StrictModeDiskReadViolation
    at android.app.SharedPreferencesImpl.getValue(SharedPreferencesImpl.java:...)
    at com.facebook.react.modules.storage.AsyncLocalStorageUtil.getItemImpl(...)
```

This tells you that a native module is doing a synchronous disk read on the main thread — common with `AsyncStorage` on older React Native versions.

#### Common React Native StrictMode Violations

- **SharedPreferences reads** on main thread (from native modules that cache config)
- **SQLite queries** on main thread (from storage libraries)
- **File system access** on main thread (from asset loading or caching)
- **Network calls** on main thread (rare but devastating — causes ANR instantly)

**Note**: StrictMode should only be enabled in debug builds. The `if (BuildConfig.DEBUG)` guard ensures it is stripped from release builds. Never ship StrictMode to production — the detection overhead is significant.

---

## 4. Xcode Debugging (iOS)

### 4.1 Instruments — Time Profiler

**When to use**: Identifying CPU-heavy native functions when JS profiler shows nothing but UI still janks.

**Why this matters**: The Time Profiler samples the call stack at regular intervals (typically 1ms) across all threads. This gives you a statistical view of where CPU time is spent. Unlike tracing profilers, it has very low overhead, making it suitable for profiling real user flows without significantly altering app behavior.

**Steps**:
1. `Product > Profile` (Cmd+I)
2. Select **Time Profiler** template
3. Select physical device + app
4. Record > perform the suspect action > stop
5. Option+click chevrons to expand call stacks

**Pro tip**: Build in Release mode for accurate profiling:
```bash
npx react-native run-ios --configuration Release
```

#### Reading Instruments Call Trees Effectively

The call tree in Instruments can be overwhelming. Here is how to navigate it efficiently:

1. **Invert the call tree**: Check "Invert Call Tree" in the bottom-left panel. This shows you the *leaf functions* (where time was actually spent) at the top, rather than making you drill down from `main()`. This is almost always what you want.

2. **Hide system libraries**: Check "Hide System Libraries" to filter out Apple framework code and focus on your app's code and React Native internals.

3. **Focus on a time range**: Click and drag on the timeline to select a specific time range (e.g., during a scroll or transition). The call tree updates to show only samples from that range.

4. **Use the "Weight" column**: Sort by "Weight" (descending) to find the functions that consumed the most CPU time. "Self Weight" is time spent in the function itself; "Weight" includes time in functions it called.

5. **Look for React Native patterns**:
   - `RCTBridge` methods — bridge communication overhead
   - `facebook::hermes` — JavaScript execution
   - `facebook::react` — React Native runtime
   - `YGNodeCalculateLayout` — Yoga layout computation
   - Methods in your own native modules

6. **Source code navigation**: Double-click any function to jump to the source code with per-line CPU time annotations. This shows exactly which lines are hot.

### 4.2 Instruments — App Launch Template

**When to use**: Analyzing cold start performance — understanding exactly what happens between the user tapping your app icon and seeing the first meaningful content.

**Why this matters**: Cold start is one of the most impactful performance metrics for user retention. On iOS, the launch sequence involves dyld (dynamic linker), runtime initialization, `UIApplicationMain`, and then your app's own initialization. For React Native, this includes loading the Hermes bytecode bundle and initializing native modules. The App Launch template captures all of this with minimal overhead.

**Steps**:
1. `Product > Profile` (Cmd+I)
2. Select **App Launch** template
3. Select your physical device
4. Click Record — Instruments will terminate the app (if running), then relaunch it
5. The recording automatically stops after a few seconds post-launch

**Interpreting the results**:

The App Launch template divides the launch into phases:

| Phase | What Happens | Optimization Targets |
|-------|-------------|---------------------|
| **Process Creation** | OS creates the process, maps memory | Reduce binary size, minimize embedded frameworks |
| **dyld** | Dynamic linker loads libraries, resolves symbols | Reduce number of dynamic frameworks, use static linking |
| **Static Initialization** | `+load` methods, `__attribute__((constructor))` functions | Defer static initialization, avoid `+load` in Objective-C |
| **UIKit Initialization** | `UIApplicationMain`, storyboard loading | Minimize storyboard complexity, lazy-load view controllers |
| **App Init** | `application:didFinishLaunchingWithOptions:` | Defer non-essential work, initialize React Native asynchronously |
| **First Frame** | First `CATransaction` commit, first meaningful paint | Optimize JS bundle size, minimize synchronous native module init |

**React Native specific considerations**:
- Look for the gap between `didFinishLaunchingWithOptions` and the first React Native view render — this is your RN initialization overhead
- Check if native modules are doing synchronous work in their `init` methods
- Verify that Hermes bytecode is being used (bytecode loads faster than parsing raw JS)

### 4.3 Instruments — Leaks & Allocations

**When to use**: Finding native memory leaks, retain cycles, or zombie objects.

**Steps**:
1. `Product > Profile` > **Leaks** instrument template
2. Allocations View shows memory over time
3. Leaks View auto-detects leaked objects with allocation stack traces
4. Use **"Generations"** mode: mark snapshots, compare object growth between actions

**Common React Native iOS leaks**:
- `AppState` / `Keyboard` listeners not cleaned up in `useEffect` return
- Native modules retaining JS callback references
- `Timer` / `setInterval` not cleared on unmount

### 4.4 Memory Graph Debugger

**When to use**: Visualizing retain cycles between native objects.

**Why this matters**: Retain cycles are the most common cause of memory leaks in Objective-C and Swift code. A retain cycle occurs when two (or more) objects hold strong references to each other, preventing either from being deallocated. The Memory Graph Debugger visualizes all object references as a graph, making cycles easy to spot.

**Steps**: `Debug > Debug Workflow > View Memory Graph Hierarchy`

- Purple squares with `!` = suspected retain cycles
- "Islands" disconnected from main graph = likely leaking
- Trace reference chains to find who's holding what

### 4.5 Sanitizers (ASan, TSan, UBSan)

**When to use**: Detecting memory corruption, data races, and undefined behavior in native code.

**Enable**: `Edit Scheme > Run > Diagnostics`:
- **Address Sanitizer (ASan)**: use-after-free, buffer overflows, double-free
- **Thread Sanitizer (TSan)**: data races between threads
- **Main Thread Checker (MTC)**: UI calls from background threads (very common in RN!)
- **Undefined Behavior Sanitizer (UBSan)**: C/C++ undefined behavior

**Important**: ASan and TSan **cannot** be used simultaneously. Run separate passes.

**Known RN issue**: heap-use-after-free detected during `RCTBridge` deallocation ([react-native#48805](https://github.com/facebook/react-native/issues/48805)).

### 4.6 Zombie Objects

**When to use**: Catching messages sent to deallocated Objective-C objects.

**Enable**: `Edit Scheme > Run > Diagnostics > Enable Zombie Objects`

In React Native, "zombie" objects happen when native modules retain references to deallocated bridge objects. Instead of crashing with `EXC_BAD_ACCESS`, zombie detection converts the crash into a descriptive error message that tells you which deallocated object received a message and what message it was.

### 4.7 dSYM Symbolication

**When to use**: Making crash reports from your crash reporting tool readable.

**Setup**:
1. Build Settings > Debug Information Format > **"DWARF with dSYM File"** (all build types)
2. For EAS builds: configure dSYM upload in `eas.json`

**Manual symbolication**:
```bash
atos -o MyApp.app/MyApp -l <load_address> <stack_address>
```

**Finding dSYMs for archived builds**:
```bash
# Find dSYMs in the Xcode archive
mdfind -name ".dSYM" | grep "MyApp"

# Or in the derived data directory
find ~/Library/Developer/Xcode/DerivedData -name "*.dSYM" -path "*MyApp*"
```

### 4.8 Metal System Trace (GPU)

**When to use**: Diagnosing GPU-related stutters, dropped frames during animations or video playback.

**Steps**: Instruments > **Metal System Trace** template

Shows parallel CPU/GPU timelines — identifies when `DrawFrame` crosses frame boundaries waiting for GPU command buffer drain. This is particularly useful for React Native apps that use heavy animations (Reanimated), video players, or OpenGL/Metal-based native modules.

### 4.9 Network Link Conditioner

**When to use**: Testing your app's behavior under poor network conditions — slow connections, packet loss, high latency.

**Why this matters**: Many native crashes and ANRs only manifest under poor network conditions. Network timeouts can cascade into unexpected native module states, and slow responses can cause race conditions that never appear on fast WiFi.

**Setup on macOS** (for Simulator):
1. Install "Additional Tools for Xcode" from [Apple Developer Downloads](https://developer.apple.com/download/all/)
2. Open `Hardware IO Tools > Network Link Conditioner.prefPane`
3. Install the preference pane
4. Open System Settings > Network Link Conditioner
5. Choose a profile (3G, Edge, High Latency DNS, etc.) or create a custom one

**Setup on physical device**:
1. On the device: `Settings > Developer > Network Link Conditioner`
2. Enable and choose a profile

**Common profiles for testing**:
| Profile | Download | Upload | Latency | Use Case |
|---------|----------|--------|---------|----------|
| 100% Loss | 0 | 0 | 0 | Offline behavior |
| 3G | 780 Kbps | 330 Kbps | 100ms | Emerging markets |
| Edge | 240 Kbps | 200 Kbps | 840ms | Worst-case mobile |
| High Latency DNS | Normal | Normal | 3000ms DNS | DNS resolution issues |
| WiFi (lossy) | 40 Mbps | 33 Mbps | 2ms + 2% loss | Unreliable WiFi |

---

## 5. Cross-Platform Native Techniques

### 5.1 Hermes CPU Sampling Profiler

Available on both platforms. The **only** way to profile JS execution at the engine level.

**Why this matters**: React DevTools shows you component render times, but the Hermes Sampling Profiler shows you what the Hermes engine is actually doing — parsing, compiling, executing, garbage collecting. This is essential for understanding why a JS function that "looks fast" in React DevTools might actually be slow due to engine-level overhead.

**Steps**:
1. Developer Menu > **"Enable Sampling Profiler"**
2. Perform the action you want to profile
3. Developer Menu > disable profiler (saves JSON trace)
4. Convert for Chrome DevTools:
```bash
npx hermes-profile-transformer <profile.cpuprofile> -o chrome-trace.json
```
5. Open `chrome://tracing` or Chrome DevTools Performance tab > load the JSON

### 5.2 react-native-release-profiler (Margelo)

Profile Hermes in **production/release builds** where standard tools are unavailable.

**Why this matters**: Many performance issues only appear in release builds. Debug builds have different JIT behavior, extra runtime checks, and dev-mode overhead that mask real performance characteristics. This library lets you capture Hermes profiles from release builds.

```bash
npm install react-native-release-profiler
```

```tsx
import { startProfiling, stopProfiling } from 'react-native-release-profiler';

// Wrap the suspect flow
startProfiling();
// ... user action ...
const path = await stopProfiling();
// Saves trace to device downloads folder
```

**Sampling interval**: 10ms default. Integrates with Chrome DevTools for visualization.

### 5.3 Tracy Profiler — React Native C++ Internals

For profiling React Native's **own C++ code** (Fabric, Yoga layout, RuntimeScheduler).

**Key instrumentation points**:
- `ReactInstance.cpp` — bridge/runtime initialization
- `RuntimeScheduler_Modern.cpp` — JS task scheduling
- Yoga's `calculateLayout()` — layout computation
- `MountingCoordinator.h` — Fabric mount operations

**Provides**: Unified view of CPU, function times, lock contentions, thread activity, and memory allocations.

Reference: [Callstack: Profiling React Native Internals with Tracy](https://www.callstack.com/blog/profiling-react-native-internals-with-tracy-for-peak-performance)

### 5.4 Expo Prebuild for Native Inspection

For Expo managed workflow, `prebuild` gives you access to native projects:

```bash
# Generate native directories
npx expo prebuild

# Open in Android Studio for native debugging
open android -a "Android Studio"

# Open in Xcode
open ios/*.xcworkspace

# Debug mode with verbose plugin logging
EXPO_DEBUG=1 npx expo prebuild
```

Once prebuilt, you have full access to all Android Studio and Xcode debugging tools described above.

### 5.5 Crashlytics NDK Tombstones

Since Firebase Crashlytics SDK v18.3.6, Android 12+ collects **tombstones** alongside minidumps:
- **Minidumps**: Precise app frame info
- **Tombstones**: On-device symbolication with system library symbols
- Combined view in Crashlytics dashboard, exportable to BigQuery

Ensure NDK crash reporting is enabled in your Firebase configuration for full native stack traces. In your `android/app/build.gradle`:

```gradle
android {
    // ...
    buildTypes {
        release {
            firebaseCrashlytics {
                nativeSymbolUploadEnabled true
            }
        }
    }
}
```

### 5.6 Debug Symbol Upload (Multi-Provider Guide)

Symbolicated crash reports are essential for debugging production crashes. Without symbols, crash stacks show raw memory addresses instead of function names and line numbers. Each crash reporting provider has its own symbol upload workflow.

#### Firebase Crashlytics

**Android** — ProGuard/R8 mapping and native symbols:
```gradle
// android/app/build.gradle
apply plugin: 'com.google.firebase.crashlytics'

android {
    buildTypes {
        release {
            firebaseCrashlytics {
                mappingFileUploadEnabled true    // ProGuard/R8 mappings
                nativeSymbolUploadEnabled true   // NDK symbols
            }
        }
    }
}
```

**iOS** — dSYM upload:
```bash
# Automatic upload via build phase script (added by Crashlytics pod)
# Manual upload:
/path/to/Pods/FirebaseCrashlytics/upload-symbols \
  -gsp ios/GoogleService-Info.plist \
  -p ios \
  /path/to/dSYMs
```

#### Sentry

**Android**:
```gradle
// android/app/build.gradle
apply from: "../../node_modules/@sentry/react-native/sentry.gradle"

// Ensure source maps and ProGuard mappings are uploaded
sentry {
    autoUploadProguardMapping = true
    uploadNativeSymbols = true
    includeNativeSources = true
}
```

**iOS**:
```bash
# Add a build phase script in Xcode (usually done automatically by Sentry's CocoaPods hook):
export SENTRY_PROPERTIES=sentry.properties
../node_modules/@sentry/cli/bin/sentry-cli upload-dsym
```

**Source maps** (both platforms):
```bash
# Upload Hermes source maps
npx sentry-cli releases files <release> upload-sourcemaps \
  --dist <dist> \
  --rewrite \
  path/to/sourcemap.js.map
```

#### Datadog

**Android**:
```gradle
// android/app/build.gradle
apply from: "../../node_modules/@datadog/mobile-react-native/datadog-sourcemaps.gradle"
```

**iOS**: Configure dSYM upload in your CI pipeline:
```bash
npx datadog-ci dsyms upload /path/to/dSYMs \
  --api-key $DATADOG_API_KEY
```

**Source maps**:
```bash
npx datadog-ci react-native upload \
  --service-name com.yourcompany.yourapp \
  --bundle ./index.android.bundle \
  --sourcemap ./index.android.bundle.map \
  --platform android \
  --release-version 1.0.0 \
  --build-version 100
```

#### Bugsnag

**Android**:
```gradle
// android/app/build.gradle
apply plugin: 'com.bugsnag.android.gradle'

bugsnag {
    uploadReactNativeMappings = true  // JS source maps
    uploadNdkMappings = true          // NDK symbols
}
```

**iOS**:
```bash
# Add to Xcode build phase (usually via BugsnagReactNative pod):
# Automatic dSYM upload is configured by the Bugsnag CocoaPods plugin
# Manual upload:
npx bugsnag-source-maps upload-ios \
  --api-key YOUR_API_KEY \
  --dsym /path/to/MyApp.app.dSYM
```

#### EAS Build Symbol Upload

For Expo/EAS builds, configure symbol uploads in `eas.json`:
```json
{
  "build": {
    "production": {
      "android": {
        "buildType": "app-bundle"
      },
      "ios": {
        "buildConfiguration": "Release"
      }
    }
  }
}
```

Then use EAS build hooks or your CI pipeline to upload symbols after each build completes. Most crash reporting SDKs provide EAS config plugins that automate this.

### 5.7 React Native New Architecture Debugging

The New Architecture (introduced incrementally from RN 0.68+, default from 0.76+) fundamentally changes how React Native bridges JavaScript and native code. This has significant implications for debugging and profiling.

#### How Fabric Changes the Debugging Model

In the old architecture, React Native used an asynchronous bridge to send serialized JSON messages between JavaScript and native. In the New Architecture:

- **Fabric** replaces the old UI Manager. Views are created and updated via C++ code that executes synchronously (or concurrently) with JavaScript via JSI.
- **The bridge is gone** (or minimal). Communication happens through direct C++ function calls.
- **Debugging implications**:
  - Bridge spy tools (like Flipper's bridge inspector) no longer apply
  - Thread boundaries are different — UI updates can happen synchronously from JS
  - The C++ layer is now the central orchestration point, making Tracy Profiler more valuable
  - Memory issues may involve C++ shared pointers rather than JNI references

#### TurboModules Debugging

TurboModules replace the old Native Modules system. Key differences for debugging:

- **Lazy initialization**: TurboModules are loaded on first use, not at startup. This changes startup profiling — a slow first call to a module now includes its initialization cost.
- **Synchronous calls**: TurboModules support synchronous JS-to-native calls via JSI. While faster (no bridge serialization), a blocking native call will freeze the JS thread.
- **Codegen types**: TurboModules use TypeScript/Flow specs to generate C++ interfaces. Type mismatches between JS and native cause compile-time errors (good) but runtime `JSINativeException` if the generated code does not match the native implementation.

**Debugging tips**:
```bash
# Look for TurboModule initialization in logcat
adb logcat | grep -i "TurboModule"

# Check for JSI exceptions
adb logcat | grep -E "(JSINativeException|TurboModuleManager)"
```

In Xcode, set a symbolic breakpoint on `facebook::react::TurboModule::invokeMethod` to intercept all TurboModule calls.

#### JSI Debugging Considerations

JSI (JavaScript Interface) is the C++ API that allows JavaScript to call C++ functions directly (and vice versa). Debugging JSI issues requires different tools:

- **C++ debuggers**: Use LLDB (Xcode) or GDB (Android Studio) to set breakpoints in JSI host functions
- **JSI host objects**: If you create custom JSI host objects, ensure proper ref-counting. A common bug is accessing a `jsi::Runtime` from a thread that doesn't own it — this causes subtle crashes
- **Memory management**: JSI objects are reference-counted. Cycles between `jsi::Object` instances and C++ `shared_ptr` are not detected by JavaScript GC — use weak references where appropriate

#### How the New Threading Model Affects Profiling

The New Architecture's concurrent rendering model introduces new threads:

| Thread | Role | Profiling Impact |
|--------|------|-----------------|
| JS Thread | Runs Hermes/JSC | Same as before |
| UI Thread | Native UI updates | Now receives synchronous calls from JS |
| Background Thread | Concurrent rendering work | New — Fabric can prepare view trees off the UI thread |
| RuntimeScheduler Thread | Schedules JS tasks with priorities | New — affects task ordering |

**Key changes for profiling**:
- A single user action may touch 3-4 threads instead of 2
- Perfetto/System Trace is even more valuable because it shows cross-thread relationships
- The `main` thread may appear less busy (work offloaded to background) but still be the bottleneck for commits
- Lock contention between threads is a new class of performance issue — use Tracy or TSan to detect

### 5.8 CI Integration for Native Performance

Automated performance testing in CI prevents regressions from shipping to production. Manual profiling catches issues you know to look for; CI catches regressions you would not have thought to check.

#### Flashlight (android-performance-profiler)

[Flashlight](https://github.com/nickt/android-performance-profiler) (formerly `android-performance-profiler` by BAM) runs automated performance benchmarks on Android:

```bash
# Install
npm install -g @perf-profiler/web-reporter
npx @perf-profiler/flashlight test --bundleId com.yourcompany.yourapp \
  --testCommand "maestro test flow.yml" \
  --iterationCount 5 \
  --duration 30000
```

**What it measures**:
- FPS (frames per second) during interactions
- CPU usage per thread
- RAM consumption over time
- Thread count and scheduling

**CI integration example** (GitHub Actions):
```yaml
- name: Run performance test
  run: |
    npx @perf-profiler/flashlight test \
      --bundleId com.yourcompany.yourapp \
      --testCommand "maestro test e2e/scroll-test.yml" \
      --iterationCount 3 \
      --duration 15000 \
      --resultsFilePath perf-results.json

- name: Check for regressions
  run: |
    # Compare against baseline
    npx @perf-profiler/web-reporter compare \
      --baseline baseline-results.json \
      --current perf-results.json \
      --threshold 10  # Fail if >10% regression
```

#### Detox for E2E with Native Profiling

[Detox](https://github.com/wix/Detox) is a gray-box E2E testing framework for React Native. While primarily a testing tool, it can be combined with native profiling:

```javascript
// e2e/performance.test.js
describe('Performance', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  it('should scroll feed without jank', async () => {
    // Start a Perfetto trace before the test
    await device.executeCommand('shell', [
      'perfetto', '-c', '-', '--txt',
      '-o', '/data/misc/perfetto-traces/test.pftrace',
      // ... config
    ]);

    // Perform the action
    await element(by.id('feed-list')).scroll(2000, 'down');

    // Stop trace and pull
    await device.executeCommand('pull',
      ['/data/misc/perfetto-traces/test.pftrace', './artifacts/']);
  });
});
```

#### Setting Up Performance Regression Testing

A complete CI performance pipeline:

1. **Define baselines**: Run performance tests on the main branch and save results as baseline artifacts
2. **Run on PRs**: Run the same tests on every PR and compare against baseline
3. **Set thresholds**: Define acceptable regression limits (e.g., FPS must not drop more than 5%, cold start must not increase more than 200ms)
4. **Alert on regression**: Fail the CI check or post a comment when thresholds are exceeded
5. **Track trends**: Store results over time to catch gradual regressions

**Example performance budget**:
```json
{
  "cold_start_ms": { "max": 2000, "warning": 1500 },
  "scroll_fps": { "min": 55, "warning": 58 },
  "memory_peak_mb": { "max": 350, "warning": 300 },
  "js_bundle_size_kb": { "max": 2048, "warning": 1800 }
}
```

---

## 6. App Optimization at Native Level

### 6.1 ProGuard / R8 Code Shrinking

R8 aggressively removes "unused" code — but React Native uses heavy reflection, so libraries can break.

**Why this matters**: R8 performs static analysis to determine which code is reachable. React Native's bridge and native module system use reflection to find and invoke native methods, which R8 cannot detect statically. Without proper keep rules, R8 will strip code that is actually needed at runtime, causing mysterious `ClassNotFoundException` or `NoSuchMethodException` crashes — but only in release builds.

```gradle
// In build.gradle release buildType:
minifyEnabled true  // Enables R8
shrinkResources true  // Removes unused resources
proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
```

**Common keep rules for React Native** (in `proguard-rules.pro`):
```
-keep class com.facebook.hermes.** { *; }
-keep class com.facebook.jni.** { *; }
-keep,allowobfuscation @interface com.facebook.proguard.annotations.DoNotStrip
-keep @com.facebook.proguard.annotations.DoNotStrip class *
```

**Impact**: 15-20% native code size reduction, but **must test release builds thoroughly**.

### 6.2 ABI Splits — Reduce APK Size by ~50%

```gradle
splits {
    abi {
        enable true
        universalApk false  // Don't generate a universal APK
        include "armeabi-v7a", "arm64-v8a", "x86", "x86_64"
    }
}
```

For apps targeting `minSdkVersion 29+`, you only need `arm64-v8a` (modern 64-bit devices).

**Better approach**: Use Android App Bundles (AAB) instead — Google Play auto-splits:
```bash
cd android && ./gradlew bundleRelease
```

### 6.3 Android Baseline Profiles — Faster Cold Start

Baseline Profiles provide AOT-compiled code paths, eliminating JIT compilation on first launch.

**Impact**: ~25-30% cold start improvement.

**How it works**: ART uses rules to pre-compile frequently used code paths. Ship profiles with the app via Android Gradle Plugin.

**Setup**: Requires Jetpack Macrobenchmark library to generate profiles. Especially impactful for React Native where large JS bundles add startup latency.

### 6.4 Hermes Bytecode Compilation

Hermes pre-compiles JS to bytecode at build time (not runtime). Ensure `hermesEnabled = true` in your `gradle.properties` (this is the default for new React Native projects).

**Verify it's working**:
```bash
# Check if the bundle is Hermes bytecode (not plain JS)
file android/app/build/generated/assets/createBundleReleaseJsAndAssets/index.android.bundle
# Should say: "Hermes JavaScript bytecode"
```

#### Hermes Bundle Size Analysis

Use `hermes-profile-transformer` to inspect what is in your Hermes bytecode bundle:

```bash
# Generate a source map during bundling
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output index.android.bundle \
  --sourcemap-output index.android.bundle.map

# Analyze the source map to see module sizes
npx source-map-explorer index.android.bundle index.android.bundle.map
```

`source-map-explorer` generates a treemap visualization showing which node modules and source files contribute most to your bundle size. Common findings:

- **Large localization files** — consider loading translations dynamically
- **Moment.js locales** — switch to `dayjs` or use `moment-locales-webpack-plugin`
- **Unused lodash functions** — use `lodash-es` with tree-shaking or individual imports
- **Duplicate dependencies** — different versions of the same library bundled by different dependencies

#### Tree-Shaking Effectiveness Measurement

Hermes does not perform tree-shaking itself — it compiles whatever Metro bundles. Metro's tree-shaking support is limited compared to webpack/rollup. To measure effectiveness:

1. **Compare bundle sizes**: Build with and without a suspect dependency and compare:
   ```bash
   # Size of the bytecode bundle
   ls -la android/app/build/generated/assets/createBundleReleaseJsAndAssets/index.android.bundle
   ```
2. **Audit imports**: Replace namespace imports with named imports:
   ```typescript
   // Bad — imports entire library
   import _ from 'lodash';
   // Good — imports only what's used
   import { debounce } from 'lodash-es';
   ```
3. **Use `@react-native-community/cli-tools`**:
   ```bash
   npx react-native-community/cli ram-bundle --platform android --dev false --entry-file index.js
   ```
   RAM bundles split the bundle into modules that can be loaded on demand, reducing initial parse time.

### 6.5 Native Module Lazy Loading

By default, React Native initializes all native modules at startup. For apps with many native modules, this adds significant cold start time.

**Measuring the impact**:
```bash
# Android: see how long each native module takes to initialize
adb logcat | grep -E "(ReactMarker|CREATE_MODULE)"
```

**Lazy loading patterns**:

1. **TurboModules (New Architecture)**: Automatically lazy-loaded on first use. This is the single biggest startup improvement when migrating to the New Architecture.

2. **Manual lazy loading (Old Architecture)**:
   ```java
   // Instead of initializing in getPackages():
   @Override
   protected List<ReactPackage> getPackages() {
       return Arrays.asList(
           new MainReactPackage(),
           // Only add packages that are needed at startup
           new CorePackage()
       );
   }

   // Load additional packages on demand via a native module
   ```

3. **Deferred native module registration**: Use `ReactPackage.getReactModuleInfoProvider()` to declare modules without instantiating them until first use.

### 6.6 Image Pipeline Optimization

Images are the largest memory consumer in most React Native apps. Optimizing the image pipeline at the native level has outsized impact.

#### Android (Fresco Configuration)

Fresco is React Native's default image library on Android. Customize it in your `MainApplication`:

```java
import com.facebook.imagepipeline.core.ImagePipelineConfig;
import com.facebook.imagepipeline.memory.PoolConfig;
import com.facebook.imagepipeline.memory.PoolFactory;

@Override
public void onCreate() {
    super.onCreate();

    ImagePipelineConfig config = ImagePipelineConfig.newBuilder(this)
        .setDownsampleEnabled(true)             // Downsample large images to view size
        .setResizeAndRotateEnabledForNetwork(true) // Resize network images
        .setBitmapMemoryCacheParamsSupplier(() ->
            new MemoryCacheParams(
                20 * 1024 * 1024,   // Max cache size (20 MB)
                50,                  // Max cache entries
                4 * 1024 * 1024,    // Max single image size (4 MB)
                Integer.MAX_VALUE,   // Max eviction queue size
                Integer.MAX_VALUE    // Max eviction queue entries
            )
        )
        .build();

    Fresco.initialize(this, config);
}
```

**Key tuning parameters**:
- `setDownsampleEnabled(true)` — reduces memory by decoding images at display size, not full resolution. This alone can cut image memory by 50-80%.
- Cache size limits — prevent Fresco from consuming unbounded memory on devices with many images
- `setResizeAndRotateEnabledForNetwork(true)` — resize images downloaded from the network before caching

#### iOS (SDWebImage / Native Image Handling)

If your app uses a library like `react-native-fast-image` (which wraps SDWebImage):

```swift
// In AppDelegate.swift
import SDWebImage

func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: ...) -> Bool {
    // Configure SDWebImage cache
    SDImageCache.shared.config.maxMemoryCost = 20 * 1024 * 1024  // 20 MB memory cache
    SDImageCache.shared.config.maxDiskSize = 100 * 1024 * 1024   // 100 MB disk cache
    SDImageCache.shared.config.maxDiskAge = 7 * 24 * 60 * 60     // 7 days

    // Enable memory-mapped file for large images
    SDImageCache.shared.config.shouldDecompressImages = true

    return true
}
```

**General image optimization tips**:
- Use WebP format instead of PNG/JPEG — 25-35% smaller at equivalent quality
- Implement progressive loading (blur placeholder > thumbnail > full image)
- Preload images for screens the user is likely to navigate to
- Clear caches when receiving memory warnings:
  ```swift
  // iOS
  override func applicationDidReceiveMemoryWarning(_ application: UIApplication) {
      SDImageCache.shared.clearMemory()
  }
  ```

### 6.7 React Native Threading Model

| Platform | Native Module Threading |
|----------|------------------------|
| iOS | Each native module gets its own serial dispatch queue |
| Android | **All native modules share one thread pool** by default |

This means on Android, a slow native module call can block ALL other native module calls. Solutions:
- Move heavy work to a dedicated `ExecutorService`
- Use `@ReactMethod(isBlockingSynchronousMethod = false)` to ensure async execution
- With New Architecture / TurboModules: synchronous calls bypass the bridge entirely (10x-1000x faster)

### 6.8 Flipper Is Dead — Use These Instead

Flipper was deprecated in RN 0.73 due to instability and build time overhead.

**Replacements**:
| Old (Flipper) | New |
|--------------|-----|
| Network inspector | `react-native-network-logger` (dev only) |
| Layout inspector | Android Studio Layout Inspector / Xcode View Hierarchy |
| Database inspector | Direct `adb` commands or Android Studio DB Inspector |
| Hermes profiler | Hermes Sampling Profiler (built-in) |
| General debugging | React Native DevTools (Chrome/Edge, zero-install) |
| State inspection | Reactotron |

---

## 7. Troubleshooting Common Issues

These are the most frequent problems developers encounter when setting up native profiling. Each includes the root cause and fix.

### 7.1 "Android Studio Profiler Shows No Data"

**Symptoms**: The Profiler connects to the app but shows empty graphs or "No data available".

**Causes and fixes**:

1. **App is not debuggable**: The Profiler requires either `android:debuggable="true"` (debug builds) or `<profileable android:shell="true" />` (release builds).
   - **Fix**: For debug builds, ensure your `build.gradle` debug buildType has `debuggable true`. For release builds, add `<profileable android:shell="true" />` to your `<application>` tag in `AndroidManifest.xml`.

2. **Wrong process selected**: React Native apps sometimes spawn multiple processes. Ensure you select the main process (your package name without a suffix).
   - **Fix**: In the Profiler session dropdown, select `com.yourcompany.yourapp` (not `:detox` or other suffixed processes).

3. **ADB connection issue**: The Profiler communicates via ADB. A flaky USB connection or ADB daemon crash can break the data stream.
   - **Fix**: Run `adb kill-server && adb start-server`, then reconnect the device and restart the Profiler session.

4. **Insufficient API level**: Some Profiler features (like native allocation tracking) require Android 10+. Check your device API level.
   - **Fix**: Use a device or emulator running Android 10 (API 29) or higher.

5. **Gradle sync required**: After modifying build configs, Android Studio may need a Gradle sync before profiling works correctly.
   - **Fix**: `File > Sync Project with Gradle Files`, then rebuild the app.

### 7.2 "Xcode Instruments Won't Attach to Process"

**Symptoms**: Instruments shows "Failed to start trace" or "Target process exited before trace could begin".

**Causes and fixes**:

1. **Code signing issue**: Instruments requires the app to be properly signed for the device.
   - **Fix**: Ensure the app runs normally on the device first (`Product > Run`). If it runs successfully, `Product > Profile` should work.

2. **Device is locked**: Instruments cannot attach to processes on a locked device.
   - **Fix**: Unlock the device and keep it awake during profiling.

3. **Entitlements mismatch**: The `get-task-allow` entitlement must be present for profiling. Release builds strip this.
   - **Fix**: For release profiling, use `Product > Profile` which builds with the correct entitlements. Do not try to attach to an App Store build.

4. **Instruments version mismatch**: Instruments and the device OS version must be compatible. Running Xcode 14 Instruments against an iOS 17 device may fail.
   - **Fix**: Update Xcode to match or exceed the device's iOS version.

5. **Zombie/old process**: A previous instance of the app may still be running.
   - **Fix**: Force-quit the app on the device, then try profiling again.

### 7.3 "ndk-stack Shows No Symbols"

**Symptoms**: Running `ndk-stack` produces output with `??:0` or `<unknown>` for every frame instead of function names and line numbers.

**Causes and fixes**:

1. **Wrong symbol path**: You must point `ndk-stack` to the **unstripped** `.so` files that match the exact build that crashed.
   - **Fix**: For debug builds, use: `android/app/build/intermediates/merged_native_libs/debug/out/lib/<abi>/`. For release builds, use: `android/app/build/intermediates/merged_native_libs/release/out/lib/<abi>/`.

2. **Wrong ABI**: The crash was on `arm64-v8a` but you pointed to `armeabi-v7a` symbols (or vice versa).
   - **Fix**: Check the crash log for the ABI (e.g., `ABI: 'arm64'`) and use the matching symbol directory.

3. **Stripped binaries**: Release builds strip debug symbols. If you did not preserve unstripped binaries, symbolication is impossible.
   - **Fix**: Configure your build to preserve unstripped libraries:
     ```gradle
     android {
         packagingOptions {
             doNotStrip "*/arm64-v8a/*.so"
             doNotStrip "*/armeabi-v7a/*.so"
         }
     }
     ```
     Or better — upload symbols to your crash reporting tool during CI.

4. **NDK version mismatch**: Using an `ndk-stack` from a different NDK version than the one used to build can cause parsing issues.
   - **Fix**: Use the same NDK version for symbolication that was used for the build. Check `android/build.gradle` for `ndkVersion`.

### 7.4 "Perfetto Trace Is Empty"

**Symptoms**: You capture a Perfetto trace and open it in `ui.perfetto.dev`, but the trace shows no app-specific data (only kernel/system events, or nothing at all).

**Causes and fixes**:

1. **Wrong `atrace_apps` value**: The `atrace_apps` field must match your app's package name exactly.
   - **Fix**: Verify the package name: `adb shell pm list packages | grep yourcompany`. Use the exact string in the Perfetto config.

2. **App not profileable**: On Android 10+, the app must be either debuggable or profileable for Perfetto to capture app events.
   - **Fix**: Add `<profileable android:shell="true" />` to your `AndroidManifest.xml` `<application>` tag.

3. **Trace categories missing**: If you do not include the right `atrace_categories`, you will not see the events you are looking for.
   - **Fix**: At minimum, include these categories for React Native profiling:
     ```
     atrace_categories: "am"    # Activity Manager
     atrace_categories: "wm"    # Window Manager
     atrace_categories: "view"  # View system
     atrace_categories: "gfx"   # Graphics
     ```

4. **Buffer too small**: If the trace buffer is too small and the trace duration is too long, early events are overwritten.
   - **Fix**: Increase `size_kb` or reduce `duration_ms`. For a 10-second trace, `63488` KB (62 MB) is usually sufficient.

5. **SELinux blocking**: On some devices, SELinux policies may prevent Perfetto from accessing app data.
   - **Fix**: Try running on a Google Pixel or an emulator with Google APIs, which have more permissive Perfetto policies.

### 7.5 "Hermes Profiler Trace Is Empty or Missing Functions"

**Symptoms**: The Hermes sampling profiler trace opens in Chrome DevTools but shows few or no functions, or all functions are labeled "(anonymous)".

**Causes and fixes**:

1. **Source maps not loaded**: Chrome DevTools needs source maps to map bytecode offsets to function names.
   - **Fix**: Ensure source maps are generated during the build and loaded alongside the trace in Chrome DevTools.

2. **Too short sampling window**: The profiler samples at intervals. Very short operations may not be captured.
   - **Fix**: Run the profiler for at least 5-10 seconds, and repeat the action multiple times to increase the chance of capturing it.

3. **Minification without source maps**: If the bundle is minified (release build) and source maps are not available, all functions appear as single-letter names.
   - **Fix**: Build with source maps: `npx react-native bundle --sourcemap-output index.android.bundle.map ...`

---

## 8. Tool Summary Matrix

| Level | Android | iOS | Cross-Platform |
|-------|---------|-----|----------------|
| **Memory** | AS Profiler, heapprofd, Perfetto | Instruments (Leaks/Allocations), Memory Graph, Zombies | Tracy |
| **CPU** | AS CPU Profiler, Perfetto, System Trace | Instruments (Time Profiler, App Launch) | Hermes Sampler, release-profiler, Tracy |
| **Threads** | Perfetto thread view, System Trace, StrictMode | Thread Sanitizer, Time Profiler | Tracy (lock contention) |
| **Crashes** | ndk-stack, Crashlytics NDK + tombstones | dSYM + atos, Address Sanitizer | Crashlytics, Sentry, Datadog, Bugsnag |
| **Graphics** | Layout Inspector, Perfetto GPU | Metal System Trace | -- |
| **Network** | Profiler Network, StrictMode | Network Link Conditioner, Instruments | react-native-network-logger |
| **Binary Size** | ProGuard/R8, ABI splits, AABs | App Thinning | Hermes bytecode, source-map-explorer |
| **Startup** | Baseline Profiles, Perfetto startup, StrictMode | Instruments (App Launch) | Hermes, expo prebuild |
| **CI** | Flashlight, Detox | Detox, XCTest Performance | Flashlight (Android), custom scripts |

---

## 9. Recommended Reading

### Comprehensive Guides
- [React Native Advanced Guide](https://github.com/anisurrahman072/React-Native-Advanced-Guide) — 12 chapters, 70+ topics including native profiling
- [React Native Official Profiling Guide](https://reactnative.dev/docs/profiling)
- [Callstack: Profiling React Native Apps with iOS and Android Tools](https://www.callstack.com/blog/profiling-react-native-apps-with-ios-and-android-tools)
- [Best React Native Debugging Tools in 2025 (SW Mansion)](https://blog.swmansion.com/best-react-native-debugging-tools-in-2025-a95dcac7a014)

### Android Deep Dives
- [Perfetto for App Start Performance](https://andrei-calazans.com/posts/analyzing-app-start-android-systrace/)
- [Perfetto Native Heap Profiler](https://perfetto.dev/docs/data-sources/native-heap-profiler)
- [Tracy for React Native Internals](https://www.callstack.com/blog/profiling-react-native-internals-with-tracy-for-peak-performance)
- [Firebase Crashlytics NDK Tombstones](https://firebase.blog/posts/2025/11/crashlytics-android-ndk-tombstones)
- [Mattermost: Cold Start Optimization](https://mattermost.com/blog/how-we-improved-our-react-native-cold-start-for-android/)
- [Android Developers: Baseline Profiles](https://developer.android.com/topic/performance/baselineprofiles/overview)
- [Android StrictMode Documentation](https://developer.android.com/reference/android/os/StrictMode)

### iOS Deep Dives
- [Apple: Diagnosing Memory, Thread, and Crash Issues](https://developer.apple.com/documentation/xcode/diagnosing-memory-thread-and-crash-issues-early)
- [Finding Memory Leaks in React Native iOS](https://www.bomberbot.com/react-native/finding-and-fixing-memory-leaks-in-your-react-native-ios-app-an-in-depth-guide/)
- [Xcode Memory Graph Debugger](https://www.donnywals.com/using-xcodes-memory-graph-to-find-memory-leaks/)
- [Apple: Reducing Your App's Launch Time](https://developer.apple.com/documentation/xcode/improving-app-launch-time)
- [WWDC: Analyze and Improve App Launch Time](https://developer.apple.com/videos/play/tech-talks/10855/)

### Architecture
- [React Native New Architecture](https://reactnative.dev/architecture/landing-page) — JSI, Fabric, TurboModules
- [RN Architecture: From Bridge to Fabric](https://blog.codeminer42.com/react-native-architecture-from-bridge-to-fabric/)
- [React Native New Architecture: TurboModules](https://reactnative.dev/docs/the-new-architecture/pillars-turbomodules)

### Tools
- [react-native-release-profiler (Margelo)](https://github.com/margelo/react-native-release-profiler) — Profile Hermes in production builds
- [ndk-stack (Android NDK)](https://developer.android.com/ndk/guides/ndk-stack) — Symbolicate native crash stacks
- [Expo: Debugging Runtime Issues](https://docs.expo.dev/debugging/runtime-issues/)
- [Flashlight (Performance Profiler)](https://github.com/nickt/android-performance-profiler) — Automated Android performance testing
- [Detox](https://github.com/wix/Detox) — Gray-box E2E testing for React Native
- [source-map-explorer](https://github.com/danvk/source-map-explorer) — Bundle size visualization

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [SIGABRT & libc Debugging](./crash-analysis.md) | Complementary guide — use this for root cause analysis of crashes found while profiling with the tools described here |
| [Profiling Tools Deep Dive](./profiling-tools-deep-dive.md) | Decision framework for when to use each tool — helps you choose between the tools documented here |
| [Architecture & Lifecycle](../foundation/architecture-lifecycle.md) | Foundation for understanding the threading model and native layers you're profiling |
| [Profiling & Debugging](../optimization/profiling-debugging.md) | JS-level profiling (React DevTools, Flashlight) that complements the native-level tools here |
