# Chapter 2: Upgrading React Native

> Newer RN versions ship critical performance fixes, not just features.

[← Monitoring](./monitoring-anr-analysis.md) | [Index](../README.md) | [New Architecture →](./new-architecture-migration.md)

**Keywords**: upgrade, version, Expo SDK, New Architecture, patch-package, upgrade-helper, breaking changes

---

## Why Update?

Each React Native release improves:
- **Native-side performance** — where most [ANRs](./monitoring-anr-analysis.md) originate
- **Thread management** — reduces main thread blocking that causes [animation jank](./performance-rendering.md#animations)
- **Bridge communication** — or eliminates it entirely via [New Architecture](./new-architecture-migration.md)
- **Framework bug fixes** — issues like ReactScrollView pointer crashes are fixed in newer versions

## New Architecture (RN 0.76+) {#new-architecture}

The biggest single performance upgrade. See [Chapter 3: New Architecture Deep Dive](./new-architecture-migration.md) for full migration playbook with production metrics.

| Old Architecture | New Architecture |
|-----------------|-----------------|
| JS ↔ Bridge ↔ Native (async, serialized) | JS ↔ JSI ↔ Native (synchronous, direct) |
| Batched UI updates | Concurrent rendering support |
| Single-threaded bridge bottleneck | Multi-threaded Fabric renderer |

**Impact**: 17-27% faster cold start ([measured data](./new-architecture-migration.md#performance-gains)), reduced [ANRs](./monitoring-anr-analysis.md), smoother [animations](./performance-rendering.md#animations).

## How to Upgrade {#how-to-upgrade}

### Step 1: Use the Upgrade Helper

1. Visit [react-native-community.github.io/upgrade-helper](https://react-native-community.github.io/upgrade-helper/)
2. Select your **current** and **target** versions
3. Apply **every** file change shown in the diff

### Step 2: Update Dependencies

After upgrading RN, third-party libraries need updating too. Incompatible versions cause:
- ANRs and crashes — see [Monitoring](./monitoring-anr-analysis.md)
- [Memory leaks](./profiling-debugging.md#memory-leaks)
- Build failures

```bash
yarn outdated
yarn why <package-name>
```

**Real example**: In a production release, upgrading `@react-native-firebase/*` from v19 to v23 (4 major versions) caused a regression in the signup flow — the modular API migration broke analytics calls.

> **See also**: [Dependencies: Safe Removal Process](./dependency-management.md#auditing) for how to safely audit and update packages.

### Step 3: Patch When Needed

Some libraries lag behind RN releases:
1. Check the library's **GitHub issues**
2. Use `patch-package` to apply community patches
3. Consider [lighter alternatives](./dependency-management.md#lighter-alternatives) if unmaintained

### Step 4: Verify

```bash
# Run your CI pipeline (type-check + lint + tests)
yarn test && yarn lint && yarn tsc

# Build for each platform
yarn android
yarn ios
```

Monitor [crash rates](./monitoring-anr-analysis.md#monitor-fixes) via Crashlytics/Sentry for 48-72 hours post-release.

## Upgrade Cadence

- **Minor versions** (0.76.1 → 0.76.2): Apply quickly, usually bug fixes
- **Major versions** (0.75 → 0.76): Plan a dedicated sprint, test thoroughly
- **Expo SDK updates**: Follow Expo's upgrade guide, which handles RN version alignment

## Common Pitfalls

- **Don't skip versions** — upgrading from 0.72 to 0.76 is harder than incremental upgrades
- **Don't delay indefinitely** — the longer you wait, the harder the upgrade. A Million Monkeys adds 30% to estimates when >2 versions behind ([source](./new-architecture-migration.md#timeline))
- **Test on real devices** — emulators don't surface all ANRs. See [device segmentation](./monitoring-anr-analysis.md#device-segmentation)
- **Watch for native module compatibility** — see [New Architecture: Audit Dependencies](./new-architecture-migration.md#migration-playbook)

## Example Audit

A production crash analysis showed that several framework bugs were fixed in newer RN versions. Upgrading RN eliminated those crashes without any app-side code changes.

From the [SIGABRT guide](../debugging/crash-analysis.md): keeping `@react-native-picker/picker` at outdated versions is a risk — always check for known SIGABRT fixes in patch releases.

---

> **See also**: [New Architecture Deep Dive](./new-architecture-migration.md) for the biggest single performance upgrade available

**Next**: [New Architecture Deep Dive →](./new-architecture-migration.md) — the biggest single performance upgrade available
