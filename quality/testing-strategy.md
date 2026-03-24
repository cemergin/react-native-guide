# Testing Strategy: Jest, RNTL & E2E

> Test what matters. The testing trophy — heavy on integration, light on unit — fits React Native better than the traditional pyramid.

[← Back to Index](../README.md)

**Keywords**: Jest, React Native Testing Library, RNTL, Maestro, Detox, E2E, mocking, native modules, Expo, testing trophy, integration testing, snapshot testing

---

<details>
<summary><strong>TL;DR</strong></summary>

- Testing trophy > testing pyramid for React Native — integration tests (RNTL) give best ROI
- Query by what users see: `getByRole` > `getByText` > `getByTestId` (last resort)
- Maestro for E2E: YAML-based, <1% flakiness, zero codebase changes, QA testers can write tests
- Mock native modules in `jest.setup.ts`, create a reusable test harness with providers
- Run unit+integration on every PR, E2E on merge, Flashlight benchmarks nightly

</details>

## The Testing Trophy for React Native

The React Native community has converged on Kent C. Dodds' **testing trophy** over the traditional testing pyramid. Integration tests provide the best ROI for mobile apps.

```
          ┌───────┐
          │  E2E  │         ← Maestro/Detox: critical user flows
         ┌┴───────┴┐
         │Integration│      ← RNTL: most tests live here
        ┌┴──────────┴┐
        │   Unit     │     ← Jest: pure functions, utilities
       ┌┴────────────┴┐
       │ Static Analysis│   ← TypeScript + ESLint
       └──────────────┘
```

**Why integration-heavy?** React Native components interact with native modules, navigation, and state in ways that unit tests can't capture. "The more your tests resemble the way your software is used, the more confidence they can give you." — Kent C. Dodds

---

## Layer 1: Static Analysis (TypeScript + ESLint)

Your first line of defense catches bugs before any test runs.

```json
// tsconfig.json — strict mode
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true
  }
}
```

```json
// eslint config (Expo SDK 53+ uses flat config)
// eslint.config.js
import expoConfig from 'eslint-config-expo';
export default [...expoConfig];
```

**What this catches**: Type mismatches, unreachable code, missing return statements, unused imports, accessibility prop issues.

---

## Layer 2: Unit Tests (Jest)

Test pure functions, utilities, and business logic in isolation.

### Jest Configuration for Expo

```json
// package.json
{
  "jest": {
    "preset": "jest-expo",
    "transformIgnorePatterns": [
      "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg)"
    ]
  }
}
```

The `jest-expo` preset handles Hermes, Metro bundler transforms, and platform-specific module resolution. Use `jest-expo/universal` for cross-platform runs (iOS, Android, web, Node).

### Mocking Native Modules

The #1 pain point in React Native testing. Native modules don't exist in the Jest environment.

```tsx
// __mocks__/react-native-mmkv.ts
export class MMKV {
  private store: Map<string, string> = new Map();
  getString(key: string) { return this.store.get(key); }
  set(key: string, value: string) { this.store.set(key, value); }
  delete(key: string) { this.store.delete(key); }
  getBoolean(key: string) { return this.store.get(key) === 'true'; }
}
```

```tsx
// jest.setup.ts — mock common native modules
jest.mock('react-native/Libraries/Animated/NativeAnimatedHelper');
jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

// Mock expo modules
jest.mock('expo-secure-store', () => ({
  getItemAsync: jest.fn(),
  setItemAsync: jest.fn(),
  deleteItemAsync: jest.fn(),
}));

jest.mock('expo-haptics', () => ({
  impactAsync: jest.fn(),
  notificationAsync: jest.fn(),
}));
```

