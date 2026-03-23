# Chapter 3: New Architecture Deep Dive

> Synthesized from production migration stories at Shopify, Discord, and A Million Monkeys.

[← Upgrading](./02-upgrading.md) | [Index](./README.md) | [Dependencies →](./04-dependencies.md)

**Keywords**: JSI, Fabric, TurboModules, New Architecture, migration, bridge, codegen, NativeModules, state batching, view flattening, rollout, Reanimated

---

## What Changed

The New Architecture replaces the async JS Bridge with **JSI** (JavaScript Interface) — synchronous, direct JS↔Native communication via C++. No more JSON serialization overhead.

| Old Architecture | New Architecture |
|-----------------|-----------------|
| JS → Bridge (async, serialized JSON) → Native | JS → JSI (synchronous, C++ references) → Native |
| Single-threaded bridge bottleneck | Fabric concurrent renderer |
| `NativeModules` API | TurboModules (codegen'd TypeScript interfaces) |

As of **RN 0.82** (Oct 2025), the old bridge is fully removed. New Architecture is the only option.

## Real-World Performance Gains

### A Million Monkeys (mid-size production app, iPhone 13 / Pixel 6)

| Metric | iOS Before → After | Android Before → After |
|--------|-------------------|----------------------|
| Cold start | 1,840ms → 1,520ms (**17% faster**) | 2,340ms → 1,780ms (**24% faster**) |
| Screen navigation | 180ms → 145ms (**19% faster**) | 245ms → 180ms (**27% faster**) |
| 60fps frame rate | 73% → 89% of frames | 61% → 84% of frames |
| Peak memory | 142MB → 128MB (**-10%**) | 187MB → 159MB (**-15%**) |

### Shopify (massive production apps, millions of merchants)

| Metric | Improvement |
|--------|------------|
| App launch (Android) | **10% faster** |
| App launch (iOS) | **3% faster** |
| Tab switching | Snappier (batched state reduces re-renders) |
| Some screens post-migration | **Up to 20% slower** (required re-optimization) |

**Key insight from Shopify**: "Not everything got faster automatically. Some screens that relied on timing assumptions in the old bridge actually regressed and needed rework."

### Where You WON'T See Improvement

- Simple forms and static content
- Isolated API requests
- Screens with minimal native interaction

## Migration Playbook

### Timeline Estimates

| App Size | Estimated Time |
|----------|---------------|
| Small (10 screens, few native modules) | 3-5 days |
| Medium (20-30 screens) | 1-2 weeks |
| Large (40+ screens, custom native code) | 2-4 weeks |
| Add 30% if >2 RN versions behind | — |

*Source: A Million Monkeys (team of 2 developers, 12 days for medium app)*

### Step 1: Enable New Architecture

**iOS** — in `Podfile`:
```ruby
ENV['RCT_NEW_ARCH_ENABLED'] = '1'
```

**Android** — in `gradle.properties`:
```properties
newArchEnabled=true
hermesEnabled=true
```

### Step 2: Audit Dependencies

Out of every ~8 native dependencies, expect:
- ~3 need version upgrades
- ~1 needs replacement (unmaintained)
- ~1 needs custom code changes
- ~3 work without changes

**Libraries that worked immediately** (from A Million Monkeys):
- React Navigation v6.3+
- Zustand, React Query
- Reanimated v3.6+

**Libraries that needed replacement**:
- `react-native-camera` → `react-native-vision-camera`
- `AsyncStorage` → `react-native-mmkv`
- Firebase v18 → v20 (breaking config changes)

### Step 3: Fix Breaking Patterns

**State batching** — New Architecture batches state updates. Code relying on intermediate state values will break:
```tsx
// BREAKS: relies on setState being processed between calls
setState(a);
// code that reads state.a here won't see the update yet
setState(b);

// FIX: combine into single update or use useEffect
setState({ a, b });
```

**Native module threading** — Set `requiresMainQueueSetup` to `false` unless strictly necessary. Legacy modules loading on main thread during animations cause "App Hang" crashes on iOS.

**View flattening** — Components with refs may get optimized out, causing `null` refs. Add `collapsable={false}` to components that need refs (Shopify created a Babel plugin for this).

**PropTypes removal** — Convert ~40 components from PropTypes to TypeScript interfaces.

**ViewPropTypes** — Replace with `ViewStyle` type definitions.

### Step 4: Clean Build Environment

```bash
# iOS: Clean derived data
rm -rf ~/Library/Developer/Xcode/DerivedData/*

# Reset Metro cache
npx react-native start --reset-cache

# Android: Clean Gradle
cd android && ./gradlew clean
```

First iOS build after migration takes ~12 minutes (vs 3-4 normally). Subsequent builds are normal.

### Step 5: Gradual Rollout

Shopify's rollout schedule:
| Day | Android | iOS |
|-----|---------|-----|
| 1 | 8% | 0% |
| 2 | 30% | 1% |
| 3 | 100% | 100% |

**Stability thresholds**:
- Above 99.80% crash-free → fix forward on next release
- 99.00-99.80% → pause and hotfix
- Below 99.00% → rollback immediately

## Common Pitfalls

1. **"Blank Screen of Doom"** — TurboModules using old `UIManager` APIs cause complete rendering failure. Debug by progressively commenting out components until you isolate the culprit.

2. **Reanimated frame drops** — At Shopify's scale, severe frame rate drops occurred. Required patches from Software Mansion (Reanimated maintainers). Make sure you're on latest Reanimated before migrating.

3. **Shadow tree desync** — Native UIManager modifications break tap gestures. Solution: manage WebView lifecycle entirely natively, outside React's tree.

4. **ANR spikes post-migration** — Both platforms can show ANR increases from Reanimated/RN patches and main-thread deadlocks. Monitor closely for 48-72 hours.

5. **Post-migration screen regressions** — Some screens get slower because they relied on old bridge timing assumptions. Profile and fix after migration, don't block on perfection.

## Actionable Checklist

- [ ] Check current RN version — if >2 versions behind, plan intermediate upgrades
- [ ] Run `npx react-native-new-architecture-check` (if available) to audit compatibility
- [ ] Audit all native modules with `yarn why` for each native dependency
- [ ] Enable New Architecture in build configs
- [ ] Fix state batching issues (search for sequential `setState` calls)
- [ ] Convert `NativeModules` to TurboModules for high-frequency modules first
- [ ] Test on low-end devices (not just emulators)
- [ ] Set up crash-free rate monitoring before rollout
- [ ] Plan gradual rollout with clear rollback thresholds
- [ ] Monitor ANR rates for 72 hours post-deployment

---

## Your App Context

Before migrating, audit your own codebase for these common issues:
- **Files using `useNativeDriver: false`** — audit each for bridge contention
- **Outdated picker or input libraries** — upgrade to latest patch versions before migration
- **Reanimated version** — ensure v4.x+ and monitor for `libreanimated.so` in crash reports
- **No `lazy: true` on navigators** — adding it reduces initial memory, making migration smoother

Framework bugs like ReactHostImpl assertion errors and NativeAnimatedNodes race conditions are typically fixed in later RN versions — upgrading first avoids chasing already-fixed issues.

> **See also**: [Monitoring: ANR Analysis](./01-monitoring.md) for post-migration crash monitoring | [Dependencies](./04-dependencies.md) for library compatibility | [Profiling](./06-profiling.md) for measuring migration impact

---

**Next**: [Dependency Management →](./04-dependencies.md)

---

Sources:
- [Shopify: Migrating to New Architecture](https://shopify.engineering/react-native-new-architecture)
- [A Million Monkeys: What Actually Changed](https://www.amillionmonkeys.co.uk/blog/react-native-new-architecture-migration)
- [Shopify: Five Years of React Native](https://shopify.engineering/five-years-of-react-native-at-shopify)
- [Discord: Supercharging Mobile](https://discord.com/blog/supercharging-discord-mobile-our-journey-to-a-faster-app)
