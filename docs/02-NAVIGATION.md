# Navigation & Routing Deep Dive

## 🗺️ Expo Router Overview

This app uses **Expo Router** - a file-based routing system where your folder structure defines your navigation.

Think of it like Next.js for React Native!

---

## 📁 File-Based Routing

### The Concept:
```
app/
├── index.tsx           →  /
├── about.tsx           →  /about
├── user/
│   ├── profile.tsx     →  /user/profile
│   └── [id].tsx        →  /user/123 (dynamic)
```

Files become routes automatically. No need to manually configure navigation!

---

## 🏗️ Your App's Route Structure

```
app/
├── _layout.tsx                    # Root layout (ClerkProvider)
├── index.tsx                      # Entry point → /
│
├── (auth)/                        # Auth routes (no auth required)
│   ├── _layout.tsx                # Auth layout wrapper
│   ├── sign-in.tsx                # /sign-in
│   └── sign-up.tsx                # /sign-up
│
└── (root)/                        # Protected routes (auth required)
    ├── _layout.tsx                # Auth guard wrapper
    │
    ├── (tabs)/                    # Bottom tab navigation
    │   ├── _layout.tsx            # Tab bar configuration
    │   ├── index.tsx              # /(root)/(tabs) or just /
    │   ├── search.tsx             # /(root)/(tabs)/search
    │   ├── create.tsx             # /(root)/(tabs)/create
    │   ├── saved.tsx              # /(root)/(tabs)/saved
    │   └── profile.tsx            # /(root)/(tabs)/profile
    │
    └── property/                  # Property detail routes
        ├── [id].tsx               # /(root)/property/123
        └── map.tsx                # /(root)/property/map
```

---

## 🎭 Route Groups: `(auth)` and `(root)`

### What are route groups?

Folders wrapped in **parentheses** `()` are **route groups**.

**Purpose:**
- Organize routes without affecting URLs
- Share layouts between routes
- Keep folder structure clean

### Example:

```
app/
├── (auth)/
│   ├── sign-in.tsx     →  /sign-in (NOT /auth/sign-in)
│   └── sign-up.tsx     →  /sign-up (NOT /auth/sign-up)
```

The `(auth)` folder **doesn't appear in the URL**!

### Why use route groups?

```typescript
// app/(auth)/_layout.tsx
export default function AuthLayout() {
  // This layout applies to ALL routes inside (auth)/
  // - sign-in.tsx
  // - sign-up.tsx
  
  return <Stack screenOptions={{ headerShown: false }} />;
}
```

All auth screens share the same layout without URL pollution.

---

## 📐 Layouts: `_layout.tsx`

Layouts are **wrapper components** that apply to child routes.

### Types of Layouts:

1. **Root Layout** - Wraps entire app
2. **Group Layout** - Wraps routes in a folder
3. **Route Layout** - Wraps a specific route

### Layout Hierarchy:

```
app/_layout.tsx                    ← Root (ClerkProvider)
    │
    ├── app/(auth)/_layout.tsx     ← Auth screens
    │   ├── sign-in.tsx
    │   └── sign-up.tsx
    │
    └── app/(root)/_layout.tsx     ← Protected routes
        │
        └── app/(root)/(tabs)/_layout.tsx  ← Tab bar
            ├── index.tsx
            ├── search.tsx
            ├── create.tsx
            ├── saved.tsx
            └── profile.tsx
```

Each `_layout.tsx` wraps its children!

---

## 🎨 Layout 1: Root Layout

**File:** `app/_layout.tsx`

```typescript
import { ClerkProvider } from "@clerk/expo";
import { tokenCache } from "@clerk/expo/token-cache";
import { Slot } from "expo-router";
import "../global.css";

const publishableKey = process.env.EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY!;

export default function RootLayout() {
  return (
    <ClerkProvider publishableKey={publishableKey} tokenCache={tokenCache}>
      <Slot />
    </ClerkProvider>
  );
}
```

### Key Concepts:

- **`<Slot />`** - Renders child routes
- **ClerkProvider** - Makes auth available to all routes
- **global.css** - NativeWind/Tailwind styles

This layout wraps **every single screen** in your app.

---

## 🔒 Layout 2: Auth Layout

**File:** `app/(auth)/_layout.tsx`

```typescript
import { useAuth } from "@clerk/expo";
import { Redirect, Stack } from "expo-router";

export default function AuthLayout() {
  const { isSignedIn, isLoaded } = useAuth();

  if (!isLoaded) return null;
  if (isSignedIn) return <Redirect href="/" />;

  return <Stack screenOptions={{ headerShown: false }} />;
}
```

### What's Happening?

