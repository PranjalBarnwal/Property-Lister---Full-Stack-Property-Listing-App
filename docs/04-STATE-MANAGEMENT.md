# State Management with Zustand

## 🏪 What is State Management?

**State** = Data that changes over time and affects what users see.

Examples in this app:
- Is user an admin? (affects which tabs show)
- What filters are applied? (affects search results)
- Is property saved? (affects heart icon)

---

## 🎯 State Management Strategy

This app uses a **hybrid approach**:

| Type | Tool | Use Case | Example |
|------|------|----------|---------|
| **Local State** | `useState` | Component-specific | Loading spinners, form inputs |
| **Global State** | `Zustand` | App-wide | User admin status, filters |
| **Server State** | Direct fetch | Database data | Properties, saved items |

---

## 📦 Zustand Overview

**Zustand** = Lightweight state management library.

### Why Zustand?
- ✅ Simple API (no boilerplate)
- ✅ No providers needed
- ✅ TypeScript support
- ✅ Tiny bundle size (1KB)
- ✅ Works with React hooks

### Comparison:

```typescript
// Redux (verbose)
const mapStateToProps = (state) => ({ count: state.count });
const mapDispatchToProps = (dispatch) => ({ increment: () => dispatch(incrementAction()) });
connect(mapStateToProps, mapDispatchToProps)(Component);

// Zustand (simple)
const count = useCountStore(state => state.count);
const increment = useCountStore(state => state.increment);
```

---

## 🏗️ Store Structure

Your app has **2 Zustand stores**:

### 1. User Store
**File:** `store/userStore.ts`

Stores: Admin status

### 2. Filter Store
**File:** `store/filterStore.ts`

Stores: Search filters (search text, type, bedrooms, price range)

---

## 👤 User Store Deep Dive

**File:** `store/userStore.ts`

```typescript
import { create } from "zustand";

interface UserStore {
  isAdmin: boolean;
  setIsAdmin: (value: boolean) => void;
}

export const useUserStore = create<UserStore>((set) => ({
  isAdmin: false,
  setIsAdmin: (value) => set({ isAdmin: value }),
}));
```

### Breakdown:

#### 1. Interface Definition
```typescript
interface UserStore {
  isAdmin: boolean;              // State
  setIsAdmin: (value: boolean) => void;  // Action
}
```

#### 2. Create Store
```typescript
create<UserStore>((set) => ({
  // Initial state
  isAdmin: false,
  
  // Action to update state
  setIsAdmin: (value) => set({ isAdmin: value }),
}))
```

**`set` function** - Updates store state

#### 3. Usage in Components:

```typescript
import { useUserStore } from "@/store/userStore";

function MyComponent() {
  // Read state
  const isAdmin = useUserStore((state) => state.isAdmin);
  
  // Get action
  const setIsAdmin = useUserStore((state) => state.setIsAdmin);
  
  // Update state
  const makeAdmin = () => {
    setIsAdmin(true);
  };
}
```

### Real Example: Tab Bar

**File:** `app/(root)/(tabs)/_layout.tsx`

```typescript
export default function TabsLayout() {
  const isAdmin = useUserStore((state) => state.isAdmin);
  
  return (
    <Tabs>
      {/* ... other tabs ... */}
      
      {isAdmin ? (
        <Tabs.Screen name="create" options={{ title: "Add Property" }} />
      ) : (
        <Tabs.Screen name="create" options={{ href: null }} />
      )}
    </Tabs>
  );
}
```

**Flow:**
```
User logs in
    ↓
useUserSync() runs
    ↓
Query Supabase for is_admin flag
    ↓
setIsAdmin(data.is_admin)
    ↓
Tab bar re-renders
    ↓
Admin tab shows/hides
```

---

## 🔍 Filter Store Deep Dive

**File:** `store/filterStore.ts`

