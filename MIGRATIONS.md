# Migration Recipes

> Step-by-step recipes for the migrations teams actually face. Each one is designed to be done in a single PR.

[← Back to Index](./README.md)

---

## FlatList → FlashList (5 minutes)

**Why**: FlashList v2 recycles views (50% less blank area), requires no `estimatedItemSize`, and has zero native dependencies.

**Step 1**: Install
```bash
yarn add @shopify/flash-list
```

**Step 2**: Replace imports and component
```tsx
// Before
import { FlatList } from 'react-native';

<FlatList
  data={items}
  renderItem={({ item }) => <ItemCard item={item} />}
  keyExtractor={(item) => item.id}
/>

// After
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={items}
  renderItem={({ item }) => <ItemCard item={item} />}
  keyExtractor={(item) => item.id}
  getItemType={(item) => item.type}  // Enables better recycling
/>
```

**Step 3**: Memoize renderItem
```tsx
const ItemCard = React.memo(({ item }: { item: Item }) => {
  // ...
});
```

**Step 4**: Measure improvement
```bash
npx @bamlab/flashlight measure  # Before and after
```

**Gotchas**:
- FlashList v2 doesn't need `estimatedItemSize` (v1 did)
- Wrap `renderItem` callbacks in `useCallback` if passing handlers