1. **Check auth status**
2. **If already signed in** → Redirect to home (prevent accessing login when logged in)
3. **If not signed in** → Render Stack navigation

### Stack Navigation:

```typescript
<Stack screenOptions={{ headerShown: false }} />
```

- **Stack** = Screen stack (like cards stacked on top of each other)
- **screenOptions** = Apply to all child screens
- **headerShown: false** = Hide default header

**Child screens:**
- `sign-in.tsx`
- `sign-up.tsx`

---

## 🛡️ Layout 3: Root Layout (Protected)

**File:** `app/(root)/_layout.tsx`

```typescript
import { useAuth } from "@clerk/expo";
import { Redirect, Slot } from "expo-router";
import { useUserSync } from "@/hooks/useUserSync";

export default function RootLayout() {
  const { isSignedIn, isLoaded } = useAuth();

  useUserSync();  // Sync user to Supabase

  if (!isLoaded) return null;
  if (!isSignedIn) return <Redirect href="/sign-in" />;

  return <Slot />;
}
```

### This is the **Auth Guard**!

**Protection Logic:**
```
User tries to access protected route
    ↓
Is auth loaded? NO → Show nothing (loading)
    ↓
Is signed in? NO → Redirect to /sign-in
    ↓
YES → Render child routes
```

**All routes inside `(root)/` are protected:**
- Home
- Search
- Create (admin only)
- Saved
- Profile
- Property details

---

## 📱 Layout 4: Tab Bar Layout

**File:** `app/(root)/(tabs)/_layout.tsx`

```typescript
import { useUserStore } from "@/store/userStore";
import { Ionicons } from "@expo/vector-icons";
import { Tabs } from "expo-router";
import { Platform } from "react-native";

export default function TabsLayout() {
  const isAdmin = useUserStore((state) => state.isAdmin);

  return (
    <Tabs
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: "#2563EB",
        tabBarInactiveTintColor: "#9CA3AF",
        tabBarStyle: {
          backgroundColor: "#FFFFFF",
          borderTopWidth: 1,
          borderTopColor: "#E5E7EB",
          height: Platform.OS === "ios" ? 85 : 65,
          paddingBottom: Platform.OS === "ios" ? 25 : 8,
          paddingTop: 8,
        },
        tabBarLabelStyle: {
          fontSize: 12,
          fontWeight: "600",
        },
      }}
    >
      <Tabs.Screen
        name="index"
        options={{
          title: "Home",
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />

      <Tabs.Screen
        name="search"
        options={{
          title: "Search",
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search" size={size} color={color} />
          ),
        }}
      />

      {isAdmin ? (
        <Tabs.Screen
          name="create"
          options={{
            title: "Add Property",
            tabBarIcon: ({ color, size }) => (
              <Ionicons name="add-circle" size={size} color={color} />
            ),
          }}
        />
      ) : (
        <Tabs.Screen
          name="create"
          options={{
            href: null,  // Hide from tabs
          }}
        />
      )}

      <Tabs.Screen
        name="saved"
        options={{
          title: "Saved",
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="heart" size={size} color={color} />
          ),
        }}
      />

      <Tabs.Screen
        name="profile"
        options={{
          title: "Profile",
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

### Key Features:

#### 1. Tab Bar Configuration:

```typescript
screenOptions={{
  headerShown: false,              // Hide screen headers
  tabBarActiveTintColor: "#2563EB", // Active tab color (blue)
  tabBarInactiveTintColor: "#9CA3AF", // Inactive tab color (gray)
  tabBarStyle: {
    height: Platform.OS === "ios" ? 85 : 65,  // iOS needs more space for safe area
  },
}}
```

#### 2. Individual Tab Configuration:

```typescript
<Tabs.Screen
  name="index"           // File name (index.tsx)
  options={{
    title: "Home",       // Label shown in tab bar
    tabBarIcon: ({ color, size }) => (
      <Ionicons name="home" size={size} color={color} />
    ),
  }}
/>
```

#### 3. Conditional Tab (Admin Only):

```typescript
{isAdmin ? (
  <Tabs.Screen name="create" options={{...}} />
) : (
  <Tabs.Screen name="create" options={{ href: null }} />
)}
```

`href: null` **hides the tab** but keeps the route accessible via code.

---

## 🚀 Navigation Methods

### 1. Router Hook

```typescript
import { useRouter } from "expo-router";

function MyComponent() {
  const router = useRouter();

  // Navigate forward
  router.push("/property/123");

  // Navigate and replace current screen
  router.replace("/sign-in");

  // Go back
  router.back();

  // Navigate to specific route
  router.navigate("/(root)/(tabs)/search");
}
```

### Methods:

| Method | Description | Example |
|--------|-------------|---------|
| `push()` | Navigate forward (can go back) | Browse property details |
| `replace()` | Replace current screen | After login/logout |
| `back()` | Go to previous screen | Back button |
| `navigate()` | Go to specific route | Deep linking |

### 2. Link Component

```typescript
import { Link } from "expo-router";

