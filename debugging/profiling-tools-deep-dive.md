# Profiling Tools Deep Dive: Memory & CPU Usage

> Profile before you optimize. The right tool for the right layer saves hours of guesswork.

[← Back to Index](../README.md)

**Keywords**: profiling, memory, CPU, Android Studio, Xcode Instruments, Perfetto, Hermes, heapprofd, Tracy, Flashlight, flamegraph, heap dump, allocation tracking, Time Profiler, Leaks

---

<details>
<summary><strong>TL;DR</strong></summary>

- Start at JS layer; if JS thread is idle during slowdown, escalate to native tools
- Memory: Hermes heap snapshots → Android Studio Profiler / Xcode Leaks → heapprofd
- CPU: React DevTools Profiler → Hermes CPU Profiler → Perfetto / Xcode Time Profiler
- Graphics memory is the hidden killer — can spike 7x during scroll without showing in heap profilers
- Always profile on real devices (especially low-end Android), never just emulators

</details>

## When Do You Need Profiling?

Not every performance issue requires profiling tools. Use this decision framework to determine when to reach for them — and which layer to target.

### Symptom → Tool Decision Matrix

| Symptom | Layer | First Tool to Reach For | Escalation Tool |
|---------|-------|------------------------|-----------------|
| UI jank during scrolling | JS → Native | React DevTools Profiler | Systrace / Perfetto |
| Slow screen transitions | JS | Hermes CPU Profiler | React DevTools flamegraph |
| App slows down over time (5+ min) | Memory (JS or Native) | Hermes heap snapshots | Android Studio Memory Profiler / Xcode Leaks |
| App killed by OS in background | Native memory | Android Studio Memory Profiler / Xcode Memory Graph | heapprofd (Android) |
| High battery drain | CPU | Flashlight (CPU %) | Xcode Energy Log / Android Studio CPU Profiler |
| Slow cold start (>2s) | Both | Perfetto startup trace | Hermes bundle analysis + Baseline Profiles |
| Crash on low-end devices only | Native memory | Android Studio Memory Profiler | ASan (AddressSanitizer) |
| Animation frame drops | UI thread | Systrace / Perfetto | Tracy (RN C++ internals) |
| Network-related freezes | JS thread | Network Logger (dev) | Sentry Performance (prod) |
| Mysterious native crash | Native | ndk-stack + tombstone analysis | ASan / TSan / UBSan |

