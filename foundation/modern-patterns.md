# Modern React Native: Architecture, Patterns & Trends (2025-2026)

> The React Native ecosystem moves fast. This guide maps the current landscape — what's standard, what's emerging, and what to choose for your next project.

[← Back to Index](../README.md)

**Keywords**: Expo Router, Zustand, Jotai, Legend State, React Query, TanStack Query, tRPC, MMKV, offline-first, file-based routing, state management, data fetching, architecture patterns, React Compiler

---

<details>
<summary><strong>TL;DR</strong></summary>

- 2026 default stack: Expo Router + Zustand + TanStack Query + React Hook Form + MMKV + Reanimated
- Separate state into 3 categories: server (TanStack Query), client (Zustand), form (RHF)
- Expo Router v4 breaking change: `navigate()` now behaves like `push()`
- MMKV is 30x faster than AsyncStorage with optional encryption
- React Compiler (SDK 54+) auto-memoizes — 83% of EAS builds use it

</details>

## The Modern React Native Stack (2026)

The community has converged on a default stack. If you're starting a new project, this is the baseline:

| Layer | Default Choice | Alternatives | When to Deviate |
|-------|---------------|-------------|-----------------|
| **Framework** | Expo (SDK 52+) | Bare RN CLI | Custom native build systems, brownfield apps |
| **Navigation** | Expo Router v4 | React Navigation 7 | Need manual control over navigation state |
| **State (client)** | Zustand | Jotai, Legend State | Atomic state → Jotai; Offline-first → Legend State |
| **State (server)** | TanStack Query (React Query) | RTK Query, SWR | Legacy Redux codebase → RTK Query |
| **Forms** | React Hook Form + Zod | Formik | Complex multi-step forms |
| **Styling** | NativeWind (Tailwind) | StyleSheet, Tamagui, Unistyles | Design system consistency → Tamagui or Unistyles |
| **Storage** | react-native-mmkv | AsyncStorage, SQLite | Large datasets → SQLite; Simple K/V → MMKV |
| **Animations** | Reanimated v3 + Gesture Handler | Moti, Animated API | Simple opacity/transform → `useNativeDriver` |
| **Build** | EAS Build | Fastlane, GitHub Actions | Self-hosted CI → Fastlane |

---

## Navigation: Expo Router vs React Navigation

### Expo Router (Recommended for Expo Projects)

Expo Router is a **file-based router** built on top of React Navigation. When you add a file to the `app/` directory, it automatically becomes a route.

```
app/
├── _layout.tsx          → Root layout (tabs, stack, drawer)
├── index.tsx            → Home screen (/)
├── settings/
│   ├── _layout.tsx      → Settings stack layout
│   ├── index.tsx        → Settings screen (/settings)
│   └── account.tsx      → Account screen (/settings/account)
└── [id].tsx             → Dynamic route (/:id)
```

**Key advantages**:
- **Automatic deep linking** — every route is a URL, works on web and mobile
- **Lazy evaluation** — routes are only loaded when needed in production
- **Type-safe** — `useLocalSearchParams<{ id: string }>()` for route params
- **Web support** — same routes work on web with SEO-friendly URLs
- **Async routes** — code-splitting in development for faster refresh

**Expo Router v4 breaking change**: `router.navigate()` now behaves identically to `router.push()` (always adds to stack). In v3, `navigate()` would intelligently pop back to existing routes. If you're upgrading, audit all `router.navigate()` calls.

### React Navigation 7 (When You Need Full Control)

React Navigation remains the foundation under Expo Router, and v7 is fully compatible with both Expo and bare RN.

**Choose React Navigation directly when**:
- You need custom transition animations per-screen
- You need programmatic navigation state manipulation
- You're in a bare React Native project without Expo
- You need navigation patterns Expo Router doesn't support yet

### Navigation Performance Tips

- **Use `lazy: true`** on stack navigators to defer rendering off-screen screens
- **Avoid deep nesting** — `settings/account` is better than Stack > Stack > Screen
- **Pre-load screens** — for critical paths, use `router.prefetch()` (Expo Router)
- **Memoize screen components** — screens are just React components, `React.memo` applies

