# Security & Data Protection

> Your React Native bundle ships to the user's device. Treat it as untrusted territory — anything in the JS bundle is readable. Secure your secrets, storage, and communication accordingly.

[← Back to Index](../README.md)

**Keywords**: security, API keys, secrets, expo-secure-store, Keychain, Keystore, MMKV encryption, certificate pinning, biometric authentication, ProGuard, R8, obfuscation, jailbreak, OWASP, RASP

---

<details>
<summary><strong>TL;DR</strong></summary>

- NEVER ship API keys in your JS bundle — even Hermes bytecode is decompilable
- Use expo-secure-store (Keychain/Keystore) for auth tokens, not AsyncStorage (unencrypted + backed up)
- Certificate pin your production API to prevent MITM attacks
- expo-local-authentication for biometric auth (Face ID, Touch ID, Android Biometric Prompt)
- freeRASP by Talsec for jailbreak/root/Frida/emulator/tamper detection (free, open-source)

</details>

## The #1 React Native Security Mistake

**Shipping API keys in your JS bundle.** This happens more often than you'd think:

```tsx
// NEVER DO THIS — keys are in the shipped bundle and trivially extractable
const API_KEY = 'sk-live-abc123def456';
const STRIPE_SECRET = 'sk_live_xxx';

// Also exposed (Hermes bytecode is decompilable):
const config = {
  apiKey: process.env.API_SECRET,  // Inlined at build time!
};
```

**Why it's dangerous**: Even with Hermes bytecode compilation, tools can extract string literals. `EXPO_PUBLIC_*` variables are explicitly designed to be public — they're embedded in the bundle. Never put secrets there.

**The fix**: Keep secrets on your server. The client should authenticate via tokens, not API keys.

---

## Secure Storage

### The Storage Decision

| Storage | Encrypted | Sync | Use For |
|---------|-----------|------|---------|
| **expo-secure-store** | Yes (Keychain/Keystore) | No | Auth tokens, API keys, sensitive user data |
| **react-native-mmkv** (encrypted) | Yes (AES-256) | No | Encrypted app state, session data |
| **react-native-mmkv** (default) | No | No | Non-sensitive preferences, cache |
| **AsyncStorage** | **No** | **Backed up** | **Never for sensitive data** — unencrypted, included in device backups |
| **SQLite** (expo-sqlite) | No (by default) | No | Structured data, offline cache |

### expo-secure-store (Recommended for Secrets)

Uses iOS Keychain and Android EncryptedSharedPreferences under the hood.

```tsx
import * as SecureStore from 'expo-secure-store';

// Store a secret
await SecureStore.setItemAsync('auth_token', token);

// Retrieve
const token = await SecureStore.getItemAsync('auth_token');

// Delete on logout
await SecureStore.deleteItemAsync('auth_token');
```

**Limits**: 2048 bytes per value on iOS. For larger data, encrypt with a key stored in SecureStore.

### MMKV with Encryption

For app state that needs encryption but not Keychain-level security:

```tsx
import { MMKV } from 'react-native-mmkv';

const secureStorage = new MMKV({
  id: 'secure-app-storage',
  encryptionKey: 'your-256-bit-key', // Store THIS key in expo-secure-store
});

// Synchronous, encrypted reads/writes
secureStorage.set('session', JSON.stringify(sessionData));
const session = JSON.parse(secureStorage.getString('session') ?? '{}');
```

### Why AsyncStorage Is Not Secure

- Stored as **plain text** JSON files on the filesystem
- Included in **device backups** (iCloud, Google backup)
- Readable by any app with root/jailbreak access
- No encryption option

