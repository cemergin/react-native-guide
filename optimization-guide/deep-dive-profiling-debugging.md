# Deep Dive: Profiling & Debugging

> Synthesized from Sentry, BAM/Theodo (Flashlight), Discord, and Callstack.

[Back to Index](./README.md) | [Reading List](./reading-list.md)

---

## The Profiling Stack

| Tool | What It Measures | When to Use |
|------|-----------------|-------------|
| **React DevTools Profiler** | Component render times, re-render causes | Finding unnecessary re-renders |
| **Hermes Profiler** | JS function execution times | Finding slow JS functions |
| **Flashlight** | FPS, CPU, RAM on real devices | Benchmarking before/after changes |
| **Sentry Performance** | Production TTI, transaction traces | Monitoring real users |
| **Android Studio Profiler** | Native thread CPU, memory | Android-specific native issues |
| **Xcode Instruments** | Native performance, memory leaks | iOS-specific native issues |

## Step-by-Step: Finding Slow Re-renders

### 1. Enable "Highlight Updates" in React DevTools

Press `j` in Metro to open React DevTools. Enable "Highlight updates when components render" in the Profiler tab. This visually shows which components re-render on each interaction.

### 2. Record a Profiler Session

1. Click "Record" in the Profiler tab
2. Perform the interaction that feels slow
3. Stop recording
4. Check "Record why each component rendered"

### 3. Read the Flamegraph

- **Tall flames** = deep component tree re-rendering (likely a parent is causing children to re-render)
- **Wide flames** = single component taking a long time
- **Grey bars** = components that didn't re-render (good)

### 4. Use JavaScript Profiler for Non-Obvious Cases

When React DevTools shows re-renders but the cause isn't clear:
1. Open Chrome DevTools → Performance tab (or Hermes Profiler)
2. Record during the slow interaction
3. Look for expensive functions in the flame chart
4. Common culprits: regex compilation, JSON parsing, lodash operations on large arrays

## Discord's Profiling Wins (Specific Examples)

### Win 1: Regex in Message Parsing
- **Symptom**: 500ms message load on iPhone XS, 1000ms on iPhone 6s
- **Root cause**: Inline regex compilation in markdown parser — created new regex object every render
- **Fix**: Made regex a `const` outside the component
- **Result**: 30% reduction → 350ms (XS), 700ms (6s)

### Win 2: Unicode Emoji Detection
- **Symptom**: Still slow after regex fix
- **Root cause**: 400+ concatenated Unicode symbols for emoji detection, extremely expensive on JSC
- **Fix**: Replaced with optimized single regex
- **Result**: 30ms (XS), 90ms (6s) — **90% total reduction, nearly 1 second off TTI**

### Win 3: Image Loading
- **Symptom**: 50ms per `UIImage.imageNamed()` call after app backgrounding
- **Root cause**: React Native passing absolute paths instead of bundle references
- **Fix**: Custom native `LocalAssetImageLoader` that reads directly from disk + manages own cache
- **Result**: 0.3ms per image (vs 50ms) — **nearly 2-second TTI reduction**

### Win 4: Screen Dimensions
- **Symptom**: Three commit passes during app startup
- **Root cause**: Using React Native's `Dimensions` module triggered multiple layout passes
- **Fix**: Used Dimensions module correctly (single read, not reactive)
- **Result**: 150ms off TTI

### Win 5: Spinning Animation (Battery)
- **Symptom**: 10% main thread CPU usage while drawer was closed
- **Root cause**: Right drawer had a spinning animation running continuously
- **Fix**: Removed the animation
- **Result**: Recovered 10% CPU, measurable battery improvement

## Bundle Size Optimization

### Analyze Your Bundle

```bash
# With Expo
EXPO_ATLAS=true bunx expo start

# Generic RN
npx react-native-bundle-visualizer
```

### Alternative Bundlers to Metro

| Bundler | Tree-shaking | HMR | Hermes Support | Best For |
|---------|-------------|-----|----------------|----------|
| **Metro** (default) | No | Yes (Fast Refresh) | Yes | Most apps |
| **Re.Pack** (Webpack) | Yes | Limited | Yes | Large apps needing code splitting |
| **react-native-esbuild** | Yes | No (live reload only) | **No** | Fast builds, no Hermes |
| **rnx-kit** (Microsoft) | Yes (beta, via esbuild) | Yes | Yes | Metro + tree-shaking |

**Key insight from Callstack**: Re.Pack's code splitting doesn't help much with Hermes because Hermes uses memory mapping to read only necessary bytecode portions directly from RAM. The bundle is already lazy-loaded at the bytecode level.

### Callstack's Results
- **78% bundle size reduction** (50MB → 11MB) in a case study
- **rnx-kit tree-shaking**: 0-20% bundle reduction on top of Metro