### The Three Layers of React Native Profiling

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 1: JavaScript                                     │
│  Tools: React DevTools, Hermes Profiler, Hermes Heap     │
│  When: Re-renders, slow functions, JS memory leaks       │
├─────────────────────────────────────────────────────────┤
│  LAYER 2: Bridge / Framework                             │
│  Tools: Systrace, Tracy, Sentry Performance              │
│  When: Cross-thread bottlenecks, serialization overhead   │
├─────────────────────────────────────────────────────────┤
│  LAYER 3: Native (Android / iOS)                         │
│  Tools: Android Studio Profiler, Xcode Instruments,      │
│         Perfetto, heapprofd, ASan                        │
│  When: Native memory leaks, CPU spikes, startup perf     │
└─────────────────────────────────────────────────────────┘
```

**Rule of thumb**: Start at Layer 1 (JS). If the profiler shows the JS thread is idle during the slowdown, the problem is in Layer 2 or 3 — escalate to native tools.

---

## Complete Tool Reference

### JS-Level Profiling Tools

| Tool | Measures | Platform | Use Case | Setup Effort |
|------|----------|----------|----------|-------------|
| **React DevTools Profiler** | Component render times, re-render causes, flamegraph | Both | Finding unnecessary re-renders, identifying which component is slow | Zero (built-in) |
| **Hermes CPU Profiler** | JS function execution times, call stacks | Both | Finding slow JS functions (regex, JSON parse, lodash) | Zero (built-in via Chrome DevTools) |
| **Hermes Heap Snapshots** | JS heap objects, retention chains | Both | Finding JS memory leaks (unmounted listeners, closures) | Zero (built-in) |
| **Flashlight (BAM)** | FPS, CPU (all threads), RAM on real devices | Both | Before/after benchmarking, CI regression testing | Low (`npx @bamlab/flashlight measure`) |
| **Sentry Performance** | Production TTI, transaction traces, vitals | Both | Monitoring real users in production | Medium (SDK install + config) |
| **React Compiler** | Auto-memoization analysis | Both | Eliminating re-render bugs at compile time (Expo SDK 54+) | Low (config flag) |

### Native-Level Profiling Tools

| Tool | Measures | Platform | Use Case | Setup Effort |
|------|----------|----------|----------|-------------|
| **Android Studio Memory Profiler** | Java/Kotlin heap, native allocations, GC events | Android | Native memory leaks, OOM investigation | Medium (Android Studio required) |
| **Android Studio CPU Profiler** | Thread activity, method traces, system traces | Android | Native thread blocking, main thread contention | Medium |
| **Xcode Instruments — Time Profiler** | CPU time per function, thread breakdown | iOS | Finding CPU-intensive native code | Medium (Xcode required) |
| **Xcode Instruments — Leaks** | Objective-C/Swift memory leaks, retain cycles | iOS | iOS native memory leaks | Medium |
| **Xcode Memory Graph Debugger** | Object graph, retain cycles visualization | iOS | Complex retain cycle analysis | Medium |
| **Perfetto** | System-wide traces: CPU scheduling, GPU, I/O, binder | Both | Cross-process analysis, startup optimization, system-level bottlenecks | High (requires trace recording setup) |
| **heapprofd** | Native heap allocations with stack traces | Android | C++ / native memory leak tracking (JNI, Hermes internals) | High |
| **Tracy** | React Native C++ internals (Fabric, JSI calls) | Both | Framework-level debugging (for RN contributors / advanced) | High (custom build required) |
| **ndk-stack** | Symbolicated native stack traces | Android | Reading tombstone / crash dumps | Low (Android NDK) |
| **ASan / TSan / UBSan** | Memory corruption, threading bugs, undefined behavior | Both | Crash-level native debugging | High (custom build flags) |

### Production Monitoring Tools

| Tool | Measures | Use Case |
|------|----------|----------|
| **Firebase Crashlytics** | Crash reports, ANR traces, session data | Production crash monitoring |
| **Sentry** | Errors, performance traces, AI-suggested fixes | Full-stack production monitoring |
| **Google Play Console** | Android Vitals, device segmentation, Gemini AI analysis | Android-specific production metrics |
| **Datadog** | APM, error tracking, full-stack correlation | Enterprise production monitoring |

> **See also**: [Monitoring & ANR Analysis](../optimization/monitoring-anr-analysis.md) for production monitoring setup

---

## Deep Dive: Memory Profiling

Memory issues are the most common cause of app instability on low-end devices. They manifest as gradual slowdowns, OS-triggered kills, and OOM crashes — often without clear stack traces.

### When Memory Profiling Is Needed

| Signal | What's Happening | Action |
|--------|-----------------|--------|
| App gets slower after 5+ minutes of use | Memory leak — objects accumulate without cleanup | Profile with Hermes heap snapshots |
| Back/forth navigation increases memory each time | Screen-level leak — components not fully unmounting | Compare heap snapshots before/after navigation |
| OS kills app frequently in background | Excessive resident memory | Profile with Android Studio / Xcode Memory Graph |
| Crash on low-RAM devices (<4GB) only | Memory pressure + inefficient allocation | Profile native memory with heapprofd or Instruments |
| Graphics memory spikes during scroll | Large images / video not recycled | Profile GPU memory in Android Studio |

### JS Memory Profiling: Hermes Heap Snapshots

**When to use**: Your app's JS heap grows over time, or navigating between screens doesn't release memory.

**Workflow**:
1. Open Chrome DevTools connected to Hermes (`j` in Metro)
2. Go to **Memory** tab
3. Take a **heap snapshot** at app idle state (baseline)
4. Perform the suspected leaky interaction (navigate to screen and back, open/close modal 10x)
5. Take another heap snapshot
6. Compare snapshots — sort by **Retained Size** to find objects that shouldn't exist

**What to look for**:
- Objects from unmounted screens still in memory
- Event listener references holding component closures
- Large arrays or maps that grow without bounds
- Timer references (`setInterval` IDs) without cleanup

**Common JS memory leak patterns**:

```tsx
// PATTERN 1: Event listener not cleaned up
useEffect(() => {
  const handler = (data) => setItems(prev => [...prev, data]);
  EventEmitter.addListener('newItem', handler);
  return () => EventEmitter.removeListener('newItem', handler); // MUST cleanup
}, []);

