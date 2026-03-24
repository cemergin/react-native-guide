# Chapter 6: Profiling & Debugging

> Measure, debug, and prevent regressions. Never optimize without data.

[← Performance](./performance-rendering.md) | [Index](../README.md) | [Reading List →](../reading-list.md)

**Keywords**: profiling, React DevTools, Hermes profiler, Flashlight, memory leak, bundle size, Metro, tree-shaking, Re.Pack, RAM bundle, startup, TTI, flamegraph, Perfetto, ndk-stack, ASan

---

<details>
<summary><strong>TL;DR</strong></summary>

- Profile first, optimize second — never guess at bottlenecks
- React DevTools Profiler for re-renders, Hermes Profiler for slow JS, Flashlight for device benchmarks
- Discord saved 3,500ms TTI with 5 specific fixes (regex, emoji, image loading, dimensions, hidden animation)
- Flashlight in CI catches performance regressions nightly
- React Compiler eliminates need for manual useMemo/useCallback (Expo SDK 54+)

</details>

## The Profiling Stack {#profiling-stack}

### JS-Level Tools

| Tool | What It Measures | When to Use |
|------|-----------------|-------------|
| **React DevTools Profiler** | Component render times, re-render causes | Finding unnecessary [re-renders](./performance-rendering.md#memoization) |
| **Hermes Profiler** | JS function execution times | Finding slow JS functions |
| **Flashlight (BAM)** | FPS, CPU, RAM on real devices | Benchmarking before/after changes |
| **Sentry Performance** | Production TTI, transaction traces | [Monitoring](./monitoring-anr-analysis.md) real users |
| **React Compiler** | Auto-memoization | Eliminating re-render bugs at compile time |

### Native-Level Tools

For crashes below the JS layer, see the dedicated guides:

| Guide | Covers |
|-------|--------|
| [Native-Layer Debugging Guide](../debugging/native-layer-debugging.md) | Android Studio profiler, Xcode Instruments, Perfetto, heapprofd, ndk-stack, Tracy |
| [SIGABRT/libc.so Guide](../debugging/crash-analysis.md) | Root cause categories, decision framework, ASan, OEM-specific crashes |

**Key native tools**: Android Studio Memory/CPU Profiler, Xcode Instruments (Time Profiler, Leaks), Perfetto, ndk-stack. Full details in [native guide](../debugging/native-layer-debugging.md#6-tool-summary-matrix).

---

## Finding Slow Re-renders {#finding-slow-re-renders}

### Step 1: Enable "Highlight Updates"

Press `j` in Metro → React DevTools → Profiler tab → Enable "Highlight updates when components render."

### Step 2: Record a Profiler Session

1. Click "Record"
2. Perform the slow interaction
3. Stop recording
4. Check "Record why each component rendered"

### Step 3: Read the Flamegraph

- **Tall flames** = deep tree re-rendering (parent causing children to re-render)
- **Wide flames** = single slow component
- **Grey bars** = didn't re-render (good)

### Step 4: JS Profiler for Non-Obvious Cases

When React DevTools shows re-renders but cause isn't clear → use Hermes Profiler or Chrome DevTools Performance tab. Common culprits: regex compilation, JSON parsing, lodash on large arrays.

> **See also**: [Performance: Memoization](./performance-rendering.md#memoization) for fixes | [React Compiler](#react-compiler) to auto-fix

---

## Discord's Profiling Wins {#discord-wins}

Real examples from Discord's iOS optimization — specific numbers, specific fixes:

| # | Symptom | Root Cause | Fix | Impact |
|---|---------|-----------|-----|--------|
| 1 | 500ms message load (XS), 1000ms (6s) | Inline regex compilation in markdown parser | Made regex a `const` | 30% reduction |
| 2 | Still slow after regex fix | 400+ Unicode symbols for emoji detection | Optimized single regex | **90% total reduction, -1s TTI** |
| 3 | 50ms per image load after backgrounding | `UIImage.imageNamed()` using absolute paths | Custom native `LocalAssetImageLoader` | 0.3ms/image (vs 50ms) **-2s TTI** |
| 4 | Three commit passes at startup | `Dimensions` module triggering multiple layout passes | Single read, not reactive | -150ms TTI |
| 5 | 10% CPU while drawer closed | Hidden spinning animation | Removed animation | -10% CPU, better battery |

**Total TTI improvement**: -3,500ms on iPhone 6, with only 3 iOS engineers.

> **See also**: [Performance: Animations](./performance-rendering.md#animations) for `useNativeDriver` | [SIGABRT guide](../debugging/crash-analysis.md) for native image loading issues

---

## Startup Optimization {#startup-optimization}

### Code-Splitting with Expo Router

```json
{ "expo": { "plugins": [["expo-router", { "asyncRoutes": { "default": "development" } }]] } }
```

### Bundle Analysis with Expo Atlas

```bash
EXPO_ATLAS=true bunx expo start
```

### RAM Bundles (Discord's Approach) {#ram-bundles}

Discord deferred ~100 of 2,600 modules:
- Modals/action sheets → dynamic imports
- Desktop-only features → stubbed out
- Native modules (media engine) → dynamic imports
- Localization (moment.js) → deferred

**Result**: -800ms TTI on iPhone 6.

### Lazy Navigation

If no navigator in your app uses `lazy: true`, adding `screenOptions={{ lazy: true }}` to stack navigators reduces initial memory footprint. See the [SIGABRT guide](../debugging/crash-analysis.md) for details on this pattern.

### Background Initialization

If your `MainApplication.kt` (Android) loads Firebase, analytics, or other SDKs synchronously at startup, consider moving those initializations to background threads. See the [SIGABRT guide](../debugging/crash-analysis.md) for the pattern.

> **See also**: [Native Debugging: Perfetto startup trace](../debugging/native-layer-debugging.md#23-perfetto--deep-system-tracing) for native-layer startup analysis | [Native Debugging: Baseline Profiles](../debugging/native-layer-debugging.md#53-android-baseline-profiles--faster-cold-start) for 25-30% cold start improvement

---

## Bundle Size Optimization {#bundle-size}

### Analyze Your Bundle

```bash
# Expo
EXPO_ATLAS=true bunx expo start

# Generic RN
npx react-native-bundle-visualizer
```

### Alternative Bundlers

| Bundler | Tree-shaking | Hermes | Best For |
|---------|-------------|--------|----------|
| **Metro** (default) | No | Yes | Most apps |
| **Re.Pack** (Webpack) | Yes | Yes | Large apps, code splitting |
| **rnx-kit** (Microsoft) | Yes (beta) | Yes | Metro + tree-shaking (0-20% reduction) |

**Key insight**: Re.Pack code splitting is irrelevant with Hermes — Hermes memory-maps bytecode and reads only needed portions from RAM. The bundle is already lazy-loaded at the bytecode level.

### Callstack's Results

- **78% bundle size reduction** (50MB → 11MB) in a case study
- See their [free ebook](https://www.callstack.com/ebooks/the-ultimate-guide-to-react-native-optimization)

> **See also**: [Dependencies: Lighter Alternatives](./dependency-management.md#lighter-alternatives) for library swaps that reduce bundle | [Dependencies: Depcheck](./dependency-management.md#auditing) for removing unused packages

---

## Flashlight: Lighthouse for Mobile {#flashlight}

[Flashlight](https://github.com/bamlab/flashlight) by BAM — measures real-device performance:

```bash
npx @bamlab/flashlight measure                                    # Manual
npx @bamlab/flashlight test --testCommand "maestro test flow.yaml" # CI
```

**Measures**: FPS, CPU (all threads), RAM
**Advantages over Flipper**: Works in production builds, no SDK needed, more metrics
**CI integration**: Run nightly with Maestro flows to catch performance regressions

> **See also**: [Performance: List Checklist](./performance-rendering.md#lists) for what to measure | [Reading List #28](../reading-list.md) for nightly E2E + perf test setup

---

## Memory Leak Detection {#memory-leaks}

### Common Patterns

```tsx
// 1. LEAK: Uncleared event listener
useEffect(() => {
  const sub = EventEmitter.addListener('event', handler);
  return () => sub.remove();  // FIX: cleanup
}, []);

// 2. LEAK: Unaborted fetch
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/data', { signal: controller.signal }).then(setData);
  return () => controller.abort();  // FIX: abort
}, []);

// 3. LEAK: Uncleared interval
useEffect(() => {
  const id = setInterval(pollData, 5000);
  return () => clearInterval(id);  // FIX: clear
}, []);
```

### Symptoms

- App runs fine for 5 minutes, then slows/crashes
- Memory climbs steadily without plateau
- Back/forth navigation increases memory each time
- Background app killed by OS frequently

### Detection Tools

| Level | Tool |
|-------|------|
| JS heap | Hermes Profiler (heap snapshots before/after navigation) |
| iOS native | Xcode Instruments → Leaks, Memory Graph Debugger |
| Android native | Android Studio Memory Profiler, heapprofd |
| Cross-platform | React DevTools (mount/unmount tracking) |

For deep native memory debugging, see [Native Debugging Guide: Memory Profiler](../debugging/native-layer-debugging.md#21-memory-profiler--native-heap-analysis) and [Xcode Leaks](../debugging/native-layer-debugging.md#32-instruments--leaks--allocations).

### OOM Patterns to Watch For

Graphics memory can spike dramatically during scrolling — e.g., jumping from 70MB to 472MB (7x increase) due to autoplay videos and animated GIFs. Product-level fixes (disable autoplay, pause off-screen media) are often more effective than code-level optimization.

> **See also**: [Monitoring](./monitoring-anr-analysis.md) for production crash patterns | [SIGABRT guide](../debugging/crash-analysis.md) for memory profile analysis

---

## React Compiler (Expo SDK 54+) {#react-compiler}

Auto-memoizes components — eliminates need for manual `useMemo`/`useCallback`:

```json
{ "expo": { "experiments": { "reactCompiler": true } } }
```

> **See also**: [Performance: Memoization](./performance-rendering.md#memoization) for manual patterns when React Compiler isn't available

---

## Checklists

### Before Optimizing
- [ ] Profile first — never optimize without data
- [ ] Measure baseline: cold start TTI, FPS during scroll, memory during navigation
- [ ] Set up Flashlight or Sentry Performance for ongoing [monitoring](./monitoring-anr-analysis.md)

### Quick Wins
- [ ] Move regex/constant creation outside components
- [ ] Add `AbortController` to all fetch calls in `useEffect`
- [ ] Clean up event listeners, intervals, WebSocket connections
- [ ] Enable React Compiler if on Expo SDK 54+
- [ ] Audit [lodash usage](./dependency-management.md#lighter-alternatives) — replace with native JS

### Deep Optimization
- [ ] Profile with React DevTools ("Record why each component rendered")
- [ ] Use Hermes Profiler for non-obvious JS bottlenecks
- [ ] Consider RAM bundles for apps with 1000+ modules
- [ ] Run Flashlight benchmarks in CI
- [ ] Test memory stability: navigate back/forth 20+ times
- [ ] Use [Perfetto](../debugging/native-layer-debugging.md#23-perfetto--deep-system-tracing) for native startup analysis
- [ ] Use [ASan](../debugging/native-layer-debugging.md#34-sanitizers-asan-tsan-ubsan) for memory corruption bugs

---

> **See also**: [Profiling Tools Deep Dive](../debugging/profiling-tools-deep-dive.md) for comprehensive memory & CPU profiling decision framework

**Next**: [Reading List →](../reading-list.md) — 28 curated articles

Sources:
- [Sentry: RN Performance Tactics](https://blog.sentry.io/react-native-performance-strategies-tools/)
- [Discord: Native iOS Performance](https://discord.com/blog/how-discord-achieves-native-ios-performance-with-react-native)
- [Callstack: Optimize Your RN JavaScript Bundle](https://www.callstack.com/blog/optimize-react-native-apps-javascript-bundle)
- [Flashlight GitHub](https://github.com/bamlab/flashlight)
