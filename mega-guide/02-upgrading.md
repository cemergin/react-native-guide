# Chapter 2: Upgrading React Native

> Newer RN versions ship critical performance fixes, not just features.

[← Monitoring](./01-monitoring.md) | [Index](./README.md) | [New Architecture →](./03-new-architecture.md)

**Keywords**: upgrade, version, Expo SDK, New Architecture, patch-package, upgrade-helper, breaking changes

---

## Why Update?

Each React Native release improves:
- **Native-side performance** — where most [ANRs](./01-monitoring.md) originate
- **Thread management** — reduces main thread blocking that causes [animation jank](./05-performance.md#animations)
- **Bridge communication** — or eliminates it entirely via [New Architecture](./03-new-architecture.md)
- **Framework bug fixes** — issues like ReactScrollView pointer crashes are fixed in newer versions

## New Architecture (RN 0.76+) {#new-architecture}

The biggest single performance upgrade. See [Chapter 3: New Architecture Deep Dive](./03-new-architecture.md) for full migration playbook with production metrics.

| Old Architecture | New Architecture |
|-----------------|-----------------|
| JS ↔ Bridge ↔ Native (async, serialized) | JS ↔ JSI ↔ Native (synchronous, direct) |
| Batched UI updates | Concurrent rendering support |
| Single-threaded bridge bottleneck | Multi-threaded Fabric renderer |

**Impact**: 17-27% faster cold start ([measured data](./03-new-architecture.md#performance-gains)), reduced [ANRs](./01-monitoring.md), smoother [animations](./05-performance.md#animations).

## How to Upgrade {#how-to-upgrade}

### Step 1: Use the Upgrade Helper

1. Visit [react-native-community.github.io/upgrade-helper](https://react-native-community.github.io/upgrade-helper/)
2. Select your **current** and **target** versions
3. Apply **every** file change shown in the diff

### Step 2: Update Dependencies

After upgrading RN, third-party libraries need updating too. Incompatible versions cause:
- ANRs and crashes — see [Monitoring](./01-monitoring.md)
- [Memory leaks](./06-profiling.md#memory-leaks)
- Build failures

```bash
yarn outdated
yarn why <package-name>
```

**Real example**: In a production release, upgrading `@react-native-firebase/*` from v19 to v23 (4 major versions) caused a regression in the signup flow — the modular API migration broke analytics calls.

> **See also**: [Dependencies: Safe Removal Process](./04-dependencies.md#auditing) for how to safely audit and update packages.

### Step 3: Patch When Needed

Some libraries lag behind RN releases:
1. Check the library's **GitHub issues**
2. Use `patch-package` to apply community patches
3. Consider [lighter alternatives](./04-dependencies.md#lighter-alternatives) if unmaintained

### Step 4: Verify

```bash
# Run your CI pipeline (type-check + lint + tests)
yarn test && yarn lint && yarn tsc

# Build for each platform
yarn android
yarn ios
```

Monitor [crash rates](./01-monitoring.md#monitor-fixes) via Crashlytics/Sentry for 48-72 hours post-release.

## Upgrade Cadence

- **Minor versions** (0.76.1 → 0.76.2): Apply quickly, usually bug fixes
- **Major versions** (0.75 → 0.76): Plan a dedicated sprint, test thoroughly
- **Expo SDK updates**: Follow Expo's upgrade guide, which handles RN version alignment

## Common Pitfalls

- **Don't skip versions** — upgrading from 0.72 to 0.76 is harder than incremental upgrades
- **Don't delay indefinitely** — the longer you wait, the harder the upgrade. A Million Monkeys adds 30% to estimates when >2 versions behind ([source](./03-new-architecture.md#timeline))
- **Test on real devices** — emulators don't surface all ANRs. See [device segmentation](./01-monitoring.md#device-segmentation)
- **Watch for native module compatibility** — see [New Architecture: Audit Dependencies](./03-new-architecture.md#migration-playbook)

## Example Audit

A production crash analysis showed that several framework bugs were fixed in newer RN versions. Upgrading RN eliminated those crashes without any app-side code changes.

From the [SIGABRT guide](../SIGABRT-libc-debugging-guide.md): keeping `@react-native-picker/picker` at outdated versions is a risk — always check for known SIGABRT fixes in patch releases.

---

**Next**: [New Architecture Deep Dive →](./03-new-architecture.md) — the biggest single performance upgrade available