// PATTERN 2: Closure over stale state in timer
useEffect(() => {
  const id = setInterval(() => {
    // This closure captures the initial `items` array
    // The old array is never GC'd while interval runs
    setItems(prev => [...prev, fetchNewItem()]);
  }, 5000);
  return () => clearInterval(id); // MUST cleanup
}, []);

// PATTERN 3: AbortController for fetch
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/data', { signal: controller.signal })
    .then(res => res.json())
    .then(setData)
    .catch(err => {
      if (err.name !== 'AbortError') throw err;
    });
  return () => controller.abort(); // MUST abort on unmount
}, []);

// PATTERN 4: WebSocket not closed
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/feed');
  ws.onmessage = (e) => setMessages(prev => [...prev, JSON.parse(e.data)]);
  return () => ws.close(); // MUST close
}, []);
```

### Android Native Memory Profiling

**When to use**: JS heap looks normal, but the app still consumes excessive memory or gets OOM-killed.

**Tool: Android Studio Memory Profiler**

1. Run your app in debug mode from Android Studio
2. Open **View → Tool Windows → Profiler**
3. Click **Memory** row to expand
4. Watch the real-time allocation graph

**What to look for**:
- **Steadily rising line** = memory leak (objects allocated but never freed)
- **Sawtooth pattern** = healthy GC behavior (allocate, collect, repeat)
- **Sudden spikes** = large allocation events (image decode, video buffer)
- **Native portion growing** = C++ / JNI leak (Hermes internals, native modules)

**Heap dump analysis**:
1. Click **Dump Java Heap** in the profiler
2. Sort by **Retained Size** (largest first)
3. Look for:
   - `Bitmap` objects with large retained size (un-recycled images)
   - `Activity` or `Fragment` instances that should be destroyed
   - `WebView` instances not properly cleaned up
   - React Native `ReactContext` leaks (rare but catastrophic)

**Tool: heapprofd (Native heap)**

For C++ / native memory leaks below the Java/Kotlin layer:

```bash
# Record native allocations for 30 seconds
adb shell perfetto -o /data/misc/perfetto-traces/heap.pftrace -c - <<EOF
buffers { size_kb: 63488 }
data_sources {
  config {
    name: "android.heapprofd"
    heapprofd_config {
      sampling_interval_bytes: 4096
      process_cmdline: "com.yourapp"
      continuous_dump_config { dump_phase_ms: 0 dump_interval_ms: 10000 }
    }
  }
}
duration_ms: 30000
EOF