```typescript
import { create } from "zustand";

export type PropertyType = "apartment" | "house" | "villa" | "studio" | null;

interface FilterState {
  // State
  search: string;
  type: PropertyType;
  bedrooms: number | null;
  minPrice: number | null;
  maxPrice: number | null;

  // Actions
  setSearch: (value: string) => void;
  setType: (value: PropertyType) => void;
  setBedrooms: (value: number | null) => void;
  setMinPrice: (value: number | null) => void;
  setMaxPrice: (value: number | null) => void;
  resetFilters: () => void;
}

export const useFilterStore = create<FilterState>((set) => ({
  // Initial state
  search: "",
  type: null,
  bedrooms: null,
  minPrice: null,
  maxPrice: null,

  // Actions
  setSearch: (value) => set({ search: value }),
  setType: (value) => set({ type: value }),
  setBedrooms: (value) => set({ bedrooms: value }),
  setMinPrice: (value) => set({ minPrice: value }),
  setMaxPrice: (value) => set({ maxPrice: value }),
  
  // Reset all filters
  resetFilters: () =>
    set({
      search: "",
      type: null,
      bedrooms: null,
      minPrice: null,
      maxPrice: null,
    }),
}));
```

### Usage Example: Search Screen

**File:** `app/(root)/(tabs)/search.tsx`

```typescript
import { useFilterStore } from "@/store/filterStore";

export default function SearchScreen() {
  const {
    search,
    type,
    bedrooms,
    minPrice,
    maxPrice,
    setSearch,
    setType,
    setBedrooms,
    setMinPrice,
    setMaxPrice,
  } = useFilterStore();

  // Fetch results when filters change
  useEffect(() => {
    fetchResults();
  }, [search, type, bedrooms, minPrice, maxPrice]);

  const fetchResults = async () => {
    let query = supabase.from("properties").select("*");

    if (search) {
      query = query.or(`title.ilike.%${search}%,city.ilike.%${search}%`);
    }

    if (type) {
      query = query.eq("type", type);
    }

    if (bedrooms) {
      query = query.eq("bedrooms", bedrooms);
    }

    if (minPrice) {
      query = query.gte("price", minPrice);
    }

    if (maxPrice) {
      query = query.lte("price", maxPrice);
    }

    const { data } = await query.order("created_at", { ascending: false });
    setResults(data ?? []);
  };

  return (
    <View>
      {/* Search input */}
      <TextInput
        value={search}
        onChangeText={setSearch}
        placeholder="Search properties..."
      />
      
      {/* Type filter chips */}
      <TouchableOpacity onPress={() => setType("apartment")}>
        <Text>Apartment</Text>
      </TouchableOpacity>
    </View>
  );
}
```

### Why Use Store for Filters?

**Problem:** User applies filters → navigates to property details → goes back → **filters are lost**

**Solution:** Store filters in Zustand → **filters persist across navigation**

---

## 🔄 Store Update Patterns

### Pattern 1: Simple Update

```typescript
const setIsAdmin = useUserStore(state => state.setIsAdmin);

// Update
setIsAdmin(true);
```

### Pattern 2: Partial Update

```typescript
// Store with multiple fields
const useUserStore = create((set) => ({
  name: "",
  email: "",
  age: 0,
  
  updateUser: (updates) => set((state) => ({ ...state, ...updates })),
}));

// Usage
updateUser({ name: "John", age: 30 });  // Only updates name and age
```

### Pattern 3: Computed Update

```typescript
const useCountStore = create((set) => ({
  count: 0,
  
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
}));
```

### Pattern 4: Reset to Initial

```typescript
const initialState = {
  search: "",
  type: null,
  bedrooms: null,
};

const useFilterStore = create((set) => ({
  ...initialState,
  
  resetFilters: () => set(initialState),
}));
```

---

## 📖 Reading Store State

### Method 1: Select Specific Field (Recommended)

```typescript
const isAdmin = useUserStore(state => state.isAdmin);
```

**Pros:**
- ✅ Component only re-renders when `isAdmin` changes
- ✅ Better performance

### Method 2: Select Multiple Fields

```typescript
const { search, type, bedrooms } = useFilterStore(state => ({
  search: state.search,
  type: state.type,
  bedrooms: state.bedrooms,
}));
```

**Pros:**
- ✅ Still performant
- ✅ Only re-renders when selected fields change

### Method 3: Select Entire Store (Not Recommended)

```typescript
const filterStore = useFilterStore();
```

**Cons:**
- ❌ Component re-renders on **any** store change
- ❌ Performance issues in large apps

---

## 🎨 Advanced Patterns

### Pattern 1: Derived State (Selectors)

```typescript
const useFilterStore = create((set, get) => ({
  search: "",
  type: null,
  bedrooms: null,
  minPrice: null,
  maxPrice: null,
  
  // ... setters ...
  
  // Derived state: Count active filters
  getActiveFilterCount: () => {
    const state = get();
    return [
      state.type !== null,
      state.bedrooms !== null,
      state.minPrice !== null,
      state.maxPrice !== null,
    ].filter(Boolean).length;
  },
}));

// Usage
const activeCount = useFilterStore(state => state.getActiveFilterCount());
```

