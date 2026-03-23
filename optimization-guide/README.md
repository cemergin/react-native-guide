# React Native Optimization Guide

A consolidated reference for building performant React Native apps, distilled from real-world optimization work.

## Contents

### Fundamentals (from article series)

| Guide | Focus Area |
|-------|-----------|
| [ANR Analysis & Monitoring](./anr-analysis.md) | Start here — instrument before optimizing |
| [Performance Fundamentals](./performance-fundamentals.md) | Animations, memoization, data fetching |
| [Dependency Management](./dependency-management.md) | Auditing, replacing, and minimizing libraries |
| [Keeping RN Updated](./keeping-rn-updated.md) | Version upgrades, New Architecture, patching |

### Deep Dives (from engineering blogs — Shopify, Discord, Sentry, Callstack, BAM)

| Guide | Focus Area | Key Sources |
|-------|-----------|-------------|
| [New Architecture Migration](./deep-dive-new-architecture.md) | JSI, Fabric, TurboModules, migration playbook | Shopify, A Million Monkeys, Discord |
| [List Performance](./deep-dive-list-performance.md) | FlashList v2, view recycling, custom lists | Shopify, Discord |
| [Profiling & Debugging](./deep-dive-profiling-debugging.md) | Re-render profiling, memory leaks, bundle optimization | Sentry, Discord, Callstack, BAM |

### Reference

| Resource | Description |
|----------|-------------|
| [Reading List](./reading-list.md) | 28 curated articles with links and summaries |

## Reading Order

These guides are designed to be read in sequence — each builds on the previous:

```
1. ANR Analysis        → Understand WHERE your app fails
   ↓
2. Keeping RN Updated  → Eliminate platform-level issues first
   ↓
3. Dependencies        → Reduce bundle size and attack surface
   ↓
4. Performance         → Optimize what remains
```

## Quick Reference: Decision Matrix

| Problem | Go To |
|---------|-------|
| App freezing / ANRs | [ANR Analysis](./anr-analysis.md) |
| Slow startup / cold start | [Profiling & Debugging](./deep-dive-profiling-debugging.md) |
| Janky animations | [Performance](./performance-fundamentals.md#1-animation-performance) |
| Unnecessary re-renders | [Profiling & Debugging](./deep-dive-profiling-debugging.md#step-by-step-finding-slow-re-renders) |
| Slow/janky lists | [List Performance](./deep-dive-list-performance.md) |
| Large bundle size | [Profiling & Debugging](./deep-dive-profiling-debugging.md#bundle-size-optimization) |
| Memory leaks | [Profiling & Debugging](./deep-dive-profiling-debugging.md#memory-leak-detection) |
| Crash on specific devices | [ANR Analysis](./anr-analysis.md#device-segmentation) |
| Migrating to New Architecture | [New Architecture](./deep-dive-new-architecture.md) |
| Outdated RN version | [Keeping Updated](./keeping-rn-updated.md#how-to-upgrade) |
| Choosing a new library | [Dependencies](./dependency-management.md#before-adding-a-library) |
| Redux boilerplate fatigue | [Performance](./performance-fundamentals.md#5-react-query) |