# Pull and open in Perfetto UI
adb pull /data/misc/perfetto-traces/heap.pftrace
# Open at ui.perfetto.dev
```

**Use cases for heapprofd**:
- Hermes engine memory growth (rare, but happens with large bytecode bundles)
- Native module memory leaks (custom C++ modules via JSI)
- Image decoding libraries allocating without freeing (Fresco, Glide)

### iOS Native Memory Profiling

**When to use**: Same as Android — JS heap is fine, but the app consumes too much memory or gets jetsammed (iOS term for OOM kill).

**Tool: Xcode Instruments — Leaks**

1. Open Xcode → **Product → Profile** (⌘I)
2. Select **Leaks** template
3. Run the app and perform the suspected leaky interaction
4. Leaks instrument flags objects that are allocated but never deallocated

**What to look for**:
- Retain cycles between `UIViewController` and closures
- `NSTimer` / `CADisplayLink` not invalidated
- Notification center observers not removed
- Strong delegate references (should be `weak`)

**Tool: Xcode Memory Graph Debugger**

For complex retain cycle analysis:
1. Run app in Xcode
2. Click the **Memory Graph** button in the debug navigator (looks like interconnected nodes)
3. Inspect the object graph — purple exclamation marks indicate leaks
4. Follow the retain chain to find the root cause

**Real-world example**: A React Native app had a `WebView` component that held a strong reference to its parent `UIViewController`. Navigating away didn't deallocate the view controller because the `WebView` → `ViewController` retain cycle kept both alive. Fix: set the `WebView.navigationDelegate` to `nil` in `viewWillDisappear`.

### Graphics Memory: The Hidden Killer

Graphics memory can spike dramatically without showing up in standard heap profilers.

**Real example from production**: Scrolling through a feed with autoplay videos caused graphics memory to jump from 70MB to 472MB (7x increase). The Java heap looked fine — the spike was entirely in GPU texture memory from decoded video frames.

**How to detect**:
- Android: Android Studio Profiler → look at the **Graphics** row (separate from Java/Native heap)
- iOS: Xcode Memory Report → check **GPU Memory** column

**Fixes are often product-level, not code-level**:
- Disable autoplay for videos outside viewport
- Pause media when scrolling fast (velocity threshold)
- Downsample thumbnails (Discord: 256×256 instead of 370×370 = 12% less memory)
- Limit concurrent decoded images (pool of 3-5 max)

---

## Deep Dive: CPU Profiling

CPU profiling helps you find what's consuming processing time — whether it's JS functions blocking the thread, native code doing expensive work, or cross-thread contention causing delays.

### When CPU Profiling Is Needed

| Signal | What's Happening | Action |
|--------|-----------------|--------|
| Animations drop below 60fps | JS or UI thread overloaded | Profile with Systrace/Perfetto to identify which thread |
| Interactions feel delayed (>100ms response) | Main thread blocked | Profile with CPU Profiler to find the blocking function |
| Battery drains faster than expected | Continuous CPU work (hidden animations, polling) | Profile with Flashlight for CPU % over time |
| Cold start takes >2 seconds | Too much synchronous work at startup | Profile with Perfetto startup trace |
| Background CPU usage >5% when idle | Rogue timers, animations, or polling | Profile with Xcode Energy Log or Flashlight |
| Specific screen is sluggish | Complex render tree or expensive computations | Profile with React DevTools flamegraph |

### JS CPU Profiling: Hermes Profiler

**When to use**: A specific interaction is slow, and you suspect JS-level computation is the bottleneck.

**Workflow**:
1. Connect Chrome DevTools to Hermes (`j` in Metro)
2. Go to **Performance** tab
3. Click **Record**
4. Perform the slow interaction
5. Stop recording
6. Analyze the flamegraph

**Reading the flamegraph**:
- **X-axis** = time duration (wider = more time spent)
- **Y-axis** = call stack depth (taller = deeper call chain)
- **Color coding**: Yellow = JS execution, Purple = layout/paint, Green = rendering

**What to look for**:
- **Wide bars at the top** = single expensive function (the bottleneck)
- **Repeated narrow bars** = function called too many times (N+1 problem)
- **Gaps between bars** = waiting for async work (not a CPU issue)

**Real examples from Discord**:
1. **Regex compilation in render** — Inline `new RegExp()` in markdown parser created a new regex object every render. Moving to `const` outside the component: **30% faster message loading**
2. **Unicode emoji detection** — 400+ concatenated Unicode symbols tested per message. Optimized to single regex: **90% total reduction, -1s TTI**
3. **Dimensions module** — Using `Dimensions` module reactively triggered 3 commit passes at startup. Single read instead: **-150ms TTI**

### React DevTools Profiler: Finding Re-render Bottlenecks

**When to use**: The app feels slow during interactions, and you suspect unnecessary re-renders.

**Workflow**:
1. Press `j` in Metro to open React DevTools
2. Go to **Profiler** tab
3. Enable "Record why each component rendered"
4. Click **Record**
5. Perform the slow interaction
6. Stop and analyze

**Reading the results**:
- **Tall flames** = Deep tree re-rendering (parent causing cascading child re-renders)
- **Wide flames** = Single slow component (expensive render function)
- **Grey bars** = Components that didn't re-render (good — memoization working)

**The "Why did this render?" column** tells you:
- "Props changed" → Check which prop, add `React.memo` or `useCallback`
- "State changed" → Expected, but check if state change was necessary
- "Parent rendered" → Add `React.memo` to prevent unnecessary cascading
- "Context changed" → Split context or memoize context value

> **See also**: [Performance: Memoization](../optimization/performance-rendering.md#memoization) for fixes | [React Compiler](../optimization/profiling-debugging.md#react-compiler) to auto-fix

### Android Native CPU Profiling

**Tool: Android Studio CPU Profiler**

1. Run app in debug mode from Android Studio
2. Open **Profiler → CPU**
3. Select **Sample Java Methods** or **Trace System Calls**
4. Record during the slow interaction
5. Analyze the flame chart

**Key threads to watch in React Native**:
| Thread | Role | Blocking Symptom |
|--------|------|-----------------|
| `main` (UI thread) | Renders views, handles touch events | Frozen UI, ANR dialog |
| `mqt_js` | Runs JavaScript via Hermes | Delayed responses, stale data |
| `mqt_native_modules` | Executes native module calls | Slow native operations (camera, file I/O) |
| `Fabric` | Handles layout calculations (New Arch) | Delayed layout updates |
| `RenderThread` | GPU rendering, animations | Dropped frames, visual glitches |

**What to look for**:
- `main` thread blocked waiting for `mqt_js` → Bridge/JSI contention
- `mqt_js` spending >16ms in a single function → Will miss 60fps frame
- `RenderThread` saturated → Too many views or complex draw operations

**Tool: Perfetto (System-wide tracing)**

For cross-process, system-level analysis. Especially valuable for cold start optimization.

```bash
# Record a startup trace
adb shell perfetto -o /data/misc/perfetto-traces/startup.pftrace -c - <<EOF
buffers { size_kb: 63488 }
data_sources {
  config { name: "linux.ftrace" ftrace_config {
    ftrace_events: "sched/sched_switch" ftrace_events: "power/cpu_frequency"
    ftrace_events: "sched/sched_wakeup" atrace_categories: "view" atrace_categories: "am"
  }}
}
duration_ms: 10000
EOF
```

**Use cases for Perfetto**:
- **Cold start analysis**: See exactly which process/thread is active during each millisecond of startup
- **Cross-thread contention**: Visualize when the JS thread is waiting for native, or vice versa
- **System-level bottlenecks**: I/O waits, CPU frequency scaling, GC pauses across all threads
- **Binder transactions**: Measure IPC overhead between your app and system services

### iOS Native CPU Profiling

**Tool: Xcode Instruments — Time Profiler**

1. **Product → Profile** (⌘I) in Xcode
2. Select **Time Profiler** template
3. Record during the slow interaction
4. Analyze the call tree

**Key insights**:
- Sort by **Self Weight** to find functions consuming the most CPU time
- Use **Invert Call Tree** to see the heaviest leaf functions first
- Filter by thread to isolate JS thread vs main thread issues
- Look for Objective-C message dispatch overhead in hot paths

**Tool: Xcode Energy Log**

For battery drain investigation:
1. Run app on a physical device connected to Xcode
2. Open **Debug Navigator → Energy Impact**
3. Watch the energy gauge over time
4. Spikes indicate CPU-intensive operations

**Common RN energy drains**:
- Hidden animations running when off-screen (Discord found 10% CPU from a spinning animation in a closed drawer)
- Aggressive polling intervals (`setInterval` at 1-5 second intervals)
- Location tracking with high accuracy when approximate would suffice
- WebSocket keepalive pings at unnecessarily high frequency

---

## Flashlight: The Swiss Army Knife

[Flashlight](https://github.com/bamlab/flashlight) by BAM deserves special mention as the most accessible profiling tool for React Native — it measures FPS, CPU, and RAM on real devices with zero SDK installation.

### When to Use Flashlight

- **Before/after benchmarking**: Measure the impact of any optimization
- **CI regression testing**: Catch performance regressions nightly
- **List performance**: Scroll benchmarks on real devices
- **Cross-device comparison**: Same test on high-end and low-end devices

### Setup

```bash
# One-time install
npm install -g @bamlab/flashlight

