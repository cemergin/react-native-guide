# React Native & Expo Cheat Sheet

> Pin this in Slack. Print it. Bookmark it. Everything you need on one page.

---

## The 2026 Stack

| Layer | Default | Alternative |
|-------|---------|------------|
| Framework | Expo SDK 52+ | Bare RN CLI |
| Navigation | Expo Router v4 | React Navigation 7 |
| Client State | Zustand (~1KB) | Jotai (atomic), Legend State (offline) |
| Server State | TanStack Query v5 | RTK Query, SWR |
| Forms | React Hook Form + Zod | Formik |
| Storage | react-native-mmkv | expo-sqlite (relational) |
| Styling | NativeWind | Tamagui, Unistyles |
| Animations | Reanimated v3+ | Moti, Animated API |
| Lists | FlashList v2 | FlatList (simple cases) |
| Build | EAS Build | Fastlane, GitHub Actions |
| Testing | Jest + RNTL + Maestro | Detox (gray-box E2E) |
| Monitoring | Sentry | Crashlytics, Datadog |

---

## Quick Commands

```bash
# Bundle analysis
EXPO_ATLAS=true bunx expo start          # Then Shift+M → Open expo-atlas

# Performance benchmark (real device)
npx @bamlab/flashlight measure            # Manual
npx @bamlab/flashlight test --testCommand "maestro test flow.yaml"  # CI

# E2E test
maestro test e2e/login-flow.yaml

# Check if OTA or binary needed
npx @expo/fingerprint                     # Hash changed → binary, same → OTA

# EAS commands
eas build --platform all                  # Build
eas submit --platform all                 # Submit to stores
eas update --branch production            # OTA update

# Dependency audit
depcheck --ignores="@types/*,react-native-*,expo-*"
yarn why <package-name>

# Debug tools
j                                         # In Metro → open React DevTools
```

---

## Profiling Decision Matrix

| Symptom | First Tool | Escalation |
|---------|-----------|-----------|
| UI jank during scroll | React DevTools Profiler | Systrace / Perfetto |
| Slow screen transitions | Hermes CPU Profiler | Flamegraph analysis |
| App slows over 5+ min | Hermes heap snapshots | Android Studio / Xcode Leaks |
| OS kills app in background | Android Studio Memory Profiler | heapprofd |
| High battery drain | Flashlight (CPU %) | Xcode Energy Log |
| Slow cold start (>2s) | Perfetto startup trace | Baseline Profiles |
| Crashes on low-end devices | Android Studio Memory Profiler | ASan |
| Animation frame drops | Systrace / Perfetto | Tracy (C++ internals) |

**Rule of thumb**: Start at JS layer. If JS thread is idle during the slowdown, escalate to native tools.

---

## State Management Decision

| Scenario | Use |
|----------|-----|
| Global UI state (theme, auth, sidebar) | Zustand |
| Complex interdependent filters | Jotai |
| Offline-first with sync | Legend State |
| API data (caching, refetching) | TanStack Query |
| Form input state | React Hook Form |
| 2-3 simple values | React Context |
| Legacy enterprise codebase | Keep Redux, adopt TQ for new |

---

## Testing Decision

| What | How | When |
|------|-----|------|
| Pure functions | Jest unit tests | Every PR |
| Component behavior | RNTL integration tests | Every PR |
| Critical user flows | Maestro E2E (YAML) | Merge to main |
| Performance regression | Flashlight + Maestro in CI | Nightly |
| Accessibility | RNTL role queries + VoiceOver | Per feature |

**Query priority**: `getByRole` > `getByText` > `getByPlaceholderText` > `getByTestId`

---

## Error Handling Layers

| Layer | Mechanism | Catches |
|-------|-----------|---------|
| 1. Local | try/catch, React Query | Expected errors |
| 2. Boundary | Error Boundaries (per screen) | Render crashes |
| 3. Global JS | `ErrorUtils.setGlobalHandler` | Uncaught exceptions |
| 4. Native | `react-native-exception-handler` | SIGSEGV, EXC_BAD_ACCESS |
| 5. Production | Sentry / Crashlytics | Everything in prod |

---

## Security Essentials

| Do | Don't |
|----|-------|
| Store tokens in `expo-secure-store` | Store tokens in AsyncStorage |
| Keep secrets on server | Ship API keys in bundle |
| Use MMKV with encryption key | Use unencrypted MMKV for secrets |
| Certificate pin production API | Trust all certificates |
| Enable Hermes + R8/ProGuard | Ship debug builds |
| Use `expo-local-authentication` for biometrics | Roll your own biometric flow |

---

## Performance Quick Wins

```tsx
// 1. Always use native driver for animations
Animated.timing(opacity, { toValue: 1, useNativeDriver: true }).start();

// 2. FlashList over FlatList
<FlashList data={items} renderItem={renderItem} getItemType={item => item.type} />

// 3. React Compiler (auto-memoization) — Expo SDK 54+
// app.json: { "expo": { "experiments": { "reactCompiler": true } } }

// 4. Abort fetch on unmount
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal }).then(setData);
  return () => controller.abort();
}, []);

// 5. Lazy navigation screens
<Tabs screenOptions={{ lazy: true }}>
```

---

## Key Metrics

| Metric | Target | Tool |
|--------|--------|------|
| Crash-free rate | >99.8% | Crashlytics / Sentry |
| Cold start TTI | <2s | Perfetto, Flashlight |
| Scroll FPS | 60fps | Flashlight |
| Bundle size | <15MB | Expo Atlas |
| Memory stability | Flat after 20 nav cycles | Hermes heap snapshots |

---

## Production Stats (Real Companies)

| Company | Achievement |
|---------|-----------|
| Shopify | 99.9% crash-free, 78% bundle reduction |
| Discord | -3,500ms TTI, 90% faster message parse |
| A Million Monkeys | -24% Android cold start in 12 days |

---

*Full guides: [README.md](./README.md) · [Reading List](./reading-list.md)*
