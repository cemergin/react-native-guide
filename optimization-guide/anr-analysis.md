# ANR Analysis & Crash Monitoring

> **Start here.** Before optimizing anything, understand where and why your app fails.

[Back to Index](./README.md) | Next: [Keeping RN Updated](./keeping-rn-updated.md)

---

## Why Analyze First?

Optimizing without data is guesswork. Crash and ANR analysis tells you:
- Which screens/features cause the most user pain
- Whether issues are code-level, library-level, or device-level
- Where to invest engineering time for maximum impact

## Monitoring Tools

| Tool | Best For | Key Strength |
|------|----------|-------------|
| **Sentry** | Stack traces, error context | AI-suggested fixes, real-time alerts |
| **Google Play Console** | Device/OS segmentation | Android vitals, Gemini AI crash analysis |
| **Firebase Crashlytics** | Real-time crash reporting | Session data, root cause clustering |
| **Mixpanel** | Behavior correlation | Connect crashes to user journeys |

You don't need all four. At minimum, you need **crash reporting** (Crashlytics or Sentry) + **device analytics** (Play Console).

## Analysis Framework

### Step 1: Segment by Dimension

Break crashes down across three axes:

- **App version** — Did a specific release introduce a regression?
- **Android/iOS version** — Is SDK 33 crashing more than SDK 34?
- **Device model** — Are low-RAM (4GB) devices disproportionately affected?

### Step 2: Identify Patterns {#device-segmentation}

| Crash Type | Likely Cause | Action |
|-----------|-------------|--------|
| Native crash (`libc.so`, `__strlen_aarch64`) | Native module or system-level bug | Check native dependencies, see [Dependencies](./dependency-management.md) |
| JS crash with stack trace | Application code error | Fix directly |
| ANR (no crash, just freeze) | Main thread blocked | Check [animation perf](./performance-fundamentals.md#1-animation-performance), heavy computation |
| Device-cluster crash | Hardware/memory constraint | Add error boundaries, reduce memory usage |

### Step 3: Prioritize by User Impact

- Fix crashes affecting the **most users first**
- Device-specific issues affecting <1% can be handled with **graceful fallbacks** rather than deep fixes
- Track crash-free user rate as your north-star metric

### Step 4: Monitor Your Fixes

After deploying a fix:
1. Watch the crash rate for that specific issue over 48-72 hours
2. Set up **regression alerts** so new releases don't reintroduce old crashes
3. Compare crash-free rates between app versions

## Google Play Console Workflow

1. Navigate to **Monitor and improve > Android vitals > Crashes and ANRs**
2. Filter by **app version** to isolate regressions
3. Check the **device** tab for hardware-specific clusters
4. Use the **Android version** distribution to prioritize OS-level fixes
5. Cross-reference with Crashlytics clusters for root cause

## Pro Tips

- **Don't ignore device-specific issues** — some crashes only occur on specific chipsets (Mediatek vs Qualcomm)
- **Low-memory devices need special attention** — 4GB RAM devices behave very differently from 8GB
- **Document reproduction steps** — when you find a crash, write down exactly how to trigger it before fixing
- **Not all crashes can be fixed** — hardware limitations exist; handle edge cases gracefully instead

---

**Next**: [Keeping RN Updated](./keeping-rn-updated.md) — eliminate platform-level issues before optimizing code
