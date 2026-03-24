[← Back to Index](../README.md)

# Expo & EAS Configuration Decision Guide

A comprehensive, decision-oriented guide for React Native / Expo developers. Every configuration option is explained with **what it does**, **why you would change it**, **trade-offs**, and **common mistakes**.

This guide covers `app.json` / `app.config.ts`, the `expo-build-properties` plugin, OTA updates, `eas.json`, and deep linking architecture.

---

## Table of Contents

1. [Choosing Your Config File Format](#1-choosing-your-config-file-format)
2. [Core Identity Properties](#2-core-identity-properties)
3. [Display & UX Configuration](#3-display--ux-configuration)
4. [iOS Configuration Decision Guide](#4-ios-configuration-decision-guide)
5. [Android Configuration Decision Guide](#5-android-configuration-decision-guide)
6. [The expo-build-properties Plugin](#6-the-expo-build-properties-plugin)
7. [OTA Updates Configuration](#7-ota-updates-configuration)
8. [Plugins Configuration Strategy](#8-plugins-configuration-strategy)
9. [eas.json Configuration Decision Guide](#9-easjson-configuration-decision-guide)
10. [New Architecture Configuration](#10-new-architecture-configuration)
11. [The extra Field -- Runtime Configuration Bridge](#11-the-extra-field--runtime-configuration-bridge)
12. [Deep Linking Architecture](#12-deep-linking-architecture)
13. [Sources](#13-sources)

---

## 1. Choosing Your Config File Format

### The Three Options

Expo supports three configuration file formats, each with different capabilities:

| Format | Dynamic Logic | TypeScript | Environment Variables | Auto-complete |
|---|---|---|---|---|
| `app.json` | No | No | No | Limited |
| `app.config.js` | Yes | No | Yes | Limited |
| `app.config.ts` | Yes | Yes | Yes | Full |

### app.json -- The Simple Default

```json
{
  "expo": {
    "name": "My App",
    "slug": "my-app",
    "version": "1.0.0"
  }
}
```

**When to use it:**
- Prototyping or very simple apps
- When you have zero need for environment-specific configuration
- When the entire team is new to Expo and you want the simplest possible setup

**Limitations:**
- No conditional logic. You cannot change config based on `NODE_ENV`, build profiles, or feature flags.
- No comments. JSON does not support comments, so you cannot document why a value is set.
- No computed values. Everything is a literal.

### app.config.js -- Dynamic Without Types

```js
export default ({ config }) => ({
  ...config,
  name: process.env.APP_ENV === 'production' ? 'My App' : 'My App (Dev)',
  slug: 'my-app',
  version: '1.0.0',
});
```

**When to use it:**
- You need environment-based config but your team does not use TypeScript
- Migrating from `app.json` and want a quick upgrade path

### app.config.ts -- The Production Choice

```ts
import { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: process.env.APP_ENV === 'production' ? 'My App' : 'My App (Dev)',
  slug: 'my-app',
  version: '1.0.0',
  ios: {
    bundleIdentifier: 'com.yourcompany.yourapp',
  },
  android: {
    package: 'com.yourcompany.yourapp',
  },
});
```

**When to use it:**
- Any production app. The type safety catches misconfigurations before they reach a build.
- When you want IDE auto-complete for every config property.
- When you need to import shared constants (API URLs, feature flags) into your config.

### File Precedence Rules

Expo resolves config files in this order:
1. `app.config.ts` (highest priority)
2. `app.config.js`
3. `app.json` (lowest priority)

If multiple files exist, the dynamic config (`app.config.ts` or `app.config.js`) receives the static `app.json` values as the `config` parameter. The dynamic config's return value is what Expo uses.

### Common Mistake: Having Both app.json and app.config.ts

This is not inherently wrong, but it causes confusion. When both exist:
- `app.json` values are passed into `app.config.ts` via the `config` parameter
- If your `app.config.ts` does not spread `...config`, the `app.json` values are silently ignored
- Team members may edit `app.json` thinking it is the source of truth, but their changes have no effect

**Recommendation:** If you use `app.config.ts`, either:
- Remove `app.json` entirely and define everything in TypeScript, or
- Keep `app.json` as a minimal base and always spread `...config` in your dynamic config

### Environment Variables in Config

In `app.config.ts`, you access environment variables via `process.env`:

```ts
const IS_PRODUCTION = process.env.APP_ENV === 'production';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: IS_PRODUCTION ? 'My App' : `My App (${process.env.APP_ENV})`,
  ios: {
    bundleIdentifier: IS_PRODUCTION
      ? 'com.yourcompany.yourapp'
      : 'com.yourcompany.yourapp.dev',
  },
});
```

These environment variables are resolved at **build time** (during `expo prebuild` or `eas build`), not at runtime. This is a critical distinction. You cannot change these values after the app is built.

---

## 2. Core Identity Properties

### name

**What it does:** The display name shown under the app icon on the user's home screen, in the app switcher, and in the app store listing.

**Why you would change it:** Different environments (dev/staging/production) benefit from different names so testers can distinguish them on-device.

**Trade-offs:**
- iOS truncates names longer than ~12 characters under the icon. Android gives slightly more room (~14 characters) but varies by launcher.
- Shorter is always safer. "My App" beats "My Application Name" on the home screen.

**Common mistakes:**
- Setting the name to your full company name + product name. It will be truncated with "..." and look unprofessional.
- Forgetting to differentiate dev/staging builds. You end up with three identical-looking "My App" icons on your test device.

```ts
// Good: environment-aware naming
name: IS_PRODUCTION ? 'My App' : `My App ${APP_ENV.toUpperCase()}`,
```

### slug

**What it does:** A URL-safe identifier for your project on Expo's servers. Used in EAS Build, EAS Update, and Expo Go URLs.

**Why you would change it:** You would not -- after initial setup. The slug is your project's permanent identity.

**Trade-offs:** None. Pick a good one and leave it.

**Common mistakes:**
- Changing the slug after publishing OTA updates. This creates a **new** project on Expo's servers. Existing users will stop receiving updates because the old project is effectively abandoned.
- Using characters that are not URL-safe (spaces, uppercase). Expo will reject them but the error message can be confusing.

### owner

**What it does:** Specifies the Expo account (personal or organization) that owns the project.

**Why you would change it:**
- When you are transferring a project from a personal account to a team/organization account
- When your CI/CD uses a robot account that differs from the project owner

**Trade-offs:**
- If owner is an organization, all members of that organization can access the project
- If owner is a personal account, only that account can manage builds and updates

**Common mistakes:**
- Not setting owner explicitly, then having builds fail in CI because the CI bot's account does not match the implicit owner.
- Transferring ownership without updating eas.json and environment variables -- builds break silently.

### version

**What it does:** The user-facing version string displayed in app store listings and the Settings > About screen. This is purely cosmetic from the OS perspective.

**Why you would change it:** Every app store submission should increment this for user visibility. Follow semantic versioning (MAJOR.MINOR.PATCH).

**The critical distinction -- version vs buildNumber vs versionCode:**

| Property | Platform | Purpose | User sees it? | Store requires unique? |
|---|---|---|---|---|
| `version` | Both | User-facing version | Yes (store listing) | No (can reuse) |
| `ios.buildNumber` | iOS | Internal build identifier | No (unless they look) | Yes (per version) |
| `android.versionCode` | Android | Internal build integer | No | Yes (must always increase) |

**Common mistakes:**
- Confusing `version` with `buildNumber`. You can submit version "2.0.0" with buildNumber "1", then "2.0.0" with buildNumber "2" (a hotfix). Users see "2.0.0" both times.
- Forgetting to increment `versionCode` on Android. Google Play rejects uploads with duplicate versionCodes, and versionCode must be **strictly increasing** across all tracks.

### scheme

**What it does:** Registers a custom URL scheme (e.g., `myapp://`) so the OS can open your app from links.

**Why you would change it:**
- You want deep linking: `myapp://profile/123` opens the profile screen
- OAuth redirect flows often require a custom scheme
- Development tools like Expo Go use the scheme to launch your app

**Trade-offs:**
- **Security:** Custom schemes are NOT secure. Any app can register the same scheme. On Android, the OS shows a disambiguation dialog. On iOS, the behavior is undefined. For sensitive flows (auth, payments), use Universal Links / App Links instead.
- **Build-time only:** This is baked into the native project at build time. You cannot change the scheme via OTA updates.

**Common mistakes:**
- Using a generic scheme like `app://` or `myapp://` that conflicts with other apps on the device.
- Relying on custom schemes for authentication without a fallback. If another app hijacks the scheme, your auth flow breaks silently.
- Not setting scheme at all, then wondering why `Linking.createURL()` returns `exp://` in development and nothing in production.

### platforms

**What it does:** Declares which platforms your app targets. Typically `["ios", "android"]`.

**Why you would change it:** If you genuinely only support one platform, removing the other prevents accidental builds.

**Common mistakes:**
- Removing a platform and forgetting to remove platform-specific config. This does not cause errors but is dead code in your config.

---

## 3. Display & UX Configuration

### orientation

**What it does:** Controls which screen orientations your app supports.

| Value | Behavior |
|---|---|
| `"default"` | Allows all orientations (portrait + landscape) |
| `"portrait"` | Locks to portrait only |
| `"landscape"` | Locks to landscape only |

**Decision framework:**
- **Most apps:** Use `"portrait"`. Users expect phone apps to be portrait. Rotation causes layout issues that are expensive to test.
- **Media/video apps:** Use `"default"`. Users expect to rotate for full-screen video.
- **Games/drawing apps:** Use `"default"` or `"landscape"` depending on your UI.
- **Tablet-focused apps:** Use `"default"`. iPad users expect rotation support; Apple may reject portrait-locked iPad apps.

**Trade-offs:**
- Locking to portrait means you do not need to handle landscape layouts at all -- significant development time savings.
- Supporting rotation means every screen must look good in both orientations. Test thoroughly.
- You can override orientation per-screen at runtime using `expo-screen-orientation`, but the app.config value sets the default.

**Common mistakes:**
- Setting `"default"` and never testing landscape. Your app renders but with overlapping elements and cut-off text.
- Forgetting that `"portrait"` on iPad may trigger App Store review pushback if you claim tablet support.

### userInterfaceStyle

**What it does:** Controls whether your app supports dark mode, light mode, or follows the system setting.

| Value | Behavior |
|---|---|
| `"light"` | Always light, ignores system dark mode |
| `"dark"` | Always dark, ignores system light mode |
| `"automatic"` | Follows system setting |

**Decision framework:**
- **"light"** is the safe default if you have not designed dark mode. Shipping a half-baked dark mode is worse than not supporting it.
- **"automatic"** is the goal for a polished app, but requires:
  - Every screen designed for both themes
  - All hardcoded colors replaced with theme-aware values
  - `expo-system-ui` installed (required for Android to respect this setting)
  - Testing in both modes

**Trade-offs:**
- `"automatic"` with incomplete dark mode support results in white text on white backgrounds -- unreadable screens that users will report immediately.
- `"light"` means dark mode users see a bright flash when opening your app at night. Not ideal but not broken.

**Common mistakes:**
- Setting `"automatic"` before implementing dark mode fully. The result is a broken mix of light and dark elements.
- Forgetting to install `expo-system-ui` on Android. The setting silently does nothing without the package.

### icon

**What it does:** The app icon displayed on the home screen and in store listings.

**Requirements:**
- **Size:** 1024x1024 pixels (Expo scales down automatically)
- **Format:** PNG
- **No transparency:** iOS fills transparent areas with black, which looks terrible
- **No rounded corners:** Both iOS and Android apply their own corner rounding. If you pre-round, you get double-rounding with a visible gap.
- **No alpha channel:** Some Android launchers render the alpha channel as black

**Common mistakes:**
- Providing a 512x512 icon. It works but looks blurry on high-density devices and in the App Store.
- Adding rounded corners to the source image. iOS applies its own squircle mask; Android applies its own shape.
- Using transparency. The black fill on iOS is a frequent source of store review rejection.

### Splash Screen (expo-splash-screen plugin)

The splash screen is configured via the `expo-splash-screen` config plugin:

```ts
plugins: [
  [
    'expo-splash-screen',
    {
      backgroundColor: '#FFFFFF',
      image: './assets/splash-icon.png',
      imageWidth: 200,
    },
  ],
],
```

**Key decisions:**

**backgroundColor:** Must match the dominant color of your splash image. If your splash image has a white background and you set `backgroundColor` to black, there is a visible color flash during load.

**The SplashScreen.preventAutoHideAsync() pattern:**

```ts
import * as SplashScreen from 'expo-splash-screen';

SplashScreen.preventAutoHideAsync();

function App() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    async function prepare() {
      await loadFonts();
      await loadInitialData();
      setReady(true);
      SplashScreen.hideAsync();
    }
    prepare();
  }, []);

  if (!ready) return null;
  return <MainApp />;
}
```

**Why this matters:** Without `preventAutoHideAsync()`, the splash screen disappears as soon as the JS bundle loads, potentially showing a blank screen while fonts and data are still loading.

**Android 12+ behavior change:** Starting with Android 12 (API 31), the OS controls the splash screen via the `SplashScreen` API. Full-screen splash images are no longer supported. Instead, Android 12+ shows:
- A centered icon (with optional branding image)
- A background color

This means your carefully designed full-screen splash illustration will not display on newer Android devices. Design your splash screen with a centered icon approach to be forward-compatible.

**Common mistakes:**
- Not calling `preventAutoHideAsync()` early enough. If any async operation runs before this call, the splash screen may have already dismissed.
- Calling `hideAsync()` inside a component that unmounts before execution. The splash screen stays forever.
- Designing a full-screen splash without realizing Android 12+ ignores it.

---

## 4. iOS Configuration Decision Guide

### bundleIdentifier

**What it does:** The unique identifier for your app in the Apple ecosystem. Format: reverse domain notation (e.g., `com.yourcompany.yourapp`).

**Why you would change it:** You would not, after your first build. The bundle ID is tied to:
- Your App Store listing
- Push notification certificates
- Keychain access groups
- Associated domains
- All provisioning profiles

**Trade-offs for the initial choice:**
- Use your company's domain in reverse: `com.yourcompany.yourapp`
- For multiple build variants: `com.yourcompany.yourapp.dev`, `com.yourcompany.yourapp.staging`
- Different bundle IDs = different apps on the device (can install dev and production side by side)
- Same bundle ID = same app, new install overwrites the old one

**Common mistakes:**
- Changing the bundle identifier after publishing. You lose your App Store listing, all reviews, download history, and existing push certificates.
- Using different bundle IDs per environment without realizing each needs its own push certificate, App Store Connect entry, and provisioning profile.
- Typos. `com.yourcomapny.yourapp` is a different app forever.

### supportsTablet

**What it does:** When `true`, your app renders natively on iPad. When `false`, iPad runs the iPhone version in a compatibility mode (small, centered, with a 2x zoom button).

**Decision framework:**
- **Set to `true` if:** You have designed responsive layouts that work on larger screens, or you plan to.
- **Set to `false` if:** Your app is phone-only and you have not tested on iPad at all.

**Trade-offs:**
- `true`: Apple expects your app to actually work well on iPad. Poor iPad support can lead to review rejection.
- `false`: iPad users get a subpar experience (a phone-sized window on their 12.9" screen). Some users will leave negative reviews.
- If your app is listed on iPad (supportsTablet: true), Apple reviews it on iPad too. Layout bugs that are invisible on iPhone become rejection reasons.

**Common mistakes:**
- Setting `true` without testing on iPad. Modals, keyboards, and navigation can behave very differently.
- Not understanding that `supportsTablet: true` means Apple tests your app on iPad during review.

### requireFullScreen (UIRequiresFullScreen)

**What it does:** When `true`, your app cannot participate in iPad Split View or Slide Over multitasking.

**Decision framework:**
- **Set to `true` if:** Your app genuinely cannot function in a split-screen context (games, camera-heavy apps, AR experiences).
- **Set to `false` (or omit) if:** Your app can handle variable window sizes. Apple increasingly expects multitasking support.

**Trade-offs:**
- `true` simplifies iPad development (no variable window sizes) but may draw scrutiny from Apple reviewers who feel you should support multitasking.
- `false` means you must handle all possible window sizes on iPad, including very narrow split-view widths (~320pt).

### associatedDomains

**What it does:** Declares the domains your app is associated with, enabling Universal Links (tap a link, open the app instead of Safari).

```ts
ios: {
  associatedDomains: [
    'applinks:yourapp.com',
    'applinks:www.yourapp.com',
  ],
},
```

**Prerequisite:** Your web server must host an Apple App Site Association (AASA) file at `https://yourapp.com/.well-known/apple-app-site-association` with the correct team ID and bundle identifier.

**Common mistakes:**
- Forgetting the `applinks:` prefix. The raw domain does not work.
- Not hosting the AASA file over HTTPS. Apple requires it.
- AASA file has incorrect team ID or bundle ID. Links silently fall back to Safari.
- Not testing with a fresh install. Universal Links are cached aggressively. Clear the cache by reinstalling the app.
- Adding `webcredentials:` domains without understanding they enable iCloud Keychain password sharing.

### entitlements

**What it does:** Declares iOS capabilities your app needs beyond the defaults.

```ts
ios: {
  entitlements: {
    'aps-environment': 'production',
    'com.apple.developer.applesignin': ['Default'],
    'com.apple.developer.in-app-payments': ['merchant.com.yourcompany.yourapp'],
  },
},
```

**Key entitlements and when you need them:**

| Entitlement | When You Need It |
|---|---|
| `aps-environment` | Push notifications (set to `"development"` or `"production"`) |
| `com.apple.developer.applesignin` | Sign in with Apple |
| `com.apple.developer.in-app-payments` | Apple Pay |
| `com.apple.developer.associated-domains` | Universal Links (usually set via associatedDomains) |
| `com.apple.developer.nfc.readersession.formats` | NFC reading |

**Common mistakes:**
- Setting `aps-environment` to `"production"` in development builds. Push tokens from the sandbox environment will not work.
- Declaring entitlements that are not enabled in your Apple Developer portal. The build will succeed but the capability will silently fail at runtime.

### infoPlist Deep Dive

The `infoPlist` field lets you set arbitrary `Info.plist` values. This is where many critical iOS behaviors are configured.

#### Permission Usage Strings

Apple requires a human-readable explanation for every sensitive permission your app requests. If missing, your app **will be rejected**.

```ts
ios: {
  infoPlist: {
    NSCameraUsageDescription: 'We need camera access to let you take profile photos.',
    NSPhotoLibraryUsageDescription: 'We need photo library access to let you choose a profile picture.',
    NSLocationWhenInUseUsageDescription: 'We use your location to show nearby stores.',
    NSMicrophoneUsageDescription: 'We need microphone access for voice messages.',
    NSFaceIDUsageDescription: 'We use Face ID to securely log you in.',
  },
},
```

**Rejection risks:**
- **Missing string:** Immediate rejection. No appeal.
- **Vague string:** "This app needs camera." Apple increasingly rejects generic descriptions. Be specific about WHY.
- **Including permissions you do not use:** If you declare `NSCameraUsageDescription` but never access the camera, Apple may reject you for requesting unnecessary permissions.
- **Third-party libraries adding permissions:** Libraries like `react-native-image-picker` automatically add camera/photo permissions. If your app does not need them, you must explicitly remove them (see Android `blockedPermissions` for the Android equivalent; on iOS, you need a config plugin to remove them from Info.plist).

#### CADisableMinimumFrameDurationOnPhone

```ts
CADisableMinimumFrameDurationOnPhone: true,
```

**What it does:** Enables ProMotion (120Hz) rendering on iPhone 13 Pro and newer. Without this, your app is capped at 60fps even on 120Hz displays.

**Trade-offs:**
- Smoother scrolling and animations on supported devices
- Slightly higher battery consumption
- No effect on older devices

**Recommendation:** Enable it. Users with ProMotion devices expect 120Hz. The battery impact is negligible for most apps.

#### CFBundleAllowMixedLocalizations

```ts
CFBundleAllowMixedLocalizations: true,
```

**What it does:** Allows your app to display strings from multiple localizations simultaneously. Without this, iOS forces a single language for all system-provided strings.

**When to use:** If your app supports multiple languages and uses third-party frameworks that provide their own localizations.

#### NSAppTransportSecurity

```ts
NSAppTransportSecurity: {
  NSAllowsArbitraryLoads: true,
},
```

**What it does:** Disables App Transport Security, allowing HTTP (non-HTTPS) connections.

**Trade-offs:**
- **Security risk:** HTTP traffic can be intercepted and modified (man-in-the-middle attacks).
- **Apple scrutiny:** If you set `NSAllowsArbitraryLoads: true`, Apple requires a justification during review. You may be rejected.
- **When it is necessary:** Loading images from user-provided URLs that may not support HTTPS, connecting to local development servers, or communicating with legacy APIs.

**Better alternative:** Instead of allowing all arbitrary loads, use `NSExceptionDomains` to whitelist specific HTTP domains:

```ts
NSAppTransportSecurity: {
  NSExceptionDomains: {
    'legacy-api.example.com': {
      NSExceptionAllowsInsecureHTTPLoads: true,
      NSIncludesSubdomains: true,
    },
  },
},
```

#### UIBackgroundModes

```ts
UIBackgroundModes: ['audio', 'location', 'fetch', 'remote-notification'],
```

**What it does:** Declares what background tasks your app performs.

| Mode | When to Use |
|---|---|
| `audio` | Audio playback, VoIP |
| `location` | GPS tracking (fitness, delivery) |
| `fetch` | Background data refresh |
| `remote-notification` | Silent push notifications that trigger background work |

**Common mistakes:**
- Declaring `location` without using it. Apple audits background modes and will reject your app.
- Declaring `audio` for a non-audio app just to keep it alive in the background. Apple explicitly rejects this.

#### UIViewControllerBasedStatusBarAppearance

```ts
UIViewControllerBasedStatusBarAppearance: true,
```

**What it does:** When `true`, each view controller controls its own status bar style. When `false`, the app-level setting applies globally.

**Recommendation:** Set to `true` for React Native apps. This allows different screens to have different status bar styles (light content on dark headers, dark content on light headers).

### usesNonExemptEncryption

```ts
ios: {
  config: {
    usesNonExemptEncryption: false,
  },
},
```

**What it does:** Declares whether your app uses encryption beyond standard HTTPS. If `false`, you skip the export compliance questionnaire every time you submit to TestFlight/App Store.

**Decision framework:**
- **Set to `false` if:** Your app only uses HTTPS for network calls, standard iOS encryption APIs for local storage, or standard authentication. This covers 95% of apps.
- **Set to `true` if:** Your app implements custom encryption algorithms, uses encryption for purposes other than authentication/HTTPS, or is subject to encryption export restrictions.

**Common mistakes:**
- Not setting this at all. Every TestFlight upload prompts you with an export compliance dialog, slowing down your CI/CD.
- Setting to `true` when you only use HTTPS. HTTPS is exempt. You are creating unnecessary paperwork.

### googleServicesFile

```ts
ios: {
  googleServicesFile: './config/GoogleService-Info.plist',
},
```

**What it does:** Points to the Firebase configuration file for iOS.

**Common mistakes:**
- Using the Android `google-services.json` file path. iOS needs `GoogleService-Info.plist`, a completely different file format.
- Having mismatched bundle IDs. The bundle ID in `GoogleService-Info.plist` must exactly match your `bundleIdentifier`.
- Committing the file to a public repository. It contains API keys (though Firebase API keys are not secret, it is still best practice to keep config files private).

---

## 5. Android Configuration Decision Guide

### package

**What it does:** The unique identifier for your app on Android (Google Play, device installs, etc.). Format: reverse domain notation (e.g., `com.yourcompany.yourapp`).

**Why it sometimes differs from iOS bundleIdentifier:**
- Historical reasons (different teams set up iOS and Android)
- Intentional differentiation for analytics/tracking
- Typos that are now permanent

**Recommendation:** Keep it identical to your iOS `bundleIdentifier` unless you have a strong reason not to. It simplifies Firebase setup, analytics, and team communication.

**Common mistakes:**
- Same as iOS bundleIdentifier: changing it after publishing loses your Google Play listing and all reviews.
- Using uppercase characters. Android package names must be lowercase.

### allowBackup

```ts
android: {
  allowBackup: false,
},
```

**What it does:** Controls whether the app's data is included in Android's auto-backup system (backs up to Google Drive).

**Decision framework:**
- **Set to `false` for:** Financial apps, apps with sensitive data, apps where security requirements prohibit data backup. Banking, healthcare, and enterprise apps should almost always disable this.
- **Set to `true` (default) for:** Consumer apps where users expect their data to persist across device changes. Games (save progress), social apps (preferences), utility apps.

**Trade-offs:**
- `false`: Users lose all app data when switching devices. They will complain. But their sensitive data cannot be extracted from backups.
- `true`: Convenient for users but the backup can be extracted on rooted devices, potentially exposing tokens, cached data, and preferences.

**Common mistakes:**
- Leaving the default (`true`) for apps that store authentication tokens in AsyncStorage. Those tokens end up in unencrypted backups.

### adaptiveIcon

```ts
android: {
  adaptiveIcon: {
    foregroundImage: './assets/adaptive-icon-foreground.png',
    backgroundImage: './assets/adaptive-icon-background.png',
    backgroundColor: '#2196F3',
    monochromeImage: './assets/adaptive-icon-monochrome.png',
  },
},
```

**What it does:** Android Adaptive Icons use two layers (foreground + background) that the OS can animate, reshape, and theme.

**Layer requirements:**
- **foregroundImage:** 1024x1024 PNG. Your logo/icon centered within the inner 66% "safe zone" (the outer 34% may be cropped by different launcher shapes).
- **backgroundImage:** 1024x1024 PNG. A pattern, gradient, or solid color. Alternatively, use `backgroundColor` for a solid color.
- **monochromeImage:** (Android 13+) A single-color silhouette of your icon. Android 13 uses this for themed icons that match the user's wallpaper colors.

**Common mistakes:**
- Placing important parts of the foreground icon outside the safe zone. Samsung's squircle, Pixel's circle, and other launchers crop differently.
- Not providing `monochromeImage`. On Android 13+ with themed icons enabled, your app shows a generic placeholder instead of your icon.
- Using the same image for `icon` and `foregroundImage`. The adaptive icon foreground needs extra padding for the safe zone.

### permissions and blockedPermissions

```ts
android: {
  permissions: [
    'CAMERA',
    'READ_EXTERNAL_STORAGE',
    'ACCESS_FINE_LOCATION',
  ],
  blockedPermissions: [
    'USE_FULL_SCREEN_INTENT',
    'READ_PHONE_STATE',
    'RECORD_AUDIO',
  ],
},
```

**What it does:**
- `permissions`: Declares which Android permissions your app requests. Expo auto-detects many of these based on installed libraries, so you often do not need to set this manually.
- `blockedPermissions`: Explicitly prevents permissions from being added to the Android manifest, even if a library tries to add them.

**Why blockedPermissions matters:**
Many React Native libraries declare permissions in their AndroidManifest.xml that get merged into your app's manifest automatically. This means your app might request `RECORD_AUDIO` even though you never use the microphone -- just because a library you installed includes it as a "just in case" declaration.

**The USE_FULL_SCREEN_INTENT problem:**
Starting with Android 14 (API 34), `USE_FULL_SCREEN_INTENT` requires special approval from Google Play. Some notification/calling libraries add this automatically. If you do not actually need full-screen intent (for incoming calls, alarms), block it:

```ts
blockedPermissions: ['USE_FULL_SCREEN_INTENT'],
```

**Common mistakes:**
- Not auditing your merged manifest. Run `expo prebuild` and check `android/app/src/main/AndroidManifest.xml` to see what permissions actually end up in your app.
- Blocking a permission that a library genuinely needs, causing a runtime crash.
- Not realizing that Google Play reviews your declared permissions. Unnecessary sensitive permissions (like `READ_PHONE_STATE`) can trigger rejection or additional review.

### intentFilters (App Links)

```ts
android: {
  intentFilters: [
    {
      action: 'VIEW',
      autoVerify: true,
      data: [
        {
          scheme: 'https',
          host: 'yourapp.com',
          pathPrefix: '/app',
        },
      ],
      category: ['BROWSABLE', 'DEFAULT'],
    },
  ],
},
```

**What it does:** Declares URL patterns that should open your app instead of a browser. With `autoVerify: true`, Android verifies domain ownership via a `/.well-known/assetlinks.json` file on your server.

**autoVerify explained:**
- `true`: Android checks your domain's `assetlinks.json` at install time. If verified, your app opens directly without a disambiguation dialog. This is an "App Link."
- `false`: Android shows a dialog asking the user which app to open. This is a regular "Deep Link."

**Common mistakes:**
- Setting `autoVerify: true` but not hosting `assetlinks.json`. Android silently falls back to showing the disambiguation dialog, and you think intent filters are broken.
- Incorrect `assetlinks.json` (wrong package name or SHA-256 fingerprint). Verification fails silently.
- Not including both `BROWSABLE` and `DEFAULT` categories. Both are required for links from browsers.
- Forgetting that `pathPrefix: '/'` matches ALL paths on that domain. Be specific.

### softwareKeyboardLayoutMode

```ts
android: {
  softwareKeyboardLayoutMode: 'pan',
},
```

**What it does:** Controls how the app adjusts when the software keyboard appears.

| Value | Behavior | Best For |
|---|---|---|
| `"resize"` | The app's layout shrinks to fit above the keyboard | Chat UIs, forms where you want all content visible |
| `"pan"` | The app pans (scrolls up) so the focused input is visible, but layout size stays the same | Apps with fixed-position elements (bottom tabs, FABs) that should not move |

**Trade-offs:**
- `"resize"`: Causes a layout reflow every time the keyboard opens/closes. Can cause janky animations. Bottom-positioned elements jump up with the keyboard.
- `"pan"`: Smoother animation but content below the focused field may be hidden behind the keyboard. The user cannot scroll to see it.

**Common mistakes:**
- Using `"resize"` with a bottom tab navigator. The tabs jump up above the keyboard on every input focus.
- Using `"pan"` for a chat screen. The send button stays hidden behind the keyboard.
- Not testing both options. The right choice depends on your specific UI.

### googleServicesFile

```ts
android: {
  googleServicesFile: './config/google-services.json',
},
```

**What it does:** Points to the Firebase configuration file for Android.

**Critical requirement:** The `package_name` in `google-services.json` must exactly match your `android.package`. If they do not match, Firebase silently fails to initialize.

**Common mistakes:**
- Using the same file for dev and production when they have different package names. Download a separate `google-services.json` for each variant from the Firebase console.
- Not adding the file to `.gitignore` in public repositories.

### versionCode Strategy

```ts
android: {
  versionCode: 42,
},
```

**What it does:** An integer that must be strictly increasing for every upload to Google Play, across ALL tracks (internal, alpha, beta, production).

**Strategy options:**
- **Sequential:** 1, 2, 3, 4... Simple, but you cannot tell which version a versionCode corresponds to.
- **Date-based:** 20250321001 (YYYYMMDD + build number). Self-documenting, but large numbers.
- **Version-encoded:** Major * 10000 + Minor * 100 + Patch. e.g., 2.3.1 = 20301. Self-documenting and compact.
- **Auto-increment via EAS:** Let `eas build` manage it with `autoIncrement: true`. Simplest for teams.

**Common mistakes:**
- Uploading versionCode 10 to the internal track, then trying to upload versionCode 5 to production. versionCode must be higher than ANY previously uploaded code, across ALL tracks.
- Two developers building locally with the same versionCode. Use EAS auto-increment to avoid this.

---

## 6. The expo-build-properties Plugin

This plugin is one of the most important in your config because it controls **native build settings** that significantly affect compatibility, performance, and app size.

```ts
plugins: [
  [
    'expo-build-properties',
    {
      android: {
        minSdkVersion: 24,
        compileSdkVersion: 35,
        targetSdkVersion: 35,
        // ...
      },
      ios: {
        deploymentTarget: '16.0',
        // ...
      },
    },
  ],
],
```

### Android Build Properties

#### minSdkVersion -- Market Coverage vs API Availability

**What it does:** The minimum Android version your app supports.

| minSdkVersion | Android Version | Market Coverage (approx.) | Key APIs Available |
|---|---|---|---|
| 21 | 5.0 Lollipop | ~99.5% | Material Design, ART runtime |
| 23 | 6.0 Marshmallow | ~98% | Runtime permissions, fingerprint |
| 24 | 7.0 Nougat | ~96% | Multi-window, Java 8 lambdas |
| 26 | 8.0 Oreo | ~92% | Notification channels, PiP |
| 28 | 9.0 Pie | ~86% | Biometric prompt, cutout API |
| 31 | 12 | ~70% | Material You, SplashScreen API |
| 33 | 13 | ~50% | Per-app language, themed icons |

**Decision framework:**
- **Expo default (typically 23-24):** Safe for most apps. Covers 96%+ of devices.
- **Raise to 26+ if:** You want to drop support for old devices that generate disproportionate crash reports and support tickets.
- **Keep low (21-23) if:** Your target market includes regions with older devices (parts of Africa, Southeast Asia, South America).

**Trade-offs:**
- Lower minSdk = more users, but more compatibility bugs and device-specific issues.
- Higher minSdk = cleaner codebase, fewer polyfills, but excludes some users.

**Common mistake:** Raising minSdkVersion without checking your analytics first. You may be cutting off 5% of your active user base.

#### compileSdkVersion and targetSdkVersion

**What they do:**
- `compileSdkVersion`: Which Android SDK version your app is **compiled against**. Determines which APIs are available at compile time.
- `targetSdkVersion`: Which Android version your app is **designed for**. Controls OS behavior changes.

**The relationship:**

```
minSdkVersion <= targetSdkVersion <= compileSdkVersion
```

**Google Play requirements:** Google Play requires `targetSdkVersion` to be within 1-2 versions of the latest Android release. As of 2025, new apps must target API 34+ (Android 14). This requirement increases annually.

**Common mistakes:**
- Setting `compileSdkVersion` higher than `targetSdkVersion` without understanding that compile gives you API access but target changes behavior. Compiling against API 35 but targeting API 33 means you can USE API 35 methods but the OS treats your app as if it is designed for API 33.
- Not updating targetSdkVersion annually. Google Play will stop accepting updates to your app.

#### usesCleartextTraffic

```ts
usesCleartextTraffic: true,
```

**What it does:** Allows HTTP (non-encrypted) network traffic on Android. Starting with Android 9 (API 28), cleartext traffic is blocked by default.

**Decision framework:**
- **Production:** `false` (default). All traffic should be HTTPS.
- **Development:** `true` if you need to connect to local servers running on HTTP.
- **Never in production** unless you have a specific, justified need (legacy API that cannot be upgraded to HTTPS).

#### enableShrinkResourcesInReleaseBuilds

```ts
enableShrinkResourcesInReleaseBuilds: true,
```

**What it does:** Removes unused resources (images, strings, layouts) from the release APK/AAB.

**Trade-offs:**
- Reduces APK size, sometimes significantly (5-20% depending on your dependencies).
- Risk: If you reference resources dynamically (by string name at runtime), the shrinker may remove them. This is rare in React Native apps but can happen with native modules.

**Common mistakes:**
- Enabling this and then loading a resource by its string name at runtime, only to discover it was stripped from the APK.

#### enableMinifyInReleaseBuilds (R8/ProGuard)

```ts
enableMinifyInReleaseBuilds: true,
```

**What it does:** Enables R8 (the modern replacement for ProGuard) to minify, optimize, and obfuscate your Java/Kotlin code.

**Trade-offs:**
- **Pros:** Significantly smaller APK size. Removes dead code. Obfuscates class/method names.
- **Cons:** Can break libraries that use reflection, JNI, or dynamic class loading. These libraries need ProGuard "keep rules" to prevent their classes from being renamed or removed.

#### extraProguardRules -- When and Why You Need Them

```ts
extraProguardRules: `
  -keep class com.stripe.android.** { *; }
  -keep class com.google.mlkit.** { *; }
  -keep class com.facebook.hermes.** { *; }
  -dontwarn com.google.android.play.core.**
`,
```

**What it does:** Adds custom ProGuard/R8 rules to prevent minification from breaking specific libraries.

**Libraries that commonly need keep rules:**

| Library | Rule Needed | Why |
|---|---|---|
| Stripe | `-keep class com.stripe.android.** { *; }` | Uses reflection for payment processing |
| Firebase ML Kit | `-keep class com.google.mlkit.** { *; }` | Dynamic class loading for ML models |
| react-native-svg | `-keep class com.horcrux.svg.** { *; }` | Native view registration |
| Branch.io | `-keep class io.branch.** { *; }` | Deep link SDK uses reflection |
| Facebook SDK | `-keep class com.facebook.** { *; }` | Dynamic class loading |
| Google Play Core | `-dontwarn com.google.android.play.core.**` | Optional dependency warnings |
| Hermes | `-keep class com.facebook.hermes.** { *; }` | JS engine internals |
| react-native-reanimated | `-keep class com.swmansion.reanimated.** { *; }` | JNI calls |

**How to know you need a rule:** If a release build crashes but a debug build works, and the crash mentions `ClassNotFoundException` or `NoSuchMethodError`, you likely need a keep rule for the affected class.

**Common mistakes:**
- Enabling minification without any keep rules and wondering why the release build crashes.
- Using overly broad keep rules (`-keep class ** { *; }`) that defeat the purpose of minification.
- Not testing the release build thoroughly after enabling minification.

#### extraMavenRepos

```ts
extraMavenRepos: [
  'https://maven.example.com/releases',
  'https://jitpack.io',
],
```

**What it does:** Adds additional Maven repositories for resolving Android dependencies.

**When you need it:** When a library depends on artifacts not available in Google's Maven repository or Maven Central.

### iOS Build Properties

#### deploymentTarget

```ts
ios: {
  deploymentTarget: '16.0',
},
```

**What it does:** The minimum iOS version your app supports.

| Target | iOS Version | Device Coverage (approx.) | Notes |
|---|---|---|---|
| 15.0 | iOS 15 | ~99% | Still supported by Apple |
| 16.0 | iOS 16 | ~96% | Good default for most apps |
| 17.0 | iOS 17 | ~85% | Cuts off older devices like iPhone 8 |
| 18.0 | iOS 18 | ~60% | Too aggressive for most apps |

**Decision framework:**
- iOS users update much faster than Android users. A deployment target of 16.0 covers the vast majority.
- Apple drops support for older iOS versions on older hardware. Check which devices are cut off by your chosen target.

**Common mistake:** Setting the target too high without checking your user analytics. Even a 4% user drop can be significant at scale.

#### useFrameworks -- "static" vs "dynamic"

```ts
ios: {
  useFrameworks: 'static',
},
```

**What it does:** Controls how CocoaPods frameworks are linked.

| Value | Behavior | Pros | Cons |
|---|---|---|---|
| `"static"` | Libraries are compiled into the app binary | Faster launch time (no dynamic linking overhead), simpler | Larger binary size, recompiles everything on changes |
| `"dynamic"` | Libraries are separate frameworks loaded at runtime | Smaller binary (shared frameworks), faster incremental builds | Slower app launch, more complex build |

**When `"static"` is the right choice:**
- Your app has Firebase, Google Maps, or other large SDK dependencies. Static linking avoids the framework loading overhead.
- Some pods only work as static libraries (they crash or fail to link as dynamic frameworks).

**When `"dynamic"` might be right:**
- You are building an app extension that shares code with the main app. Dynamic frameworks can be shared between the app and the extension.

**Common mistake:** Switching between static and dynamic without cleaning the build folder. Run `expo prebuild --clean` after changing this.

#### buildReactNativeFromSource

```ts
ios: {
  buildReactNativeFromSource: true,
},
```

**What it does:** Compiles React Native from source code instead of using prebuilt binaries.

**When you need it:** Required for the New Architecture (Fabric renderer, TurboModules) in certain SDK versions. Allows custom C++ code integration.

**Trade-offs:**
- Significantly longer build times (5-15 minutes added)
- Full control over React Native's native code
- Required for some New Architecture features

---

## 7. OTA Updates Configuration

Over-the-air (OTA) updates let you push JavaScript bundle changes to users without going through the app store. This is one of Expo's most powerful features and one of the most misconfigured.

### Core Configuration

```ts
updates: {
  enabled: true,
  url: 'https://u.expo.dev/your-project-id',
  checkAutomatically: 'ON_LOAD',
  fallbackToCacheTimeout: 0,
},
```

### updates.enabled

- `true`: The app checks for and downloads OTA updates.
- `false`: OTA updates are completely disabled. Users only get new code via app store updates.

**When to disable:** If you do not use EAS Update and want to avoid the network request on every app launch.

### updates.url

The URL of your update server. For EAS Update, this is `https://u.expo.dev/<your-project-id>`.

### checkAutomatically -- Strategy Table

| Value | Behavior | Best For | Trade-off |
|---|---|---|---|
| `ON_LOAD` | Checks for updates every time the app starts | Most apps; ensures users get fixes quickly | Network request on every cold start; adds latency if fallbackToCacheTimeout > 0 |
| `ON_ERROR_RECOVERY` | Only checks after a crash/error recovery | Apps where startup speed is critical | Users may run outdated code for a long time |
| `WIFI_ONLY` | Checks only on Wi-Fi connections | Apps used in areas with expensive mobile data | Users on cellular never get updates automatically |
| `NEVER` | Never checks automatically; you must trigger manually | Apps that want full control (check in background, show UI) | Requires you to implement update checking logic manually |

**Manual update checking (when checkAutomatically is NEVER):**

```ts
import * as Updates from 'expo-updates';

async function checkForUpdate() {
  const update = await Updates.checkForUpdateAsync();
  if (update.isAvailable) {
    await Updates.fetchUpdateAsync();
    // Optionally ask user, then:
    await Updates.reloadAsync();
  }
}
```

### fallbackToCacheTimeout -- The Startup Speed vs Freshness Trade-off

**What it does:** How long (in milliseconds) the app waits for a new update to download before falling back to the cached (existing) bundle.

| Value | Behavior | Best For |
|---|---|---|
| `0` | Immediately loads cached bundle; downloads update in background for next launch | Fast startup; most apps |
| `5000` | Waits up to 5 seconds for a new update to download | Apps where running latest code is critical (e.g., regulatory compliance) |
| `30000` | Waits up to 30 seconds | Almost never appropriate; terrible UX |

**Decision framework:**
- **0 (recommended for most apps):** Users see the app instantly. If an update is available, it downloads in the background and applies on next launch. This means users are always one launch behind the latest update.
- **>0:** Users may see a blank/splash screen for up to N seconds while the update downloads. Only use this if running outdated code is dangerous (financial calculations, security patches).

**Common mistakes:**
- Setting a high timeout thinking it ensures users always get the latest code. It does not -- if the download takes longer than the timeout, they still get the cached version.
- Setting timeout to 0 and not understanding why a critical fix does not apply until the user's second app launch.

### Runtime Version Policies -- Comparison Table

The runtime version determines which OTA updates are compatible with which native builds. This is the most critical update configuration decision.

| Policy | Value Example | How It Works | Best For | Risk |
|---|---|---|---|---|
| `appVersion` | `"2.3.1"` | Matches the `version` field. Only updates for the same app version are applied. | Apps that rarely change native code | Forgetting to update the version when native code changes, causing incompatible updates |
| `nativeVersion` | `"2.3.1(42)"` | Combines `version` + build number. More granular than appVersion. | Apps with frequent native changes | Same risk as appVersion but less likely due to build number inclusion |
| `fingerprint` | `"sha256:abc123..."` | Hashes the entire native project. Automatically detects native changes. | Most apps (recommended) | Larger hash strings; any native file change (even comments) creates a new runtime version |
| Custom string | `"1.0.0-rc.1"` | You manually set a string. Full control. | Advanced teams with custom versioning needs | You must remember to update it whenever native code changes |

**Decision framework:**
- **Start with `fingerprint`.** It automatically detects when native code changes and prevents incompatible OTA updates. This is the safest option.
- **Use `appVersion` if:** You want simple, predictable runtime versions and your team is disciplined about versioning.
- **Use custom if:** You have a complex release process (multiple native variants, feature branches with native changes).

**Common mistakes with each policy:**

| Policy | Common Mistake |
|---|---|
| `appVersion` | Pushing an OTA update that uses a new native module but version hasn't changed. App crashes. |
| `nativeVersion` | Same as above but less common because build numbers change with native builds. |
| `fingerprint` | Being surprised that a "minor" native change (updating a Pod version) invalidates all existing OTA updates. |
| Custom | Forgetting to update the runtime version after a native change. The OTA update crashes on older native builds. |

---

## 8. Plugins Configuration Strategy

### Plugin Execution Order

Plugins are executed **sequentially, in array order**. If two plugins modify the same native file, the later one's changes take precedence.

```ts
plugins: [
  'plugin-a',  // Runs first
  'plugin-b',  // Runs second, may override plugin-a's changes
  ['plugin-c', { option: true }],  // Runs third, with options
],
```

**This matters when:**
- Two plugins modify `AndroidManifest.xml` or `Info.plist`
- A plugin removes something that a later plugin expects to exist
- You are debugging why a plugin's changes are not reflected in the native project

### Debug Tip: Inspecting Plugin Output

```bash
EXPO_DEBUG=1 npx expo prebuild --clean
```

This outputs detailed logs showing what each plugin modifies. Essential for debugging plugin conflicts.

### Notable Plugins for Production Apps

#### expo-build-properties (covered in Section 6)

The most important plugin. Every production app needs it.

#### MultiDex

**When you need it:** When your app exceeds the 65,536 method limit (the DEX limit). This is common with large apps that include Firebase, Google Maps, Facebook SDK, and other heavy libraries.

**How to tell you need it:** Your build fails with:

```
Cannot fit requested classes in a single dex file
```

Since `minSdkVersion` 21+ includes native MultiDex support, you typically do not need a separate MultiDex plugin. But if your minSdkVersion is below 21, you need it.

#### Launch Mode Configuration

For apps that use deep linking heavily, configuring the Android launch mode prevents duplicate activities:

```ts
// In a custom config plugin or via expo-build-properties
android: {
  launchMode: 'singleTask',
},
```

| Mode | Behavior | Best For |
|---|---|---|
| `standard` | New instance created for every intent | Simple apps without deep links |
| `singleTop` | Reuses existing instance if it is on top of the stack | Most apps |
| `singleTask` | Reuses existing instance, clears activities above it | Deep link-heavy apps |
| `singleInstance` | Only one instance ever, in its own task | Special cases (payment flows, OAuth callbacks) |

**Common mistake:** Using `standard` mode with deep links. Every deep link creates a new instance of your app's main activity, filling the back stack with duplicates.

#### Stripe Plugin

```ts
plugins: [
  [
    '@stripe/stripe-react-native',
    {
      merchantIdentifier: 'merchant.com.yourcompany.yourapp',
      enableGooglePay: true,
    },
  ],
],
```

**Key considerations:**
- `merchantIdentifier` must match what you configured in Apple Developer portal
- The plugin adds necessary entitlements and Swift bridging headers automatically
- Google Pay requires additional setup in the Google Pay Business Console

---

## 9. eas.json Configuration Decision Guide

The `eas.json` file configures EAS Build, EAS Submit, and EAS Update. It lives in your project root.

### CLI Configuration

```json
{
  "cli": {
    "appVersionSource": "remote",
    "promptToConfigurePushNotifications": false
  }
}
```

#### appVersionSource -- "remote" vs "local"

| Value | Behavior | Pros | Cons |
|---|---|---|---|
| `"remote"` | EAS servers track version/buildNumber. Auto-incremented in the cloud. | No merge conflicts, CI-friendly, always correct | Version numbers are invisible in your codebase; you must run `eas build:version:get` to check |
| `"local"` | Version is read from `app.json`/`app.config.ts`. You manage it manually. | Visible in git, part of code review | Merge conflicts, easy to forget incrementing, race conditions with parallel builds |

**Recommendation:** `"remote"` for teams. `"local"` for solo developers who want full visibility.

#### promptToConfigurePushNotifications

Set to `false` in CI to prevent interactive prompts. Set to `true` (or omit) for local development where you want the setup wizard.

### Build Profile Strategy

#### The Inheritance Pattern

```json
{
  "build": {
    "base": {
      "node": "20.11.0",
      "env": {
        "APP_ENV": "production"
      }
    },
    "development": {
      "extends": "base",
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "APP_ENV": "development"
      }
    },
    "preview": {
      "extends": "base",
      "distribution": "internal",
      "env": {
        "APP_ENV": "staging"
      }
    },
    "production": {
      "extends": "base",
      "autoIncrement": true
    }
  }
}
```

**Key concepts:**

**extends:** A profile can inherit from another profile. Only override what differs. This prevents configuration drift between profiles.

**developmentClient:** When `true`, the build includes the Expo Dev Client (development tools, Metro bundler connection). Never use this for user-facing builds -- it adds significant size and security surface.

**distribution:** Controls who can install the build.

| Value | Result | Use Case |
|---|---|---|
| `"internal"` | Ad-hoc / internal distribution (registered devices on iOS) | QA testing, internal dogfooding |
| `"store"` | Signed for App Store / Google Play submission | Production releases |

#### autoIncrement -- Which Profiles Need It

| Profile | autoIncrement | Why |
|---|---|---|
| development | `false` | Dev builds are disposable. Version does not matter. |
| preview/staging | `false` or `true` | `true` if QA needs to distinguish builds. `false` if builds are always overwritten. |
| production | `true` | **Always.** App stores require unique, increasing build numbers. Forgetting this causes rejected uploads. |

#### resourceClass -- When "large" Is Worth the Cost

```json
{
  "production": {
    "resourceClass": "large"
  }
}
```

| Resource Class | Build Time (typical) | Cost | When to Use |
|---|---|---|---|
| `"default"` | 15-30 min | Standard | Most builds |
| `"medium"` | 10-20 min | Higher | Medium-complexity apps |
| `"large"` | 8-15 min | Highest | Apps with heavy native dependencies, New Architecture builds |

**Decision:** Only upgrade if your build times are painful. Monitor your actual build times before spending more.

#### buildType -- "apk" vs "app-bundle"

For Android `distribution: "internal"` builds:

| buildType | Result | Use Case |
|---|---|---|
| `"apk"` | A standalone APK file | Internal testing, direct installation without Play Store |
| `"app-bundle"` (default) | AAB format | Store submission, Google Play required format |

**Common mistake:** Trying to install an AAB directly on a device. AABs are for Google Play only. Use APK for internal testing.

### iOS Provisioning

| Distribution | Provisioning | Devices | Use Case |
|---|---|---|---|
| `"internal"` | Ad-hoc (default) | Registered UDIDs only (max 100) | Small team testing |
| `"internal"` + `"universal"` enterprise | Universal | Any device in your org | Large enterprise teams |
| `"store"` | App Store | Anyone via App Store/TestFlight | Public release |

**Common mistake:** Not registering test device UDIDs for ad-hoc builds. The build succeeds but the IPA will not install on unregistered devices.

### Environment Variables Strategy

There are three ways to provide environment variables to EAS Build:

#### 1. eas.json env field

```json
{
  "build": {
    "production": {
      "env": {
        "API_URL": "https://api.yourapp.com",
        "APP_ENV": "production"
      }
    }
  }
}
```

**Safe to commit:** Only non-secret values (API URLs, feature flags, environment names).

#### 2. .env files (via expo-env plugin or custom config)

```
# .env.production
API_URL=https://api.yourapp.com
SENTRY_DSN=https://xxx@sentry.io/123
```

**Use for:** Values that differ per environment but are not secrets. Add `.env.*` to `.gitignore` if they contain anything sensitive.

#### 3. EAS Secrets (cloud-stored)

```bash
eas secret:create --name SIGNING_KEY --value "your-secret-value" --scope project
```

**Use for:** Actual secrets -- API keys, signing credentials, private tokens. These are stored encrypted on EAS servers and injected during build.

**What is safe to commit:**

| Source | Safe to Commit | Example Values |
|---|---|---|
| eas.json `env` | Yes | `API_URL`, `APP_ENV`, `FEATURE_FLAG_X` |
| .env files | Depends | Public API endpoints: yes. Secret keys: no |
| EAS Secrets | N/A (cloud-stored) | `SENTRY_AUTH_TOKEN`, `GOOGLE_SERVICES_JSON_BASE64` |

#### Accessing Env Vars in app.config.ts

```ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  extra: {
    apiUrl: process.env.API_URL ?? 'https://api.yourapp.com',
    appEnv: process.env.APP_ENV ?? 'development',
  },
});
```

These become available at runtime via:

```ts
import Constants from 'expo-constants';
const apiUrl = Constants.expoConfig?.extra?.apiUrl;
```

### Submit Configuration

```json
{
  "submit": {
    "production": {
      "android": {
        "track": "internal",
        "releaseStatus": "draft",
        "rollout": 0.1
      },
      "ios": {
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDEF1234"
      }
    }
  }
}
```

#### Android Tracks

| Track | Visibility | Use Case |
|---|---|---|
| `"internal"` | Internal testers only (up to 100) | First-pass testing, CI/CD auto-deploy |
| `"alpha"` | Closed testing group | Broader QA team |
| `"beta"` | Open testing (anyone can opt in) | Public beta programs |
| `"production"` | All users | Public release |

**Staged rollouts:**

```json
{
  "android": {
    "track": "production",
    "rollout": 0.1
  }
}
```

This releases to 10% of users. Monitor crash rates and user feedback, then gradually increase:

```bash
# Increase rollout
eas submit --platform android --profile production
# Update rollout percentage in eas.json to 0.25, 0.5, 1.0
```

**Rollout as a safety mechanism:** If your 10% rollout shows a spike in crashes (via Firebase Crashlytics, Sentry, etc.), you can halt the rollout without affecting the other 90%.

#### iOS TestFlight Configuration

For iOS, you submit to TestFlight (internal/external testing) or the App Store:

```json
{
  "submit": {
    "production": {
      "ios": {
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDEF1234"
      }
    }
  }
}
```

**Auto-submit configuration:**

```json
{
  "build": {
    "production": {
      "autoSubmit": true,
      "submitProfile": "production"
    }
  }
}
```

When `autoSubmit` is `true`, a successful build automatically triggers submission. Useful for CI/CD pipelines where you want builds to flow straight to TestFlight or Google Play Internal testing.

### Cache Configuration

```json
{
  "build": {
    "production": {
      "cache": {
        "key": "production-v1",
        "paths": [
          "android/.gradle",
          "ios/Pods"
        ]
      }
    }
  }
}
```

**When custom caching helps:**
- You have large native dependencies (Firebase, Google Maps) that are slow to download/compile
- Your builds consistently spend 5+ minutes on the same dependency resolution

**When it does not help:**
- `node_modules` are NOT cached by EAS Build. They are always freshly installed. Do not add them to cache paths.
- If your native dependencies change frequently, cached data is invalidated anyway.

**Common mistake:** Aggressively caching everything and then wondering why a dependency update is not reflected in builds. Use a versioned cache key (`production-v2`) to bust the cache when needed.

---

## 10. New Architecture Configuration

### What is the New Architecture?

React Native's New Architecture replaces the legacy bridge with:
- **Fabric:** A new rendering system that supports synchronous operations, concurrent features, and direct C++ communication
- **TurboModules:** Lazy-loaded native modules that are faster and type-safe
- **JSI (JavaScript Interface):** Direct communication between JS and native code without serialization

### Configuration

```ts
// In app.config.ts
{
  newArchEnabled: true,
}

// In expo-build-properties plugin (for iOS):
ios: {
  buildReactNativeFromSource: true,
},
```

### Current Status

- **Expo SDK 52+:** New Architecture is opt-in via `newArchEnabled: true`
- **Expo SDK 55+:** New Architecture is always on. The flag is ignored.
- **React Native 0.76+:** New Architecture is the default in bare React Native

### Benefits

- Synchronous native calls (no more async-only bridge)
- Concurrent rendering support (React 18 features)
- Type-safe native module interfaces
- Better memory management
- Faster startup (lazy module loading)

### Risks

- Some third-party libraries do not yet support the New Architecture. Check [reactnative.directory](https://reactnative.directory/) for compatibility.
- Longer build times (building React Native from source)
- Different rendering behavior may expose bugs in your layout code
- Debugging tools are still catching up

### Decision Framework

| Situation | Recommendation |
|---|---|
| New project on latest SDK | Enable it. The ecosystem is ready. |
| Existing large app | Test in a preview build first. Check all third-party libraries for compatibility. |
| Using many native libraries | Audit each library for New Architecture support before enabling. |
| SDK 55+ | It is enabled automatically. Focus on compatibility testing. |

### Legacy Architecture Sunset

React Native core team has announced that the legacy architecture (Bridge) will be removed in a future version. The timeline:
- **SDK 52-54:** Opt-in, both architectures available
- **SDK 55+:** New Architecture is always on
- **Future React Native versions:** Bridge code will be removed entirely

**Recommendation:** Start migrating now if you have not already. Test with `newArchEnabled: true` in a preview build to identify incompatible libraries.

---

## 11. The extra Field -- Runtime Configuration Bridge

### What It Does

The `extra` field in your Expo config is the bridge between **build-time configuration** and **runtime code**. Values you put here are embedded in the app's JavaScript bundle and accessible at runtime.

```ts
// app.config.ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  extra: {
    apiUrl: process.env.API_URL ?? 'https://api.yourapp.com',
    appEnv: process.env.APP_ENV ?? 'development',
    enableNewFeature: process.env.ENABLE_NEW_FEATURE === 'true',
    sentryDsn: 'https://public-key@sentry.io/123',
    analyticsId: 'UA-12345678-1',
    eas: {
      projectId: 'your-eas-project-id',
    },
  },
});
```

### Accessing at Runtime

```ts
import Constants from 'expo-constants';

const apiUrl = Constants.expoConfig?.extra?.apiUrl;
const isNewFeatureEnabled = Constants.expoConfig?.extra?.enableNewFeature;
```

### What Goes in extra vs Env Vars vs Hardcoded

| Data | Where to Put It | Why |
|---|---|---|
| API base URLs | `extra` (via env vars) | Changes per environment, needed at runtime |
| Feature flags | `extra` | Can be toggled per build profile |
| Analytics IDs | `extra` | Different IDs for dev vs production |
| Sentry DSN | `extra` | Public key, safe to embed, needed at runtime |
| EAS project ID | `extra.eas.projectId` | Required for EAS Update |
| Private API keys | EAS Secrets only | NEVER in extra |
| Database passwords | EAS Secrets only | NEVER in extra |
| OAuth client secrets | EAS Secrets only | NEVER in extra |

### Security Warning

**Everything in `extra` is embedded in your JavaScript bundle.** This means:
- Anyone can decompile your app and read these values
- They are visible in the Hermes bytecode bundle
- They are transmitted in OTA updates (in plaintext)

**Never put in extra:**
- Private API keys or secrets
- Database connection strings
- OAuth client secrets
- Encryption keys
- Anything that would be damaging if a user extracted it

**Safe to put in extra:**
- Public API URLs
- Public analytics IDs (Google Analytics, Sentry DSN with public key)
- Feature flags
- Environment identifiers

### Common Patterns

#### Environment-Aware API URLs

```ts
const API_URLS = {
  development: 'http://localhost:3000',
  staging: 'https://staging-api.yourapp.com',
  production: 'https://api.yourapp.com',
};

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  extra: {
    apiUrl: API_URLS[process.env.APP_ENV ?? 'development'],
  },
});
```

#### Feature Flags

```ts
extra: {
  features: {
    newOnboarding: process.env.APP_ENV === 'production',
    debugMenu: process.env.APP_ENV !== 'production',
    betaFeatureX: process.env.ENABLE_BETA_X === 'true',
  },
},
```

---

## 12. Deep Linking Architecture

Deep linking allows external sources (web links, push notifications, other apps, QR codes) to navigate directly to specific content in your app. There are three layers, each with different capabilities and security characteristics.

### The Three-Layer Strategy

| Layer | Example | Security | Setup Complexity | Fallback |
|---|---|---|---|---|
| Custom Scheme | `myapp://profile/123` | None (any app can register) | Low | No web fallback |
| Universal Links (iOS) | `https://yourapp.com/profile/123` | High (domain verification) | Medium | Opens in Safari |
| App Links (Android) | `https://yourapp.com/profile/123` | High (domain verification) | Medium | Opens in browser |

**Recommended approach:** Implement all three. Custom scheme for development/testing, Universal Links + App Links for production.

### Layer 1: Custom Scheme

**Configuration:**

```ts
// app.config.ts
{
  scheme: 'myapp',
}
```

**How it works:** The OS registers `myapp://` as a protocol your app handles. Any `myapp://path/here` link opens your app.

**Security concern:** On Android, any app can register the same scheme. If two apps register `myapp://`, the OS shows a disambiguation dialog. On iOS, the behavior is undefined (one app wins, unpredictably). This makes custom schemes unsuitable for security-sensitive flows (OAuth callbacks, password resets).

**Use for:**
- Development and testing
- Expo Go compatibility
- Non-sensitive navigation (opening a product page)

**Do NOT use for:**
- OAuth redirect URIs in production (use Universal Links)
- Any flow where another app intercepting the link would be harmful

### Layer 2: Universal Links (iOS)

Universal Links use your HTTPS domain to open your app. Because Apple verifies domain ownership, only YOUR app can handle links to YOUR domain.

**Step 1: App configuration**

```ts
ios: {
  associatedDomains: [
    'applinks:yourapp.com',
    'applinks:www.yourapp.com',
  ],
},
```

**Step 2: Server configuration**

Host this file at `https://yourapp.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appIDs": ["TEAMID.com.yourcompany.yourapp"],
        "components": [
          {
            "/": "/profile/*",
            "comment": "Matches profile deep links"
          },
          {
            "/": "/product/*",
            "comment": "Matches product deep links"
          }
        ]
      }
    ]
  }
}
```

**Requirements:**
- File must be served over HTTPS with `Content-Type: application/json`
- No redirects allowed (the file must be at the exact URL)
- `TEAMID` is your Apple Developer Team ID
- File must be accessible without authentication

**How Apple verifies:**
- When the app is installed, iOS downloads the AASA file from each domain in `associatedDomains`
- iOS caches this verification for approximately 24 hours (not at every app launch)
- If verification fails, links open in Safari instead of the app

**Fallback behavior:** If the app is not installed, the link opens in Safari. Your web page should handle this gracefully (show the content or prompt to install the app).

**Common mistakes:**
- AASA file has a redirect (e.g., `yourapp.com` redirects to `www.yourapp.com`). Apple does not follow redirects.
- AASA file is behind a CDN that adds caching headers that serve a stale version.
- Forgetting to update the AASA file when you change your bundle identifier or team ID.
- Testing with `cmd+click` or long-press. Universal Links only work with regular taps on links from OTHER apps (Safari, Mail, Messages). They do not work from within your own app's web views.

### Layer 3: App Links (Android)

Android App Links are the Android equivalent of Universal Links.

**Step 1: App configuration**

```ts
android: {
  intentFilters: [
    {
      action: 'VIEW',
      autoVerify: true,
      data: [
        {
          scheme: 'https',
          host: 'yourapp.com',
          pathPrefix: '/profile',
        },
        {
          scheme: 'https',
          host: 'yourapp.com',
          pathPrefix: '/product',
        },
      ],
      category: ['BROWSABLE', 'DEFAULT'],
    },
  ],
},
```

**Step 2: Server configuration**

Host this file at `https://yourapp.com/.well-known/assetlinks.json`:

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.yourcompany.yourapp",
      "sha256_cert_fingerprints": [
        "AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90"
      ]
    }
  }
]
```

**Getting the SHA-256 fingerprint:**

```bash
# For EAS builds
eas credentials --platform android
# Look for the SHA-256 fingerprint of your signing key

# For local debug builds
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android
```

**Important:** Different signing keys (debug, upload, Google Play app signing) have different fingerprints. You need ALL relevant fingerprints in your `assetlinks.json`:
- Your upload key fingerprint (for builds you submit)
- Google Play's app signing key fingerprint (Google re-signs your app)
- Debug key fingerprint (for local development)

**Verification testing:**

```bash
# Check if Android has verified your domain
adb shell pm get-app-links com.yourcompany.yourapp
```

**Common mistakes:**
- Only including the upload key fingerprint but not the Google Play app signing key. The Play Store version of your app fails to verify.
- Not including `autoVerify: true`. Without it, Android shows a disambiguation dialog instead of opening your app directly.
- `assetlinks.json` not being valid JSON. Trailing commas, missing brackets, etc.

### Deep Link Routing in Your App

Once links reach your app, you need to route them to the correct screen:

```ts
import * as Linking from 'expo-linking';
import { useEffect } from 'react';

function useDeepLinkHandler(navigation) {
  useEffect(() => {
    // Handle link that opened the app
    Linking.getInitialURL().then((url) => {
      if (url) handleDeepLink(url, navigation);
    });

    // Handle links while app is running
    const subscription = Linking.addEventListener('url', ({ url }) => {
      handleDeepLink(url, navigation);
    });

    return () => subscription?.remove();
  }, [navigation]);
}

function handleDeepLink(url: string, navigation) {
  const parsed = Linking.parse(url);
  // parsed.path = 'profile/123'
  // parsed.queryParams = { ref: 'notification' }

  if (parsed.path?.startsWith('profile/')) {
    const userId = parsed.path.split('/')[1];
    navigation.navigate('Profile', { userId });
  }
}
```

### Branch.io and Other Deep Link Services

For complex deep linking needs (deferred deep links, attribution, cross-platform consistency), consider a service like Branch.io, Adjust, or AppsFlyer.

**What they add:**
- **Deferred deep linking:** User clicks a link, installs the app, and is taken to the correct content after installation.
- **Attribution:** Track which campaign/source brought the user.
- **Cross-platform links:** A single link works on iOS, Android, and web.

**Trade-off:** Additional SDK dependency, potential latency on link resolution, vendor lock-in.

### Testing Deep Links

```bash
# iOS Simulator
xcrun simctl openurl booted "myapp://profile/123"
xcrun simctl openurl booted "https://yourapp.com/profile/123"

# Android Emulator
adb shell am start -a android.intent.action.VIEW -d "myapp://profile/123"
adb shell am start -a android.intent.action.VIEW -d "https://yourapp.com/profile/123"

# Verify Android App Links
adb shell pm get-app-links com.yourcompany.yourapp

# Verify iOS AASA file is accessible
curl -I https://yourapp.com/.well-known/apple-app-site-association
```

---

## 13. Sources

### Official Expo Documentation
- [Expo Application Config (app.json / app.config.js)](https://docs.expo.dev/versions/latest/config/app/)
- [EAS Build Configuration (eas.json)](https://docs.expo.dev/build/eas-json/)
- [EAS Submit Configuration](https://docs.expo.dev/submit/eas-json/)
- [EAS Update](https://docs.expo.dev/eas-update/introduction/)
- [Runtime Version Policies](https://docs.expo.dev/eas-update/runtime-versions/)
- [expo-build-properties](https://docs.expo.dev/versions/latest/sdk/build-properties/)
- [expo-splash-screen](https://docs.expo.dev/versions/latest/sdk/splash-screen/)
- [expo-updates](https://docs.expo.dev/versions/latest/sdk/updates/)
- [Expo Config Plugins](https://docs.expo.dev/config-plugins/introduction/)
- [Environment Variables in Expo](https://docs.expo.dev/guides/environment-variables/)
- [Deep Linking Guide](https://docs.expo.dev/guides/deep-linking/)
- [Linking API](https://docs.expo.dev/versions/latest/sdk/linking/)

### Apple Documentation
- [Apple App Site Association (AASA)](https://developer.apple.com/documentation/xcode/supporting-associated-domains)
- [Universal Links](https://developer.apple.com/documentation/uikit/inter-process_communication/allowing_apps_and_websites_to_link_to_your_content)
- [Information Property List (Info.plist)](https://developer.apple.com/documentation/bundleresources/information_property_list)
- [App Transport Security](https://developer.apple.com/documentation/bundleresources/information_property_list/nsapptransportsecurity)
- [Export Compliance](https://developer.apple.com/documentation/bundleresources/information_property_list/itsappusesnonexemptencryption)
- [UIBackgroundModes](https://developer.apple.com/documentation/bundleresources/information_property_list/uibackgroundmodes)
- [ProMotion Display Support](https://developer.apple.com/documentation/quartzcore/cadisplaylink)

### Android Documentation
- [Android App Links](https://developer.android.com/training/app-links)
- [Digital Asset Links (assetlinks.json)](https://developers.google.com/digital-asset-links/v1/getting-started)
- [Android Adaptive Icons](https://developer.android.com/develop/ui/views/launch/icon_design_adaptive)
- [Android Manifest Permissions](https://developer.android.com/reference/android/Manifest.permission)
- [R8 / ProGuard Shrinking](https://developer.android.com/build/shrink-code)
- [Google Play Target API Level Requirements](https://developer.android.com/google/play/requirements/target-sdk)
- [Android SplashScreen API (API 31+)](https://developer.android.com/develop/ui/views/launch/splash-screen)
- [Android Backup](https://developer.android.com/guide/topics/data/autobackup)

### React Native Documentation
- [React Native New Architecture](https://reactnative.dev/docs/the-new-architecture/landing-page)
- [React Native Linking](https://reactnative.dev/docs/linking)
- [React Native Directory (Library Compatibility)](https://reactnative.directory/)

### Community Resources
- [Expo Forums](https://forums.expo.dev/)
- [EAS Build Changelog](https://expo.dev/changelog)
- [Expo GitHub Issues](https://github.com/expo/expo/issues)

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Expo EAS Complete Guide](./eas-complete-guide.md) | Companion guide — EAS Build, Submit, Update pipeline that uses the configuration documented here |
| [Dependency Management](../optimization/dependency-management.md) | Library audit process — affects expo-build-properties and plugin configuration |
| [New Architecture Deep Dive](../optimization/new-architecture-migration.md) | New Architecture enablement that changes build configuration documented here |