<Link href="/sign-up">
  <Text>Sign Up</Text>
</Link>
```

**Use when:**
- Static navigation
- No logic needed
- Simple tap-to-navigate

### 3. Redirect Component

```typescript
import { Redirect } from "expo-router";

if (!isSignedIn) {
  return <Redirect href="/sign-in" />;
}
```

**Use when:**
- Conditional redirects
- Auth guards
- Permission checks

---

## 🎯 Dynamic Routes: `[id].tsx`

**File:** `app/(root)/property/[id].tsx`

### What is `[id]`?

Square brackets `[]` create **dynamic routes** that match any value.

### Examples:

```
/property/123       →  [id].tsx  (id = "123")
/property/abc-def   →  [id].tsx  (id = "abc-def")
/property/anything  →  [id].tsx  (id = "anything")
```

### Accessing Route Parameters:

```typescript
import { useLocalSearchParams } from "expo-router";

export default function PropertyDetailScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  
  console.log(id);  // "123"
  
  // Fetch property data using id
  const fetchProperty = async () => {
    const { data } = await supabase
      .from("properties")
      .select("*")
      .eq("id", id)
      .single();
  };
}
```

### Navigating to Dynamic Routes:

```typescript
// Method 1: Template literal
router.push(`/property/${propertyId}`);

// Method 2: Using parentheses (type-safe)
router.push({
  pathname: "/(root)/property/[id]",
  params: { id: propertyId }
});
```

---

## 🔗 Query Parameters

### Passing Query Params:

```typescript
router.push({
  pathname: "/(root)/(tabs)/search",
  params: { 
    openFilters: "true",
    city: "Mumbai"
  }
});

// Results in: /search?openFilters=true&city=Mumbai
```

### Reading Query Params:

```typescript
const { openFilters, city } = useLocalSearchParams<{ 
  openFilters?: string;
  city?: string;
}>();

useEffect(() => {
  if (openFilters === "true") {
    setShowFilters(true);
  }
}, [openFilters]);
```

### Example Use Case:

**Home screen search bar:**
```typescript
<TouchableOpacity
  onPress={() =>
    router.push("/(root)/(tabs)/search?openFilters=true")
  }
>
  <Ionicons name="options-outline" />
</TouchableOpacity>
```

**Search screen:**
```typescript
const { openFilters } = useLocalSearchParams<{ openFilters?: string }>();

useEffect(() => {
  if (openFilters === "true") {
    setShowFilters(true);  // Open filter modal
  }
}, [openFilters]);
```

---

## 🧭 Navigation Flow Examples

### Example 1: User Signs Up

```
1. User opens app
   → app/index.tsx
   → Not signed in
   → Redirect to /sign-in

2. User taps "Sign Up"
   → <Link href="/sign-up">
   → Navigate to app/(auth)/sign-up.tsx

3. User completes sign-up
   → router.replace("/")
   → Navigate to app/index.tsx

4. Auth check passes
   → Redirect to /(root)/(tabs)
   → Navigate to app/(root)/(tabs)/index.tsx
   → Home screen renders
```

### Example 2: User Browses Property

```
1. User on home screen
   → app/(root)/(tabs)/index.tsx

2. User taps property card
   → router.push(`/(root)/property/${property.id}`)
   → Navigate to app/(root)/property/[id].tsx

3. User views details
   → Can tap "Back" button
   → router.back()
   → Return to home screen

4. Alternative: User taps "View on Map"
   → router.push({
       pathname: "/(root)/property/map",
       params: { latitude, longitude, title, address }
     })
   → Navigate to app/(root)/property/map.tsx
```

### Example 3: Admin Adds Property

```
1. Admin taps "Add Property" tab
   → Navigate to app/(root)/(tabs)/create.tsx

