# Error Handling & Crash Prevention

> In React Native, there's no browser refresh button. When your app crashes, the user must force-quit and relaunch. Layered error handling isn't optional — it's survival.

[← Back to Index](../README.md)

**Keywords**: error boundary, ErrorUtils, global handler, Sentry, Crashlytics, crash prevention, retry logic, graceful degradation, exception handler, recovery

---

## The Error Handling Stack

React Native errors come from multiple layers. Each requires a different catching mechanism.

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 5: Production Monitoring                          │
│  Sentry, Crashlytics, Datadog                           │
│  Catches: Everything that reaches production             │
├─────────────────────────────────────────────────────────┤
│  LAYER 4: Native Crash Handler                           │
│  react-native-exception-handler                         │
│  Catches: Native fatal crashes (SIGSEGV, EXC_BAD_ACCESS)│
├─────────────────────────────────────────────────────────┤
│  LAYER 3: Global JS Error Handler                        │
│  ErrorUtils.setGlobalHandler                            │
│  Catches: Uncaught JS exceptions, unhandled rejections  │
├─────────────────────────────────────────────────────────┤
│  LAYER 2: Error Boundaries                               │
│  React componentDidCatch / getDerivedStateFromError      │
│  Catches: Render errors, lifecycle errors                │
├─────────────────────────────────────────────────────────┤
│  LAYER 1: Local Error Handling                           │
│  try/catch, React Query error states, form validation   │
│  Catches: Expected errors in specific operations         │
└─────────────────────────────────────────────────────────┘
```

### What Catches What

| Error Type | Caught By | Example |
|-----------|-----------|---------|
| Render / lifecycle errors | Error Boundaries | Component throws during render |
| Event handler errors | try/catch | Button press handler throws |
| Async / Promise rejections | Global rejection handler, React Query | Failed API call |
| Uncaught JS exceptions | `ErrorUtils.setGlobalHandler` | TypeError in setTimeout |
| Native crashes (SIGSEGV) | `react-native-exception-handler`, Sentry | Memory corruption, null pointer |
| Network failures | Retry logic, React Query | Server timeout, offline |

---

## Layer 1: Local Error Handling

### try/catch for Event Handlers

Error boundaries don't catch errors in event handlers. Use try/catch:

```tsx
function SubmitButton({ onSubmit }: { onSubmit: () => Promise<void> }) {
  const [error, setError] = useState<string | null>(null);

  const handlePress = async () => {
    try {
      setError(null);
      await onSubmit();
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong');
      // Report to Sentry without crashing
      Sentry.captureException(err);
    }
  };

  return (
    <>
      {error && <Text style={styles.error}>{error}</Text>}
      <Button title="Submit" onPress={handlePress} />
    </>
  );
}
```

### React Query Error Handling

React Query handles server state errors elegantly — no manual try/catch needed for API calls:

```tsx
function UserProfile({ userId }: { userId: string }) {
  const { data, error, isError, refetch } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
    retry: 3,                          // Auto-retry 3 times
    retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000), // Exponential backoff
  });

  if (isError) {
    return (
      <ErrorCard
        message={error.message}
        onRetry={refetch}
      />
    );
  }

  return <ProfileView user={data} />;
}
```

### Network Retry with Exponential Backoff

For non-React Query network calls:

```tsx
async function fetchWithRetry(
  url: string,
  options?: RequestInit,
  maxRetries = 3,
): Promise<Response> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok && response.status >= 500) {
        throw new Error(`Server error: ${response.status}`);
      }
      return response;
    } catch (err) {
      if (attempt === maxRetries) throw err;
      // Exponential backoff with jitter
      const delay = Math.min(1000 * 2 ** attempt, 10000);
      const jitter = delay * 0.1 * Math.random();
      await new Promise(resolve => setTimeout(resolve, delay + jitter));
    }
  }
  throw new Error('Unreachable');
}
```

---

## Layer 2: Error Boundaries

### The Critical Difference from Web

On the web, a crashed page can be refreshed. In React Native, an unhandled render error **crashes the entire app**. Users must force-quit and relaunch. This makes error boundaries significantly more important.

### Per-Screen Error Boundaries

Wrap each screen (or major section) in its own error boundary. This isolates crashes to one screen while keeping the rest of the app functional.

```tsx
// components/ErrorBoundary.tsx
import React, { Component, ErrorInfo } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import * as Sentry from '@sentry/react-native';

