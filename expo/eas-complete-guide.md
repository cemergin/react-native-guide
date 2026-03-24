[← Back to Index](../README.md)

# The Definitive EAS (Expo Application Services) Guide

> A comprehensive reference for React Native / Expo developers covering EAS Build, EAS Submit, EAS Update, config plugins, crash reporting, and more. Last updated: March 2026.

---

## Table of Contents

1. [What is EAS?](#1-what-is-eas)
2. [EAS Build](#2-eas-build)
3. [EAS Submit](#3-eas-submit)
4. [EAS Update (OTA)](#4-eas-update-ota)
5. [Expo Prebuild & Config Plugins](#5-expo-prebuild--config-plugins)
6. [EAS Metadata](#6-eas-metadata)
7. [Debug Symbols & Crash Reporting](#7-debug-symbols--crash-reporting)
8. [Performance & Optimization](#8-performance--optimization)
9. [EAS Workflows](#9-eas-workflows)
10. [Pricing](#10-pricing)
11. [Sources](#11-sources)

---

## 1. What is EAS?

**Expo Application Services (EAS)** is a suite of cloud-hosted services designed for React Native and Expo applications. EAS handles the most painful parts of mobile development: building native binaries, submitting to app stores, and delivering over-the-air updates.

### The Three Pillars

| Service | Purpose | Key Benefit |
|---------|---------|-------------|
| **EAS Build** | Compile your app in the cloud into `.apk`, `.aab`, or `.ipa` files | No local Xcode/Android Studio setup required |
| **EAS Submit** | Upload binaries to Google Play Store and Apple App Store | Automate the tedious store submission process |
| **EAS Update** | Push JavaScript/asset updates over-the-air (OTA) | Ship bug fixes in minutes, not days |

### How EAS Fits into the Expo Ecosystem

```
┌─────────────────────────────────────────────────────┐
│                   Your App Code                      │
│          (TypeScript, React Native, Expo)             │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────▼────────┐
              │  expo prebuild   │  ← Continuous Native Generation
              │  (Config Plugins)│
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    ┌────▼────┐  ┌─────▼─────┐  ┌───▼────┐
    │EAS Build│  │EAS Submit │  │EAS     │
    │ (cloud) │──│ (stores)  │  │Update  │
    └─────────┘  └───────────┘  │ (OTA)  │
                                └────────┘
```

### When to Use EAS vs Local Builds

| Scenario | Recommendation |
|----------|---------------|
| You don't have a Mac but need iOS builds | **EAS Build** (only option) |
| CI/CD pipeline for your team | **EAS Build** (consistent, reproducible) |
| Quick iteration on native code locally | **Local build** (`npx expo run:ios/android`) |
| One-off debugging of native crashes | **Local build** with Xcode/Android Studio |
| You need to submit to stores from CI | **EAS Submit** |
| Hot-fixing a JS bug in production | **EAS Update** |
| Your team has limited mobile experience | **EAS** (handles credential management, signing) |

---

## 2. EAS Build

### Architecture: How Cloud Builds Work

When you run `eas build`, the following sequence occurs:

```
1. UPLOAD        Your project is packaged and uploaded to EAS servers
                 (respecting .easignore / .gitignore)

2. PREBUILD      `npx expo prebuild` runs to generate native projects
                 (android/ and ios/ directories) from your app config
                 and config plugins

3. INSTALL DEPS  Package manager installs JS dependencies (npm/yarn/pnpm)
                 Native dependencies installed (CocoaPods for iOS,
                 Gradle for Android)

4. COMPILE       Native compilation:
                 - Android: Gradle assembles APK or AAB
                 - iOS: xcodebuild archives and exports IPA

5. SIGN          Binary is signed with your credentials
                 (keystore for Android, provisioning profile for iOS)

6. ARTIFACT      Signed binary is uploaded and a download URL is provided
```

### Server Infrastructure

| Platform | Environment | Cloud Provider |
|----------|-------------|----------------|
| Android | Ubuntu Linux | Google Cloud Platform (GCP) |
| iOS | macOS (Apple Silicon) | Apple hardware (required by Apple license) |

### Resource Classes

| Resource Class | Platform | vCPUs | RAM | Use Case |
|---------------|----------|-------|-----|----------|
| `medium` | Android | 4 | 16 GB | Default; suitable for most apps |
| `large` | Android | 8 | 32 GB | Large apps, faster builds |
| `medium` | iOS | 4 | 16 GB | Default; suitable for most apps |
| `large` | iOS | 8 | 32 GB | Large apps, faster builds |

Configure in `eas.json`:

```json
{
  "build": {
    "production": {
      "resourceClass": "large"
    }
  }
}
```

### Build Images

EAS provides pre-configured build images with specific tool versions. These are updated periodically.

| Image | OS | Xcode | Node | NDK | JDK | Notes |
|-------|-----|-------|------|-----|-----|-------|
| `default` (latest) | Ubuntu 22.04 / macOS 14 | 16.x | 20.x | 27.x | 17 | Recommended for new projects |
| `latest` | Ubuntu 22.04 / macOS 14 | 16.x | 20.x | 27.x | 17 | Alias for newest image |
| SDK 52+ default | Ubuntu 22.04 / macOS 14 | 16.x | 20.x | 27.x | 17 | Matched to Expo SDK |
| SDK 51 default | Ubuntu 22.04 / macOS 14 | 15.x | 18.x | 26.x | 17 | Legacy |

Configure the image in `eas.json`:

```json
{
  "build": {
    "production": {
      "image": "latest"
    }
  }
}
```

> **Tip:** Pin your build image version in production profiles to avoid surprises when EAS updates the `default` image.

### eas.json Complete Reference

The `eas.json` file lives at your project root and configures all EAS services.

#### Top-Level Properties

| Property | Type | Description |
|----------|------|-------------|
| `cli` | object | CLI configuration |
| `cli.version` | string | Required CLI version (semver range) |
| `cli.requireCommit` | boolean | Require clean git working tree before build |
| `cli.appVersionSource` | string | `"local"` or `"remote"` - where to read app version |
| `cli.promptToConfigurePushNotifications` | boolean | Prompt for push notification setup |
| `build` | object | Build profile definitions |
| `submit` | object | Submit profile definitions |

#### Build Profile Properties (Common)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `distribution` | string | `"store"` | `"store"`, `"internal"`, or `"simulator"` |
| `developmentClient` | boolean | `false` | Build a development client (debug build) |
| `prebuildCommand` | string | - | Custom command to run instead of default prebuild |
| `resourceClass` | string | `"medium"` | `"medium"` or `"large"` |
| `image` | string | `"default"` | Build server image to use |
| `node` | string | - | Node.js version override |
| `pnpm` | string | - | pnpm version override |
| `yarn` | string | - | Yarn version override |
| `bun` | string | - | Bun version override |
| `env` | object | `{}` | Environment variables for this profile |
| `autoIncrement` | boolean/string | `false` | Auto-increment build number. `true`, `"version"`, or `"buildNumber"` |
| `cache` | object | - | Caching configuration |
| `cache.key` | string | - | Cache key for custom caching |
| `cache.paths` | string[] | - | Additional paths to cache |
| `cache.disabled` | boolean | `false` | Disable caching entirely |
| `buildCacheProvider` | string | - | `"eas"` or `"none"` (SDK 53+) |
| `config` | string | - | Path to custom app config file |
| `channel` | string | - | EAS Update channel to embed |
| `appVersionSource` | string | - | Override `cli.appVersionSource` per profile |
| `extends` | string | - | Inherit from another build profile |
| `credentialsSource` | string | `"remote"` | `"remote"` or `"local"` |
| `releaseChannel` | string | - | (Legacy) Classic updates release channel |

#### Android-Specific Build Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `buildType` | string | `"app-bundle"` | `"app-bundle"` (`.aab`) or `"apk"` |
| `gradleCommand` | string | - | Custom Gradle command (e.g., `:app:assembleRelease`) |
| `applicationArchivePath` | string | - | Glob pattern for build output |
| `withoutCredentials` | boolean | `false` | Build without signing credentials |
| `ndk` | string | - | NDK version override |

#### iOS-Specific Build Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `simulator` | boolean | `false` | Build for iOS Simulator |
| `scheme` | string | - | Xcode scheme to build |
| `buildConfiguration` | string | - | Xcode build configuration (`Debug`/`Release`) |
| `applicationArchivePath` | string | - | Glob pattern for build output |
| `autoIncrement` | boolean/string | `false` | Auto-increment `buildNumber` |
| `image` | string | `"default"` | macOS image version |
| `bundler` | string | - | Ruby Bundler version |
| `cocoapods` | string | - | CocoaPods version override |
| `appStoreConnectApiKeyPath` | string | - | Path to App Store Connect API key |

### Build Profiles Explained

Build profiles let you define different configurations for different use cases. Three conventional profiles are `development`, `preview`, and `production`.

```json
{
  "cli": {
    "version": ">= 14.0.0",
    "appVersionSource": "remote"
  },
  "build": {
    "base": {
      "node": "20.18.0",
      "env": {
        "APP_ENV": "production"
      }
    },
    "development": {
      "extends": "base",
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      },
      "env": {
        "APP_ENV": "development"
      },
      "channel": "development"
    },
    "preview": {
      "extends": "base",
      "distribution": "internal",
      "autoIncrement": true,
      "channel": "preview",
      "env": {
        "APP_ENV": "staging"
      }
    },
    "production": {
      "extends": "base",
      "distribution": "store",
      "autoIncrement": true,
      "channel": "production",
      "resourceClass": "large",
      "env": {
        "APP_ENV": "production"
      }
    }
  }
}
```

**Profile purposes:**

| Profile | Distribution | Signing | Use Case |
|---------|-------------|---------|----------|
| `development` | `internal` | Development cert | Day-to-day development with dev client |
| `preview` | `internal` | Ad Hoc / Enterprise | QA testing, stakeholder review |
| `production` | `store` | Distribution cert | App Store / Play Store releases |

Run a specific profile:

```bash
eas build --profile preview --platform ios
eas build --profile production --platform all
```

### Credential Management

#### Android: Keystore

EAS can auto-generate and manage your Android keystore, or you can provide your own.

**Auto-generated (recommended for new apps):**
```bash
eas build --platform android
# EAS generates a keystore and stores it securely
```

**Using a local keystore:**

Create a `credentials.json` at your project root:

```json
{
  "android": {
    "keystore": {
      "keystorePath": "./release.keystore",
      "keystorePassword": "your-keystore-password",
      "keyAlias": "your-key-alias",
      "keyPassword": "your-key-password"
    }
  }
}
```

Then set `credentialsSource` to `"local"` in your build profile.

**Managing credentials:**
```bash
# View current credentials
eas credentials

# Download your keystore from EAS servers
eas credentials --platform android
# Select "Download Keystore"
```

> **Critical:** Back up your production keystore. If you lose it, you cannot update your app on the Play Store. Use `eas credentials` to download a backup.

#### iOS: Provisioning Profiles & Certificates

**Auto-managed (recommended):**

EAS handles the full Apple provisioning flow automatically:
- Creates/reuses Distribution Certificates
- Creates/manages Provisioning Profiles
- Registers devices for ad hoc distribution

```bash
eas build --platform ios
# Follow prompts for Apple ID login
```

**Manual management:**

Create a `credentials.json`:

```json
{
  "ios": {
    "distributionCertificate": {
      "path": "./certs/dist_cert.p12",
      "password": "cert-password"
    },
    "provisioningProfilePath": "./certs/profile.mobileprovision"
  }
}
```

**App Store Connect API Key (recommended for CI):**

Instead of Apple ID + password + 2FA, use an API key:

```json
{
  "ios": {
    "appStoreConnectApiKeyPath": "./keys/AuthKey_XXXXXX.p8",
    "appStoreConnectApiKeyId": "XXXXXX",
    "appStoreConnectApiKeyIssuerId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
}
```

### Environment Variables & Secrets

EAS supports three visibility levels for environment variables:

| Visibility | Available During | Accessible In | Set Via |
|-----------|-----------------|---------------|---------|
| **Plain text** (`env` in eas.json) | Build time | Build logs (visible) | `eas.json` |
| **Sensitive** (EAS Secrets) | Build time | Build logs (hidden) | `eas secret:create` |
| **Built-in** | Always | Build process | Automatic |

**Managing secrets:**

```bash
# Create a secret
eas secret:create --scope project --name API_KEY --value "sk-abc123" --type string

# Create a file secret (e.g., google-services.json)
eas secret:create --scope project --name GOOGLE_SERVICES_JSON --value ./google-services.json --type file

# List secrets
eas secret:list

# Delete a secret
eas secret:delete --name API_KEY
```

**Scopes:**
- `--scope project` - Available to all builds of this project
- `--scope account` - Available to all projects in your EAS account

**Using secrets in your app:**

```javascript
// In app.config.js
export default {
  extra: {
    apiUrl: process.env.API_URL ?? 'https://api.example.com',
  },
};
```

**Built-in environment variables:**

| Variable | Description |
|----------|-------------|
| `EAS_BUILD` | `"true"` when running on EAS Build |
| `EAS_BUILD_PROFILE` | Current build profile name |
| `EAS_BUILD_GIT_COMMIT_HASH` | Git commit SHA |
| `EAS_BUILD_ID` | Unique build identifier |
| `EAS_BUILD_NPM_CACHE_URL` | URL for npm cache |
| `EAS_BUILD_PLATFORM` | `"android"` or `"ios"` |
| `EAS_BUILD_RUNNER` | `"eas-build"` on EAS servers |
| `EAS_BUILD_USERNAME` | EAS account username |

**Common pitfalls:**
- Secrets set via `eas secret:create` are NOT available in `app.config.js` during local development. Use `.env` files locally with a library like `dotenv`.
- File-type secrets are written to a temporary path during build. Access the path via the environment variable.
- Never commit `credentials.json` or `.p8` key files to version control. Add them to `.gitignore`.

### Caching

#### Built-in Caching

EAS Build automatically caches:
- **Maven** dependencies (Android, `~/.gradle` and local Maven repos)
- **CocoaPods** specs and pods (iOS, `~/Library/Caches/CocoaPods` and `Pods/`)
- **npm/yarn/pnpm** packages

#### Custom Cache Configuration

Add extra paths to the cache:

```json
{
  "build": {
    "production": {
      "cache": {
        "key": "my-custom-cache-v1",
        "paths": [
          "node_modules/.cache",
          "~/.gradle/caches"
        ]
      }
    }
  }
}
```

Change the `key` to invalidate the cache when needed.

#### Build Cache Providers (SDK 53+)

Build cache providers cache the entire derived build artifacts, dramatically speeding up subsequent builds when native code hasn't changed.

```json
{
  "build": {
    "production": {
      "buildCacheProvider": "eas"
    }
  }
}
```

When enabled, if your native fingerprint hasn't changed since the last build, EAS can reuse the previous build artifact instead of recompiling from scratch. This can reduce build times from 15-30 minutes to under 2 minutes.

### Monorepo Setup

EAS Build supports monorepo projects (Yarn Workspaces, npm workspaces, pnpm workspaces, Turborepo, Nx).

**Key configuration steps:**

1. **Run `eas build` from the app directory**, not the monorepo root:

```bash
cd apps/mobile
eas build --platform ios
```

2. **Configure `.easignore`** at the monorepo root to exclude unnecessary packages:

```
# .easignore
apps/web
apps/admin
packages/backend
*.test.ts
__tests__
```

If `.easignore` exists, it replaces `.gitignore` for determining which files to upload. Your `.easignore` must include everything from `.gitignore` that you want excluded, plus additional monorepo-specific exclusions.

3. **Set `EAS_NO_VCS=1`** if you want to build without git (useful in some CI setups):

```bash
EAS_NO_VCS=1 eas build --platform ios
```

4. **Package manager considerations:**

| Package Manager | Monorepo Support | Notes |
|----------------|-----------------|-------|
| Yarn (Classic) | Workspaces via `workspaces` in root `package.json` | Most mature support |
| Yarn (Berry/v3+) | Workspaces | May need `.yarnrc.yml` nodeLinker config |
| pnpm | `pnpm-workspace.yaml` | Set `node-linker=hoisted` in `.npmrc` for React Native compatibility |
| npm | `workspaces` in root `package.json` | Works with npm 7+ |

### CI/CD Integration

#### GitHub Actions with `expo-github-action`

```yaml
# .github/workflows/eas-build.yml
name: EAS Build
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - run: yarn install

      # Fire-and-forget: starts the build and exits immediately
      - run: eas build --platform all --profile production --non-interactive

      # OR wait for the build to complete (blocks until done):
      - run: eas build --platform all --profile production --non-interactive --wait
```

**Fire-and-forget vs `--wait`:**

| Mode | Command | CI Minutes Used | Use Case |
|------|---------|----------------|----------|
| Fire-and-forget | `eas build --non-interactive` | ~1 min | You don't need the artifact in CI |
| Wait | `eas build --non-interactive --wait` | 15-40 min | You need to chain submit or run tests on the artifact |

**Setting up `EXPO_TOKEN`:**
1. Go to [expo.dev](https://expo.dev) > Account Settings > Access Tokens
2. Create a Robot token (recommended) or Personal token
3. Add it as a GitHub Actions secret named `EXPO_TOKEN`

### Build Lifecycle Hooks

EAS Build provides npm-script hooks that run at specific points during the build. Add these to your `package.json`:

```json
{
  "scripts": {
    "eas-build-pre-install": "echo 'Runs before npm install'",
    "eas-build-post-install": "echo 'Runs after npm install'",
    "eas-build-pre-upload-artifacts": "echo 'Runs after build, before upload'"
  }
}
```

| Hook | When It Runs | Common Use Cases |
|------|-------------|-----------------|
| `eas-build-pre-install` | Before `npm install` / `yarn install` | Install system dependencies, set up tools |
| `eas-build-post-install` | After dependencies are installed, before native build | Patch packages, generate files, run codegen |
| `eas-build-pre-upload-artifacts` | After native build, before artifact upload | Copy dSYMs, run post-build scripts |

**Example: Installing a system dependency on the build server:**

```json
{
  "scripts": {
    "eas-build-pre-install": "apt-get update && apt-get install -y cmake"
  }
}
```

> **Note:** Hooks run with the same user permissions as the build process. On iOS builders (macOS), you may need `sudo` for system-level installations.

### Custom Build Images

While you cannot provide a fully custom Docker image, you can:

1. **Override tool versions** in `eas.json`:
```json
{
  "build": {
    "production": {
      "node": "20.18.0",
      "yarn": "1.22.22",
      "ios": {
        "cocoapods": "1.15.2",
        "image": "macos-sonoma-14.5-xcode-16.0"
      },
      "android": {
        "ndk": "27.1.12297006",
        "image": "ubuntu-22.04-jdk-17-ndk-27"
      }
    }
  }
}
```

2. **Use npm hooks** to install additional system dependencies:
```json
{
  "scripts": {
    "eas-build-pre-install": "./scripts/install-build-deps.sh"
  }
}
```

**Limitations:**
- You cannot change the base OS
- You cannot pre-install large native toolchains
- iOS builds must run on macOS (Apple requirement)
- Xcode version is tied to the macOS image version

---

## 3. EAS Submit

EAS Submit automates uploading your built binaries to the Google Play Store and Apple App Store.

### Google Play Store

#### Prerequisites

1. **Create a Google Cloud Service Account** with the "Service Account User" role
2. **Grant the service account access** to your Google Play Console (Settings > API Access)
3. **Download the JSON key file**

#### Configuration

```json
{
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./keys/play-store-service-account.json",
        "track": "internal",
        "releaseStatus": "completed",
        "changesNotSentForReview": false
      }
    }
  }
}
```

#### Android Submit Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `serviceAccountKeyPath` | string | - | Path to Google Play service account JSON key |
| `track` | string | `"internal"` | `"internal"`, `"alpha"`, `"beta"`, `"production"` |
| `releaseStatus` | string | `"completed"` | `"completed"`, `"draft"`, `"halted"`, `"inProgress"` |
| `rollout` | number | - | Percentage for staged rollout (0-1). Required when `releaseStatus` is `"inProgress"` |
| `changesNotSentForReview` | boolean | `false` | Set `true` if changes don't need review |
| `applicationId` | string | - | Override application ID |

#### Tracks Explained

| Track | Visibility | Use Case |
|-------|-----------|----------|
| `internal` | Up to 100 internal testers | Quick team testing |
| `alpha` | Closed testing group | Limited external testing |
| `beta` | Open testing (anyone can opt in) | Public beta |
| `production` | All users | Public release |

#### Staged Rollouts

```json
{
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./keys/play-store-service-account.json",
        "track": "production",
        "releaseStatus": "inProgress",
        "rollout": 0.1
      }
    }
  }
}
```

This rolls out to 10% of production users. Increase the rollout via Play Console or subsequent submissions.

### Apple App Store

#### Authentication Options

**Option 1: Apple ID (interactive, not recommended for CI)**
```bash
eas submit --platform ios
# Prompts for Apple ID and password
```

**Option 2: App Store Connect API Key (recommended)**

```json
{
  "submit": {
    "production": {
      "ios": {
        "ascApiKeyPath": "./keys/AuthKey_XXXXXX.p8",
        "ascApiKeyIssuerId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "ascApiKeyId": "XXXXXX",
        "appleTeamId": "XXXXXXXXXX"
      }
    }
  }
}
```

#### iOS Submit Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `ascApiKeyPath` | string | - | Path to App Store Connect `.p8` API key |
| `ascApiKeyIssuerId` | string | - | Issuer ID from App Store Connect |
| `ascApiKeyId` | string | - | Key ID from App Store Connect |
| `appleId` | string | - | Your Apple ID email (if not using API key) |
| `appleTeamId` | string | - | Apple Developer Team ID |
| `appName` | string | - | App name (for first submission) |
| `bundleIdentifier` | string | - | iOS bundle identifier override |
| `metadataPath` | string | - | Path to store metadata directory |
| `language` | string | `"en-US"` | Primary language |

#### TestFlight Configuration

Builds submitted to App Store Connect automatically appear in TestFlight. To assign to specific test groups:

```bash
# Submit and specify TestFlight group
eas submit --platform ios --latest
```

Then manage TestFlight groups in App Store Connect. EAS Submit uploads the binary; group assignment is managed in the App Store Connect UI or via the `metadataPath` configuration.

### Auto-Submit After Build

Chain build and submit in a single command:

```bash
# Using CLI flag
eas build --platform ios --profile production --auto-submit

# Or configure in eas.json
```

```json
{
  "build": {
    "production": {
      "autoSubmit": true,
      "channel": "production"
    }
  },
  "submit": {
    "production": {
      "ios": {
        "ascApiKeyPath": "./keys/AuthKey_XXXXXX.p8",
        "ascApiKeyIssuerId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "ascApiKeyId": "XXXXXX"
      },
      "android": {
        "serviceAccountKeyPath": "./keys/play-store-service-account.json",
        "track": "internal"
      }
    }
  }
}
```

### Common Pitfalls

1. **First Android submission must be done manually.** Google Play requires you to upload the very first APK/AAB through the Play Console UI. After that, EAS Submit can handle subsequent uploads.

2. **App Store Connect app record must exist.** Create the app record in App Store Connect before your first EAS Submit. Alternatively, EAS Submit can create it if you provide `appName` and `bundleIdentifier`.

3. **Service account permissions.** The Google Play service account needs "Release manager" permissions at minimum. Missing permissions produce cryptic 403 errors.

4. **API key expiry.** App Store Connect API keys don't expire, but they can be revoked. Store them securely and keep track of which key is in use.

---

## 4. EAS Update (OTA)

### Architecture: Two-Layer Model

Every Expo app has two layers:

```
┌─────────────────────────────────────┐
│         UPDATE LAYER (JS)           │  ← Can be updated OTA
│  JavaScript bundle + assets         │
│  (your React components, logic,     │
│   images, fonts)                    │
├─────────────────────────────────────┤
│         NATIVE LAYER                │  ← Requires new build
│  Native modules, config plugins,    │
│  android/ and ios/ directories,     │
│  native dependencies                │
└─────────────────────────────────────┘
```

**What CAN be updated via OTA:**
- JavaScript/TypeScript code changes
- New React components and screens
- Style changes
- Image and font asset additions/changes
- Bug fixes in JS logic
- Updated API endpoints or feature flags

**What CANNOT be updated via OTA:**
- Adding/removing/upgrading native modules
- Changes to `app.json` / `app.config.js` that affect native config
- New permissions (camera, location, etc.)
- Changes to native code (Java/Kotlin/Swift/Objective-C)
- Expo SDK version upgrades
- Config plugin changes

### How Updates Are Fetched

```
1. APP LAUNCH     App starts and loads the embedded (built-in) bundle

2. MANIFEST       App requests the update manifest from EAS Update CDN
   CHECK          (checks channel + runtime version)

3. COMPARE        If a newer update exists:
                  - Download new JS bundle + changed assets
                  - Store locally on device

4. APPLY          Next app launch loads the downloaded update
                  (or immediately, depending on configuration)
```

Update behavior is controlled by the `expo-updates` configuration:

```json
// app.json
{
  "expo": {
    "updates": {
      "url": "https://u.expo.dev/your-project-id",
      "checkAutomatically": "ON_LAUNCH",
      "fallbackToCacheTimeout": 0
    }
  }
}
```

| `checkAutomatically` | Behavior |
|---------------------|----------|
| `"ON_LAUNCH"` | Check for updates every time the app launches |
| `"ON_ERROR_RECOVERY"` | Only check after a crash recovery |
| `"NEVER"` | Only check when you call `Updates.checkForUpdateAsync()` manually |

### Channels, Branches, and Runtime Versions

This is the most important conceptual model to understand for EAS Update.

```
CHANNEL ──── maps to ──── BRANCH ──── contains ──── UPDATES
(embedded       (configurable      (ordered list of
 in build)       mapping)           published bundles)
```

**Channel:** A string embedded in the native build at compile time. Think of it as a "slot" that a build listens to. Set via `eas.json`:

```json
{
  "build": {
    "production": {
      "channel": "production"
    },
    "preview": {
      "channel": "preview"
    }
  }
}
```

**Branch:** A named stream of updates (like a git branch). By default, a branch with the same name as the channel is created automatically.

**Runtime Version:** A version identifier that ensures native compatibility. A build will only accept updates with a matching runtime version.

```
Build (channel="production", runtimeVersion="1.0.0")
  └── Receives updates from branch "production"
        where runtimeVersion="1.0.0"

Build (channel="production", runtimeVersion="2.0.0")
  └── Receives updates from branch "production"
        where runtimeVersion="2.0.0"
```

**Channel-to-branch mapping** can be customized:

```bash
# Point the "production" channel to a specific branch
eas channel:edit production --branch release-v1.5

# Roll back by pointing to a previous branch
eas channel:edit production --branch release-v1.4
```

### Runtime Version Policies

| Policy | Value | How It's Computed | Best For |
|--------|-------|-------------------|----------|
| `appVersion` | `"appVersion"` | Uses `version` from app config (e.g., `"1.2.3"`) | Simple version-based tracking |
| `nativeVersion` | `"nativeVersion"` | Uses platform-specific build numbers (`versionCode`/`buildNumber`) | Teams that bump build numbers on native changes |
| `fingerprint` | `"fingerprint"` | Hash of all native-impacting configuration | **Recommended.** Automatic, precise |
| Custom | `{ "policy": "custom" }` | You provide the value | Full control |

Configure in `app.json`:

```json
{
  "expo": {
    "runtimeVersion": {
      "policy": "fingerprint"
    }
  }
}
```

**The `fingerprint` policy** (recommended) automatically computes a hash of your native configuration including:
- Expo SDK version
- Native module versions
- Config plugin outputs
- `ios.bundleIdentifier` and `android.package`
- Native dependencies in `package.json`

If any of these change, the fingerprint changes, and old builds won't receive the new update (they'll need a new native build).

### Publishing Updates

```bash
# Publish to a specific branch
eas update --branch production --message "Fix checkout button crash"

# Publish to the channel mapped to a build profile
eas update --channel production --message "Fix checkout button crash"

# Publish for a specific platform only
eas update --branch production --platform ios --message "iOS-only fix"
```

### EAS Update vs CodePush Comparison

| Feature | EAS Update | CodePush (App Center) |
|---------|-----------|----------------------|
| **Status** | Actively developed | Deprecated (March 2025) |
| **Expo integration** | Native, first-party | Third-party, manual setup |
| **Runtime version safety** | Built-in fingerprint policy | Manual version targeting |
| **Rollback** | Channel-to-branch remapping + automatic crash recovery | Manual rollback |
| **Rollouts** | Gradual percentage rollouts | Percentage rollouts |
| **CDN** | Global CDN (Cloudflare) | Azure CDN |
| **Cost** | Included in EAS plans | Was free (now sunsetting) |
| **Bundle format** | Expo-specific | CodePush-specific |
| **Hermes support** | Full support | Required extra config |
| **Bare workflow** | Supported | Supported |
| **Web support** | N/A (web deploys differently) | N/A |

### Rollback Strategies

#### 1. Publish a New Fix Update
The simplest approach: fix the bug and publish a new update.
```bash
# Fix the code, then:
eas update --branch production --message "Fix regression from previous update"
```

#### 2. Remap Channel to a Previous Branch
Instantly revert all users to a previous known-good state:
```bash
# Point production channel to a previous branch
eas channel:edit production --branch release-v1.4-hotfix
```

#### 3. Roll Back to the Embedded Bundle
Force the app to use the bundle that was compiled into the native build:
```bash
# Publish a special "rollback" directive
eas update:rollback --channel production
```

#### 4. Automatic Crash Recovery
Built into `expo-updates`: if the app crashes immediately after applying an update, it automatically rolls back to the previous working update on the next launch. This is a safety net, not a strategy to rely on.

### Rollouts (Gradual Deployment)

Deploy updates to a percentage of users before going to 100%:

```bash
# Roll out to 10% of users
eas update --branch production --message "New feature" --rollout-percentage 10

# Increase to 50%
eas update:rollout --branch production --percentage 50

# Complete the rollout to 100%
eas update:rollout --branch production --end
```

### Bundle Diffing (SDK 55+)

Starting with SDK 55, EAS Update supports bundle diffing, which computes the difference between the currently installed update and the new one. Instead of downloading the entire JS bundle, the app only downloads the changed bytes.

**Impact:** ~75% smaller download sizes for typical updates, leading to faster update application and lower bandwidth usage.

This is enabled automatically for apps on SDK 55+ using `expo-updates` v0.26+. No configuration required.

### What CAN vs CANNOT Be Updated via OTA

| Can Update (JS Layer) | Cannot Update (Native Layer) |
|----------------------|----------------------------|
| React components, screens, navigation | Adding/removing native modules |
| Business logic, API calls | Native code (Swift/Kotlin/Java/ObjC) |
| Styles, layouts, animations | New native permissions |
| Image and font assets | `app.json` changes affecting native config |
| Feature flags, A/B test config | Expo SDK version changes |
| Localization strings | Config plugin changes |
| Bug fixes in JS code | Splash screen configuration |

---

## 5. Expo Prebuild & Config Plugins

### Continuous Native Generation (CNG)

CNG is the philosophy behind modern Expo development: **your native `android/` and `ios/` directories are generated from your app configuration, not maintained by hand.**

Instead of manually editing `AndroidManifest.xml`, `Info.plist`, `build.gradle`, and Xcode project files, you declare your configuration in `app.json`/`app.config.js` and use config plugins to modify native files programmatically.

**Benefits:**
- Upgrade Expo SDK without manual native file migration
- Share native configuration as installable plugins
- Reproducible native projects from a single source of truth
- Easier onboarding (new developers don't need to understand native project structure)

### How `npx expo prebuild` Works

```bash
npx expo prebuild
```

Step-by-step:

1. **Read app config** — Parses `app.json` or `app.config.js`
2. **Determine template** — Selects the native project template matching your Expo SDK version
3. **Generate native projects** — Creates `android/` and `ios/` directories from the template
4. **Apply built-in config** — Sets app name, bundle ID, version, icons, splash screen, etc.
5. **Run config plugins** — Executes all plugins listed in the `plugins` array (in order)
6. **Write files** — Finalizes native project files to disk

```bash
# Prebuild for a specific platform
npx expo prebuild --platform ios

# Clean prebuild (delete existing native dirs first)
npx expo prebuild --clean

# Skip dependency installation
npx expo prebuild --no-install
```

> **When does prebuild run?** EAS Build runs prebuild automatically as part of the cloud build process. You only need to run it locally if you want to inspect/debug the generated native project.

### Config Plugin System

Config plugins are functions that modify the generated native project. Many Expo and third-party packages ship with their own config plugins.

#### Using Config Plugins

```json
// app.json
{
  "expo": {
    "plugins": [
      "expo-camera",
      "expo-location",
      ["expo-image-picker", { "photosPermission": "Allow access to select photos" }],
      "./my-custom-plugin"
    ]
  }
}
```

Plugins can be:
- A package name (string) — if the package exports a config plugin
- A tuple `[packageName, options]` — to pass configuration options
- A local file path — for custom plugins

#### Available Mods (Modification Targets)

Config plugins use a "mod" system to target specific native files:

| Mod | Platform | File Modified |
|-----|----------|--------------|
| `withAndroidManifest` | Android | `AndroidManifest.xml` |
| `withMainActivity` | Android | `MainActivity.java/kt` |
| `withMainApplication` | Android | `MainApplication.java/kt` |
| `withProjectBuildGradle` | Android | Project-level `build.gradle` |
| `withAppBuildGradle` | Android | App-level `build.gradle` |
| `withSettingsGradle` | Android | `settings.gradle` |
| `withStringsXml` | Android | `strings.xml` |
| `withAndroidColors` | Android | `colors.xml` |
| `withAndroidStyles` | Android | `styles.xml` |
| `withInfoPlist` | iOS | `Info.plist` |
| `withEntitlementsPlist` | iOS | `*.entitlements` |
| `withXcodeProject` | iOS | Xcode `project.pbxproj` |
| `withPodfile` | iOS | `Podfile` |
| `withPodfileProperties` | iOS | `Podfile.properties.json` |
| `withDangerousMod` | Both | Direct file system access (escape hatch) |

### Creating a Custom Config Plugin

Here's a complete example of a config plugin that adds a custom intent filter to Android and a URL scheme to iOS:

```javascript
// plugins/withDeepLinks.js
const {
  withAndroidManifest,
  withInfoPlist,
} = require('@expo/config-plugins');

function withDeepLinks(config, { scheme = 'myapp', host = 'example.com' } = {}) {
  // Android: Add intent filter
  config = withAndroidManifest(config, (config) => {
    const mainActivity =
      config.modResults.manifest.application[0].activity.find(
        (activity) => activity.$['android:name'] === '.MainActivity'
      );

    if (mainActivity) {
      const intentFilter = {
        action: [{ $: { 'android:name': 'android.intent.action.VIEW' } }],
        category: [
          { $: { 'android:name': 'android.intent.category.DEFAULT' } },
          { $: { 'android:name': 'android.intent.category.BROWSABLE' } },
        ],
        data: [
          {
            $: {
              'android:scheme': 'https',
              'android:host': host,
            },
          },
        ],
      };

      mainActivity['intent-filter'] = [
        ...(mainActivity['intent-filter'] || []),
        intentFilter,
      ];
    }

    return config;
  });

  // iOS: Add URL scheme
  config = withInfoPlist(config, (config) => {
    const existingSchemes = config.modResults.CFBundleURLTypes || [];
    existingSchemes.push({
      CFBundleURLSchemes: [scheme],
    });
    config.modResults.CFBundleURLTypes = existingSchemes;
    return config;
  });

  return config;
}

module.exports = withDeepLinks;
```

Use it in `app.json`:

```json
{
  "expo": {
    "plugins": [
      ["./plugins/withDeepLinks", { "scheme": "myapp", "host": "example.com" }]
    ]
  }
}
```

### Plugin Execution Order

Plugins execute in the order they appear in the `plugins` array. This matters when multiple plugins modify the same native file.

```json
{
  "plugins": [
    "plugin-a",   // Runs first
    "plugin-b",   // Runs second (sees changes from plugin-a)
    "plugin-c"    // Runs third (sees changes from plugin-a and plugin-b)
  ]
}
```

> **Important:** Some plugins may conflict if they modify the same configuration. For example, two plugins that both set `UIBackgroundModes` in `Info.plist` might overwrite each other. The last plugin to write wins.

### Managed vs Bare Workflow in 2025-2026

The traditional distinction between "managed" and "bare" workflows has largely dissolved:

| Era | Managed Workflow | Bare Workflow |
|-----|-----------------|---------------|
| **2020-2022** | No `android/`/`ios/` dirs, limited native access | Full native project, manual maintenance |
| **2023-2024** | CNG + config plugins, prebuild generates native dirs | Same, but you maintain native dirs yourself |
| **2025-2026** | **The default.** CNG handles everything. | Only for apps that truly need manual native customization |

**Today's recommendation:**
- Use CNG (no committed `android/`/`ios/` directories) for all new projects
- Use config plugins for native customization
- Only eject to a bare workflow if you have a native module that cannot be expressed as a config plugin
- Even "bare" projects can use EAS Build, EAS Submit, and EAS Update

---

## 6. EAS Metadata

EAS Metadata (beta) lets you manage app store listings (descriptions, screenshots, categories) as code.

### store.config.json Structure

Create a `store.config.json` at your project root:

```json
{
  "$schema": "https://raw.githubusercontent.com/expo/eas-cli/main/packages/eas-cli/schema/metadata-0.json",
  "configVersion": 0,
  "apple": {
    "info": {
      "en-US": {
        "title": "My Awesome App",
        "subtitle": "Do awesome things",
        "description": "A longer description of what the app does...",
        "keywords": ["awesome", "productivity", "tools"],
        "marketingUrl": "https://example.com",
        "supportUrl": "https://example.com/support",
        "privacyPolicyUrl": "https://example.com/privacy",
        "privacyChoicesUrl": "https://example.com/privacy-choices"
      }
    },
    "categories": ["PRODUCTIVITY", "UTILITIES"],
    "copyright": "2026 Your Company",
    "age_rating": {
      "gambling": false,
      "unrestricted_web_access": false,
      "alcohol_tobacco_drugs": "NONE",
      "medical_treatment": "NONE",
      "sexual_content": "NONE",
      "graphic_violence": "NONE",
      "profanity": "NONE",
      "horror_fear": "NONE",
      "mature_suggestive": "NONE",
      "cartoon_fantasy_violence": "NONE",
      "realistic_violence": "NONE"
    }
  },
  "google": {
    "info": {
      "en-US": {
        "title": "My Awesome App",
        "shortDescription": "Do awesome things",
        "fullDescription": "A longer description of what the app does...",
        "video": "https://youtube.com/watch?v=abc123"
      }
    }
  }
}
```

### Dynamic Configuration with store.config.js

For dynamic values, use a JavaScript configuration file:

```javascript
// store.config.js
const version = require('./package.json').version;

module.exports = {
  configVersion: 0,
  apple: {
    info: {
      'en-US': {
        title: 'My App',
        description: `Version ${version} brings new features...`,
        releaseNotes: `What's new in ${version}:\n- Bug fixes\n- Performance improvements`,
      },
    },
  },
};
```

### Localization Support

Add entries for each locale you support:

```json
{
  "apple": {
    "info": {
      "en-US": {
        "title": "My App",
        "description": "English description"
      },
      "es-ES": {
        "title": "Mi Aplicacion",
        "description": "Descripcion en espanol"
      },
      "ja": {
        "title": "My App (Japanese title)",
        "description": "Japanese description"
      }
    }
  }
}
```

**Sync metadata to stores:**

```bash
# Push local metadata to App Store Connect / Google Play
eas metadata:push

# Pull current store metadata to local config
eas metadata:pull
```

---

## 7. Debug Symbols & Crash Reporting

Properly uploading debug symbols is critical for readable crash reports. Without them, crash stacks show memory addresses instead of function names.

### iOS dSYM Files with EAS Build

dSYM (Debug Symbol) files are generated during iOS builds. EAS Build can automatically upload them to your crash reporting service.

**Finding dSYMs:**
```bash
# List recent builds and their artifacts
eas build:list --platform ios

# Download build artifacts (includes dSYMs)
eas build:view <build-id>
```

dSYMs are included in the build artifact. Most crash reporting integrations (Sentry, Datadog, Crashlytics) have config plugins that handle dSYM upload automatically during the build process.

### Android ProGuard Mapping Files

When ProGuard/R8 code shrinking is enabled (default for release builds), Android produces mapping files that translate obfuscated stack traces.

These are generated at `android/app/build/outputs/mapping/release/mapping.txt` during the build.

### Sentry Integration

Sentry is the most common crash reporting service for Expo apps.

**Setup:**

```bash
npx expo install @sentry/react-native expo-sentry
```

```json
// app.json
{
  "expo": {
    "plugins": [
      [
        "@sentry/react-native/expo",
        {
          "organization": "your-org",
          "project": "your-project"
        }
      ]
    ]
  }
}
```

Set the auth token as an EAS Secret:

```bash
eas secret:create --scope project --name SENTRY_AUTH_TOKEN --value "sntrys_..."
```

**Initialization:**

```typescript
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'https://your-dsn@sentry.io/your-project-id',
  tracesSampleRate: 0.2,
  enableAutoSessionTracking: true,
});
```

The `@sentry/react-native/expo` config plugin automatically:
- Uploads dSYMs during iOS builds
- Uploads ProGuard mappings during Android builds
- Uploads source maps for JavaScript stack traces

### Datadog Integration

```bash
npx expo install expo-datadog @datadog/mobile-react-native
```

```json
// app.json
{
  "expo": {
    "plugins": [
      [
        "expo-datadog",
        {
          "iosDsyms": {
            "datadogGradlePluginVersion": "1.14.0"
          },
          "errorTracking": {
            "iosDsyms": {
              "enabled": true
            },
            "androidProguard": {
              "enabled": true
            },
            "androidNdkSymbolication": {
              "enabled": true
            }
          }
        }
      ]
    ]
  }
}
```

Set Datadog secrets:

```bash
eas secret:create --scope project --name DATADOG_API_KEY --value "your-api-key"
eas secret:create --scope project --name DATADOG_SITE --value "datadoghq.com"
```

### Firebase Crashlytics Integration

```bash
npx expo install @react-native-firebase/app @react-native-firebase/crashlytics
```

```json
// app.json
{
  "expo": {
    "plugins": [
      "@react-native-firebase/app",
      "@react-native-firebase/crashlytics",
      [
        "expo-build-properties",
        {
          "ios": {
            "useFrameworks": "static"
          }
        }
      ]
    ],
    "android": {
      "googleServicesFile": "./google-services.json"
    },
    "ios": {
      "googleServicesFile": "./GoogleService-Info.plist"
    }
  }
}
```

### Plugin Ordering Pitfalls

When using multiple crash reporting or monitoring plugins, **order matters**.

```json
{
  "plugins": [
    "expo-datadog",
    "@sentry/react-native/expo"
  ]
}
```

**`expo-datadog` must come before `@sentry/react-native`** if you use both. The Datadog plugin modifies the Xcode build phases for dSYM upload, and Sentry's plugin does the same. If Sentry runs first, Datadog's build phase may not be positioned correctly, causing dSYM upload failures for Datadog.

General rule: check each plugin's documentation for ordering requirements. When in doubt, place more "foundational" plugins (build properties, Firebase) before "observability" plugins (Sentry, Datadog).

---

## 8. Performance & Optimization

### Faster Builds

| Strategy | Impact | How |
|----------|--------|-----|
| **Use `large` resource class** | 30-50% faster | Set `"resourceClass": "large"` in build profile |
| **Enable build caching** | Up to 90% faster (cache hit) | Set `"buildCacheProvider": "eas"` (SDK 53+) |
| **Reduce config plugins** | Fewer prebuild steps | Audit and remove unused plugins |
| **Cache custom paths** | Faster dependency resolution | Configure `cache.paths` in build profile |
| **Pin dependency versions** | Consistent, cacheable installs | Use lockfiles, avoid `^` ranges for native deps |
| **Use `.easignore`** | Faster upload | Exclude test files, docs, unused packages |
| **Parallel platform builds** | Build both at once | `eas build --platform all` |

### @expo/fingerprint

The fingerprint library computes a hash of all native-impacting configuration. It's used internally by the `fingerprint` runtime version policy, but you can also use it directly in CI.

**What it hashes:**
- `app.json` / `app.config.js` (native-relevant fields)
- `package.json` dependencies (native modules only)
- Config plugin output
- iOS `Podfile.lock` (if present)
- Android Gradle files (if present)
- Expo SDK version
- Custom native code in `android/` and `ios/` (if present)

**Using in CI for build decisions:**

```bash
npx @expo/fingerprint ./
```

```javascript
// scripts/should-build.js
const { createFingerprintAsync } = require('@expo/fingerprint');

async function main() {
  const fingerprint = await createFingerprintAsync(process.cwd());
  const currentHash = fingerprint.hash;

  // Compare with the last deployed fingerprint
  const lastDeployedHash = process.env.LAST_DEPLOYED_FINGERPRINT;

  if (currentHash === lastDeployedHash) {
    console.log('No native changes detected. Skipping build, using EAS Update.');
    process.exit(0);
  } else {
    console.log('Native changes detected. Triggering full build.');
    process.exit(1);
  }
}

main();
```

**CI workflow example:**

```yaml
# .github/workflows/smart-deploy.yml
jobs:
  check-native-changes:
    runs-on: ubuntu-latest
    outputs:
      needs-build: ${{ steps.fingerprint.outputs.changed }}
    steps:
      - uses: actions/checkout@v4
      - run: npm install @expo/fingerprint
      - id: fingerprint
        run: |
          HASH=$(npx @expo/fingerprint ./)
          if [ "$HASH" != "${{ vars.LAST_FINGERPRINT }}" ]; then
            echo "changed=true" >> $GITHUB_OUTPUT
          else
            echo "changed=false" >> $GITHUB_OUTPUT
          fi

  build:
    needs: check-native-changes
    if: needs.check-native-changes.outputs.needs-build == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: eas build --platform all --profile production --non-interactive

  update:
    needs: check-native-changes
    if: needs.check-native-changes.outputs.needs-build == 'false'
    runs-on: ubuntu-latest
    steps:
      - run: eas update --branch production --message "JS-only update"
```

### Expo Modules API: Custom Native Modules

When you need native functionality not covered by existing libraries, use the Expo Modules API to create custom native modules with Swift and Kotlin.

```bash
npx create-expo-module my-native-module
```

This generates a module with the following structure:

```
my-native-module/
├── src/
│   └── MyNativeModule.ts          # TypeScript interface
├── ios/
│   └── MyNativeModule.swift       # Swift implementation
├── android/
│   └── src/main/java/.../
│       └── MyNativeModule.kt      # Kotlin implementation
├── expo-module.config.json
└── package.json
```

**Swift implementation example:**

```swift
// ios/MyNativeModule.swift
import ExpoModulesCore

public class MyNativeModule: Module {
  public func definition() -> ModuleDefinition {
    Name("MyNativeModule")

    Function("getDeviceName") {
      return UIDevice.current.name
    }

    AsyncFunction("fetchNativeData") { (url: String) -> String in
      let data = try await URLSession.shared.data(from: URL(string: url)!)
      return String(data: data.0, encoding: .utf8) ?? ""
    }

    Events("onStatusChange")

    View(MyNativeView.self) {
      Prop("color") { (view, color: UIColor) in
        view.backgroundColor = color
      }
    }
  }
}
```

**Kotlin implementation example:**

```kotlin
// android/src/main/java/.../MyNativeModule.kt
package com.example.mynativemodule

import expo.modules.kotlin.modules.Module
import expo.modules.kotlin.modules.ModuleDefinition

class MyNativeModule : Module() {
  override fun definition() = ModuleDefinition {
    Name("MyNativeModule")

    Function("getDeviceName") {
      android.os.Build.MODEL
    }

    AsyncFunction("fetchNativeData") { url: String ->
      // Kotlin coroutine-based async work
      val connection = java.net.URL(url).openConnection()
      connection.getInputStream().bufferedReader().readText()
    }

    Events("onStatusChange")
  }
}
```

**TypeScript interface:**

```typescript
// src/MyNativeModule.ts
import { requireNativeModule } from 'expo-modules-core';

const MyNativeModule = requireNativeModule('MyNativeModule');

export function getDeviceName(): string {
  return MyNativeModule.getDeviceName();
}

export async function fetchNativeData(url: string): Promise<string> {
  return MyNativeModule.fetchNativeData(url);
}
```

---

## 9. EAS Workflows

EAS Workflows (beta) let you define multi-step CI/CD pipelines that orchestrate builds, submissions, updates, and tests.

### YAML Configuration

Workflow files live in `.eas/workflows/`:

```yaml
# .eas/workflows/production-deploy.yml
name: Production Deploy

on:
  push:
    branches: ['main']

jobs:
  build-android:
    type: build
    platform: android
    profile: production

  build-ios:
    type: build
    platform: ios
    profile: production

  submit-android:
    type: submit
    needs: [build-android]
    platform: android
    profile: production

  submit-ios:
    type: submit
    needs: [build-ios]
    platform: ios
    profile: production

  notify:
    type: custom
    after: [submit-android, submit-ios]
    steps:
      - run: curl -X POST $SLACK_WEBHOOK -d '{"text":"Production build submitted!"}'
```

### Job Types

| Type | Description | Produces |
|------|-------------|---------|
| `build` | Runs `eas build` with specified profile | Build artifact |
| `submit` | Runs `eas submit` with specified profile | Store submission |
| `update` | Runs `eas update` | OTA update |
| `maestro` | Runs Maestro E2E tests against a build | Test results |
| `custom` | Runs arbitrary shell commands | Custom output |

### Job Dependencies: `needs` vs `after`

| Keyword | Behavior | Use Case |
|---------|----------|----------|
| `needs` | Waits for listed jobs to **succeed**. Receives their outputs. Fails if dependency fails. | Submit needs a successful build |
| `after` | Waits for listed jobs to **complete** (success or failure). Always runs. | Notification after builds, regardless of result |

**Example with Maestro testing:**

```yaml
# .eas/workflows/test-and-deploy.yml
name: Test and Deploy

on:
  push:
    branches: ['main']

jobs:
  build-preview:
    type: build
    platform: ios
    profile: preview

  e2e-tests:
    type: maestro
    needs: [build-preview]
    flow_path: ./maestro/critical-flows.yml

  build-production:
    type: build
    needs: [e2e-tests]
    platform: ios
    profile: production

  submit-production:
    type: submit
    needs: [build-production]
    platform: ios
    profile: production
```

**Run a workflow:**

```bash
eas workflow:run .eas/workflows/production-deploy.yml
```

---

## 10. Pricing

### Plan Comparison (as of 2026)

| Feature | Free | Production | Enterprise |
|---------|------|------------|------------|
| **Price** | $0/month | $99/month | Custom |
| **Build credits/month** | 30 | Varies by plan | Custom |
| **iOS builds** | Yes (queued) | Priority | Dedicated |
| **Android builds** | Yes (queued) | Priority | Dedicated |
| **Large resource class** | No | Yes | Yes |
| **EAS Update** | 10K users | Included | Custom limits |
| **EAS Submit** | Yes | Yes | Yes |
| **EAS Metadata** | Yes | Yes | Yes |
| **Team members** | 1 | Unlimited | Unlimited |
| **Priority support** | Community | Email | Dedicated |
| **Custom SLA** | No | No | Yes |
| **Build concurrency** | 1 | Up to 4 | Custom |
| **Build cache** | Limited | Full | Full |

> **Note:** Pricing tiers and specific numbers change frequently. Check [expo.dev/pricing](https://expo.dev/pricing) for the latest information.

### Build Credits

Each build consumes credits based on platform and resource class:

| Platform | Resource Class | Credits Per Build |
|----------|---------------|-------------------|
| Android | Medium | 1 |
| Android | Large | 2 |
| iOS | Medium | 2 |
| iOS | Large | 4 |

Credits reset monthly. Overages are billed per-credit on paid plans. Free tier builds are queued behind paid users during peak times.

### Usage-Based Overages

On paid plans, if you exceed your monthly credit allocation:
- Additional Android medium builds: ~$1-2 per build
- Additional iOS medium builds: ~$2-4 per build
- Additional EAS Update monthly active users beyond plan limits are billed per-user
- Exact overage pricing depends on your plan and negotiated rates

---

## 11. Sources

### Official Expo Documentation

- [EAS Build Introduction](https://docs.expo.dev/build/introduction/)
- [EAS Build Configuration (eas.json)](https://docs.expo.dev/build/eas-json/)
- [EAS Build Server Infrastructure](https://docs.expo.dev/build-reference/infrastructure/)
- [EAS Build Credentials](https://docs.expo.dev/app-signing/managed-credentials/)
- [EAS Build Environment Variables](https://docs.expo.dev/build-reference/variables/)
- [EAS Build Caching](https://docs.expo.dev/build-reference/caching/)
- [EAS Build Monorepos](https://docs.expo.dev/guides/monorepos/)
- [EAS Build Lifecycle Hooks](https://docs.expo.dev/build-reference/npm-hooks/)
- [EAS Submit Introduction](https://docs.expo.dev/submit/introduction/)
- [EAS Submit to Google Play](https://docs.expo.dev/submit/android/)
- [EAS Submit to App Store](https://docs.expo.dev/submit/ios/)
- [EAS Update Introduction](https://docs.expo.dev/eas-update/introduction/)
- [EAS Update Runtime Versions](https://docs.expo.dev/eas-update/runtime-versions/)
- [EAS Update Rollouts](https://docs.expo.dev/eas-update/rollouts/)
- [EAS Update Rollback](https://docs.expo.dev/eas-update/rollbacks/)
- [Expo Prebuild](https://docs.expo.dev/workflow/prebuild/)
- [Config Plugins Introduction](https://docs.expo.dev/config-plugins/introduction/)
- [Creating Config Plugins](https://docs.expo.dev/config-plugins/plugins-and-mods/)
- [EAS Metadata](https://docs.expo.dev/eas/metadata/)
- [EAS Workflows](https://docs.expo.dev/eas/workflows/)
- [Expo Modules API](https://docs.expo.dev/modules/overview/)
- [@expo/fingerprint](https://docs.expo.dev/versions/latest/sdk/fingerprint/)
- [EAS Pricing](https://expo.dev/pricing)

### Crash Reporting Integration

- [Sentry Expo Guide](https://docs.sentry.io/platforms/react-native/manual-setup/expo/)
- [Datadog React Native Setup](https://docs.datadoghq.com/real_user_monitoring/mobile_and_tv_monitoring/setup/expo/)
- [Firebase Crashlytics with Expo](https://rnfirebase.io/crashlytics/usage)
- [expo-datadog Plugin](https://github.com/DataDog/expo-datadog)

### Community Resources

- [Expo Blog](https://blog.expo.dev/)
- [Expo GitHub Repository](https://github.com/expo/expo)
- [EAS CLI Source Code](https://github.com/expo/eas-cli)
- [Expo Discord Community](https://chat.expo.dev/)
- [Expo Forums](https://forums.expo.dev/)

---

*This guide is intended as a reference for React Native / Expo developers. For the most up-to-date information, always check the [official Expo documentation](https://docs.expo.dev/).*

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Expo App Config Decision Guide](./app-config-decision-guide.md) | Companion guide — every app.config.ts and eas.json property explained with WHAT/WHY/TRADE-OFFS |
| [Monitoring & ANR Analysis](../optimization/monitoring-anr-analysis.md) | Post-deploy monitoring setup — crash reporting integration with EAS builds |
| [Upgrading React Native](../optimization/upgrading-react-native.md) | Version upgrade strategy that affects EAS build configuration |
