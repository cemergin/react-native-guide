# Deep Dive: List Performance — FlatList → FlashList v2

> Synthesized from Shopify Engineering (FlashList creators) and Discord Engineering.

[Back to Index](./README.md) | [Reading List](./reading-list.md)

---

## Why Lists Are the #1 Performance Problem

Lists are the most common performance bottleneck in React Native apps. A chat app (like Discord) or a merchant dashboard (like Shopify) can mount thousands of views in a single list. The default `FlatList` creates and destroys views as you scroll — **FlashList recycles them**.

## FlashList v2: What Changed

FlashList v2 is a complete ground-up rewrite for the New Architecture. Key changes:

| Feature | FlatList | FlashList v1 | FlashList v2 |
|---------|---------|-------------|-------------|
| View recycling | No | Yes (native) | Yes (JS-only) |
| Requires `estimatedItemSize` | N/A | **Yes** | **No** |
| Native code dependency | N/A | Yes (`AutoLayoutView`) | **None** |
| Blank area while scrolling | High | Low | **50% less than v1** |
| Web support | N/A | Limited | **Full** |
| Horizontal list resizing | N/A | Broken post-mount | **Works** |

### How v2 Eliminates Estimates

Three-pillar approach:
1. **Progressive rendering** — Mounts only 1-2 items initially, then renders more as layout is understood
2. **Predictions** — Uses measured items to estimate unmeasured items of the same type
3. **Corrections** — Adjusts layouts pre-paint via `useLayoutEffect` to prevent gaps/overlaps

### Pixel-Perfect `scrollToIndex()`

v2's algorithm:
1. Calculates target using layout map
2. Computes neighboring item layouts
3. Continuously refines position as items measure
4. Lands exactly on target — every time

## Discord's List Evolution

Discord went through **four generations** of list solutions:

### Generation 1: FlatList/SectionList
- **Problem**: Mounted ~2,000 views in large servers
- **Result**: Unusable on older devices

### Generation 2: react-native-large-list
- 60ms rendering, ~400 views mounted
- **Problem**: Locked CPU on 100,000+ rows

### Generation 3: recyclerlistview
- 30ms server switch, 10ms channel switch, 60 FPS scroll
- **Problem**: Blank frames on mount

### Generation 4: Custom FastList → FastestList
- Built internal component reusing web list logic
- Added sticky headers and view recycling
- **Result**: 70-90ms render time savings
- Later rebuilt as **FastestList** using native `RecyclerView` on Android
- **Achievement**: Eliminated blanking even on $50 phones

### Discord's Additional List Optimizations

| Technique | Impact |
|-----------|--------|
| Pre-filling recycling pools before chat opens | Eliminated blank frames |
| Lazy inflation for spoilers and upload overlays | Reduced initial mount cost |
| Recycling media mosaics | 12% less memory in chat |
| Up to 60% reduction in slow frames | Smoother scrolling |

## Migration Guide: FlatList → FlashList v2

### Step 1: Install
```bash
yarn add @shopify/flash-list
```

### Step 2: Replace FlatList

```tsx
// Before
import { FlatList } from 'react-native';

<FlatList
  data={items}
  renderItem={({ item }) => <ItemComponent item={item} />}
  keyExtractor={(item) => item.id}
/>

// After
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={items}
  renderItem={({ item }) => <ItemComponent item={item} />}
  keyExtractor={(item) => item.id}
  // No estimatedItemSize needed in v2!
/>
```

### Step 3: Add `getItemType` for Mixed Lists

If your list has multiple item types (headers, items, footers), tell FlashList:

```tsx
<FlashList
  data={items}
  renderItem={renderItem}
  getItemType={(item) => item.type} // 'header' | 'item' | 'footer'
/>
```

This enables better recycling — items of the same type reuse each other's views.

### Step 4: Profile

Use Flashlight to measure before/after:
```bash
npx @bamlab/flashlight measure
```

## Performance Checklist for Lists

- [ ] Replace `FlatList` with `FlashList` (v2 if on New Architecture)
- [ ] Add `getItemType` for lists with mixed content types
- [ ] Memoize `renderItem` components with `React.memo`
- [ ] Avoid inline functions in `renderItem` (use `useCallback`)
- [ ] Use `keyExtractor` with stable, unique keys (not array index)
- [ ] For images in lists, use pre-cached thumbnails (Discord: 256x256 instead of 370x370)
- [ ] Pre-fill recycling pools for immediate display (no blank frames)
- [ ] Test on low-end Android devices ($50 phones)
- [ ] Profile with Flashlight or React DevTools Profiler

## When to Build Custom (Discord's Approach)

Consider a custom list when:
- You have 1,000+ items with complex interactions
- You need platform-specific native recycling (e.g., Android `RecyclerView`)
- Sticky headers with gesture-driven interactions are required
- You're building a chat interface with real-time updates

For most apps, **FlashList v2 is the right choice**. Only Discord-scale apps (millions of MAU, 100k+ row lists) need custom solutions.

---

Sources:
- [Shopify: FlashList v2](https://shopify.engineering/flashlist-v2)
- [Discord: Native iOS Performance](https://discord.com/blog/how-discord-achieves-native-ios-performance-with-react-native)
- [Discord: Supercharging Mobile](https://discord.com/blog/supercharging-discord-mobile-our-journey-to-a-faster-app)