> **See also**: [Performance: Lists](./optimization/performance-rendering.md#lists)

---

## AsyncStorage → MMKV (15 minutes)

**Why**: 30x faster reads/writes, synchronous API, optional AES-256 encryption.

**Step 1**: Install
```bash
yarn add react-native-mmkv
npx expo prebuild  # If using Expo (needs native module)
```

**Step 2**: Create storage instance
```tsx
// lib/storage.ts
import { MMKV } from 'react-native-mmkv';

export const storage = new MMKV({ id: 'app-storage' });

// For sensitive data, use encrypted instance
// Store the encryption key in expo-secure-store
export const secureStorage = new MMKV({
  id: 'secure-storage',
  encryptionKey: 'your-key-from-secure-store',
});
```

**Step 3**: Replace AsyncStorage calls
```tsx
// Before (async)
import AsyncStorage from '@react-native-async-storage/async-storage';
const value = await AsyncStorage.getItem('key');
await AsyncStorage.setItem('key', 'value');
await AsyncStorage.removeItem('key');

// After (sync!)
import { storage } from './lib/storage';
const value = storage.getString('key');
storage.set('key', 'value');
storage.delete('key');
```

**Step 4**: Create Zustand persistence adapter
```tsx
// lib/storage.ts (add to existing file)
export const zustandStorage = {
  getItem: (name: string) => storage.getString(name) ?? null,
  setItem: (name: string, value: string) => storage.set(name, value),
  removeItem: (name: string) => storage.delete(name),
};
```

**Step 5**: Remove AsyncStorage
```bash
yarn remove @react-native-async-storage/async-storage
```

**Gotchas**:
- MMKV requires a dev client build (won't work in Expo Go)
- MMKV stores are NOT included in device backups (good for security)
- `storage.getString()` returns `undefined` (not `null`) when key doesn't exist

> **See also**: [Security: Secure Storage](./quality/security.md)

---

## Redux (API data) → TanStack Query (30 minutes per endpoint)

**Why**: Eliminates action types, creators, reducer cases, saga handlers, and selectors for every API call. Auto-caching, background refetch, optimistic updates for free.

**Step 1**: Install
```bash
yarn add @tanstack/react-query
```

**Step 2**: Add QueryClientProvider
```tsx
// app/_layout.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // 5 minutes
      gcTime: 30 * 60 * 1000,       // 30 minutes
      retry: 3,
    },
  },
});

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <Stack />
    </QueryClientProvider>
  );
}
```

**Step 3**: Migrate one endpoint at a time
```tsx
// Before: Redux
// actions/userActions.ts — action types, creators
// reducers/userReducer.ts — loading, error, data states
// sagas/userSaga.ts — fetch logic
// selectors/userSelectors.ts — select from store
// TOTAL: 4 files, ~80-120 lines

// After: TanStack Query
// hooks/useUser.ts — 10 lines
export function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
  });
}

// Usage in component (identical to before)
function ProfileScreen({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useUser(userId);
  // ...
}
```

**Step 4**: Migrate mutations
```tsx
export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: UserUpdate) => api.updateUser(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['user'] });
    },
  });
}
```

**Strategy** (don't rewrite everything):
1. Use TanStack Query for **all new features**
2. Migrate existing screens **when they need changes anyway**
3. Keep Redux for **global client state** (auth, settings, theme)
4. Delete Redux slices as you migrate their screens

**Gotchas**:
- `cacheTime` was renamed to `gcTime` in TanStack Query v5
- Don't mix Redux and TanStack Query for the **same** data
- Set up React Query DevTools for debugging

> **See also**: [Modern Patterns: Data Fetching](./foundation/modern-patterns.md#data-fetching-tanstack-query--trpc)

---

## Expo Go → Development Build (20 minutes)

**Why**: Dev builds support custom native modules, push notifications, deep linking, and any library with native code.

**Step 1**: Install expo-dev-client
```bash
npx expo install expo-dev-client
```

**Step 2**: Create a development build
```bash
# Cloud build (EAS)
eas build --profile development --platform all

# Or local build
npx expo prebuild
npx expo run:ios
npx expo run:android
```

**Step 3**: Install the dev build on your device
- EAS provides a QR code / download link after build
- Install the `.apk` (Android) or use `eas device:create` (iOS)

**Step 4**: Start the development server
```bash
npx expo start --dev-client
```

**When to switch**:
- You need `react-native-mmkv`, camera, or any native module
- You need push notifications
- You need deep linking with custom URL schemes
- Expo Go's built-in modules aren't enough

**Gotchas**:
- You need to rebuild when adding new native dependencies
- JS-only changes still use fast refresh (no rebuild needed)
- iOS dev builds require an Apple Developer account

---

## CodePush → EAS Update (1 hour)

**Why**: Microsoft App Center shut down in March 2025. EAS Update is the recommended replacement.

**Step 1**: Remove CodePush
```bash
yarn remove react-native-code-push
# Remove CodePush.sync() calls from your app
# Remove CodePush wrapper HOC
```

**Step 2**: Install expo-updates
```bash
npx expo install expo-updates
```

**Step 3**: Configure eas.json
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

**Step 4**: Set runtime version policy
```json
// app.json
{
  "expo": {
    "runtimeVersion": {
      "policy": "fingerprint"  // Auto-detects native changes
    }
  }
}
```

**Step 5**: Deploy an update
```bash
eas update --branch production --message "Fix checkout bug"
```

**Step 6**: Staged rollout (optional)
```bash
# Roll out to 10% first
eas update --branch production --rollout-percentage 10

# Monitor for 24 hours, then expand
eas update --branch production --rollout-percentage 100
```

**Key differences from CodePush**:
| Feature | CodePush | EAS Update |
|---------|----------|-----------|
| Mandatory updates | Built-in | Via `expo-updates` API |
| Rollback | Automatic | `eas update:rollback` |
| Staged rollout | Percentage-based | Percentage-based |
| Binary detection | Manual version check | `@expo/fingerprint` (automatic) |
| Hermes bytecode | No | Yes (differential updates) |

> **See also**: [EAS Complete Guide](./expo/eas-complete-guide.md)

---

## Class Components → Hooks (per component)

**When**: Touching a legacy class component for any reason — convert it while you're there.

**Step 1**: Convert state
```tsx
// Before
class UserProfile extends Component {
  state = { user: null, loading: true };

  componentDidMount() {
    this.fetchUser();
  }

  fetchUser = async () => {
    const user = await api.getUser(this.props.userId);
    this.setState({ user, loading: false });
  };
}

// After
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    api.getUser(userId).then(user => {
      setUser(user);
      setLoading(false);
    });
  }, [userId]);
}
```

**Step 2**: Convert lifecycle methods
| Class Lifecycle | Hook Equivalent |
|----------------|----------------|
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate(prevProps)` | `useEffect(() => { ... }, [dep])` |
| `componentWillUnmount` | `useEffect(() => { return () => cleanup }, [])` |
| `shouldComponentUpdate` | `React.memo(Component)` |
| `getDerivedStateFromProps` | `useMemo` or compute in render |

**Step 3**: Even better — use TanStack Query
```tsx
// Best: let React Query handle the whole lifecycle
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
  });

  if (isLoading) return <Skeleton />;
  return <ProfileView user={user} />;
}
```

**Gotchas**:
- Error boundaries must remain as class components (no hook equivalent yet)
- Convert one component at a time — don't batch
- Run tests after each conversion

---

## React Navigation → Expo Router (project-wide)

**When**: You're already on Expo and want file-based routing with automatic deep linking.

**Step 1**: Install
```bash
npx expo install expo-router expo-linking expo-constants
```

**Step 2**: Create `app/` directory structure from existing screens
```
// Before (React Navigation)
screens/
├── HomeScreen.tsx
├── SettingsScreen.tsx
└── ProfileScreen.tsx

// After (Expo Router)
app/
├── _layout.tsx          // Tab navigator
├── index.tsx            // HomeScreen → /
├── settings.tsx         // SettingsScreen → /settings
└── profile.tsx          // ProfileScreen → /profile
```

**Step 3**: Convert navigator to layout
```tsx
// Before: React Navigation
const Tab = createBottomTabNavigator();
<Tab.Navigator>
  <Tab.Screen name="Home" component={HomeScreen} />
  <Tab.Screen name="Settings" component={SettingsScreen} />
</Tab.Navigator>

// After: app/_layout.tsx
import { Tabs } from 'expo-router';
export default function Layout() {
  return (
    <Tabs>
      <Tabs.Screen name="index" options={{ title: 'Home' }} />
      <Tabs.Screen name="settings" options={{ title: 'Settings' }} />
    </Tabs>
  );
}
```

**Step 4**: Replace navigation calls
```tsx
// Before
navigation.navigate('Profile', { userId: '123' });

// After
import { router } from 'expo-router';
router.push({ pathname: '/profile', params: { userId: '123' } });
```

**Step 5**: Remove React Navigation (if fully migrated)
```bash
yarn remove @react-navigation/native @react-navigation/bottom-tabs @react-navigation/stack
```

**Gotchas**:
- Expo Router v4: `router.navigate()` = `router.push()` (changed from v3)
- Deep linking is free — every file-based route is a URL
- Use `(group)` folders for route groups that don't affect the URL path

> **See also**: [Modern Patterns: Navigation](./foundation/modern-patterns.md#navigation-expo-router-vs-react-navigation)

---

*Each recipe is designed to be one PR. Don't batch multiple migrations — ship them separately for easier rollback.*
