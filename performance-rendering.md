# Chapter 5: Performance & Rendering

> Small improvements compound. Each optimization reduces jank, re-renders, or wasted work.

[← Dependencies](./dependency-management.md) | [Index](./README.md) | [Profiling →](./profiling-debugging.md)

**Keywords**: animation, useNativeDriver, Reanimated, useMemo, useCallback, React.memo, re-render, FlashList, FlatList, view recycling, React Query, Redux, data fetching, startup, TTI, cold start

---

## Animations {#animations}

### The Problem

Without native driver, animations flow through the JS bridge:
```
JS thread → Bridge → Native thread → UI
```
Any heavy JS work (API calls, saga processing) causes animation jank — a leading cause of perceived sluggishness in [ANR analysis](./monitoring-anr-analysis.md).

### The Fix

```tsx
Animated.timing(opacity, {
  toValue: 1,
  duration: 1000,
  useNativeDriver: true,  // Bypasses the bridge entirely
}).start();
```

### Audit Your App for `useNativeDriver: false`

Search your codebase for files using `useNativeDriver: false`. Common culprits include progress bars, toggle buttons, skeleton loaders, pin indicators, and spinning loaders.

If the animation only transforms `opacity` or `transform` → switch to `useNativeDriver: true`.

### Going Further: React Native Reanimated {#reanimated}

For complex gesture-driven animations, Reanimated runs entirely on the UI thread via **worklets**. No bridge at all.

**Shopify's experience**: At scale, Reanimated caused severe frame drops during [New Architecture migration](./new-architecture-migration.md#common-pitfalls). Ensure you're on latest Reanimated before migrating.

**Discord's experience**: They moved drawer interactions to `react-native-gesture-handler` + Reanimated — "buttery smooth" result. Also found a spinning animation consuming 10% CPU while hidden — removing it was a bigger win than any optimization. See [Profiling: Discord Wins](./profiling-debugging.md#discord-wins).

### Limitations

`useNativeDriver` only works with **non-layout** properties: `opacity`, `transform`. For `width`/`height`/`margin` animations → use Reanimated's `Layout` animations.

