# Chapter 1: Monitoring & ANR Analysis

> **Start here.** Before optimizing anything, understand where and why your app fails.

[Index](./README.md) | Next: [Upgrading RN →](./02-upgrading.md)

**Keywords**: ANR, crash, Crashlytics, Sentry, Google Play Console, crash-free rate, device segmentation, monitoring, tombstone, SIGABRT, SIGSEGV

---

## Why Analyze First?

Optimizing without data is guesswork. Crash and ANR analysis tells you:
- Which screens/features cause the most user pain
- Whether issues are code-level, library-level, or device-level
- Where to invest engineering time for maximum impact

**Real example**: In a production crash analysis, 381 of 428 crashes were a single unsymbolicated `libc.so` native crash — uploading NDK symbols was the highest-leverage action, not code changes.

## Monitoring Tools {#tools}

| Tool | Best For | Key Strength |
|------|----------|-------------|
| **Firebase Crashlytics** | Real-time crash reporting | Session data, root cause clustering, NDK tombstones |
| **Sentry** | Stack traces, error context | AI-suggested fixes, real-time alerts |
| **Google Play Console** | Device/OS segmentation | Android vitals, Gemini AI crash analysis |
| **Datadog** | Error tracking + APM | Full-stack correlation |
| **Mixpanel** | Behavior correlation | Connect crashes to user journeys |

At minimum: **crash reporting** (Crashlytics or Sentry) + **device analytics** (Play Console).

> **See also**: [Profiling & Debugging](./06-profiling.md#profiling-stack) for dev-time profiling tools, vs these production monitoring tools.

## Analysis Framework {#analysis-framework}

### Step 1: Segment by Dimension

Break crashes down across three axes:

- **App version** — Did a specific release introduce a regression?
- **Android/iOS version** — Is SDK 33 crashing more than SDK 34?
- **Device model** — Are low-RAM (4GB) devices disproportionately affected?

**Real example**: In a production crash analysis, one issue spanned Android 9-16 with no OS specificity — pointing to memory pressure, not an OS bug. Another was confined to Android 13 only — an AOSP bug.

### Step 2: Identify Patterns {#device-segmentation}

| Crash Type | Likely Cause | Action |
|-----------|-------------|--------|
| Native crash (`libc.so`, `__strlen_aarch64`) | Native module or system-level bug | See [SIGABRT debugging guide](../SIGABRT-libc-debugging-guide.md) |
| JS crash with stack trace | Application code error | Fix directly |
| ANR (no crash, just freeze) | Main thread blocked | Check [animations](./05-performance.md#animations), heavy computation |
| Device-cluster crash (80%+ one OEM) | Hardware/OEM constraint | See [SIGABRT guide: OEM crashes](../SIGABRT-libc-debugging-guide.md#category-d-oem-specific-crashes-huawei-xiaomi-samsung) |
| All versions crash simultaneously | Google Play System update | See [SIGABRT guide: Category E](../SIGABRT-libc-debugging-guide.md#category-e-google-play-system-updates-os-level-bug) |
| Startup crash (<1 second) | Native init, memory pressure | See [native debugging](../native-layer-debugging-guide.md#21-memory-profiler--native-heap-analysis) |

### Step 3: Prioritize by User Impact

- Fix crashes affecting the **most users first**
- Device-specific issues affecting <1% → **graceful fallbacks** rather than deep fixes
- Track **crash-free user rate** as your north-star metric (target: >99.8%)

**Real example from Shopify**: They use 99.80% crash-free as the "fix forward" threshold and rollback below 99.00%. See [New Architecture: Rollout strategy](./03-new-architecture.md#gradual-rollout).

### Step 4: Monitor Your Fixes {#monitor-fixes}

After deploying a fix:
1. Watch the crash rate for that specific issue over **48-72 hours**
2. Set up **regression alerts** so new releases don't reintroduce old crashes
3. Compare crash-free rates between app versions
4. For OTA fixes, verify via `eas update --branch production` that the fix reaches users

## Native Crash Debugging

For crashes below the JavaScript layer (`libc.so`, `libhermes.so`, `libhwui.so`):

| Resource | Covers |
|----------|--------|
| [SIGABRT/libc.so Debugging Guide](../SIGABRT-libc-debugging-guide.md) | Root cause categories, decision framework |
| [Native-Layer Debugging Guide](../native-layer-debugging-guide.md) | Android Studio profiler, Xcode instruments, Perfetto, ndk-stack |
| [Profiling: Memory Leaks](./06-profiling.md#memory-leaks) | JS-level memory leak patterns and detection |
| [New Architecture: Common Pitfalls](./03-new-architecture.md#common-pitfalls) | Migration-specific crashes (Blank Screen of Doom, shadow tree desync) |

### SIGABRT Decision Tree

From the [SIGABRT guide](../SIGABRT-libc-debugging-guide.md#6-decision-framework):

```
SIGABRT/SIGSEGV crash detected
│
├── Stack trace has specific .so file? → Identify library → upgrade
├── Crash-free rate dropped across ALL versions? → Google Play System update
├── 80%+ crashes on one OEM? → OEM bug → try Hermes, disable HW accel
├── Started after a specific release? → Binary search dependencies
├── Push notification launch crash? → PlatformConstants issue (RN 0.76+)
├── Memory profiling shows excessive growth? → OOM path
└── Run with ASan → identify specific memory corruption
```

## Google Play Console Workflow

1. Navigate to **Monitor and improve > Android vitals > Crashes and ANRs**
2. Filter by **app version** to isolate regressions
3. Check the **device** tab for hardware-specific clusters
4. Use the **Android version** distribution to prioritize OS-level fixes
5. Cross-reference with Crashlytics clusters for root cause

> **Pro tip from [SIGABRT guide](../SIGABRT-libc-debugging-guide.md)**: Google Play Console often has more complete crash logs than Crashlytics — it includes the "Thread terminating due to..." message that Crashlytics may truncate.

## Checklist

- [ ] Crash reporting SDK configured (Crashlytics, Sentry, Datadog, etc.)
- [ ] NDK debug symbols uploaded — see [SIGABRT guide](../SIGABRT-libc-debugging-guide.md)
- [ ] Crash-free rate dashboard visible to the team
- [ ] Regression alerts configured for new releases
- [ ] Device segmentation reviewed monthly
- [ ] Non-actionable issues muted to reduce noise

---

**Next**: [Upgrading React Native →](./02-upgrading.md) — eliminate platform-level issues before optimizing code