```tsx
// Lazy tab navigation (screens only mount when focused)
<Tabs screenOptions={{ lazy: true }}>
  <Tabs.Screen name="home" />
  <Tabs.Screen name="search" />
  <Tabs.Screen name="profile" />
</Tabs>
```

> **See also**: [Performance: Startup](../optimization/profiling-debugging.md#startup-optimization) for lazy navigation impact on cold start

---

## State Management Decision Framework

### The Three Categories of State

Modern React Native separates state into three distinct categories. Mixing them is the #1 architecture mistake.

```
┌─────────────────────────────────────────────┐
│  SERVER STATE (async, cached, shared)        │
│  "Data from your API"                        │
│  Tool: TanStack Query (React Query)          │
│  Examples: user profile, product list,       │
│            notifications, feed data          │
├─────────────────────────────────────────────┤
│  CLIENT STATE (sync, local, transient)       │
│  "UI state and app state"                    │
│  Tool: Zustand / Jotai / Legend State        │
│  Examples: theme, auth token, sidebar open,  │
│            selected filters, cart items      │
├─────────────────────────────────────────────┤
│  FORM STATE (ephemeral, validated)           │
│  "What the user is currently typing"         │
│  Tool: React Hook Form + Zod                 │
│  Examples: login form, search input,         │
│            checkout flow, profile edit        │
└─────────────────────────────────────────────┘
```

### Zustand: The Default Client State Manager

Zustand is the most popular choice in 2026 — ~1KB bundle, zero boilerplate, works with React Native out of the box.

```tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { zustandStorage } from './mmkv-storage'; // MMKV adapter

interface AuthStore {
  token: string | null;
  user: User | null;
  setAuth: (token: string, user: User) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      setAuth: (token, user) => set({ token, user }),
      logout: () => set({ token: null, user: null }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => zustandStorage), // MMKV for persistence
    }
  )
);
```

**When to use Zustand**:
- Global UI state (theme, auth, sidebar, modals)
- State shared across many screens
- State that persists between sessions (with MMKV middleware)
- You want the simplest possible API

### Jotai: Atomic State for Complex Dependencies

Jotai (~2.5KB) shines when state atoms depend on each other — computed values, derived state, interdependent filters.

```tsx
import { atom, useAtom } from 'jotai';

// Base atoms
const filtersAtom = atom({ category: 'all', priceRange: [0, 100] });
const sortAtom = atom<'price' | 'rating' | 'newest'>('newest');

// Derived atom — recomputes when filters or sort change
const filteredProductsAtom = atom(async (get) => {
  const filters = get(filtersAtom);
  const sort = get(sortAtom);
  const response = await fetch(`/api/products?category=${filters.category}&sort=${sort}`);
  return response.json();
});
```

**When to use Jotai over Zustand**:
- Many small, interdependent pieces of state
- Computed/derived state that auto-updates
- You prefer atomic composition over store-based patterns
- Complex filter/sort UIs where state pieces depend on each other

### Legend State: Offline-First Persistence

Legend State is purpose-built for offline-first React Native apps — persistence with MMKV, automatic conflict resolution, and sync primitives.

```tsx
import { observable } from '@legendapp/state';
import { synced } from '@legendapp/state/sync';
import { ObservablePersistMMKV } from '@legendapp/state/persist-plugins/mmkv';

const tasks$ = observable(
  synced({
    initial: [] as Task[],
    persist: {
      name: 'tasks',
      plugin: ObservablePersistMMKV,
    },
    // Changes made offline are queued and synced when online
    sync: {
      get: () => fetch('/api/tasks').then(r => r.json()),
      set: ({ value }) => fetch('/api/tasks', { method: 'POST', body: JSON.stringify(value) }),
    },
  })
);
```

**When to use Legend State**:
- Offline-first apps (field work, travel, unreliable connectivity)
- Apps that need local-first data with background sync
- You want persistence built into the state layer (not bolted on)
- Complex sync/conflict resolution requirements

### State Management Decision Matrix

| Scenario | Recommended | Why |
|----------|------------|-----|
| New Expo project, standard app | Zustand + TanStack Query | Simplest, smallest, most docs |
| Complex filter/sort UI | Jotai + TanStack Query | Atomic dependencies shine here |
| Offline-first (field app, travel) | Legend State + MMKV | Built-in persistence and sync |
| Legacy Redux codebase | Keep Redux + adopt TanStack Query for new API calls | Don't rewrite what works |
| Enterprise, large team | Redux Toolkit + RTK Query | Strict patterns, established tooling |
| Ephemeral UI state only | React Context | No library needed for 2-3 values |

### Redux: When to Keep, When to Migrate

Redux Toolkit is still the right choice for large enterprise codebases where strict action/reducer patterns aid team coordination. But for new projects in 2026, it's rarely chosen.

**Migration path** (from the [Performance guide](../optimization/performance-rendering.md#data-fetching)):
1. Install TanStack Query for all new API calls
2. Use Zustand for new client state
3. Migrate existing screens when they need changes anyway
4. Keep Redux for global state that's deeply integrated

---

## Data Fetching: TanStack Query + tRPC

### TanStack Query (React Query v5): The Server State Standard

TanStack Query handles caching, background refetching, pagination, optimistic updates, and error/loading states — all the things you'd otherwise build yourself with Redux + Sagas.

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Fetch with automatic caching and background refetch
function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
    staleTime: 5 * 60 * 1000,     // Fresh for 5 minutes
    gcTime: 30 * 60 * 1000,       // Keep in cache 30 minutes
  });
}