> **See also**: [Profiling: Finding Slow Re-renders](./profiling-debugging.md#finding-slow-re-renders) for diagnosing animation-related jank

---

## Memoization {#memoization}

| Hook | Memoizes | Use When |
|------|----------|----------|
| `useMemo` | A **computed value** | Expensive calculations, filtered/sorted lists |
| `useCallback` | A **function reference** | Passing callbacks to `React.memo` children |

```tsx
// useMemo — only recomputes when transactions changes
const sorted = useMemo(() => transactions.sort((a, b) => b.date - a.date), [transactions]);

// useCallback — stable function identity for memoized children
const handlePress = useCallback((id: string) => {
  navigation.navigate('Details', { id });
}, [navigation]);
```

**When NOT to memoize**: Simple computations, functions not passed to memoized children, values that change every render.

**React Compiler (Expo SDK 54+)**: Auto-memoizes at compile time — eliminates this entire class of bugs. See [Profiling: React Compiler](./profiling-debugging.md#react-compiler).

> **See also**: [Profiling: Finding Slow Re-renders](./profiling-debugging.md#finding-slow-re-renders) for when memoization actually matters

---

## Lists {#lists}

Lists are the **#1 performance bottleneck** in React Native. FlatList creates/destroys views on scroll. FlashList recycles them.

### FlashList v2 vs FlatList

| Feature | FlatList | FlashList v2 |
|---------|---------|-------------|
| View recycling | No | Yes (JS-only, no native deps) |
| Requires `estimatedItemSize` | N/A | **No** (v2 eliminates estimates) |
| Blank area while scrolling | High | **50% less than FlashList v1** |
| Web support | N/A | **Full** |

### Migration: FlatList → FlashList

```tsx
// Before
import { FlatList } from 'react-native';
<FlatList data={items} renderItem={renderItem} keyExtractor={keyExtractor} />

// After
import { FlashList } from '@shopify/flash-list';
<FlashList data={items} renderItem={renderItem} keyExtractor={keyExtractor}
  getItemType={(item) => item.type}  // enables better recycling
/>
```

### Discord's List Evolution (4 Generations)

| Gen | Solution | Result | Problem |
|-----|----------|--------|---------|
| 1 | FlatList | ~2,000 views | Unusable on old devices |
| 2 | react-native-large-list | ~400 views | CPU locked on 100k+ rows |
| 3 | recyclerlistview | 60 FPS scroll | Blank frames on mount |
| 4 | Custom FastList → FastestList (native RecyclerView) | No blanking on $50 phones | — |

**Key techniques from Discord**: Pre-fill recycling pools, lazy inflation for spoilers, recycling media mosaics (12% less memory), thumbnail downsizing (256x256 vs 370x370).

For most apps, **FlashList v2 is the right choice**. Only Discord-scale apps (100k+ rows) need custom solutions.

### List Performance Checklist

- [ ] Replace `FlatList` with `FlashList`
- [ ] Add `getItemType` for mixed content types
- [ ] Wrap `renderItem` components in `React.memo`
- [ ] Use `useCallback` for functions passed to list items
- [ ] Use stable `keyExtractor` (not array index)
- [ ] Pre-cache thumbnails at smaller sizes
- [ ] Test on low-end Android devices ($50 phones)
- [ ] Profile with [Flashlight](./profiling-debugging.md#flashlight)

> **See also**: [Profiling: Flashlight](./profiling-debugging.md#flashlight) for measuring list FPS | [Dependencies](./dependency-management.md) for FastImage vs standard Image

---

## Data Fetching {#data-fetching}

### React Query vs Redux for API Data

Redux + Sagas requires action types, creators, reducer cases, saga handlers, and selectors **for every endpoint**. React Query gives you caching, loading states, and background refetching for free:

```tsx
const { data, isLoading } = useQuery(['userData', userId], () => fetchUserData(userId), {
  staleTime: 5 * 60 * 1000,     // Fresh for 5 minutes
  cacheTime: 30 * 60 * 1000,    // Cache for 30 minutes
});
```

### Adoption Strategy (many production apps use Redux + Sagas)

1. Install `@tanstack/react-query` + `QueryClientProvider`
2. Use React Query for **new screens/features**
3. Migrate existing screens when they need changes anyway
4. Keep Redux for **global state** (auth, settings, offline data)

### When to Keep Redux

- Offline-first data needing persistence
- Global UI state (modals, toasts, navigation)
- Complex state machines with interdependencies
- **Expensify's approach**: They built [react-native-onyx](https://github.com/Expensify/react-native-onyx) — an offline-first state layer. See [Reading List](./reading-list.md).

---

## Code Organization

If your app uses a **design token system** (semantic colors, spacing tokens, typography components), prefer tokens over raw styles:
```tsx
// Example using a design token system
<Container padding="md" margin="sm">
  <Heading variant="primary">Title</Heading>
  <BodyText variant="secondary">Description</BodyText>
</Container>
```

This approach supersedes the generic advice of "separate your StyleSheet files" — tokens enforce consistency and reduce style duplication.

---

## Network Logger (Dev Only)

A floating dev-only button for real-time API inspection. Useful when [ANR analysis](./monitoring-anr-analysis.md) points to network-related freezes. Gate behind `__DEV__`:

```tsx
if (__DEV__) return <NetworkLoggerButton />;
return null;
```

> **See also**: [Native Debugging Guide](./native-layer-debugging-guide.md#56-flipper-is-dead--use-these-instead) for Flipper replacements including `react-native-network-logger`

---

## Pre-Ship Checklist

- [ ] Animations use `useNativeDriver: true` or Reanimated
- [ ] Expensive computations wrapped in `useMemo`
- [ ] Callbacks to memoized children use `useCallback`
- [ ] Lists use FlashList with `getItemType`
- [ ] No new [unnecessary dependencies](./dependency-management.md#before-adding)
- [ ] API fetching uses caching (React Query or equivalent)
- [ ] Tested on a low-end device (not just emulator)
- [ ] [ANR rates](./monitoring-anr-analysis.md) monitored post-release

---

> **See also**: [Profiling Tools Deep Dive](./profiling-tools-deep-dive.md) for measuring list and rendering performance with Flashlight

**Next**: [Profiling & Debugging →](./profiling-debugging.md)

Sources:
- [Shopify: FlashList v2](https://shopify.engineering/flashlist-v2)
- [Discord: Native iOS Performance](https://discord.com/blog/how-discord-achieves-native-ios-performance-with-react-native)
- [Discord: Supercharging Mobile](https://discord.com/blog/supercharging-discord-mobile-our-journey-to-a-faster-app)