**Use case:** Calculate values based on store state without storing redundant data.

### Pattern 2: Async Actions

```typescript
const useUserStore = create((set) => ({
  user: null,
  loading: false,
  
  fetchUser: async (userId) => {
    set({ loading: true });
    
    const { data } = await supabase
      .from("users")
      .select("*")
      .eq("id", userId)
      .single();
    
    set({ user: data, loading: false });
  },
}));

// Usage
const fetchUser = useUserStore(state => state.fetchUser);

useEffect(() => {
  fetchUser("user_123");
}, []);
```

### Pattern 3: Middleware (Logging)

```typescript
const log = (config) => (set, get, api) =>
  config(
    (...args) => {
      console.log("Before:", get());
      set(...args);
      console.log("After:", get());
    },
    get,
    api
  );

const useCountStore = create(
  log((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
  }))
);
```

**Output:**
```
Before: { count: 0 }
After: { count: 1 }
```

### Pattern 4: Persist to Storage

```typescript
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";
import AsyncStorage from "@react-native-async-storage/async-storage";

const useFilterStore = create(
  persist(
    (set) => ({
      search: "",
      type: null,
      setSearch: (value) => set({ search: value }),
      setType: (value) => set({ type: value }),
    }),
    {
      name: "filter-storage",
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

**Result:** Filters persist across app restarts!

---

## 🆚 Local State vs. Global State

### When to Use Local State (useState):

```typescript
// ✅ Form inputs
const [email, setEmail] = useState("");

// ✅ Loading states
const [loading, setLoading] = useState(false);

// ✅ Modal visibility
const [showModal, setShowModal] = useState(false);

// ✅ Component-specific data
const [expanded, setExpanded] = useState(false);
```

**Rule:** If only **one component** needs it → Use local state

### When to Use Global State (Zustand):

```typescript
// ✅ User authentication status
const isAdmin = useUserStore(state => state.isAdmin);

// ✅ App-wide filters
const { search, type } = useFilterStore();

// ✅ Theme preference
const theme = useThemeStore(state => state.theme);

// ✅ Shopping cart
const cart = useCartStore(state => state.items);
```

**Rule:** If **multiple components** need it → Use global state

---

## 🎭 Real-World Example: Filter Modal

**File:** `components/FilterModal.tsx`

```typescript
export default function FilterModal({ visible, onClose }) {
  const {
    type,
    bedrooms,
    minPrice,
    maxPrice,
    setType,
    setBedrooms,
    setMinPrice,
    setMaxPrice,
    resetFilters,
  } = useFilterStore();

  const [localMin, setLocalMin] = useState(minPrice ? String(minPrice) : "");
  const [localMax, setLocalMax] = useState(maxPrice ? String(maxPrice) : "");

  const handleApply = () => {
    setMinPrice(localMin ? Number(localMin) : null);
    setMaxPrice(localMax ? Number(localMax) : null);
    onClose();
  };

  const handleReset = () => {
    setLocalMin("");
    setLocalMax("");
    resetFilters();
    onClose();
  };

  return (
    <Modal visible={visible}>
      {/* Type chips */}
      <TouchableOpacity onPress={() => setType("apartment")}>
        <Text>Apartment</Text>
      </TouchableOpacity>
      
      {/* Bedroom selector */}
      <TouchableOpacity onPress={() => setBedrooms(2)}>
        <Text>2 Bedrooms</Text>
      </TouchableOpacity>
      
      {/* Price inputs */}
      <TextInput value={localMin} onChangeText={setLocalMin} />
      <TextInput value={localMax} onChangeText={setLocalMax} />
      
      {/* Actions */}
      <Button onPress={handleApply}>Apply</Button>
      <Button onPress={handleReset}>Reset</Button>
    </Modal>
  );
}
```

### State Strategy:

| Data | Storage | Reason |
|------|---------|--------|
| `type`, `bedrooms` | **Global (Zustand)** | Immediately update search results |
| `minPrice`, `maxPrice` | **Local (useState)** | Only update when user taps "Apply" |

**Why?**
- Type/bedrooms → User expects instant results
- Price → User wants to adjust range before applying

---

## 🧪 Testing Zustand Stores

### Test 1: Initial State

```typescript
import { useUserStore } from "@/store/userStore";