// Mutation with optimistic update
function useUpdateProfile() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: ProfileUpdate) => api.updateProfile(data),
    onMutate: async (newData) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['user'] });
      // Snapshot previous value
      const previous = queryClient.getQueryData(['user']);
      // Optimistically update
      queryClient.setQueryData(['user'], (old) => ({ ...old, ...newData }));
      return { previous };
    },
    onError: (err, newData, context) => {
      // Rollback on error
      queryClient.setQueryData(['user'], context?.previous);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['user'] });
    },
  });
}
```

### tRPC: End-to-End Type Safety

If you control both the backend (Node.js/TypeScript) and the React Native frontend, tRPC provides **zero-runtime type safety** — change a server function signature and TypeScript catches the error in your mobile app instantly.

```tsx
// Server: define your API
const appRouter = router({
  user: router({
    getById: publicProcedure
      .input(z.object({ id: z.string() }))
      .query(({ input }) => db.user.findUnique({ where: { id: input.id } })),
    update: protectedProcedure
      .input(z.object({ name: z.string(), bio: z.string().optional() }))
      .mutation(({ input, ctx }) => db.user.update({ where: { id: ctx.userId }, data: input })),
  }),
});

// Client: fully typed, no code generation
const { data: user } = trpc.user.getById.useQuery({ id: '123' });
//    ^? { id: string; name: string; email: string; ... } — fully inferred
```

**When to use tRPC**:
- You own both backend and mobile frontend
- Both are TypeScript
- You want zero-config type safety (no code generation like GraphQL)
- Your team is small-medium and can share a monorepo

**When NOT to use tRPC**:
- Backend is not TypeScript (use REST + React Query)
- Public API consumed by third parties (use REST or GraphQL)
- Team is too large for monorepo coordination

### Data Fetching Decision Matrix

| Scenario | Approach |
|----------|---------|
| REST API you don't control | TanStack Query + fetch/axios |
| TypeScript backend you own | tRPC + TanStack Query |
| GraphQL API | Apollo Client or urql |
| Real-time data (chat, feeds) | TanStack Query + WebSocket subscription |
| Offline-first sync | Legend State synced observables |
| Legacy Redux + Sagas | Keep for existing, TanStack Query for new endpoints |

---

## Frontend Architecture Patterns

### Pattern 1: Feature-Based File Structure

The era of grouping by type (`components/`, `screens/`, `hooks/`) is over. Feature-based organization scales better:

```
app/                          → Expo Router file-based routing
├── (tabs)/
│   ├── _layout.tsx
│   ├── home.tsx
│   ├── search.tsx
│   └── profile.tsx
├── auth/
│   ├── sign-in.tsx
│   └── sign-up.tsx
└── _layout.tsx

