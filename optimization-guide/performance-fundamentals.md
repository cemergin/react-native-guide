# Performance Fundamentals

> Small improvements compound. Each optimization here reduces jank, re-renders, or wasted work.

[Back to Index](./README.md) | Previous: [Dependencies](./dependency-management.md)

---

## 1. Animation Performance {#1-animation-performance}

### The Problem

Without native driver, animations flow through the JS bridge:
```
JS thread → Bridge → Native thread → UI
```
This means any heavy JS work (API calls, saga processing) causes animation jank.

### The Fix

```tsx
// Always set useNativeDriver: true
Animated.timing(opacity, {
  toValue: 1,
  duration: 1000,
  useNativeDriver: true,  // Bypasses the bridge entirely
}).start();
```

With native driver, animations execute directly on the native UI thread — no bridge involved.

### Going Further: React Native Reanimated

For complex gesture-driven animations, Reanimated runs animation logic entirely on the UI thread via **worklets**:
- No bridge communication at all
- Animations survive JS thread blocking
- Gesture-driven interactions stay at 60fps

### Limitations of `useNativeDriver`

Only works with **non-layout** properties:
- `opacity`, `transform` (translate, scale, rotate)
- Does NOT work with `width`, `height`, `margin`, `padding`, `flex`

For layout animations, use Reanimated's `Layout` animations or `LayoutAnimation`.

### Connection to ANRs

Animation jank is a leading cause of perceived sluggishness. If your [ANR analysis](./anr-analysis.md) shows UI thread blocking, this is the first thing to check.

---

## 2. Memoization: useMemo vs useCallback {#2-memoization}

### When to Use Each

| Hook | Memoizes | Use When |
|------|----------|----------|
| `useMemo` | A **computed value** | Expensive calculations, filtered/sorted lists |
| `useCallback` | A **function reference** | Passing callbacks to `React.memo` children |

### useMemo — Cache Expensive Work

```tsx
// Only recomputes when `transactions` changes
const sortedTransactions = useMemo(
  () => transactions.sort((a, b) => b.date - a.date),
  [transactions]
);
```

### useCallback — Stabilize Function Identity

```tsx
// Without useCallback, a new function is created every render,
// causing memoized children to re-render unnecessarily
const handlePress = useCallback((id: string) => {
  navigation.navigate('Details', { id });
}, [navigation]);
```

### When NOT to Memoize

- Simple computations that are fast anyway
- Functions not passed to memoized children
- Values that change on every render (defeats the purpose)

**Rule of thumb**: Only memoize when you have a **measurable** performance problem or are passing to `React.memo` wrapped children.

---

## 3. TypeScript

Already enforced in this codebase. Key benefits as a performance tool:
- Catches bugs at compile time → fewer runtime crashes → fewer [ANRs](./anr-analysis.md)
- Self-documenting interfaces reduce misuse of components
- Safer refactoring when [replacing libraries](./dependency-management.md#lighter-alternatives)

---

## 4. Code Organization

### Styles

If your app uses a **design token system** (custom container components, typography components like `H1`, `Body`, `Caption`), prefer tokens over raw styles:

```tsx
// Use semantic tokens from your design system
<Container padding="md" margin="sm">
  <Heading variant="primary">Title</Heading>
  <Body variant="secondary">Description</Body>
</Container>

// Over raw StyleSheet values
// style={{ padding: 16, marginBottom: 8 }}
```

Design tokens ensure consistency and make global style changes easy.

---

## 5. React Query for Data Fetching {#5-react-query}

### The Problem with Redux for API Data

```tsx
// Redux approach: lots of boilerplate for every API call
const dispatch = useDispatch();
const data = useSelector(state => state.user.data);
const loading = useSelector(state => state.user.loading);
const error = useSelector(state => state.user.error);

useEffect(() => {
  dispatch(fetchUserData());
}, []);
```

For each API endpoint, you need: action types, action creators, reducer cases, saga handlers, selectors.

### The React Query Alternative

```tsx
const { data, isLoading, error } = useQuery(
  ['userData', userId],
  () => fetchUserData(userId),
  {
    staleTime: 5 * 60 * 1000,     // Consider fresh for 5 minutes
    cacheTime: 30 * 60 * 1000,    // Keep in cache for 30 minutes
  }
);
```

**What you get for free**:
- Automatic caching and deduplication
- Background refetching on focus/reconnect
- Loading, error, and success states
- Optimistic updates
- Automatic garbage collection of stale data

### Adoption Strategy for Existing Redux Apps

Don't rewrite everything. Use React Query for **new features** while keeping existing Redux/Saga flows stable:

1. Install `@tanstack/react-query` and set up `QueryClientProvider`
2. Use React Query for new screens/features
3. Gradually migrate existing screens when they need changes anyway
4. Keep Redux for **global app state** (auth, settings, offline data) — it's still good at that

### When to Keep Redux

- Offline-first data that needs persistence
- Global UI state (modals, toasts, navigation state)
- Complex state machines with many interdependencies
- Data shared across many screens that mutates client-side

---

## 6. Network Logger (Dev Only)

A floating dev-only button that opens `react-native-network-logger` for real-time API inspection.

### When It's Useful

- Debugging on **physical devices** where Chrome DevTools network tab isn't available
- Checking request/response payloads without Flipper
- Verifying caching behavior (see headers, response times)
- Debugging auth token issues (inspect Authorization headers)

### Key Detail

Gate behind `__DEV__` so it never ships to production:
```tsx
if (__DEV__) {
  return <NetworkLoggerButton />;
}
return null;
```

### Connection to ANRs

If your [ANR analysis](./anr-analysis.md) points to network-related freezes, the network logger helps identify:
- Slow endpoints blocking the UI
- Redundant API calls that could be cached
- Large payloads that should be paginated

---

## Performance Checklist

Before shipping a new feature, verify:

- [ ] Animations use `useNativeDriver: true` or Reanimated
- [ ] Expensive computations are wrapped in `useMemo`
- [ ] Callbacks passed to memoized children use `useCallback`
- [ ] No new [unnecessary dependencies](./dependency-management.md#before-adding-a-library) added
- [ ] API data fetching uses caching (React Query or equivalent)
- [ ] Tested on a low-end device (not just emulator)
- [ ] [ANR rates](./anr-analysis.md) monitored post-release