> **See also**: [Expo: Mocking Native Calls](https://docs.expo.dev/modules/mocking/) for the official reference

### What to Unit Test

- Pure utility functions (formatters, validators, parsers)
- State management logic (Zustand store actions)
- API response transformers
- Business rules and calculations

### What NOT to Unit Test

- Component rendering (use RNTL instead)
- Navigation flows (use E2E)
- Native module behavior (mock it, don't test the mock)

---

## Layer 3: Integration Tests (React Native Testing Library)

RNTL is the standard for testing React Native components in a way that resembles how users interact with your app.

### Core Philosophy

**Query by what the user sees**, not by implementation details:

```tsx
// GOOD: queries what the user sees/hears
const button = screen.getByRole('button', { name: 'Submit' });
const input = screen.getByPlaceholderText('Enter email');
const message = screen.getByText('Welcome back!');

// AVOID: queries implementation details
const button = screen.getByTestId('submit-btn');  // Last resort only
```

**Priority order**: `getByRole` > `getByText` > `getByPlaceholderText` > `getByDisplayValue` > `getByTestId`

### Testing with Providers (Reusable Render)

Most React Native components need providers (navigation, query client, theme). Create a reusable render utility:

```tsx
// test-utils.tsx
import { render, RenderOptions } from '@testing-library/react-native';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { NavigationContainer } from '@react-navigation/native';

function AllProviders({ children }: { children: React.ReactNode }) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return (
    <QueryClientProvider client={queryClient}>
      <NavigationContainer>
        {children}
      </NavigationContainer>
    </QueryClientProvider>
  );
}

const customRender = (ui: React.ReactElement, options?: RenderOptions) =>
  render(ui, { wrapper: AllProviders, ...options });

export * from '@testing-library/react-native';
export { customRender as render };
```

### Testing Async Operations

```tsx
import { render, screen, waitFor } from './test-utils';

test('loads and displays user profile', async () => {
  // Mock API response
  server.use(
    rest.get('/api/user/123', (req, res, ctx) =>
      res(ctx.json({ name: 'Jane', email: 'jane@example.com' }))
    ),
  );

  render(<ProfileScreen userId="123" />);

  // Wait for async data
  expect(await screen.findByText('Jane')).toBeTruthy();
  expect(screen.getByText('jane@example.com')).toBeTruthy();
});
```

### Testing Expo Router

```tsx
import { renderRouter } from 'expo-router/testing-library';

test('navigates to settings from home', async () => {
  const { getByText } = renderRouter({
    index: () => <HomeScreen />,
    'settings/index': () => <SettingsScreen />,
  });

  fireEvent.press(getByText('Settings'));
  expect(await screen.findByText('Account')).toBeTruthy();
});
```

> **See also**: [Expo Router Testing Docs](https://docs.expo.dev/router/reference/testing/) for the full API

---

## Layer 4: End-to-End Testing

### Maestro (Recommended for Most Teams)

Maestro has overtaken Detox as the dominant E2E framework in 2025-2026. YAML-based, <1% flakiness, zero codebase changes.

```yaml
# e2e/login-flow.yaml
appId: com.yourapp
---
- launchApp
- tapOn: "Sign In"
- tapOn: "Email"
- inputText: "user@example.com"
- tapOn: "Password"
- inputText: "securepassword"
- tapOn: "Log In"
- assertVisible: "Welcome back"
- assertVisible: "Home"
```

```bash
# Run locally
maestro test e2e/login-flow.yaml

# Run in CI with Flashlight for performance
npx @bamlab/flashlight test --testCommand "maestro test e2e/scroll-benchmark.yaml"
```

**Maestro strengths**: YAML simplicity, works on release builds, no SDK needed, manual QA testers can write tests, <1% flakiness.

**Maestro limitations**: Black-box only (can't access component state), no programmatic assertions in JS, limited gesture support compared to Detox.

### Detox (When You Need Gray-Box Control)

Choose Detox when you need:
- Programmatic JS test authoring (complex assertions)
- Direct access to native APIs during tests
- Synchronization with React Native's bridge/runtime
- Fine-grained gesture simulation

```tsx
// e2e/login.test.ts (Detox)
describe('Login', () => {
  beforeAll(async () => {
    await device.launchApp();
  });

  it('should login successfully', async () => {
    await element(by.id('email-input')).typeText('user@example.com');
    await element(by.id('password-input')).typeText('securepassword');
    await element(by.text('Log In')).tap();
    await expect(element(by.text('Welcome back'))).toBeVisible();
  });
});
```

### Maestro vs Detox Decision Matrix

| Factor | Maestro | Detox |
|--------|---------|-------|
| Setup time | Minutes | Hours (native build config) |
| Test language | YAML | JavaScript/TypeScript |
| Flakiness | <1% | 5-15% (depends on sync) |
| Codebase changes | Zero | testID props, build configs |
| Release build testing | Yes | Limited |
| Who can write tests | QA + devs | Devs only |
| Complex assertions | Limited | Full JS power |
| CI cost | Lower (shorter runs) | Higher (native builds) |
| Best for | Most teams | Teams needing programmatic control |

---

## CI Integration

### GitHub Actions Example

```yaml
name: Test
on: [push, pull_request]
jobs:
  unit-and-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: yarn }
      - run: yarn install --frozen-lockfile
      - run: yarn test --coverage --ci
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/ }

  e2e-maestro:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: yarn }
      - run: yarn install --frozen-lockfile
      - run: npx expo prebuild --platform android
      - run: curl -Ls "https://get.maestro.mobile.dev" | bash
      - run: maestro test e2e/
```

---

## What to Test — Decision Framework

| What | How | Priority |
|------|-----|----------|
| Pure business logic (validators, formatters) | Jest unit tests | High |
| Component rendering with data | RNTL integration tests | High |
| Critical user flows (login, checkout, onboarding) | Maestro E2E | High |
| Navigation between screens | RNTL with expo-router/testing-library | Medium |
| Error states and edge cases | RNTL (mock error responses) | Medium |
| Accessibility (screen reader) | RNTL role queries + manual VoiceOver | Medium |
| List scroll performance | Flashlight + Maestro in CI | Low (nightly) |
| Visual regression | Snapshot tests (use sparingly) | Low |

---

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Do Instead |
|-------------|---------------|------------|
| Testing implementation details (state, hooks) | Breaks on refactor, gives false confidence | Test behavior the user sees |
| 100% code coverage as a goal | Incentivizes low-value tests | Cover critical paths, not lines |
| Snapshot testing everything | Snapshots break constantly, nobody reviews diffs | Use for stable UI (icons, design tokens) only |
| Mocking everything | Tests pass but don't test real behavior | Mock boundaries (API, native), not internals |
| E2E for everything | Slow, flaky, expensive | E2E for critical flows only; RNTL for the rest |
| No test infrastructure (providers, utils) | Copy-paste provider setup in every test | Build a reusable test harness |

---

## Checklist

### Project Setup
- [ ] `jest-expo` preset configured (or `jest-expo/universal`)
- [ ] `jest.setup.ts` with common native module mocks
- [ ] Reusable `test-utils.tsx` with provider wrappers
- [ ] `transformIgnorePatterns` configured for all RN dependencies
- [ ] TypeScript strict mode enabled

### Per-Feature
- [ ] Unit tests for business logic / pure functions
- [ ] RNTL integration tests for component behavior
- [ ] Error state tests (network failure, empty state, invalid input)
- [ ] Accessibility: queries use `getByRole` / `getByText` where possible

### CI Pipeline
- [ ] Unit + integration tests run on every PR
- [ ] E2E tests (Maestro) run on merge to main
- [ ] Performance benchmarks (Flashlight) run nightly
- [ ] Coverage report uploaded as artifact

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Error Handling & Crash Prevention](./error-handling.md) | Test your error boundaries and recovery flows |
| [Performance & Rendering](../optimization/performance-rendering.md) | What to measure in performance tests |
| [Profiling & Debugging](../optimization/profiling-debugging.md) | Flashlight CI integration for performance regression testing |
| [Monitoring & ANR Analysis](../optimization/monitoring-anr-analysis.md) | Production monitoring that validates your testing caught the right issues |

---

Sources:
- [Expo: Unit Testing with Jest](https://docs.expo.dev/develop/unit-testing/)
- [Expo: Router Testing Library](https://docs.expo.dev/router/reference/testing/)
- [Expo: How to Build a Solid Test Harness](https://expo.dev/blog/how-to-build-a-solid-test-harness-for-expo-apps)
- [Expo: Mocking Native Calls](https://docs.expo.dev/modules/mocking/)
- [React Native: Testing Overview](https://reactnative.dev/docs/testing-overview)
- [Kent C. Dodds: Testing Trophy](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
- [Jupiter: Maestro vs Detox](https://life.jupiter.money/choosing-between-maestro-and-detox-on-jupiter-qa-automation-7b94e6f8759d)
- [Callstack: React Native Harness](https://www.callstack.com/blog/introducing-react-native-harness-fast-real-device-testing-for-native-modules)
- [Yuri Kan: RNTL Best Practices](https://yrkan.com/blog/react-native-testing-library/)
