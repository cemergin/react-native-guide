# React Native & Expo: The Complete Guide

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/nicemogul/react-native-guide/pulls)
[![React Native](https://img.shields.io/badge/React%20Native-0.76+-61DAFB?logo=react)](https://reactnative.dev)
[![Expo](https://img.shields.io/badge/Expo-SDK%2052+-000020?logo=expo)](https://expo.dev)

> From architecture internals to production optimization — everything you need to build, debug, and ship performant React Native apps.

When your app crashes with a `SIGABRT` in `libc.so`, you need to profile a memory leak on a $50 Android phone, or you want to master EAS builds and OTA updates — these guides have you covered. They bridge the gap between React Native's JS world and everything underneath.

---

## Pick Your Starting Point

### By Experience Level

**New to React Native (0-2 years)**
1. [Performance & Rendering](performance-rendering.md) — core optimization principles, no setup required
2. [Monitoring & ANR Analysis](monitoring-anr-analysis.md) — understand why your app feels slow
3. [Upgrading React Native](upgrading-react-native.md) — version upgrade strategy
4. [Architecture & Lifecycle](react-native-architecture-lifecycle.md) — how RN actually works (deep dive, optional at this stage)

**Experienced with React Native (2-5 years)**
1. [Monitoring & ANR Analysis](monitoring-anr-analysis.md) — production observability
2. [New Architecture Migration](new-architecture-migration.md) — 17-27% faster cold start
3. [Profiling Tools Deep Dive](profiling-tools-deep-dive.md) — memory & CPU profiling decision framework
4. [Native-Layer Debugging](native-layer-debugging-guide.md) — crash analysis and profiling tools

**Optimizing Production Apps (Team Lead / 5+ years)**
1. [SIGABRT & libc Debugging](SIGABRT-libc-debugging-guide.md) — native crash root causes
2. [Profiling Tools Deep Dive](profiling-tools-deep-dive.md) — when and how to profile memory & CPU
3. [Optimization Series](#optimization-series-read-in-order) — comprehensive reference (7 chapters)
4. [Expo EAS Complete Guide](expo-eas-complete-guide.md) — build pipeline mastery

### By Problem

| Problem | Go To | Key Metric |
|---------|-------|-----------|
| App freezing / ANRs | [Monitoring](monitoring-anr-analysis.md) | Crash-free rate |
| Slow cold start (>2s) | [Profiling Deep Dive: Use Case 3](profiling-tools-deep-dive.md#use-case-3-cold-start-takes-3-seconds) | TTI |
| Janky animations | [Performance: Animations](performance-rendering.md#animations) | FPS |
| App slows after 5+ min use | [Profiling Deep Dive: Memory](profiling-tools-deep-dive.md#deep-dive-memory-profiling) | Heap growth |
| Unnecessary re-renders | [Profiling: Re-renders](profiling-debugging.md#finding-slow-re-renders) | Render count |
| Slow/janky lists | [Performance: Lists](performance-rendering.md#lists) | Blank area, view count |
| Large bundle size | [Profiling: Bundle](profiling-debugging.md#bundle-size) | Bundle MB |
| Memory leaks | [Profiling Deep Dive: Memory](profiling-tools-deep-dive.md#deep-dive-memory-profiling) | Heap snapshots |
| Native crash (SIGABRT/SIGSEGV) | [SIGABRT Guide](SIGABRT-libc-debugging-guide.md) | Signal analysis |
| Battery drain | [Profiling Deep Dive: Use Case 4](profiling-tools-deep-dive.md#use-case-4-battery-drain-reported-by-users) | CPU % idle |
| Crashes on low-end devices | [Profiling Deep Dive: Use Case 5](profiling-tools-deep-dive.md#use-case-5-crashes-only-on-low-end-devices) | Peak memory |
| Migrating to New Architecture | [New Architecture](new-architecture-migration.md) | Timeline |
| Outdated RN version | [Upgrading](upgrading-react-native.md) | Version delta |
| Choosing a new library | [Dependencies](dependency-management.md) | Bundle impact |
| EAS build/deploy issues | [EAS Guide](expo-eas-complete-guide.md) | Build config |
| app.config.ts confusion | [App Config Guide](expo-app-config-decision-guide.md) | WHAT/WHY/TRADE-OFFS |
| Which state manager to pick | [Modern Patterns](modern-rn-architecture-patterns.md#state-management-decision-framework) | Decision matrix |
| Navigation architecture | [Modern Patterns](modern-rn-architecture-patterns.md#navigation-expo-router-vs-react-navigation) | Expo Router vs RN7 |
| Data fetching strategy | [Modern Patterns](modern-rn-architecture-patterns.md#data-fetching-tanstack-query--trpc) | TanStack Query + tRPC |
| Starting a new project | [Modern Patterns](modern-rn-architecture-patterns.md#the-modern-react-native-stack-2026) | 2026 default stack |

---

## All Guides

### Foundation

| Guide | What It Covers |
|-------|---------------|
| [Architecture & Lifecycle](react-native-architecture-lifecycle.md) | Old vs New Architecture, threading model, rendering pipeline, Hermes internals, cold start sequences |
| [Modern RN Patterns](modern-rn-architecture-patterns.md) | 2025-2026 stack: Expo Router, Zustand/Jotai/Legend State, TanStack Query, tRPC, MMKV, React Compiler |

### Optimization Series (read in order)

| # | Guide | What You'll Learn |
|---|-------|------------------|
| 1 | [Monitoring & ANR Analysis](monitoring-anr-analysis.md) | Instrument before optimizing. Crashlytics, Sentry, Play Console |
| 2 | [Upgrading React Native](upgrading-react-native.md) | Version upgrades, New Architecture enablement, patches |
| 3 | [New Architecture Migration](new-architecture-migration.md) | JSI, Fabric, TurboModules — Shopify & Discord metrics |
| 4 | [Dependency Management](dependency-management.md) | Audit, replace, minimize libraries. Bundle impact |
| 5 | [Performance & Rendering](performance-rendering.md) | Animations, memoization, re-renders, React Query, lists |
| 6 | [Profiling & Debugging](profiling-debugging.md) | Tools, memory leaks, bundle analysis, Flashlight, Discord wins |
| 7 | [Reading List](reading-list.md) | 37 curated articles, verified March 2026 |

### Deep Dives

| Guide | When You Need It |
|-------|-----------------|
| [Profiling Tools Deep Dive](profiling-tools-deep-dive.md) | Memory & CPU profiling decision framework, use case playbooks |
| [SIGABRT & libc Debugging](SIGABRT-libc-debugging-guide.md) | Native crash root causes, decision framework, production fixes |
| [Native-Layer Debugging](native-layer-debugging-guide.md) | Android Studio, Xcode, Perfetto, ASan, Tracy walkthroughs |

### Expo & Build

| Guide | When You Need It |
|-------|-----------------|
| [Expo EAS Complete Guide](expo-eas-complete-guide.md) | EAS Build, Submit, Update, Workflows, CI/CD, pricing |
| [Expo App Config Guide](expo-app-config-decision-guide.md) | Every app.config.ts and eas.json property with WHAT/WHY/TRADE-OFFS |

---

## Reading Flow

```
Architecture → Modern Patterns → Optimization Series (1-7) → Deep Dives as needed
                                                            ↘ Expo guides for build/deploy
```

---

## Production Stats

Real numbers from production apps referenced across these guides:

| Company | Metric | Value | Guide |
|---------|--------|-------|-------|
| Shopify | Crash-free sessions | 99.9% | [Monitoring](monitoring-anr-analysis.md) |
| Shopify | Screen load P75 | <500ms | [Performance](performance-rendering.md) |
| Shopify | Native modules audited | 40+ | [New Arch](new-architecture-migration.md) |
| Shopify | Bundle size reduction | 78% (50→11MB) | [Profiling](profiling-debugging.md) |
| Discord | iOS engineers | 3 | [Lists](performance-rendering.md#lists) |
| Discord | TTI improvement (iPhone 6) | -3,500ms | [Profiling](profiling-debugging.md#discord-wins) |
| Discord | Message parse optimization | 90% faster | [Profiling](profiling-debugging.md#discord-wins) |
| Discord | Memory reduction (lists) | -14% | [Lists](performance-rendering.md#lists) |
| A Million Monkeys | Android cold start | -24% | [New Arch](new-architecture-migration.md) |
| A Million Monkeys | Migration timeline (2 devs) | 12 days | [New Arch](new-architecture-migration.md) |
| Moment→Day.js | Size reduction | 99% (232→2KB) | [Dependencies](dependency-management.md) |

---

## Concept Cross-Index

Find where each concept is covered across all guides:

| Concept | Primary Guide | Also Covered In |
|---------|--------------|----------------|
| **ANR / App Not Responding** | [Monitoring](monitoring-anr-analysis.md) | [Performance](performance-rendering.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **Animations / useNativeDriver** | [Performance](performance-rendering.md#animations) | [New Arch](new-architecture-migration.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **ASan / Sanitizers** | [Native Debugging](native-layer-debugging-guide.md) | [SIGABRT Guide](SIGABRT-libc-debugging-guide.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **Bundle size** | [Profiling](profiling-debugging.md#bundle-size) | [Dependencies](dependency-management.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **Cold start / TTI** | [Profiling](profiling-debugging.md#startup-optimization) | [New Arch](new-architecture-migration.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **Crashlytics / Sentry** | [Monitoring](monitoring-anr-analysis.md) | [EAS Guide](expo-eas-complete-guide.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **EAS Build / Submit / Update** | [EAS Guide](expo-eas-complete-guide.md) | [App Config](expo-app-config-decision-guide.md) |
| **Fabric / JSI / TurboModules** | [New Architecture](new-architecture-migration.md) | [Architecture](react-native-architecture-lifecycle.md), [Upgrading](upgrading-react-native.md) |
| **Flashlight (BAM)** | [Profiling](profiling-debugging.md#flashlight) | [Performance](performance-rendering.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **FlashList v2** | [Performance](performance-rendering.md#lists) | [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **Hermes engine** | [Architecture](react-native-architecture-lifecycle.md) | [Profiling](profiling-debugging.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |
| **Memory leaks** | [Profiling Deep Dive](profiling-tools-deep-dive.md#deep-dive-memory-profiling) | [Profiling](profiling-debugging.md#memory-leaks), [Native Debugging](native-layer-debugging-guide.md) |
| **Memory profiling (native)** | [Profiling Deep Dive](profiling-tools-deep-dive.md#android-native-memory-profiling) | [Native Debugging](native-layer-debugging-guide.md) |
| **CPU profiling** | [Profiling Deep Dive](profiling-tools-deep-dive.md#deep-dive-cpu-profiling) | [Profiling](profiling-debugging.md), [Native Debugging](native-layer-debugging-guide.md) |
| **Memoization** | [Performance](performance-rendering.md#memoization) | [Profiling](profiling-debugging.md#react-compiler) |
| **New Architecture** | [New Architecture](new-architecture-migration.md) | [Architecture](react-native-architecture-lifecycle.md), [Upgrading](upgrading-react-native.md) |
| **Perfetto** | [Profiling Deep Dive](profiling-tools-deep-dive.md) | [Native Debugging](native-layer-debugging-guide.md) |
| **React Compiler** | [Profiling](profiling-debugging.md#react-compiler) | [Performance](performance-rendering.md#memoization) |
| **React Query** | [Performance](performance-rendering.md#data-fetching) | [Modern Patterns](modern-rn-architecture-patterns.md) |
| **Navigation / Expo Router** | [Modern Patterns](modern-rn-architecture-patterns.md#navigation-expo-router-vs-react-navigation) | [Performance](performance-rendering.md) |
| **State management (Zustand/Jotai)** | [Modern Patterns](modern-rn-architecture-patterns.md#state-management-decision-framework) | [Performance](performance-rendering.md#data-fetching) |
| **Data fetching (TanStack Query)** | [Modern Patterns](modern-rn-architecture-patterns.md#data-fetching-tanstack-query--trpc) | [Performance](performance-rendering.md#data-fetching) |
| **MMKV / Storage** | [Modern Patterns](modern-rn-architecture-patterns.md#pattern-4-mmkv-over-asyncstorage) | [SIGABRT Guide](SIGABRT-libc-debugging-guide.md) |
| **SIGABRT / SIGSEGV** | [SIGABRT Guide](SIGABRT-libc-debugging-guide.md) | [Monitoring](monitoring-anr-analysis.md), [Profiling Deep Dive](profiling-tools-deep-dive.md) |

---

## Who Is This For?

- **React Native / Expo developers** who want to understand how RN works under the hood
- **Mobile engineers** hitting native crashes they can't explain from the JS layer
- **Team leads** responsible for production stability and release rollouts
- **New team members** who need a comprehensive onboarding resource
- **Anyone** who has stared at a tombstone trace from `libc.so` and thought "now what?"

No deep Android/C++ experience required — the guides explain native concepts as they come up.

---

## What's Not Covered Yet (Roadmap)

These topics are planned for future additions. Contributions welcome!

| Topic | Status | Priority |
|-------|--------|----------|
| ~~Navigation patterns~~ | **Done** — [Modern Patterns Guide](modern-rn-architecture-patterns.md) | ~~High~~ |
| ~~State management architecture~~ | **Done** — [Modern Patterns Guide](modern-rn-architecture-patterns.md) | ~~High~~ |
| Testing strategy (Jest, RNTL, Maestro, Detox) | Planned | High |
| Security & data protection | Planned | High |
| Push notifications (FCM/APNS) | Planned | Medium |
| Local storage & offline patterns | Planned | Medium |
| Accessibility (a11y) | Planned | Medium |
| Error boundaries & crash prevention | Planned | Medium |
| Advanced animations & gestures | Planned | Medium |
| Internationalization (i18n) | Planned | Low |
| Custom native modules & JSI | Planned | Low |
| Multi-platform (web support) | Planned | Low |

---

## Contributing

Contributions are welcome! These guides are living documents — React Native evolves quickly and new patterns emerge with every major release.

**Ways to contribute**:
- **Add a new root cause or fix** you discovered in production
- **Improve an existing section** with better explanations or updated tooling
- **Share a debugging war story** (even as a brief case study)
- **Fill a gap** from the roadmap above
- **Fix typos, broken links, or outdated version references**
- **Add iOS-specific coverage** (current guides lean Android)

Please open a PR with a clear description of what you're adding or changing.

---

## Sources & Acknowledgments

These guides were compiled from 20+ sources across the React Native community, including:

- **Shopify Engineering** — FlashList, New Architecture migration at scale, production metrics
- **Discord Engineering** — High-scale performance optimization, custom list implementations
- **Callstack** — Bundle optimization, performance methodology, free ebooks
- **BAM/Theodo** — Flashlight profiling tool, re-render methodology
- **Sentry** — Performance monitoring, debugging tooling
- **Oscar Franco** — JSI, native modules, C++ internals
- **GitHub Issues** — crash reports from react-native, reanimated, hermes-engine, expo
- **Android & Apple Developer Docs** — Perfetto, Instruments, profiling APIs
- **Production incidents** — real-world crash patterns from shipping RN apps at scale

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---

<p align="center">
  <em>If these guides saved you a few hours of debugging, consider giving the repo a star.</em>
</p>
