# Version Compatibility Matrix

> Last verified: March 2026. When your build fails or crashes only on certain setups, check here first.

[← Back to Index](./README.md)

---

## React Native × Expo SDK

| Expo SDK | React Native | Hermes | New Architecture | React | Node.js |
|----------|-------------|--------|-----------------|-------|---------|
| **SDK 55** | 0.79+ | Default | Default (mandatory from 0.82) | 19.x | 20+ |
| **SDK 54** | 0.76+ | Default | Default | 18.3 | 20+ |
| **SDK 52** | 0.76+ | Default | Default | 18.3 | 18+ |
| **SDK 51** | 0.74 | Default | Opt-in | 18.2 | 18+ |
| SDK 50 | 0.73 | Default | Opt-in | 18.2 | 18+ |
| SDK 49 | 0.72 | Default | Opt-in | 18.2 | 16+ |

**Key milestone**: RN 0.82 (Oct 2025) fully removed the old Bridge. New Architecture is the only option.

---

## Core Libraries

| Library | Min RN Version | New Arch Support | Notes |
|---------|---------------|-----------------|-------|
| **React Navigation** 7.x | 0.72+ | Yes | Fully compatible |
| **React Navigation** 6.x | 0.63+ | Yes (6.3+) | Still maintained |
| **Expo Router** v4 | SDK 52+ | Yes | `navigate()` behavior changed from v3 |
| **Expo Router** v3 | SDK 51+ | Yes | `navigate()` pops back to existing |
| **Reanimated** 4.x | 0.76+ | **Required** (Fabric only) | Must be on New Arch |
| **Reanimated** 3.x | 0.72+ | Yes | Works on both architectures |
| **Gesture Handler** 2.x | 0.68+ | Yes | Required for Reanimated gestures |
| **FlashList** 2.x | 0.72+ | Yes (JS-only recycling) | No native dependency in v2 |
| **FlashList** 1.x | 0.63+ | Partial | Native `AutoLayoutView` dependency |

---

## State Management & Data

| Library | Bundle Size | RN Compatibility | Notes |
|---------|------------|-----------------|-------|
| **Zustand** 5.x | ~1KB | Any RN version | MMKV persist middleware available |
| **Jotai** 2.x | ~2.5KB | Any RN version | Atomic state, async atoms |
| **Legend State** 3.x | ~5KB | 0.72+ | MMKV plugin for offline persistence |
| **Redux Toolkit** 2.x | ~15KB | Any RN version | RTK Query for server state |
| **TanStack Query** 5.x | ~12KB | Any RN version | `gcTime` replaces `cacheTime` in v5 |
| **React Hook Form** 7.x | ~9KB | Any RN version | Zod integration via `@hookform/resolvers` |

---

## Storage

| Library | Sync/Async | Encrypted | Max Value Size | Notes |
|---------|-----------|-----------|---------------|-------|
| **react-native-mmkv** 3.x | Sync | Optional (AES-256) | ~unlimited | 30x faster than AsyncStorage |
| **expo-secure-store** | Async | Always (Keychain/Keystore) | 2KB (iOS) | For auth tokens and secrets only |
| **@react-native-async-storage** 2.x | Async | No | ~6MB (default) | **Not for sensitive data** |
| **expo-sqlite** 14+ | Async | No (by default) | DB size limit | WAL mode, Drizzle ORM support |
| **WatermelonDB** 0.27+ | Sync (observable) | No | DB size limit | Lazy loading, native SQLite thread |

---

## Testing

| Tool | Type | RN Version | Notes |
|------|------|-----------|-------|
| **Jest** 30.x | Unit/Integration | Any | Native TS config support |
| **jest-expo** | Jest preset | SDK 49+ | Use `jest-expo/universal` for cross-platform |
| **@testing-library/react-native** 12.x | Integration | 0.72+ | `getByRole` queries preferred |
| **Maestro** 1.38+ | E2E (black-box) | Any | YAML tests, <1% flakiness |
| **Detox** 20.x | E2E (gray-box) | 0.72+ | Needs native build config |
| **Flashlight** 0.10+ | Performance | Any | Real-device FPS/CPU/RAM |

---

## Monitoring & Error Tracking

| Tool | JS Errors | Native Crashes | Performance | Bundle Impact |
|------|-----------|---------------|-------------|--------------|
| **Sentry** 6.x | Yes | Yes | Yes (traces) | ~200KB |
| **Crashlytics** (Firebase) | Yes (`recordError`) | Yes | Separate SDK | ~150KB |
| **Datadog** RUM | Yes | Yes | Yes | ~300KB |
| **Bugsnag** 8.x | Yes | Yes | Yes | ~180KB |

---

## Build & Animation

| Library | Bundle Size | GPU Accelerated | Notes |
|---------|------------|----------------|-------|
| **Reanimated** 3.x/4.x | ~150KB | Yes (UI thread) | Worklets, shared values |
| **Moti** 0.29+ | ~5KB (+ Reanimated) | Via Reanimated | Declarative API |
| **Lottie** 7.x | ~1.5MB | Partial | Heavy — consider Rive |
| **Rive** 8.x | ~850KB | Yes | State machines, lighter than Lottie |
| **@shopify/react-native-skia** | ~2MB | Yes (GPU) | 2D drawing, shaders |

---

## Common Version Conflicts

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Invariant Violation: TurboModuleRegistry` | Reanimated 3.x on New Arch without Fabric | Upgrade to Reanimated 4.x or enable Fabric |
| `Cannot find module 'react-native/Libraries/Renderer/...'` | React version mismatch | Match React version to RN (see table above) |
| `Duplicate class kotlin.collections...` | Multiple Kotlin versions in gradle | Add `ext.kotlinVersion` to `build.gradle` |
| `SDK version mismatch` | Expo SDK ≠ RN version | Use `npx expo install` to auto-resolve |
| `Unable to resolve module` after upgrade | Metro cache stale | `npx react-native start --reset-cache` |
| `libreanimated.so` crash | Reanimated version incompatible with RN | Match Reanimated to RN version |
| Pod install fails after RN upgrade | Cached Pods from old version | `cd ios && pod deintegrate && pod install` |
| `ViewPropTypes` deprecated warning | Library using removed API | Upgrade library or patch with `deprecated-react-native-prop-types` |

---

## Upgrade Checklist

When upgrading React Native or Expo SDK:

1. Check this matrix for compatible library versions
2. Run `npx expo install --check` to auto-fix Expo package versions
3. Run `npx expo-doctor` to diagnose config issues
4. Clean caches: Metro (`--reset-cache`), iOS (`pod deintegrate`), Android (`./gradlew clean`)
5. Build both platforms before merging
6. Monitor crash-free rate for 72 hours post-deploy

> **See also**: [Upgrading React Native](./optimization/upgrading-react-native.md) for the full upgrade guide

---

*Last updated: March 2026. Submit a PR if you find version info that's out of date.*