### RAM Bundles (Discord's Approach)

Discord deferred ~100 of 2,600 modules using RAM bundles:
- Modal/action sheet/alert APIs → dynamic imports
- Desktop-only features → stubbed out
- Native modules (media engine) → dynamic imports
- Localization (moment.js) → deferred

**Result**: 800ms TTI reduction on iPhone 6, 3,500ms total TTI improvement

## Flashlight: Lighthouse for Mobile

[Flashlight](https://github.com/bamlab/flashlight) by BAM measures real-device performance:

```bash
# Install
npm install -g @bamlab/flashlight

# Measure on connected device
npx @bamlab/flashlight measure

# Run with Maestro flow for automated benchmarks
npx @bamlab/flashlight test --testCommand "maestro test flow.yaml"
```

**What it measures**: FPS, CPU (all threads), RAM usage
**Advantages over Flipper**: Works in production builds, no SDK installation, more metrics
**CI integration**: Run nightly with Maestro flows to catch performance regressions

## Memory Leak Detection

### Common Leak Patterns in React Native

1. **Uncleared event listeners**
```tsx
// LEAK
useEffect(() => {
  const sub = EventEmitter.addListener('event', handler);
  // Missing cleanup!
}, []);

// FIX
useEffect(() => {
  const sub = EventEmitter.addListener('event', handler);
  return () => sub.remove();
}, []);
```

2. **Unaborted fetch requests**
```tsx
// LEAK
useEffect(() => {
  fetch('/api/data').then(setData);
  // Component unmounts, but fetch response still calls setData
}, []);

// FIX
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/data', { signal: controller.signal }).then(setData);
  return () => controller.abort();
}, []);
```

3. **Uncleared intervals/timeouts**
```tsx
// LEAK
useEffect(() => {
  setInterval(pollData, 5000);
}, []);

// FIX
useEffect(() => {
  const id = setInterval(pollData, 5000);
  return () => clearInterval(id);
}, []);
```

4. **WebSocket connections**
```tsx
// FIX
useEffect(() => {
  const ws = new WebSocket(url);
  return () => ws.close();
}, []);
```

### Detection Tools

1. **Hermes Profiler** — take heap snapshots before/after navigation
2. **Xcode Instruments → Leaks** — for iOS native memory leaks
3. **Android Studio Profiler** — for Android native memory leaks
4. **Flipper → React DevTools** — track component mount/unmount

### Symptoms That Indicate a Leak

- App runs fine for 5 minutes, then slows/crashes
- Memory usage climbs steadily in profiler without plateau
- Navigating back and forth between screens increases memory each time
- Background app gets killed by OS more frequently than expected

## React Compiler (Expo SDK 54+)

Automatically optimizes components — no manual `useMemo`/`useCallback`:

```json
// app.json
{
  "expo": {
    "experiments": {
      "reactCompiler": true
    }
  }
}
```

This eliminates an entire class of re-render bugs by auto-memoizing at compile time.

## Actionable Checklist

### Before Optimizing
- [ ] Profile first — never optimize without data
- [ ] Measure baseline: cold start TTI, FPS during scroll, memory during navigation
- [ ] Set up Flashlight or Sentry Performance for ongoing monitoring

### Quick Wins
- [ ] Move regex/constant creation outside components
- [ ] Add `AbortController` to all fetch calls in `useEffect`
- [ ] Clean up all event listeners, intervals, WebSocket connections
- [ ] Enable React Compiler if on Expo SDK 54+
- [ ] Audit lodash usage — replace with native JS methods where possible

### Deep Optimization
- [ ] Profile with React DevTools Profiler ("Record why each component rendered")
- [ ] Use Hermes Profiler for non-obvious JS bottlenecks
- [ ] Consider RAM bundles for apps with 1000+ modules
- [ ] Run Flashlight benchmarks in CI to catch regressions
- [ ] Test memory stability by navigating back/forth 20+ times

---

Sources:
- [Sentry: RN Performance Tactics](https://blog.sentry.io/react-native-performance-strategies-tools/)
- [Sentry: New Way of RN Debugging](https://blog.sentry.io/the-new-way-of-react-native-debugging/)
- [Discord: Native iOS Performance](https://discord.com/blog/how-discord-achieves-native-ios-performance-with-react-native)
- [Callstack: Optimize Your RN JavaScript Bundle](https://www.callstack.com/blog/optimize-react-native-apps-javascript-bundle)
- [BAM: Measuring RN Performance with Flashlight](https://apps.theodo.com/article/measuring-react-native-performance-with-flashlight)
- [Flashlight GitHub](https://github.com/bamlab/flashlight)
