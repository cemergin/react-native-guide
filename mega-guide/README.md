# React Native Mega Guide

> Production-tested optimization knowledge from Shopify, Discord, Sentry, Callstack, and BAM — organized for search, deep-linking, and action.

---

## How to Use This Guide

**By problem** → Jump to the [Decision Matrix](#decision-matrix)
**By topic** → Browse the [Table of Contents](#table-of-contents)
**By keyword** → See the [Concept Index](#concept-index)
**By source** → Check the [Reading List](./07-reading-list.md)

---

## Table of Contents {#table-of-contents}

| # | Chapter | What You'll Learn |
|---|---------|------------------|
| 01 | [Monitoring & ANR Analysis](./01-monitoring.md) | Instrument before optimizing. Crashlytics, Sentry, Play Console workflows |
| 02 | [Upgrading React Native](./02-upgrading.md) | Version upgrades, New Architecture enablement, patch strategies |
| 03 | [New Architecture Deep Dive](./03-new-architecture.md) | JSI, Fabric, TurboModules — with Shopify & Discord production metrics |
| 04 | [Dependency Management](./04-dependencies.md) | Audit, replace, and minimize libraries. Bundle size impact |
| 05 | [Performance & Rendering](./05-performance.md) | Animations, memoization, re-renders, React Query, lists, startup |
| 06 | [Profiling & Debugging](./06-profiling.md) | Tools, memory leaks, bundle analysis, Flashlight, war stories |
| 07 | [Reading List](./07-reading-list.md) | 28 curated articles with links — all verified March 2026 |

---

## Decision Matrix {#decision-matrix}

Jump directly to what you need:

| Problem | Go To | Key Metric |
|---------|-------|-----------|
| App freezing / ANRs | [Monitoring](./01-monitoring.md#analysis-framework) | Crash-free rate |
| Slow cold start | [Profiling: Startup](./06-profiling.md#startup-optimization) | TTI (Time to Interactive) |
| Janky animations | [Performance: Animations](./05-performance.md#animations) | FPS, useNativeDriver |
| Unnecessary re-renders | [Profiling: Re-renders](./06-profiling.md#finding-slow-re-renders) | Render count, flamegraph |
| Slow/janky lists | [Performance: Lists](./05-performance.md#lists) | Blank area, view count |
| Large bundle size | [Profiling: Bundle](./06-profiling.md#bundle-size) | Bundle MB, dep count |
| Memory leaks | [Profiling: Memory](./06-profiling.md#memory-leaks) | Heap growth over time |
| Crash on specific devices | [Monitoring: Segmentation](./01-monitoring.md#device-segmentation) | Device-cluster rate |
| Migrating to New Arch | [New Architecture](./03-new-architecture.md#migration-playbook) | Timeline, compat matrix |
| Outdated RN version | [Upgrading](./02-upgrading.md#how-to-upgrade) | Current vs target version |
| Choosing a new library | [Dependencies: Checklist](./04-dependencies.md#before-adding) | Bundle impact, maintenance |
| Redux boilerplate fatigue | [Performance: Data Fetching](./05-performance.md#data-fetching) | Lines of code, caching |
| CI performance regression | [Profiling: Flashlight](./06-profiling.md#flashlight) | FPS/CPU/RAM in CI |
| Post-release regression | [Monitoring: Fix Monitoring](./01-monitoring.md#monitor-fixes) | 48-72hr crash rate |

---

## Concept Index {#concept-index}

Quick lookup — find where each concept is covered:

| Concept | Primary | Also Mentioned In |
|---------|---------|------------------|
| **ANR** | [01-monitoring](./01-monitoring.md) | [05-performance](./05-performance.md#animations), [03-new-arch](./03-new-architecture.md#common-pitfalls) |
| **Animations / useNativeDriver** | [05-performance](./05-performance.md#animations) | [03-new-arch](./03-new-architecture.md#reanimated) |
| **Bundle size** | [06-profiling](./06-profiling.md#bundle-size) | [04-dependencies](./04-dependencies.md#lighter-alternatives) |
| **Cold start / TTI** | [06-profiling](./06-profiling.md#startup-optimization) | [03-new-arch](./03-new-architecture.md#performance-gains) |
| **Crashlytics / Sentry** | [01-monitoring](./01-monitoring.md#tools) | [06-profiling](./06-profiling.md#profiling-stack) |
| **Depcheck** | [04-dependencies](./04-dependencies.md#auditing) | — |
| **Fabric / JSI / TurboModules** | [03-new-architecture](./03-new-architecture.md) | [02-upgrading](./02-upgrading.md#new-architecture) |
| **Flashlight (BAM)** | [06-profiling](./06-profiling.md#flashlight) | [05-performance](./05-performance.md#lists) |
| **FlashList v2** | [05-performance](./05-performance.md#lists) | [03-new-arch](./03-new-architecture.md) |
| **Hermes** | [06-profiling](./06-profiling.md#bundle-size) | [02-upgrading](./02-upgrading.md) |
| **Memory leaks** | [06-profiling](./06-profiling.md#memory-leaks) | [01-monitoring](./01-monitoring.md) |
| **Memoization (useMemo/useCallback)** | [05-performance](./05-performance.md#memoization) | [06-profiling](./06-profiling.md#react-compiler) |
| **Metro bundler** | [06-profiling](./06-profiling.md#bundle-size) | [04-dependencies](./04-dependencies.md) |
| **New Architecture** | [03-new-architecture](./03-new-architecture.md) | [02-upgrading](./02-upgrading.md#new-architecture) |
| **RAM Bundles** | [06-profiling](./06-profiling.md#ram-bundles) | — |
| **React Compiler** | [06-profiling](./06-profiling.md#react-compiler) | [05-performance](./05-performance.md#memoization) |
| **React Query** | [05-performance](./05-performance.md#data-fetching) | — |
| **Reanimated** | [05-performance](./05-performance.md#animations) | [03-new-arch](./03-new-architecture.md#common-pitfalls) |
| **Rollout strategy** | [03-new-architecture](./03-new-architecture.md#gradual-rollout) | [01-monitoring](./01-monitoring.md#monitor-fixes) |
| **View recycling** | [05-performance](./05-performance.md#lists) | — |

---

## Reading Order

```
START HERE
    │
    ▼
┌─────────────────────────┐
│  01 Monitoring          │ ← Instrument first. Know WHERE it hurts.
│  (ANRs, crashes, tools) │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  02 Upgrading RN        │ ← Eliminate platform-level issues.
│  (versions, patches)    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  03 New Architecture    │ ← Biggest single perf upgrade.
│  (JSI, Fabric, migrate) │   17-27% faster cold start.
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  04 Dependencies        │ ← Reduce bundle, remove dead weight.
│  (audit, swap, custom)  │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  05 Performance         │ ← Optimize rendering, lists, data.
│  (animations, lists,    │
│   memoization, queries) │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  06 Profiling           │ ← Measure, debug, prevent regressions.
│  (tools, memory, bundle,│
│   CI benchmarks)        │
└────────────┘
```

---

## Stats At a Glance

Real numbers from production apps referenced in this guide:

| Company | Metric | Value | Chapter |
|---------|--------|-------|---------|
| Shopify | Crash-free sessions | 99.9% | [01](./01-monitoring.md) |
| Shopify | Screen load P75 | <500ms | [05](./05-performance.md) |
| Shopify | Native modules audited | 40+ | [03](./03-new-architecture.md) |
| Discord | iOS engineers | 3 | [05](./05-performance.md#lists) |
| Discord | TTI improvement (iPhone 6) | -3,500ms | [06](./06-profiling.md#startup-optimization) |
| Discord | Message parse optimization | 90% faster | [06](./06-profiling.md#discord-wins) |
| Discord | Memory reduction (lists) | -14% | [05](./05-performance.md#lists) |
| A Million Monkeys | Android cold start | -24% | [03](./03-new-architecture.md#performance-gains) |
| A Million Monkeys | Migration timeline (2 devs) | 12 days | [03](./03-new-architecture.md#timeline) |
| Callstack | Bundle size reduction | 78% (50→11MB) | [06](./06-profiling.md#bundle-size) |
| Moment→Day.js | Size reduction | 99% (232→2KB) | [04](./04-dependencies.md#lighter-alternatives) |
