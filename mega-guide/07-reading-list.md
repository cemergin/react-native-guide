# Chapter 7: Reading List

> Curated from engineering blogs by real practitioners. Every link verified March 2026.

[← Profiling](./06-profiling.md) | [Index](./README.md)

**Keywords**: articles, blogs, Shopify, Discord, Callstack, BAM, Sentry, Expensify, Oscar Franco, Flashlight, FlashList

---

## Architecture & Internals

| # | Article | Author | Key Takeaway |
|---|---------|--------|-------------|
| 1 | [Five Years of React Native at Shopify](https://shopify.engineering/five-years-of-react-native-at-shopify) | Shopify Engineering | 99.9% crash-free at scale, code sharing strategy |
| 2 | [Migrating to React Native's New Architecture](https://shopify.engineering/react-native-new-architecture) | Shopify Engineering | Production migration of millions-of-users apps |
| 3 | [New Architecture: What Actually Changed](https://www.amillionmonkeys.co.uk/blog/react-native-new-architecture-migration) | A Million Monkeys | Honest metrics: 17% faster Android launch, compatibility gotchas |
| 4 | [Client Guide to React Native Modules](https://ospfranco.com/client-guide-to-react-native-modules/) | Oscar Franco | Best JSI/Fabric/TurboModules explainer |
| 5 | [JSI Cheatsheet (Parts 1-3)](https://ospfranco.com/post/2023/08/15/jsi-cheatsheet-part-1-jsi/) | Oscar Franco | Hands-on C++ reference for JSI modules |
| 6 | [Rust Modules in React Native](https://ospfranco.com/post/2024/05/08/react-native-rust-module-guide/) | Oscar Franco | Bypassing JS for perf-critical code via Rust + JSI |
| 7 | [React Native 0.82 - A New Era](https://reactnative.dev/blog/2025/10/08/react-native-0.82) | Meta/RN Team | Bridge fully removed, New Architecture only |

## Performance & Profiling

| # | Article | Author | Key Takeaway |
|---|---------|--------|-------------|
| 8 | [FlashList v2: A Ground-Up Rewrite](https://shopify.engineering/flashlist-v2) | Shopify Engineering | No estimatedItemSize, 50% less blank area |
| 9 | [RN Performance Tactics](https://blog.sentry.io/react-native-performance-strategies-tools/) | Sentry | Profiling with Sentry + RN DevTools on 0.80+ |
| 10 | [Handle Slow Re-renders](https://apps.theodo.com/en/article/handle-slow-re-renders-a-not-so-obvious-solution-using-react-devtools-and-javascript-profiler) | Alexandre Moureaux / BAM | Step-by-step re-render profiling methodology |
| 11 | [Measure RN Performance with Flashlight](https://apps.theodo.com/article/measuring-react-native-performance-with-flashlight) | BAM/Theodo | "Lighthouse for mobile" — FPS, CPU, RAM |
| 12 | [Flashlight (GitHub)](https://github.com/bamlab/flashlight) | BAM | Open-source, CI-integrated perf benchmarks |
| 13 | [WebGPU, Skia, and Beyond](https://shopify.engineering/webgpu-skia-web-graphics) | Shopify Engineering | 50% faster iOS animations, 200% faster Android |

## Debugging & Crash Analysis

| # | Article | Author | Key Takeaway |
|---|---------|--------|-------------|
| 14 | [Memory Leak You Don't See Until Production](https://medium.com/@silverskytechnology/the-react-native-memory-leak-you-dont-see-until-production-8d62a18d840a) | Silversky Technology | React lifecycle vs native side effects mismatch |
| 15 | [App Crashed in Front of 500 Users](https://medium.com/@mohantaankit2002/the-day-my-react-native-app-crashed-in-front-of-500-users-and-how-i-fixed-the-memory-leaks-7bc7a275a032) | Ankit | Production war story with diagnosis and fix |
| 16 | [Debugging Memory Leaks (2025)](https://medium.com/react-native-journal/debugging-memory-leaks-in-react-native-2025-a-step-by-step-guide-f8d4e1976fe7) | RN Journal | Flipper + Hermes profiler on New Architecture |
| 17 | [New Way of RN Debugging](https://blog.sentry.io/the-new-way-of-react-native-debugging/) | Sentry | New RN DevTools + production monitoring |
| 18 | [Debugging RN Android Performance](https://gitnation.com/contents/debugging-rn-android-performance) | Alexandre Moureaux | Android Studio tracing + Hermes profiler |

## Build Systems & Bundle Optimization

| # | Article | Author | Key Takeaway |
|---|---------|--------|-------------|
| 19 | [Optimize Your RN JavaScript Bundle](https://www.callstack.com/blog/optimize-react-native-apps-javascript-bundle) | Callstack | Metro optimization, why Re.Pack doesn't help with Hermes |
| 20 | [Ultimate Guide to RN Optimization](https://www.callstack.com/ebooks/the-ultimate-guide-to-react-native-optimization) | Callstack (free ebook) | 78% bundle size reduction case study |
| 21 | [RN Optimization with DMAIC](https://www.callstack.com/ebooks/react-native-performance-optimization-with-dmaic-guide) | Callstack (free ebook) | Data-driven performance methodology |
| 22 | [Why Metro?](https://docs.expo.dev/guides/why-metro/) | Expo | Metro architecture + Hermes bytecode |
| 23 | [Measure Hermes with Flashlight](https://www.theodo.com/blog/measure-hermes-engine-performance-in-react-native-with-flashlight) | Theodo/BAM | Hermes-specific benchmarking |

## Architecture Patterns & Testing

| # | Article | Author | Key Takeaway |
|---|---------|--------|-------------|
| 24 | [react-native-onyx (GitHub)](https://github.com/Expensify/react-native-onyx) | Expensify | Production offline-first state management |
| 25 | [Supercharging Discord Mobile](https://discord.com/blog/supercharging-discord-mobile-our-journey-to-a-faster-app) | Discord, Mar 2025 | Ongoing perf work for millions of MAU |
| 26 | [Native iOS Performance with RN](https://discord.com/blog/how-discord-achieves-native-ios-performance-with-react-native) | Discord | 99.9% crash free, 3 iOS engineers |
| 27 | [Maestro vs Detox at Jupiter](https://life.jupiter.money/choosing-between-maestro-and-detox-on-jupiter-qa-automation-7b94e6f8759d) | Jupiter Money | Real team comparison with setup/maintenance costs |
| 28 | [Nightly E2E + Perf Tests](https://blog.theodo.com/2023/11/nightly-performance-flashlight/) | Theodo/BAM | CI performance regression testing |

## Blogs to Follow

- **[ospfranco.com](https://ospfranco.com/)** — JSI, native modules, C++ internals
- **[Shopify Engineering](https://shopify.engineering/topics/mobile)** — FlashList, Skia, production RN
- **[Callstack](https://www.callstack.com/insights/performance-optimization)** — Performance, Re.Pack, debugging
- **[Theodo/BAM Apps](https://apps.theodo.com/)** — Flashlight, profiling methodology
- **[Sentry Blog](https://blog.sentry.io)** — Monitoring, debugging, observability
- **[Discord Engineering](https://discord.com/blog)** — High-scale RN performance
