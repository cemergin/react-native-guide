# Keeping React Native Updated

> Newer RN versions ship critical performance fixes, not just features.

[Back to Index](./README.md) | Previous: [ANR Analysis](./anr-analysis.md) | Next: [Dependencies](./dependency-management.md)

---

## Why Update?

Each React Native release improves:
- **Native-side performance** — where most [ANRs](./anr-analysis.md) originate
- **Code execution and rendering logic**
- **Thread management** — reduces main thread blocking
- **Bridge communication** — or eliminates it entirely (New Architecture)

## New Architecture (RN 0.76+)

The biggest single performance upgrade available. Enabled by default in 0.76+:

| Old Architecture | New Architecture |
|-----------------|-----------------|
| JS ↔ Bridge ↔ Native (async, serialized) | JS ↔ JSI ↔ Native (synchronous, direct) |
| Batched UI updates | Concurrent rendering support |
| Single-threaded bridge bottleneck | Multi-threaded fabric renderer |

**Impact**: Reduced ANRs, smoother animations (see [Performance](./performance-fundamentals.md#1-animation-performance)), faster startup.

## How to Upgrade {#how-to-upgrade}

### Step 1: Use the Upgrade Helper

1. Visit [react-native-community.github.io/upgrade-helper](https://react-native-community.github.io/upgrade-helper/)
2. Select your **current** and **target** versions
3. Apply **every** file change shown in the diff

### Step 2: Update Dependencies

After upgrading RN, third-party libraries need updating too. Incompatible versions cause:
- ANRs and crashes (see [ANR Analysis](./anr-analysis.md))
- Memory leaks
- Build failures

```bash
# Check which dependencies need updating
yarn outdated

# For each critical library, check compatibility
yarn why <package-name>
```

### Step 3: Patch When Needed

Some libraries lag behind RN releases. When that happens:
1. Check the library's **GitHub issues** — someone has likely solved it
2. Use `patch-package` to apply community patches
3. Consider [lighter alternatives](./dependency-management.md#lighter-alternatives) if the library is unmaintained

### Step 4: Verify

```bash
# Build both platforms
yarn android
yarn ios

# Run full test suite
yarn test

# Check crash rates post-release
# Monitor via Crashlytics/Sentry for 48-72 hours
```

## Upgrade Cadence

- **Minor versions** (0.76.1 → 0.76.2): Apply quickly, usually bug fixes
- **Major versions** (0.75 → 0.76): Plan a dedicated sprint, test thoroughly
- **Expo SDK updates**: Follow Expo's upgrade guide, which handles RN version alignment

## Common Pitfalls

- **Don't skip versions** — upgrading from 0.72 to 0.76 is harder than 0.72→0.73→...→0.76
- **Don't delay indefinitely** — the longer you wait, the harder the upgrade
- **Test on real devices** — emulators don't surface all ANRs (see [device segmentation](./anr-analysis.md#device-segmentation))

---

**Next**: [Dependency Management](./dependency-management.md) — reduce bundle size and attack surface
