# React Native & Expo: Architecture, Debugging & Optimization Guides

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/nicemogul/react-native-guide/pulls)
[![React Native](https://img.shields.io/badge/React%20Native-0.70+-61DAFB?logo=react)](https://reactnative.dev)
[![Expo](https://img.shields.io/badge/Expo-SDK%2049+-000020?logo=expo)](https://expo.dev)

> A practical, community-sourced collection of guides for understanding, debugging, and optimizing React Native / Expo apps — from architecture internals to native-layer crash analysis to EAS build pipelines.

When your app crashes with a `SIGABRT` in `libc.so`, you need to understand the rendering pipeline, or you want to master EAS builds and OTA updates — these guides have you covered. They bridge the gap between React Native's JS world and everything underneath.

---

## Who Is This For?

- **React Native / Expo developers** who want to understand how RN actually works under the hood
- **Mobile engineers** hitting native crashes they can't explain from the JS layer
- **Team leads** responsible for production stability and release rollouts
- **New team members** who need a comprehensive onboarding resource for RN internals and Expo tooling
- **Anyone** who has stared at a tombstone trace from `libc.so` and thought "now what?"

No deep Android/C++ experience required — the guides are written for React Native developers and explain native concepts as they come up.

---

## What's Inside

| Guide | What It Covers | When You Need It |
|-------|---------------|-----------------|
| [**React Native Architecture & Lifecycle**](react-native-architecture-lifecycle.md) | Old vs New Architecture, threading model, rendering pipeline, Hermes internals, Android/iOS app lifecycle, Bridge vs Bridgeless, event system, memory model | You want to understand how React Native actually works under the hood — essential foundation for all other guides |
| [**SIGABRT & libc Debugging Guide**](SIGABRT-libc-debugging-guide.md) | Signal 6 (SIGABRT) and Signal 11 (SIGSEGV) crashes from `libc.so` — root causes, debugging strategies, production fixes, and a decision framework | Your app is crashing in production with native signals and you need to find the root cause and ship a fix |
| [**Native-Layer Debugging & Optimization Guide**](native-layer-debugging-guide.md) | Memory profiling, CPU profiling, Perfetto tracing, heap analysis, sanitizers, Hermes profiling, Tracy integration, Expo prebuild inspection, crash symbolication, ProGuard/R8, ABI splits, baseline profiles | You need to profile, optimize, or debug your app at the native layer using Android Studio, Xcode, or cross-platform tools |
| [**Expo EAS Complete Guide**](expo-eas-complete-guide.md) | EAS Build, Submit, Update (OTA), Prebuild & Config Plugins, Metadata, debug symbols, crash reporting integration, EAS Workflows, pricing | You're building with Expo and need to master the build pipeline, OTA updates, credential management, or CI/CD |
| [**Expo App Config Decision Guide**](expo-app-config-decision-guide.md) | Every app.config.ts and eas.json property explained with WHAT/WHY/TRADE-OFFS/MISTAKES — not boilerplate, but decision-oriented guidance for iOS, Android, OTA, plugins, deep linking, and more | You want to understand what each config option actually does and make informed choices for your specific app |
| [**Mega Guide**](mega-guide/) | 7-chapter series: monitoring, upgrading RN, New Architecture migration, dependencies, performance & rendering, profiling & debugging, curated reading list — with Shopify/Discord/Callstack production metrics | You want a structured learning path through all aspects of RN optimization |
| [**Optimization Guide**](optimization-guide/) | Concise optimization reference: ANR analysis, dependency management, performance fundamentals, list performance, New Architecture deep dive, profiling & debugging, curated reading list | You want quick-reference optimization advice organized by topic |

### At a Glance

**React Native Architecture & Lifecycle (1,700+ lines):**
- Old Architecture (Bridge) vs New Architecture (JSI/Fabric/TurboModules/Codegen) with ASCII diagrams
- Complete threading model — all 6 threads explained with blocking consequences
- Rendering pipeline deep dive — Old Renderer vs Fabric (Render/Commit/Mount phases)
- Hermes engine internals — bytecode, GenGC, no-JIT rationale
- Full Android cold start sequence (Zygote → SoLoader → Bridge → JS → Render)
- Full iOS cold start sequence (dyld → AppDelegate → RCTBridge → Bundle → Render)
- Bridge vs Bridgeless comparison, React 18 concurrent features
- Event system, gesture handling, memory model across platforms

**SIGABRT & libc Debugging Guide (1,400+ lines):**
- 9+ root cause categories (buggy native libs, Hermes, OEM-specific, OOM, R8 over-stripping, and more)
- 10+ debugging strategies (from logcat filtering to AddressSanitizer)
- 8+ production-ready fixes with code examples
- Crash reporting tool comparison (Crashlytics vs Sentry vs Datadog vs Bugsnag)
- Decision framework flowchart for systematic triage
- "Audit your own app" checklist

**Native-Layer Debugging & Optimization Guide (1,400+ lines):**
- Android Studio Memory Profiler, CPU Profiler, and Layout Inspector walkthroughs
- Perfetto system tracing for cross-process analysis
- Heap dumps and native memory tracking with heapprofd
- ASan / TSan / UBSan sanitizer setup for both platforms
- Hermes CPU profiling and Chrome DevTools integration
- Tracy profiler for React Native C++ internals
- Xcode Instruments, Memory Graph Debugger, Metal System Trace
- ProGuard/R8 configuration, ABI splits, baseline profiles, Flipper replacements

**Expo EAS Complete Guide (1,900+ lines):**
- EAS Build architecture, server infrastructure, resource classes, build images
- Complete eas.json reference with all properties
- Credential management (Android keystore, iOS provisioning)
- EAS Submit for Google Play and App Store
- EAS Update (OTA) — channels, branches, runtime versions, rollbacks, rollouts
- Prebuild & Config Plugins — CNG, custom plugins, managed vs bare
- Debug symbol upload for Sentry, Datadog, Crashlytics
- EAS Workflows, CI/CD integration, pricing

**Expo App Config Decision Guide (1,600+ lines):**
- Every app.config.ts property with WHAT/WHY/TRADE-OFFS/MISTAKES framework
- iOS deep dive: bundleIdentifier, supportsTablet, Universal Links, infoPlist, entitlements
- Android deep dive: package, allowBackup, adaptive icons, permissions, App Links, keyboard modes
- expo-build-properties: minSdkVersion coverage, R8/ProGuard keep rules for 10+ libraries
- OTA strategy: checkAutomatically options, fallbackToCacheTimeout decisions, runtime version policies
- eas.json: build profile inheritance, environment variable strategy, staged rollouts
- Deep linking architecture: custom scheme + Universal Links + App Links complete setup

---

## Quick Start

**"I want to understand how React Native works under the hood."**
Start with the [Architecture & Lifecycle Guide](react-native-architecture-lifecycle.md). It explains the threading model, rendering pipeline, Hermes engine, and app startup sequence — essential context for everything else.

**"My app is crashing with a native signal (SIGABRT, SIGSEGV) and I don't know why."**
Start with the [SIGABRT & libc Debugging Guide](SIGABRT-libc-debugging-guide.md). It walks you through identifying the crash source, matching it to known root causes, and applying targeted fixes.

**"My app is slow, uses too much memory, or I need to profile native performance."**
Start with the [Native-Layer Debugging & Optimization Guide](native-layer-debugging-guide.md). It covers the full toolkit for profiling and optimizing at the native layer.

**"I need to set up EAS builds, OTA updates, or CI/CD for my Expo app."**
Start with the [Expo EAS Complete Guide](expo-eas-complete-guide.md). It covers the entire EAS pipeline from build configuration to store submission to over-the-air updates.

**"I want to understand what each app.config.ts / eas.json property actually does before I change it."**
Start with the [Expo App Config Decision Guide](expo-app-config-decision-guide.md). Every property explained with trade-offs and common mistakes — not boilerplate, but informed decision-making.

**"I want to proactively audit my app before the next release."**
Read all four guides. The architecture guide gives you the mental model, the SIGABRT guide has an audit checklist, the optimization guide covers profiling, and the EAS guide ensures your build pipeline is solid.

---

## Contributing

Contributions are welcome! These guides are living documents — React Native's native layer evolves quickly and new crash patterns emerge with every major release.

Ways to contribute:

- **Add a new root cause or fix** you discovered in production
- **Improve an existing section** with better explanations or updated tooling
- **Share a debugging war story** that could help others (even as a brief case study)
- **Fix typos, broken links, or outdated version references**
- **Add coverage for iOS-specific native debugging** (these guides currently focus on Android)

Please open a PR with a clear description of what you're adding or changing. If you're adding a new root cause, include the crash signature, the environment where you saw it, and the fix that worked.

---

## Sources & Acknowledgments

These guides were compiled from 20+ sources across the React Native community, including:

- **GitHub Issues** — crash reports and fixes from `react-native`, `reanimated`, `react-native-screens`, `hermes-engine`, `expo`, and other core repositories
- **Omkar Salapurkar** — [detailed writeup on SIGABRT crashes in React Native Android](https://medium.com/@nicemogul) and native-layer debugging approaches
- **Callstack Blog** — articles on React Native performance, Hermes internals, and native module debugging
- **Margelo** — creators of [react-native-release-profiler](https://github.com/margelo/react-native-release-profiler) and Tracy integration tooling for React Native
- **Software Mansion** — Reanimated and React Native Screens debugging documentation
- **Stack Overflow** — community answers on `libc.so` crashes, tombstone analysis, and Android NDK debugging
- **Android Developer Documentation** — Perfetto, Android Studio profiling tools, baseline profiles, and ProGuard/R8 guides
- **Production incidents** — real-world crash patterns and fixes from shipping React Native apps at scale

Thank you to everyone who files detailed bug reports, writes debugging blog posts, and builds the tooling that makes native-layer debugging accessible to the React Native community.

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---

<p align="center">
  <em>If these guides saved you a few hours of debugging, consider giving the repo a star.</em>
</p>