features/                     → Business logic, organized by domain
├── auth/
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   └── useSession.ts
│   ├── api/
│   │   └── auth.api.ts
│   ├── stores/
│   │   └── auth.store.ts
│   └── components/
│       ├── LoginForm.tsx
│       └── SocialButtons.tsx
├── products/
│   ├── hooks/
│   │   └── useProducts.ts
│   ├── api/
│   │   └── products.api.ts
│   └── components/
│       ├── ProductCard.tsx
│       └── ProductList.tsx
└── shared/                   → Cross-feature utilities
    ├── components/
    ├── hooks/
    └── utils/
```

**Why this works**:
- Adding a feature = adding a folder (not touching 5+ directories)
- Deleting a feature = deleting a folder
- Each feature is testable in isolation
- Clear ownership boundaries for teams

### Pattern 2: Service Layer Abstraction

Separate API communication from UI components. Never call `fetch()` directly in a component.

```tsx
// features/products/api/products.api.ts
class ProductsAPI {
  async getAll(filters: ProductFilters): Promise<Product[]> {
    const params = new URLSearchParams(filters as any);
    const res = await apiClient.get(`/products?${params}`);
    return res.data;
  }

  async getById(id: string): Promise<Product> {
    const res = await apiClient.get(`/products/${id}`);
    return res.data;
  }
}

export const productsAPI = new ProductsAPI();

// features/products/hooks/useProducts.ts
export function useProducts(filters: ProductFilters) {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: () => productsAPI.getAll(filters),
  });
}

// Screen only knows about the hook, not the API
function ProductsScreen() {
  const { data, isLoading } = useProducts({ category: 'electronics' });
  // ...
}
```

### Pattern 3: React Compiler (Expo SDK 54+)

The React Compiler auto-memoizes components at compile time — eliminating the need for manual `useMemo`, `useCallback`, and `React.memo` in most cases.

```json
// app.json — enable React Compiler
{
  "expo": {
    "experiments": {
      "reactCompiler": true
    }
  }
}
```

**What it changes**:
- **Before**: You manually wrap expensive computations in `useMemo` and callbacks in `useCallback`
- **After**: The compiler does this automatically, and does it better (more granular)
- **Migration**: Enable it, run your tests, remove manual memoization that the compiler handles

As of January 2026, ~83% of SDK 54 projects built with EAS Build use React Compiler.

> **See also**: [React Compiler](../optimization/profiling-debugging.md#react-compiler) | [Performance: Memoization](../optimization/performance-rendering.md#memoization) for when you still need manual memoization

### Pattern 4: MMKV Over AsyncStorage

MMKV is 30x faster than AsyncStorage and supports encryption. It's the default storage choice in 2026.

```tsx
import { MMKV } from 'react-native-mmkv';

export const storage = new MMKV({
  id: 'app-storage',
  encryptionKey: 'your-encryption-key', // optional
});

// Synchronous reads (unlike AsyncStorage)
const token = storage.getString('auth.token');
const onboarded = storage.getBoolean('onboarding.completed');