interface Props {
  children: React.ReactNode;
  fallback?: React.ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    Sentry.captureException(error, { extra: { componentStack: errorInfo.componentStack } });
    this.props.onError?.(error, errorInfo);
  }

  resetError = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <View style={styles.container}>
          <Text style={styles.title}>Something went wrong</Text>
          <Text style={styles.message}>{this.state.error?.message}</Text>
          <TouchableOpacity style={styles.button} onPress={this.resetError}>
            <Text style={styles.buttonText}>Try Again</Text>
          </TouchableOpacity>
        </View>
      );
    }
    return this.props.children;
  }
}
```

### Wrapping Navigation Stacks

```tsx
// app/_layout.tsx (Expo Router)
export default function RootLayout() {
  return (
    <ErrorBoundary>
      <QueryClientProvider client={queryClient}>
        <Stack>
          <Stack.Screen name="(tabs)" />
          <Stack.Screen name="modal" />
        </Stack>
      </QueryClientProvider>
    </ErrorBoundary>
  );
}

// Per-screen boundaries for isolation
// app/(tabs)/home.tsx
export default function HomeScreen() {
  return (
    <ErrorBoundary fallback={<HomeErrorFallback />}>
      <HomeFeed />
    </ErrorBoundary>
  );
}
```

---

## Layer 3: Global JS Error Handler

### ErrorUtils.setGlobalHandler

This catches any uncaught JS exception — the last line of defense before a crash.

```tsx
// app/_layout.tsx or index.ts — run at app initialization
import * as Sentry from '@sentry/react-native';

// Capture the original handler
const originalHandler = ErrorUtils.getGlobalHandler();

ErrorUtils.setGlobalHandler((error: Error, isFatal?: boolean) => {
  // Report to Sentry
  Sentry.captureException(error, {
    extra: { isFatal },
    level: isFatal ? 'fatal' : 'error',
  });

  if (__DEV__) {
    // In development, show the red box
    originalHandler(error, isFatal);
  } else if (isFatal) {
    // In production, show a user-friendly message before crash
    Alert.alert(
      'Unexpected Error',
      'The app encountered an error and needs to restart. We\'ve been notified.',
      [{ text: 'OK', onPress: () => {} }]
    );
  }
});
```

### Handling Unhandled Promise Rejections

```tsx
// Polyfill for tracking unhandled rejections
if (typeof global.addEventListener === 'undefined') {
  // React Native doesn't have window.addEventListener
  const tracking = require('promise/setimmediate/rejection-tracking');
  tracking.enable({
    allRejections: true,
    onUnhandled: (id: number, error: Error) => {
      Sentry.captureException(error, { extra: { promiseId: id } });
    },
  });
}
```

---

## Layer 4: Native Crash Handler

For crashes that happen below the JS layer (C++, Objective-C, Kotlin), JS error handlers don't work. You need `react-native-exception-handler`:

```tsx
import { setNativeExceptionHandler, setJSExceptionHandler } from 'react-native-exception-handler';

// JS exceptions (backup for ErrorUtils)
setJSExceptionHandler((error, isFatal) => {
  Sentry.captureException(error);
  if (isFatal) {
    Alert.alert('Error', 'An unexpected error occurred. Please restart the app.');
  }
}, true); // true = allow in production

// Native exceptions (SIGSEGV, EXC_BAD_ACCESS, etc.)
setNativeExceptionHandler((errorString) => {
  // This runs in a limited context — can't show UI on iOS
  // On Android, can optionally show a dialog and restart
  Sentry.captureMessage(`Native crash: ${errorString}`, 'fatal');
}, false, true); // (handler, forceAppQuit, executeDefaultHandler)
```

> **See also**: [Crash Analysis Guide](../debugging/crash-analysis.md) for root cause analysis of native crashes

---

## Layer 5: Production Monitoring

### Sentry Integration

```tsx
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  tracesSampleRate: __DEV__ ? 1.0 : 0.2,  // 20% of production transactions
  enableAutoSessionTracking: true,
  attachStacktrace: true,
  environment: __DEV__ ? 'development' : 'production',
});

// Wrap root component
export default Sentry.wrap(RootLayout);
```

### Crashlytics Integration

```tsx
import crashlytics from '@react-native-firebase/crashlytics';