> **See also**: [Modern Patterns: MMKV](../foundation/modern-patterns.md#pattern-4-mmkv-over-asyncstorage) for migration guide

---

## Authentication

### Biometric Authentication (expo-local-authentication)

```tsx
import * as LocalAuthentication from 'expo-local-authentication';

async function authenticateWithBiometrics(): Promise<boolean> {
  // Check hardware support
  const hasHardware = await LocalAuthentication.hasHardwareAsync();
  if (!hasHardware) return false;

  // Check if enrolled (has fingerprint/face set up)
  const isEnrolled = await LocalAuthentication.isEnrolledAsync();
  if (!isEnrolled) return false;

  // Authenticate
  const result = await LocalAuthentication.authenticateAsync({
    promptMessage: 'Verify your identity',
    cancelLabel: 'Use password',
    disableDeviceFallback: false, // Allow PIN/password as fallback
  });

  return result.success;
}

// Usage: protect sensitive screens
function SensitiveScreen() {
  const [authenticated, setAuthenticated] = useState(false);

  useEffect(() => {
    authenticateWithBiometrics().then(setAuthenticated);
  }, []);

  if (!authenticated) return <AuthPrompt />;
  return <SensitiveContent />;
}
```

### Token Management Pattern

```tsx
// Secure token storage + refresh
class AuthService {
  async getAccessToken(): Promise<string | null> {
    const token = await SecureStore.getItemAsync('access_token');
    if (!token) return null;

    // Check expiration
    const payload = JSON.parse(atob(token.split('.')[1]));
    if (payload.exp * 1000 < Date.now()) {
      return this.refreshToken();
    }
    return token;
  }

  async refreshToken(): Promise<string | null> {
    const refreshToken = await SecureStore.getItemAsync('refresh_token');
    if (!refreshToken) return null;

    try {
      const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken }),
      });
      const { accessToken, newRefreshToken } = await response.json();
      await SecureStore.setItemAsync('access_token', accessToken);
      await SecureStore.setItemAsync('refresh_token', newRefreshToken);
      return accessToken;
    } catch {
      await this.logout(); // Force re-login on refresh failure
      return null;
    }
  }

  async logout() {
    await SecureStore.deleteItemAsync('access_token');
    await SecureStore.deleteItemAsync('refresh_token');
  }
}
```

---

## Network Security

### Certificate Pinning

Prevents man-in-the-middle attacks by verifying the server's SSL certificate against a known hash.

```tsx
// Using react-native-ssl-pinning
import { fetch as pinnedFetch } from 'react-native-ssl-pinning';

const response = await pinnedFetch('https://api.yourapp.com/data', {
  method: 'GET',
  sslPinning: {
    certs: ['your-cert-sha256-hash'], // SHA-256 of the server's public key
  },
  headers: { Authorization: `Bearer ${token}` },
});
```

**Extracting the certificate hash**:
```bash
# Get SHA-256 pin from your server
openssl s_client -connect api.yourapp.com:443 | \
  openssl x509 -pubkey -noout | \
  openssl pkey -pubin -outform der | \
  openssl dgst -sha256 -binary | \
  openssl enc -base64
```

**Gotchas**:
- Pin the **public key**, not the certificate (certificates rotate, keys usually don't)
- Include a **backup pin** (in case you rotate keys)
- Certificate pinning **blocks Charles Proxy / Proxyman** in debug — disable in dev builds

### HTTPS Enforcement

```tsx
// api-client.ts — enforce HTTPS
function createApiClient(baseUrl: string) {
  if (!__DEV__ && !baseUrl.startsWith('https://')) {
    throw new Error('Production API must use HTTPS');
  }
  // ... configure axios/fetch
}
```

---

## Code Protection

### Hermes Bytecode (Built-in)

Hermes compiles JavaScript to bytecode at build time. This isn't encryption, but it raises the bar significantly:
- Source code is not directly readable
- Variable names and comments are stripped
- Decompilation produces less readable output than plain JS

**This happens automatically** when Hermes is enabled (default in Expo).

### Android ProGuard / R8

R8 (the default in modern Android builds) obfuscates Java/Kotlin code and removes unused code.

```groovy
// android/app/build.gradle
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

**Critical keep rules for React Native**:

```proguard
# proguard-rules.pro
-keep class com.facebook.hermes.unicode.** { *; }
-keep class com.facebook.jni.** { *; }
-keep class com.facebook.react.** { *; }

# Keep native module registrations
-keepclassmembers class * {
    @com.facebook.react.bridge.ReactMethod <methods>;
}

# Keep Reanimated
-keep class com.swmansion.reanimated.** { *; }
```

### JavaScript Obfuscation (Extra Layer)

For high-security apps, add JS-level obfuscation on top of Hermes:

```bash
# Using react-native-obfuscating-transformer
npm install react-native-obfuscating-transformer --save-dev
```

```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const config = getDefaultConfig(__dirname);

if (!__DEV__) {
  config.transformer.minifierPath = 'react-native-obfuscating-transformer';
  config.transformer.minifierConfig = {
    obfuscate: true,
    compact: true,
  };
}

module.exports = config;
```

---

## Runtime Protection (RASP)

### freeRASP by Talsec (Recommended)

Open-source Runtime Application Self-Protection that detects:
- Root / jailbreak
- Frida / dynamic instrumentation
- Emulator / simulator
- App tampering / repackaging
- Debugger attachment
- Unofficial installer (sideloading)

```tsx
import { useFreeRasp, Threat } from 'react-native-freerasp';

function App() {
  useFreeRasp({
    androidConfig: {
      packageName: 'com.yourapp',
      certificateHashes: ['your-signing-cert-hash'],
    },
    iosConfig: {
      appBundleId: 'com.yourapp',
      appTeamId: 'YOUR_TEAM_ID',
    },
    onThreatDetected: (threat: Threat) => {
      switch (threat) {
        case Threat.Jailbreak:
        case Threat.Root:
          // Log but don't necessarily block — many legitimate users have rooted devices
          analytics.track('security_threat', { type: threat });
          break;
        case Threat.Tampering:
        case Threat.UnofficialStore:
          // This is more serious — likely a pirated/modified build
          Alert.alert('Security Warning', 'This app may have been tampered with.');
          break;
        case Threat.Debugger:
          // Only alert in production — expected in development
          if (!__DEV__) analytics.track('debugger_detected');
          break;
      }
    },
  });

  return <MainApp />;
}
```

---

## OWASP Mobile Top 10 for React Native

| # | Risk | RN-Specific Mitigation |
|---|------|----------------------|
| M1 | Improper Credential Usage | expo-secure-store for tokens, never hardcode keys |
| M2 | Inadequate Supply Chain | Audit dependencies, use lockfiles, Depcheck |
| M3 | Insecure Auth/Authorization | Server-side validation, biometric + token combo |
| M4 | Insufficient I/O Validation | Validate all API responses, sanitize user input |
| M5 | Insecure Communication | HTTPS only, certificate pinning, no HTTP in production |
| M6 | Inadequate Privacy Controls | Minimize data collection, encrypted storage |
| M7 | Insufficient Binary Protection | Hermes bytecode, R8/ProGuard, freeRASP |
| M8 | Security Misconfiguration | Remove debug flags, disable backup for sensitive data |
| M9 | Insecure Data Storage | Never AsyncStorage for secrets, use Keychain/Keystore |
| M10 | Insufficient Cryptography | Use platform crypto (Keychain/Keystore), not JS crypto |

> **See also**: [OWASP: Securing React Native with MAS](https://owasp.org/blog/2024/10/02/Securing-React-Native-Mobile-Apps-with-OWASP-MAS)

---

## Checklist

### Secrets
- [ ] No API keys or secrets in JS bundle
- [ ] No secrets in `EXPO_PUBLIC_*` environment variables
- [ ] Secrets stored server-side, client uses tokens
- [ ] `.env` files in `.gitignore`

### Storage
- [ ] Auth tokens in expo-secure-store (not AsyncStorage)
- [ ] MMKV encryption key stored in SecureStore
- [ ] `android:allowBackup="false"` for sensitive apps
- [ ] No sensitive data in plain-text logs

### Network
- [ ] HTTPS enforced for all API calls
- [ ] Certificate pinning for production API endpoints
- [ ] Backup pin configured for key rotation

### Authentication
- [ ] Biometric auth for sensitive operations
- [ ] Token refresh with secure storage
- [ ] Session timeout for inactive users
- [ ] Logout clears all secure storage

### Binary Protection
- [ ] Hermes enabled (bytecode compilation)
- [ ] R8/ProGuard enabled with keep rules
- [ ] freeRASP or equivalent RASP SDK
- [ ] Debug flags removed in release builds

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Error Handling](./error-handling.md) | Secure error reporting — don't leak sensitive data in error messages |
| [Crash Analysis (SIGABRT)](../debugging/crash-analysis.md) | Security-related crashes (memory corruption, buffer overflow) |
| [Modern Patterns: MMKV](../foundation/modern-patterns.md#pattern-4-mmkv-over-asyncstorage) | MMKV encryption setup and AsyncStorage migration |
| [Expo App Config](../expo/app-config-decision-guide.md) | Config-level security settings (allowBackup, permissions) |
| [Dependency Management](../optimization/dependency-management.md) | Supply chain security — auditing and managing dependencies |

---

Sources:
- [React Native: Security Docs](https://reactnative.dev/docs/security)
- [OWASP: Securing React Native with MAS](https://owasp.org/blog/2024/10/02/Securing-React-Native-Mobile-Apps-with-OWASP-MAS)
- [Oscar Franco: React Native Security Guide](https://ospfranco.com/react-native-security-guide/)
- [Expo: SecureStore Documentation](https://docs.expo.dev/versions/latest/sdk/securestore/)
- [Expo: LocalAuthentication Documentation](https://docs.expo.dev/versions/latest/sdk/local-authentication/)
- [freeRASP by Talsec](https://github.com/talsec/Free-RASP-ReactNative)
- [LogRocket: Biometric Auth with Expo](https://blog.logrocket.com/implementing-react-native-biometric-authentication-expo/)