// Zustand persistence adapter
export const zustandStorage = {
  getItem: (name: string) => storage.getString(name) ?? null,
  setItem: (name: string, value: string) => storage.set(name, value),
  removeItem: (name: string) => storage.delete(name),
};
```

**Why MMKV over AsyncStorage**:
- **Synchronous** — no `await`, reads are instant
- **Encrypted** — optional AES-256 encryption
- **Faster** — 30x read/write performance
- **Smaller** — binary storage vs JSON serialization

> **See also**: [SIGABRT Guide](../debugging/crash-analysis.md) for AsyncStorage security concerns (unencrypted backups)

---

## Emerging Trends to Watch

### Universal Apps (Write Once, Run Everywhere)

Expo's "universal" vision is maturing — same codebase for iOS, Android, and web:
- **Expo Router** handles web + mobile routing from one file tree
- **NativeWind** brings Tailwind CSS to React Native with web parity
- **Tamagui** offers design-system components that compile to native and web

### Server Components for React Native

React Server Components are coming to React Native (experimental). This would allow:
- Fetching data on the server before sending UI to the device
- Smaller bundle sizes (server-only code not shipped to client)
- Better separation of data fetching and rendering

### Local-First / Offline-First

The local-first movement is gaining traction:
- **Legend State** — state management with built-in sync
- **PowerSync** — SQLite-based sync for React Native
- **Expo SQLite** — first-party SQLite support with sync primitives
- **CRDT-based** approaches for conflict-free collaboration

### React Compiler Adoption

React Compiler is rapidly becoming standard:
- 83% of SDK 54 EAS Build projects use it (Jan 2026)
- Eliminates entire categories of performance bugs
- Reduces need for performance expertise on the team

---

## Checklist: Modern RN Project Setup

### New Project Baseline
- [ ] Expo SDK 52+ with `npx create-expo-app`
- [ ] Expo Router for navigation (file-based routing)
- [ ] React Compiler enabled in app.json
- [ ] Zustand for client state (with MMKV persistence)
- [ ] TanStack Query for server state
- [ ] React Hook Form + Zod for form validation
- [ ] react-native-mmkv for local storage
- [ ] Reanimated v3 for animations
- [ ] FlashList for lists
- [ ] EAS Build for CI/CD

### Architecture Setup
- [ ] Feature-based file structure (not type-based)
- [ ] Service layer for API calls (separate from components)
- [ ] Separate server state (TanStack Query) from client state (Zustand)
- [ ] Error boundaries at feature boundaries
- [ ] TypeScript strict mode enabled

### Performance Baseline
- [ ] Profile cold start on low-end device (<2s target)
- [ ] Flashlight benchmark for list screens
- [ ] Bundle analysis with Expo Atlas
- [ ] Crash monitoring (Crashlytics or Sentry)

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [Performance & Rendering](../optimization/performance-rendering.md) | Optimization patterns for animations, lists, memoization, and React Query adoption |
| [Profiling Tools Deep Dive](../debugging/profiling-tools-deep-dive.md) | When and how to profile the patterns described here |
| [Dependency Management](../optimization/dependency-management.md) | Evaluating and managing the libraries in this stack |
| [New Architecture Deep Dive](../optimization/new-architecture-migration.md) | The New Architecture that enables many of these modern patterns |
| [Expo EAS Complete Guide](../expo/eas-complete-guide.md) | Build and deploy pipeline for Expo projects |
| [Expo App Config Decision Guide](../expo/app-config-decision-guide.md) | Configuration reference for the Expo features discussed here |

---

Sources:
- [Expo: Best Practices for Reducing Lag](https://expo.dev/blog/best-practices-for-reducing-lag-in-expo-apps)
- [Expo: Offline-First Apps with Legend State](https://expo.dev/blog/offline-first-apps-with-expo-and-legend-state)
- [Expo: From Web to Native with React](https://expo.dev/blog/from-web-to-native-with-react)
- [Expo Router Documentation](https://docs.expo.dev/router/introduction/)
- [Callstack: Why Your Bundle Size Matters and How Expo Atlas Helps](https://www.callstack.com/blog/knowing-your-apps-bundle-contents-native-performance)
- [Callstack: React Native Wrapped 2025](https://www.callstack.com/blog/react-native-wrapped-2025-a-month-by-month-recap-of-the-year)
- [K-Optional: Maximizing Performance in React Native + Expo](https://koptional.com/resource/optimizing-react-native-expo/)
- [Oflight: React Native State Management Guide 2026](https://www.oflight.co.jp/en/columns/react-native-state-management-2026)
- [tRPC: New TanStack React Query Integration](https://trpc.io/blog/introducing-tanstack-react-query-client)
- [React Navigation 7 vs Expo Router](https://viewlytics.ai/blog/react-navigation-7-vs-expo-router)
- [Sentry: RN Performance Tactics](https://blog.sentry.io/react-native-performance-strategies-tools/)
- [Bergue: How I Reduced My RN Bundle Size by 70%](https://medium.com/@bergueeg/how-i-reduced-my-react-native-app-bundle-size-by-70-7505e67d0373)