// Log non-fatal errors
try {
  await riskyOperation();
} catch (error) {
  crashlytics().recordError(error as Error);
  crashlytics().log('riskyOperation failed');
  // Don't rethrow — handle gracefully
}

// Add context for debugging
crashlytics().setAttributes({
  screen: 'checkout',
  userId: user.id,
  cartSize: String(cart.items.length),
});
```

### Choosing Between Sentry and Crashlytics

| Factor | Sentry | Crashlytics |
|--------|--------|-------------|
| JS + native visibility | Excellent (unified view) | Native-focused |
| Source map support | Automatic with plugin | Manual upload |
| AI-suggested fixes | Yes | No |
| Performance monitoring | Built-in | Separate (Firebase Performance) |
| Cost | Free tier (5K events/mo) | Free (part of Firebase) |
| Overhead | <1% CPU, 0.8% battery | Similar |
| Best for | Full-stack JS visibility | Firebase ecosystem apps |

---

## Common Crash Prevention Patterns

### 1. Optional Chaining for API Responses

```tsx
// CRASHES: if response shape changes
const name = response.data.user.profile.name;

// SAFE: graceful fallback
const name = response.data?.user?.profile?.name ?? 'Unknown';
```

### 2. Safe JSON Parsing

```tsx
function safeJsonParse<T>(json: string, fallback: T): T {
  try {
    return JSON.parse(json) as T;
  } catch {
    Sentry.captureMessage(`JSON parse failed: ${json.substring(0, 100)}`);
    return fallback;
  }
}
```

### 3. Image Error Handling

```tsx
<Image
  source={{ uri: imageUrl }}
  onError={() => setImageUrl(FALLBACK_IMAGE)}
  defaultSource={require('./placeholder.png')}
/>
```

### 4. Navigation Guard

```tsx
function safeNavigate(path: string) {
  try {
    router.push(path);
  } catch (error) {
    Sentry.captureException(error);
    router.replace('/');  // Fallback to home
  }
}
```

---

## Checklist

### Setup
- [ ] Error boundaries wrapping each screen / major section
- [ ] `ErrorUtils.setGlobalHandler` configured
- [ ] `react-native-exception-handler` for native crashes
- [ ] Sentry or Crashlytics initialized at app root
- [ ] Source maps uploaded for production builds

### Per-Feature
- [ ] try/catch in all event handlers that can fail
- [ ] React Query with retry + error fallback for API calls
- [ ] Optional chaining for all external data
- [ ] Loading, error, and empty states for every data-dependent screen
- [ ] Image components have `onError` + `defaultSource`

### Production
- [ ] Crash-free rate dashboard visible to team (target: >99.8%)
- [ ] Regression alerts for new releases
- [ ] Non-fatal error tracking (not just crashes)
- [ ] User context attached to crash reports (screen, user ID, actions)

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Crash Analysis (SIGABRT)](../debugging/crash-analysis.md) | When error handling fails — root cause analysis for native crashes |
| [Profiling Tools Deep Dive](../debugging/profiling-tools-deep-dive.md) | Memory and CPU issues that lead to crashes |
| [Testing Strategy](./testing-strategy.md) | Test your error boundaries and recovery flows |
| [Monitoring & ANR Analysis](../optimization/monitoring-anr-analysis.md) | Production monitoring that surfaces errors |
| [Security](./security.md) | Secure error reporting (don't leak sensitive data in error messages) |

---

Sources:
- [Andrei Calazans: Global Error Handlers in RN](https://andrei-calazans.com/posts/how-to-global-catch-errors/)
- [Carlos Cuesta: Error Boundaries in Production at Factorial](https://carloscuesta.me/blog/managing-react-native-crashes-with-error-boundaries)
- [Sentry: React Native Documentation](https://docs.sentry.io/platforms/react-native/)
- [React Native Firebase: Crashlytics](https://rnfirebase.io/crashlytics/usage)
- [react-native-exception-handler](https://github.com/a7ul/react-native-exception-handler)
- [Youssouf El Azizi: Complete Error Handling Guide](https://elazizi.com/posts/handling-errors-in-react-native-a-complete-guide/)
- [DZone: Stop React Native Crashes](https://dzone.com/articles/react-native-error-handling-guide)
