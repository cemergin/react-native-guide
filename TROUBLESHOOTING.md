# Troubleshooting FAQ

> The error messages you Google at 2am, with fixes that actually work.

[← Back to Index](./README.md)

---

## Build Errors

### `Unable to resolve module` / `Module not found`

**Cause**: Metro bundler cache is stale after installing/removing packages.

**Fix**:
```bash
npx react-native start --reset-cache
# Or with Expo:
npx expo start --clear
```

If that doesn't work, nuke everything:
```bash
rm -rf node_modules && yarn install
cd ios && pod deintegrate && pod install && cd ..
npx react-native start --reset-cache
```

---

### `Duplicate class kotlin.collections.AbstractMutableList` (Android)

**Cause**: Multiple Kotlin versions pulled by different dependencies.

**Fix**: Pin Kotlin version in `android/build.gradle`:
```groovy
ext {
    kotlinVersion = "1.9.22"  // Match your highest dependency requirement
}
```

---

### Pod install fails: `CocoaPods could not find compatible versions`

**Cause**: Cached Pods from a previous RN version.

**Fix**:
```bash
cd ios
pod deintegrate
rm -rf Pods Podfile.lock
pod install --repo-update
cd ..
```

---

### `SDK version mismatch` (Expo)

**Cause**: Dependencies don't match your Expo SDK version.

**Fix**:
```bash
npx expo install --check    # Shows mismatched versions
npx expo install --fix       # Auto-fixes to compatible versions
npx expo-doctor             # Diagnoses additional issues
```

---

### Xcode build fails: `Multiple commands produce '...'`

**Cause**: Duplicate resource files or conflicting build phases.

**Fix**:
```bash
rm -rf ~/Library/Developer/Xcode/DerivedData/*
cd ios && pod deintegrate && pod install
```

---

## Runtime Crashes

### `SIGABRT in libc.so` / `Signal 6`

**Cause**: Multiple possible — native memory corruption, buggy native library, OOM.

**Fix**: Follow the decision tree in the [Crash Analysis Guide](./debugging/crash-analysis.md):
1. Check if a specific `.so` file is in the stack trace → upgrade that library
2. Check if crash-free rate dropped across ALL versions → Google Play system update
3. Check if 80%+ crashes are on one OEM → OEM-specific bug
4. Run with ASan to identify exact memory issue

---

### `Invariant Violation: requireNativeComponent "RCTView" was not found`

**Cause**: New Architecture / Fabric mismatch — component expects old renderer.

**Fix**:
- Ensure New Architecture is enabled in both iOS and Android configs
- Upgrade the crashing library to a Fabric-compatible version
- If a custom native module, migrate to TurboModules

---

### `TurboModuleRegistry: module not found`

**Cause**: A TurboModule isn't properly registered or the native build is stale.

**Fix**:
```bash
cd android && ./gradlew clean && cd ..
cd ios && pod deintegrate && pod install && cd ..
npx react-native start --reset-cache
```

---

### App crashes on startup (<1 second)

**Cause**: Native initialization failure — too many SDKs loading synchronously, memory pressure, or missing native module.

**Fix**:
1. Check `adb logcat` / Xcode console for the first error
2. Move Firebase/analytics/SDK initialization to background threads
3. Check if a native module is missing (common after upgrading)
4. Profile with Perfetto startup trace → [Profiling Tools Guide](./debugging/profiling-tools-deep-dive.md)

---

### `ViewPropTypes will be removed from React Native` / `ViewPropTypes` crash

**Cause**: Library using deprecated `ViewPropTypes` removed in RN 0.73+.

**Fix**:
```bash
# Install the compatibility shim
yarn add deprecated-react-native-prop-types

# Or better: upgrade the library to a version that doesn't use PropTypes
```

---

## Performance Issues

### ANR (App Not Responding) on Android

**Cause**: Main thread blocked for >5 seconds.

**Common culprits**:
1. Synchronous SDK initialization at startup → move to background thread
2. Heavy computation in component render → move to `useMemo` or worker
3. `useNativeDriver: false` animations → switch to `true` or Reanimated
4. Large list without virtualization → switch to FlashList

**Debug**: [Monitoring Guide](./optimization/monitoring-anr-analysis.md) for systematic ANR analysis.

---

### Memory grows continuously (leak)

**Cause**: Uncleaned listeners, intervals, or fetches.