# Manual measurement (interactive)
npx @bamlab/flashlight measure

# Automated with Maestro (for CI)
npx @bamlab/flashlight test --testCommand "maestro test flow.yaml"

# Compare two measurements
npx @bamlab/flashlight report compare baseline.json optimized.json
```

### CI Integration Pattern

```yaml
# .github/workflows/perf.yml
name: Nightly Performance
on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily
jobs:
  benchmark:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npx expo prebuild
      - run: npx @bamlab/flashlight test --testCommand "maestro test e2e/scroll-benchmark.yaml" --output results.json
      - run: npx @bamlab/flashlight report compare baseline.json results.json --threshold 10
```

**Threshold**: Alert if FPS drops >10% or CPU increases >10% from baseline.

> **See also**: [Profiling & Debugging: Flashlight](../optimization/profiling-debugging.md#flashlight) for more details | [Reading List #11-12](../reading-list.md) for BAM's articles

---

## Use Case Playbooks

### Use Case 1: "My app gets slow after 5 minutes"

**Likely cause**: Memory leak (JS or native)

**Playbook**:
1. Open Hermes heap snapshots → take baseline snapshot
2. Use the app for 5 minutes (trigger the reported slow path)
3. Take second snapshot → compare retained sizes
4. If JS heap grew significantly → find the un-cleaned listener/timer/fetch
5. If JS heap is stable but app is still slow → escalate to Android Studio / Xcode native memory profiler
6. Check graphics memory separately (images, video thumbnails)

### Use Case 2: "Scrolling is janky on Android"

**Likely cause**: Too many views, slow render, or main thread contention

**Playbook**:
1. Run Flashlight → measure FPS during scroll on a low-end device
2. If FPS < 50 → check if using FlatList (switch to FlashList)
3. If already FlashList → React DevTools Profiler to check re-render count per scroll
4. If re-renders are fine → Android Studio CPU Profiler → check main thread for blocking calls
5. If main thread has long `nativeFabricUIManager` calls → too many views → simplify item layout

### Use Case 3: "Cold start takes 3+ seconds"

**Likely cause**: Too much synchronous initialization, large bundle, or no baseline profiles

**Playbook**:
1. Perfetto startup trace → identify where time is spent
2. Check bundle size (`EXPO_ATLAS=true bunx expo start`) → if >15MB, audit dependencies
3. Check `MainApplication.kt` / `AppDelegate` for synchronous SDK initialization → move to background
4. Check if baseline profiles are configured → can reduce cold start 25-30%
5. If on New Architecture → verify Hermes bytecode precompilation is working
6. Consider lazy loading screens not needed at startup (`lazy: true` on navigators)

### Use Case 4: "Battery drain reported by users"

**Likely cause**: Continuous CPU work when app should be idle

**Playbook**:
1. Flashlight → measure CPU % when app is idle (should be <2%)
2. If >5% idle → check for hidden animations (spinning loaders, progress indicators off-screen)
3. Check for aggressive `setInterval` / polling (reduce frequency or use push notifications)
4. Xcode Energy Log → check for location tracking, background audio, or network keepalives
5. Check for `useNativeDriver: false` animations that keep JS thread active

### Use Case 5: "Crashes only on low-end devices"

**Likely cause**: Memory pressure (OOM or native memory corruption)

**Playbook**:
1. Check crash reports → is it SIGKILL (OOM) or SIGSEGV (memory corruption)?
2. If OOM → Android Studio Memory Profiler on a 3-4GB RAM device → watch peak memory
3. If peak memory >300MB → audit image sizes, video buffers, cached data
4. If SIGSEGV → enable ASan build → reproduce → ASan will pinpoint the exact line
5. Cross-reference with [SIGABRT Debugging Guide](./crash-analysis.md) for root cause categories

---

## Profiling Anti-Patterns

Avoid these common mistakes when profiling:

| Anti-Pattern | Why It's Wrong | Do This Instead |
|-------------|---------------|-----------------|
| Profiling in debug mode only | Debug builds are 5-10x slower, different behavior | Profile release builds (Flashlight, Sentry Performance) |
| Profiling on emulator | Emulators don't reflect real device performance | Always profile on physical devices, especially low-end |
| Optimizing without baseline | No way to know if changes helped | Take before/after measurements with Flashlight |
| Profiling on high-end device only | Performance issues hide on fast hardware | Test on $50 Android phones (2-3GB RAM) |
| Fixing symptoms instead of root cause | "Add more memoization" without understanding why | Use profiler to find the ACTUAL bottleneck first |
| Over-profiling everything | Diminishing returns, wasted time | Profile only when there's a user-reported or measured problem |

---

## Checklist: When to Profile

### Before Every Release
- [ ] Measure cold start TTI on a low-end device
- [ ] Measure scroll FPS on list-heavy screens
- [ ] Check memory stability: navigate back/forth 20+ times
- [ ] Run Flashlight benchmark and compare to previous release

### When Users Report Issues
- [ ] Reproduce the issue with profiling tools attached
- [ ] Identify which layer (JS / Bridge / Native) is the bottleneck
- [ ] Take before measurement → fix → take after measurement
- [ ] Monitor crash-free rate for 48-72 hours post-fix

### After Major Changes
- [ ] After upgrading React Native version → full profiling pass
- [ ] After New Architecture migration → memory + CPU + ANR monitoring
- [ ] After adding a new native dependency → check memory and startup impact
- [ ] After bundle size change >1MB → startup profiling

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Native-Layer Debugging Guide](./native-layer-debugging.md) | Step-by-step walkthroughs for Android Studio, Xcode Instruments, Perfetto, ASan |
| [SIGABRT & libc Debugging Guide](./crash-analysis.md) | Root cause framework for native crashes found during profiling |
| [Profiling & Debugging](../optimization/profiling-debugging.md) | JS-level profiling (React DevTools, Flashlight, bundle analysis, Discord case studies) |
| [Performance & Rendering](../optimization/performance-rendering.md) | Optimization patterns to apply after profiling identifies the bottleneck |
| [Monitoring & ANR Analysis](../optimization/monitoring-anr-analysis.md) | Production monitoring that tells you WHEN to profile |

---

**See also**: [Native-Layer Debugging Guide](./native-layer-debugging.md) for detailed tool walkthroughs | [Monitoring](../optimization/monitoring-anr-analysis.md) for production metrics that trigger profiling

Sources:
- [Sentry: RN Performance Tactics](https://blog.sentry.io/react-native-performance-strategies-tools/)
- [Discord: Native iOS Performance](https://discord.com/blog/how-discord-achieves-native-ios-performance-with-react-native)
- [Discord: Supercharging Mobile](https://discord.com/blog/supercharging-discord-mobile-our-journey-to-a-faster-app)
- [Callstack: Optimize Your RN JavaScript Bundle](https://www.callstack.com/blog/optimize-react-native-apps-javascript-bundle)
- [BAM: Measuring RN Performance with Flashlight](https://apps.theodo.com/article/measuring-react-native-performance-with-flashlight)
- [Android Developer: Perfetto](https://developer.android.com/topic/performance/tracing)
- [Apple Developer: Instruments](https://developer.apple.com/instruments/)
