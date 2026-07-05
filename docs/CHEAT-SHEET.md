# Property Lister - Quick Reference Cheat Sheet

## 🚀 Common Commands

```bash
# Start development server
npm start

# Run on iOS
npm run ios

# Run on Android
npm run android

# Run on Web
npm run web

# Clear cache and restart
npx expo start -c

# Install dependencies
npm install
```

---

## 📁 Directory Structure (Quick Reference)

```
app/
├── _layout.tsx              # Root (ClerkProvider)
├── index.tsx                # Entry (auth redirect)
├── (auth)/
│   ├── sign-in.tsx          # Login
│   └── sign-up.tsx          # Register
└── (root)/
    ├── _layout.tsx          # Auth guard
    ├── (tabs)/
    │   ├── index.tsx        # Home
    │   ├── search.tsx       # Search
    │   ├── create.tsx       # Add (admin)
    │   ├── saved.tsx        # Saved
    │   └── profile.tsx      # Profile
    └── property/
        ├── [id].tsx         # Details
        └── map.tsx          # Map

components/
├── FeaturedCard.tsx         # Featured carousel item
├── PropertyCard.tsx         # List item
└── FilterModal.tsx          # Filter sheet

hooks/
├── useUserSync.ts           # Sync Clerk → Supabase
├── useSupabase.ts           # Auth Supabase client
└── useSavedProperty.ts      # Save/unsave logic

store/
├── userStore.ts             # User state (isAdmin)
└── filterStore.ts           # Search filters

lib/
├── supabase.ts              # Supabase setup
└── utils.ts                 # Helpers (formatPrice)

types/
└── index.ts                 # TypeScript types
```

---

## 🔐 Authentication Snippets

### Get Auth Status
```typescript
import { useAuth } from "@clerk/expo";

const { isSignedIn, userId } = useAuth();
```

### Get User Info
```typescript
import { useUser } from "@clerk/expo";

const { user } = useUser();
console.log(user?.firstName);
```

### Sign Out
```typescript
import { useAuth } from "@clerk/expo";

const { signOut } = useAuth();
await signOut();
```

### Check if Admin
```typescript
import { useUserStore } from "@/store/userStore";

const isAdmin = useUserStore(state => state.isAdmin);
```

---

## 🗄️ Supabase Snippets

### Public Query (No Auth)
```typescript
import { supabase } from "@/lib/supabase";

const { data, error } = await supabase
  .from("properties")
  .select("*")
  .eq("is_featured", true);
```

### Authenticated Query
```typescript
import { useSupabase } from "@/hooks/useSupabase";

const authSupabase = useSupabase();

const { data, error } = await authSupabase
  .from("saved_properties")
  .select("*")
  .eq("user_clerk_id", userId);
```

### Insert
```typescript
const { data, error } = await authSupabase
  .from("properties")
  .insert({ title: "New Property", price: 5000000 })
  .select()
  .single();
```

### Update
```typescript
const { error } = await authSupabase
  .from("properties")
  .update({ is_sold: true })
  .eq("id", propertyId);
```

### Delete
```typescript
const { error } = await authSupabase
  .from("properties")
  .delete()
  .eq("id", propertyId);
```

### Upload Image
```typescript
const { error } = await authSupabase.storage
  .from("property-images")
  .upload(filename, buffer, { contentType: "image/jpeg" });

const { data } = authSupabase.storage
  .from("property-images")
  .getPublicUrl(filename);
```

---

## 🧭 Navigation Snippets

### Navigate to Route
```typescript
import { useRouter } from "expo-router";

const router = useRouter();

router.push("/property/123");              // Go forward
router.replace("/sign-in");                // Replace current
router.back();                             // Go back
```

### Navigate with Params
```typescript
router.push({
  pathname: "/property/[id]",
  params: { id: "123" }
});
```

### Get Route Params
```typescript
import { useLocalSearchParams } from "expo-router";

const { id } = useLocalSearchParams<{ id: string }>();
```

### Link Component
```typescript
import { Link } from "expo-router";

<Link href="/sign-up">
  <Text>Sign Up</Text>
</Link>
```

---

## 🏪 Zustand Snippets

### Create Store
```typescript
import { create } from "zustand";

interface MyStore {
  value: string;
  setValue: (v: string) => void;
}

export const useMyStore = create<MyStore>((set) => ({
  value: "",
  setValue: (v) => set({ value: v }),
}));
```

### Use Store
```typescript
// Read state
const value = useMyStore(state => state.value);

// Get action
const setValue = useMyStore(state => state.setValue);

// Update state
setValue("new value");
```

### Reset Store
```typescript
const resetFilters = useFilterStore(state => state.resetFilters);
resetFilters();
```

---

## 🎨 Common UI Patterns

### Touchable Card
```typescript
<TouchableOpacity
  onPress={handlePress}
  className="bg-white rounded-2xl p-4"
  style={{
    shadowColor: "#000",
    shadowOpacity: 0.08,
    elevation: 3,
  }}
>
  <Text>Content</Text>
</TouchableOpacity>
```

### Loading State
```typescript
{loading ? (
  <ActivityIndicator size="large" color="#2563EB" />
) : (
  <FlatList data={items} />
)}
```