**Quick check**:
```tsx
// Most common leaks — search your codebase for these:
useEffect(() => {
  // Missing return cleanup?
  EventEmitter.addListener('event', handler);
  setInterval(fn, 5000);
  fetch(url).then(setData);
  const ws = new WebSocket(url);
}, []);
```

Every `useEffect` that subscribes to something needs a cleanup return. See [Error Handling: Common Patterns](./quality/error-handling.md).

---

### Scroll jank / dropped frames in lists

**Fix priority**:
1. Replace `FlatList` with `FlashList` (biggest single win)
2. Add `getItemType` for mixed content lists
3. Wrap `renderItem` in `React.memo`
4. Stabilize callbacks with `useCallback`
5. Reduce item complexity (fewer nested views)
6. Test on low-end device ($50 Android phone)

See [Performance: Lists](./optimization/performance-rendering.md#lists).

---

### Cold start >3 seconds

**Fix priority**:
1. Check bundle size with Expo Atlas → if >15MB, audit dependencies
2. Move SDK initialization to background threads
3. Add `lazy: true` to stack navigators
4. Enable baseline profiles (Android) for 25-30% improvement
5. Check for synchronous `require()` calls at top of entry file

See [Profiling: Startup](./optimization/profiling-debugging.md#startup-optimization).

---

## Expo-Specific Issues

### `expo-notifications`: Taps not detected when app is killed

**Cause**: Using `addNotificationResponseReceivedListener` which doesn't fire from killed state.

**Fix**: Use `useLastNotificationResponse()` hook instead:
```tsx
import * as Notifications from 'expo-notifications';

const response = Notifications.useLastNotificationResponse();

useEffect(() => {
  if (response) {
    const url = response.notification.request.content.data?.url;
    if (url) router.push(url);
  }
}, [response]);
```

---

### `expo-secure-store`: Value too large (iOS)

**Cause**: iOS Keychain has a 2048-byte limit per item.

**Fix**: For larger data, encrypt it with a key stored in SecureStore:
```tsx
// Store the encryption key in SecureStore (small)
await SecureStore.setItemAsync('encryption_key', generatedKey);

// Store the encrypted data in MMKV (no size limit)
const encrypted = encrypt(largeData, generatedKey);
mmkv.set('large_secure_data', encrypted);
```

---

### EAS Build fails: `Credentials not found`

**Cause**: Missing or expired signing credentials.

**Fix**:
```bash
# Regenerate credentials
eas credentials

# Or let EAS manage them automatically
eas build --platform ios --auto-submit
```

See [EAS Complete Guide](./expo/eas-complete-guide.md) for credential management.

---

### `Cannot read property 'navigate' of undefined` (Expo Router)

**Cause**: Using `router` outside of navigation context, or Expo Router v4 behavior change.

**Fix**:
```tsx
// Ensure you're using the router from expo-router
import { router } from 'expo-router';

// In v4, router.navigate() = router.push() (breaking change from v3)
// If you want v3 "smart navigate" behavior, manually check history
```

---

## React Native DevTools

### DevTools won't connect / blank profiler

**Fix**:
```bash
# 1. Make sure Metro is running
npx expo start

# 2. Press 'j' in the terminal to open DevTools
# 3. If still blank, try direct connection:
# Open chrome://inspect in Chrome
# Click "Configure" → add localhost:8081
```

---

### Hermes profiler shows no data

**Cause**: Profiler needs Hermes-specific connection.

**Fix**: Use the React Native DevTools (press `j` in Metro), NOT Chrome DevTools Performance tab. Hermes profiling works through the built-in DevTools profiler.

---

## Quick Diagnostic Commands

```bash
# Check what's eating your bundle
EXPO_ATLAS=true bunx expo start

# Check for dependency issues
npx expo-doctor
npx expo install --check

# Check native module compatibility
npx react-native info

# Clean everything
watchman watch-del-all 2>/dev/null
rm -rf node_modules .expo
yarn install
npx react-native start --reset-cache

# Android: full clean
cd android && ./gradlew clean && cd ..

# iOS: full clean
cd ios && pod deintegrate && rm -rf Pods Podfile.lock && pod install && cd ..
rm -rf ~/Library/Developer/Xcode/DerivedData/*
```

---

*Can't find your error here? Check the [Crash Analysis Guide](./debugging/crash-analysis.md) for native crashes or [Error Handling Guide](./quality/error-handling.md) for JS error patterns.*