2. Admin fills form and submits
   → Property created in Supabase
   → router.replace("/(root)/(tabs)")
   → Return to home screen (can't go back to form)
```

---

## 🎨 Navigation Patterns

### Pattern 1: Back Button

```typescript
import { useRouter } from "expo-router";

<TouchableOpacity onPress={() => router.back()}>
  <Ionicons name="arrow-back" size={20} />
</TouchableOpacity>
```

### Pattern 2: Navigate After Action

```typescript
const handleSubmit = async () => {
  // Save data
  await supabase.from("properties").insert(property);
  
  // Navigate away
  router.replace("/(root)/(tabs)");
};
```

**Use `replace`** when user shouldn't go back (form submission, logout).

### Pattern 3: Deep Linking

```typescript
// From push notification or external link
router.push({
  pathname: "/(root)/property/[id]",
  params: { id: "abc123" }
});
```

### Pattern 4: Conditional Navigation

```typescript
const handleLogin = async () => {
  await signIn();
  
  if (isAdmin) {
    router.replace("/(root)/(tabs)/create");
  } else {
    router.replace("/(root)/(tabs)");
  }
};
```

---

## 📊 Navigation State Management

### Focus Detection:

```typescript
import { useFocusEffect } from "expo-router";

useFocusEffect(
  useCallback(() => {
    // This runs when screen comes into focus
    fetchProperties();
    
    return () => {
      // Cleanup when screen loses focus
    };
  }, [])
);
```

**Use cases:**
- Refresh data when user returns to screen
- Clear form when navigating away
- Pause/resume operations

### Navigation Events:

```typescript
import { useNavigation } from "expo-router";

const navigation = useNavigation();

useEffect(() => {
  const unsubscribe = navigation.addListener("focus", () => {
    console.log("Screen focused");
  });

  return unsubscribe;
}, [navigation]);
```

---

## 🔀 Navigation Flow Diagram

```
app/
├── _layout.tsx (ClerkProvider)
    │
    ├── index.tsx
    │   ├─ isSignedIn? YES → Redirect to /(root)/(tabs)
    │   └─ isSignedIn? NO  → Redirect to /sign-in
    │
    ├── (auth)/_layout.tsx (Stack)
    │   ├── sign-in.tsx
    │   └── sign-up.tsx
    │
    └── (root)/_layout.tsx (Auth Guard)
        ├── (tabs)/_layout.tsx (Tab Bar)
        │   ├── index.tsx     [Home Tab]
        │   ├── search.tsx    [Search Tab]
        │   ├── create.tsx    [Create Tab - Admin Only]
        │   ├── saved.tsx     [Saved Tab]
        │   └── profile.tsx   [Profile Tab]
        │
        └── property/
            ├── [id].tsx      (Property Details)
            └── map.tsx       (Full Screen Map)
```

---

## 🚦 Route Protection Patterns

### Pattern 1: Layout-Level Protection

```typescript
// app/(root)/_layout.tsx
if (!isSignedIn) return <Redirect href="/sign-in" />;
```

**Protects:** All routes inside `(root)/`

### Pattern 2: Screen-Level Protection

```typescript
// app/(root)/(tabs)/create.tsx
export default function CreateScreen() {
  const isAdmin = useUserStore(state => state.isAdmin);
  
  if (!isAdmin) {
    return <Redirect href="/(root)/(tabs)" />;
  }
  
  return <CreatePropertyForm />;
}
```

**Protects:** Individual screen

### Pattern 3: UI-Level Protection

```typescript
// app/(root)/(tabs)/_layout.tsx
{isAdmin ? (
  <Tabs.Screen name="create" />
) : (
  <Tabs.Screen name="create" options={{ href: null }} />
)}
```

**Hides:** Tab from UI, but route still accessible via code

---

## 🎯 Best Practices

### 1. Use `replace` for Final Navigation
```typescript
// ❌ Don't
router.push("/");  // User can go back to login

// ✅ Do
router.replace("/");  // Can't go back to login
```

### 2. Handle Loading States
```typescript
if (!isLoaded) return null;  // Wait for auth to load
```

### 3. Type Your Route Params
```typescript
const { id } = useLocalSearchParams<{ id: string }>();
```

### 4. Use Route Groups for Organization
```
(auth)/    → Login flows
(root)/    → App screens
(admin)/   → Admin-only screens
```

### 5. Refresh Data on Focus
```typescript
useFocusEffect(
  useCallback(() => {
    fetchData();
  }, [])
);
```

---

## 🐛 Common Navigation Issues

### Issue: Screen flashes during redirect
**Cause:** Not waiting for auth to load
**Fix:**
```typescript
if (!isLoaded) return null;
```

### Issue: Can go back to login after login
**Cause:** Using `push` instead of `replace`
**Fix:**
```typescript
router.replace("/");  // Not router.push("/")
```

### Issue: Tab not showing for admin
**Cause:** `isAdmin` not set in userStore
**Fix:** Check `useUserSync` is running and setting admin status

### Issue: 404 on dynamic route
**Cause:** Wrong pathname format
**Fix:**
```typescript
// ❌ Wrong
router.push("/property/" + id);

// ✅ Correct
router.push(`/(root)/property/${id}`);
```

---

**Next:** Read `03-DATA-FETCHING.md` to learn how to fetch data from Supabase!