### Conditional Render
```typescript
{isAdmin && <AdminButton />}
```

### Badge
```typescript
<View className="bg-blue-50 px-3 py-1 rounded-full">
  <Text className="text-blue-600 text-xs font-semibold">
    Featured
  </Text>
</View>
```

### Icon with Text
```typescript
<View className="flex-row items-center gap-1">
  <Ionicons name="location-outline" size={13} color="#6B7280" />
  <Text className="text-xs text-gray-500">Mumbai</Text>
</View>
```

---

## 🎯 Common Hooks

### Refresh on Focus
```typescript
import { useFocusEffect } from "expo-router";

useFocusEffect(
  useCallback(() => {
    fetchData();
  }, [])
);
```

### Debounce Search
```typescript
useEffect(() => {
  const timer = setTimeout(() => {
    fetchResults(searchQuery);
  }, 500);
  
  return () => clearTimeout(timer);
}, [searchQuery]);
```

### Form State
```typescript
const [form, setForm] = useState({
  title: "",
  price: "",
});

const updateForm = (fields) => {
  setForm(prev => ({ ...prev, ...fields }));
};

updateForm({ title: "New Title" });
```

---

## 🔧 Utility Functions

### Format Price
```typescript
import { formatPrice } from "@/lib/utils";

formatPrice(5000000);   // "₹50L"
formatPrice(15000000);  // "₹1.5Cr"
```

### Truncate Text
```typescript
<Text numberOfLines={1}>
  {longText}
</Text>
```

---

## 🐛 Debugging Tips

### Console Logs
```typescript
console.log("Data:", data);
console.error("Error:", error);
console.warn("Warning:", warning);
```

### Check Auth State
```typescript
console.log("Signed In:", isSignedIn);
console.log("User ID:", userId);
console.log("Is Admin:", isAdmin);
```

### Check Query Results
```typescript
const { data, error } = await supabase.from("properties").select("*");
console.log("Data:", data);
console.log("Error:", error);
```

### Network Requests
Open React Native debugger and check Network tab.

---

## 🚨 Common Errors & Fixes

### "No suitable key was found to decode JWT"
**Fix:** Add Clerk JWKS URL to Supabase JWT settings

### "Row level security policy violation"
**Fix:** Add RLS policies to Supabase tables

### Images not uploading
**Fix:** Check `property-images` bucket exists and has correct permissions

### Can't navigate after action
**Fix:** Use `router.replace()` instead of `router.push()`

### State not updating
**Fix:** Use `set()` function in Zustand, don't mutate directly

---

## 📊 Database Schema

### properties
```sql
id              uuid PRIMARY KEY
title           text NOT NULL
price           integer NOT NULL
type            text NOT NULL
bedrooms        integer
bathrooms       integer
area_sqft       integer
address         text
city            text
latitude        float
longitude       float
images          text[]
is_featured     boolean DEFAULT false
is_sold         boolean DEFAULT false
created_at      timestamp DEFAULT now()
```

### users
```sql
id              uuid PRIMARY KEY
clerk_id        text UNIQUE NOT NULL
email           text NOT NULL
first_name      text
last_name       text
avatar_url      text
is_admin        boolean DEFAULT false
created_at      timestamp DEFAULT now()
```

### saved_properties
```sql
id              uuid PRIMARY KEY
user_clerk_id   text NOT NULL
property_id     uuid NOT NULL
created_at      timestamp DEFAULT now()
```

---

## 🎨 Color Palette

```typescript
Primary Blue:    #2563EB
Gray 50:         #F9FAFB
Gray 100:        #F3F4F6
Gray 200:        #E5E7EB
Gray 300:        #D1D5DB
Gray 400:        #9CA3AF
Gray 500:        #6B7280
Gray 600:        #4B5563
Gray 700:        #374151
Gray 800:        #1F2937
Gray 900:        #111827
Red:             #EF4444
Amber:           #D97706
```

---

## 🔗 Useful Links

- [Expo Docs](https://docs.expo.dev/)
- [Expo Router](https://expo.github.io/router/)
- [Supabase Docs](https://supabase.com/docs)
- [Clerk Docs](https://clerk.com/docs)
- [Zustand Docs](https://docs.pmnd.rs/zustand)
- [NativeWind Docs](https://www.nativewind.dev/)
- [React Native Docs](https://reactnative.dev/)
- [Ionicons](https://ionic.io/ionicons)

---

## 📝 SQL Snippets

### Make User Admin
```sql
UPDATE users
SET is_admin = true
WHERE email = 'admin@example.com';
```

### View All Admins
```sql
SELECT email, first_name, last_name
FROM users
WHERE is_admin = true;
```

### View Featured Properties
```sql
SELECT title, price, is_sold
FROM properties
WHERE is_featured = true
ORDER BY created_at DESC;
```

### View User's Saved Properties
```sql
SELECT p.title, p.price
FROM saved_properties sp
JOIN properties p ON sp.property_id = p.id
WHERE sp.user_clerk_id = 'user_123';
```

---

**Keep this cheat sheet handy for quick reference!** 🚀

For detailed explanations, see the full documentation in `docs/README.md`.