// Get initial state without React
const initialState = useUserStore.getState();

expect(initialState.isAdmin).toBe(false);
```

### Test 2: Actions

```typescript
const { setIsAdmin } = useUserStore.getState();

setIsAdmin(true);

const updatedState = useUserStore.getState();
expect(updatedState.isAdmin).toBe(true);
```

### Test 3: Reset Store (For Tests)

```typescript
beforeEach(() => {
  useFilterStore.setState({
    search: "",
    type: null,
    bedrooms: null,
    minPrice: null,
    maxPrice: null,
  });
});
```

---

## 🎯 Best Practices

### 1. Keep Stores Focused

```typescript
// ❌ Don't: Single store for everything
const useAppStore = create((set) => ({
  user: null,
  filters: {},
  cart: [],
  theme: "light",
  // ... 50 more fields
}));

// ✅ Do: Separate stores by domain
useUserStore    // User-related
useFilterStore  // Filter-related
useCartStore    // Cart-related
useThemeStore   // Theme-related
```

### 2. Use Descriptive Action Names

```typescript
// ❌ Don't
set(true);

// ✅ Do
setIsAdmin(true);
```

### 3. Type Your Stores

```typescript
// ✅ Always define interface
interface UserStore {
  isAdmin: boolean;
  setIsAdmin: (value: boolean) => void;
}

// ✅ Use it in create
create<UserStore>((set) => ({ ... }))
```

### 4. Avoid Nested Objects in Zustand

```typescript
// ❌ Don't
const useUserStore = create((set) => ({
  user: {
    profile: {
      name: "",
      age: 0,
    },
  },
  setName: (name) => set((state) => ({
    user: {
      ...state.user,
      profile: {
        ...state.user.profile,
        name,
      },
    },
  })),
}));

// ✅ Do: Flatten structure
const useUserStore = create((set) => ({
  profileName: "",
  profileAge: 0,
  setProfileName: (name) => set({ profileName: name }),
}));
```

### 5. Don't Store Server Data

```typescript
// ❌ Don't store properties in Zustand
const usePropertyStore = create((set) => ({
  properties: [],
  setProperties: (props) => set({ properties: props }),
}));

// ✅ Do: Fetch directly and use local state
const [properties, setProperties] = useState([]);

useEffect(() => {
  const fetchProperties = async () => {
    const { data } = await supabase.from("properties").select("*");
    setProperties(data ?? []);
  };
  fetchProperties();
}, []);
```

**Why?** Server data can become stale. Fetch fresh data instead.

---

## 🐛 Common Mistakes

### Mistake 1: Selecting Entire Store

```typescript
// ❌ Component re-renders on any store change
const filterStore = useFilterStore();

// ✅ Component only re-renders when search changes
const search = useFilterStore(state => state.search);
```

### Mistake 2: Mutating State Directly

```typescript
// ❌ Don't mutate
const setFilters = useFilterStore(state => state.filters);
setFilters.search = "new value";  // Won't trigger re-render!

// ✅ Use action
const setSearch = useFilterStore(state => state.setSearch);
setSearch("new value");
```

### Mistake 3: Not Resetting State

```typescript
// ❌ Filters persist when user logs out
// Next user sees previous user's filters!

// ✅ Reset on logout
const handleLogout = async () => {
  await signOut();
  useFilterStore.getState().resetFilters();
  router.replace("/sign-in");
};
```

---

## 🚀 Performance Tips

### Tip 1: Use Shallow Comparison

```typescript
import { shallow } from "zustand/shallow";

// Re-renders only when search OR type changes
const { search, type } = useFilterStore(
  (state) => ({ search: state.search, type: state.type }),
  shallow
);
```

### Tip 2: Split Large Stores

```typescript
// ❌ One large store
const useLargeStore = create(() => ({
  // 100 fields...
}));

// ✅ Multiple small stores
const useUserStore = create(() => ({ ... }));
const useFilterStore = create(() => ({ ... }));
const useCartStore = create(() => ({ ... }));
```

### Tip 3: Memoize Selectors

```typescript
const selector = useCallback(
  (state) => state.items.filter(item => item.active),
  []
);

const activeItems = useStore(selector);
```

---

**Next:** Read `05-COMPONENTS.md` to understand reusable UI components!
