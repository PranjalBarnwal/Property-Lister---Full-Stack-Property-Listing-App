# Architecture Diagram

## 🏗️ System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mobile App (React Native + Expo)         │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │  iOS App     │  │ Android App  │  │   Web App    │        │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │
│         └──────────────────┴──────────────────┘                │
│                           │                                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │      Expo Router (Navigation)         │
        │  File-based routing, layouts, guards  │
        └───────────────┬───────────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │                               │
        ▼                               ▼
┌───────────────┐              ┌────────────────┐
│  Clerk Auth   │◄────────────►│  Zustand Store │
│  (External)   │              │  (Local State) │
└───────┬───────┘              └────────────────┘
        │
        │ JWT Token
        │
        ▼
┌──────────────────────────────────────────────┐
│           Supabase Backend                   │
│                                              │
│  ┌─────────────┐      ┌──────────────────┐  │
│  │ PostgreSQL  │      │  Supabase Storage│  │
│  │  Database   │      │  (Image Files)   │  │
│  │             │      │                  │  │
│  │ - users     │      │ - property-      │  │
│  │ - properties│      │   images/        │  │
│  │ - saved_    │      │                  │  │
│  │   properties│      │                  │  │
│  └─────────────┘      └──────────────────┘  │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │  Row Level Security (RLS)            │   │
│  │  - JWT validation                    │   │
│  │  - User-specific access control      │   │
│  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

---

## 🔄 Complete User Journey Flow

### New User Sign-Up Flow

```
┌─────────────┐
│ User Opens  │
│    App      │
└──────┬──────┘
       │
       ▼
┌──────────────┐
│ app/index.tsx│ ◄── Entry point
│ Check Auth   │
└──────┬───────┘
       │
       │ Not Signed In
       ▼
┌─────────────────┐
│  /sign-up       │
│  Sign Up Screen │
└────────┬────────┘
         │
         │ 1. Enter details
         │ 2. Submit to Clerk
         ▼
┌──────────────────┐
│ Clerk Service    │
│ - Create user    │
│ - Send OTP email │
└────────┬─────────┘
         │
         │ 3. User enters OTP
         ▼
┌──────────────────┐
│ Clerk Verifies   │
│ - Email verified │
│ - Session created│
│ - JWT generated  │
└────────┬─────────┘
         │
         │ 4. Redirect to app
         ▼
┌──────────────────┐
│ /(root)/_layout  │ ◄── Auth guard
│ - Check auth     │
│ - Run useUserSync│
└────────┬─────────┘
         │
         │ 5. Sync user to Supabase
         ▼
┌────────────────────────────────┐
│ useUserSync()                  │
│                                │
│ 1. Query Supabase:             │
│    SELECT * FROM users         │
│    WHERE clerk_id = ?          │
│                                │
│ 2. User not found?             │
│    INSERT INTO users           │
│    (clerk_id, email, ...)      │
│                                │
│ 3. Update Zustand:             │
│    setIsAdmin(user.is_admin)   │
└────────┬───────────────────────┘
         │
         │ 6. Ready!
         ▼
┌─────────────────┐
│ Home Screen     │
│ Browse listings │
└─────────────────┘
```

---

## 🔐 Authentication Flow Detailed

```
┌──────────────────────────────────────────────────────┐
│                   Authentication Layer                │
│                                                       │
│  ┌────────────┐              ┌──────────────┐       │
│  │   Login    │─────────────►│ Clerk Auth   │       │
│  │   Screen   │              │  Service     │       │
│  └────────────┘              └──────┬───────┘       │
│                                     │               │
│                                     │ Creates JWT   │
│                                     │               │
│                                     ▼               │
│                          ┌──────────────────┐      │
│                          │  JWT Token       │      │
│                          │  {               │      │
│                          │   sub: "user_id" │      │
│                          │   email: "..."   │      │
│                          │   role: "auth"   │      │
│                          │  }               │      │
│                          └────────┬─────────┘      │
└──────────────────────────────────┼──────────────────┘
                                   │
                    Attached to every Supabase request
                                   │
                                   ▼
┌──────────────────────────────────────────────────────┐
│                   Supabase Backend                    │
│                                                       │
│  ┌────────────────────────────────────┐             │
│  │  JWT Validation (JWKS)             │             │
│  │  - Verify signature with Clerk key │             │
│  │  - Extract claims                  │             │
│  └────────────┬───────────────────────┘             │
│               │                                      │
│               ▼                                      │
│  ┌────────────────────────────────────┐             │
│  │  Row Level Security (RLS)          │             │
│  │                                    │             │
│  │  Query: SELECT * FROM saved_props  │             │
│  │  Policy: WHERE user_id =           │             │
│  │           auth.jwt() ->> 'sub'     │             │
│  │                                    │             │
│  │  ✓ Match → Allow                   │             │
│  │  ✗ No match → Deny                 │             │
│  └────────────────────────────────────┘             │
└───────────────────────────────────────────────────────┘
```

---

## 📊 Data Flow: Browse Properties

```
┌─────────────────┐
│   Home Screen   │
│                 │
│ useEffect(() => │
│   fetchProps()  │
│ }, [])          │
└────────┬────────┘
         │
         │ Public query (no auth needed)
         ▼
┌──────────────────────────────────┐
│  supabase                        │
│    .from("properties")           │
│    .select("*")                  │
│    .eq("is_featured", true)      │
│    .order("created_at", "desc")  │
└────────┬─────────────────────────┘
         │
         ▼
┌──────────────────────┐
│  Supabase Database   │
│                      │
│  Properties Table:   │
│  ┌────┬───────────┐  │
│  │ id │  title    │  │
│  │ 1  │ "Villa"   │  │
│  │ 2  │ "Apt"     │  │
│  └────┴───────────┘  │
│                      │
│  RLS: Public read OK │
└────────┬─────────────┘
         │
         │ Returns Property[]
         ▼
┌─────────────────────┐
│  setState(data)     │
│                     │
│ properties = [      │
│   { id: 1, ... },   │
│   { id: 2, ... }    │
│ ]                   │
└────────┬────────────┘
         │
         │ Render
         ▼
┌─────────────────────┐
│  <FlatList>         │
│    {properties.map} │
│    <PropertyCard/>  │
│  </FlatList>        │
└─────────────────────┘
```

---

## ❤️ Data Flow: Save Property

```
┌─────────────────────┐
│  PropertyCard       │
│  [Heart Icon Tap]   │
└──────────┬──────────┘
           │
           │ useSavedProperty hook
           ▼
┌──────────────────────────────────┐
│  toggleSave()                    │
│                                  │
│  1. Is property saved?           │
│     ├─ YES: DELETE FROM saved_   │
│     └─ NO:  INSERT INTO saved_   │
└──────────┬───────────────────────┘
           │
           │ Authenticated request
           │ (includes JWT)
           ▼
┌──────────────────────────────────┐
│  authSupabase                    │
│    .from("saved_properties")     │
│    .insert({                     │
│      user_clerk_id: userId,      │
│      property_id: propertyId     │
│    })                            │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  Supabase Database               │
│                                  │
│  RLS Policy Check:               │
│  WHERE user_clerk_id =           │
│        auth.jwt() ->> 'sub'      │
│                                  │
│  ✓ Match → INSERT allowed        │
│  ✗ No match → 403 Forbidden      │
└──────────┬───────────────────────┘
           │
           │ Success
           ▼
┌──────────────────────┐
│  UI Updates          │
│  - Heart fills red   │
│  - setIsSaved(true)  │
└──────────────────────┘
```

---

## 🏢 Data Flow: Admin Create Property

```
┌─────────────────────┐
│  Create Screen      │
│  (Admin Only)       │
│                     │
│ 1. Upload images    │
│ 2. Fill form        │
│ 3. Submit           │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────────────────┐
│  Upload Images to Storage        │
│                                  │
│  for each image:                 │
│    authSupabase.storage          │
│      .from("property-images")    │
│      .upload(filename, buffer)   │
│                                  │
│  Returns: Public URLs            │
└──────────┬───────────────────────┘
           │
           │ URLs collected
           ▼
┌──────────────────────────────────┐
│  Insert Property                 │
│                                  │
│  authSupabase                    │
│    .from("properties")           │
│    .insert({                     │
│      title, price, images: [...],│
│      is_featured, ...            │
│    })                            │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  Supabase RLS Check              │
│                                  │
│  Policy: Only admins can insert  │
│                                  │
│  EXISTS (                        │
│    SELECT 1 FROM users           │
│    WHERE clerk_id =              │
│      auth.jwt() ->> 'sub'        │
│    AND is_admin = true           │
│  )                               │
│                                  │
│  ✓ Admin → INSERT allowed        │
│  ✗ Not admin → 403 Forbidden     │
└──────────┬───────────────────────┘
           │
           │ Success
           ▼
┌──────────────────────┐
│  Navigate to Home    │
│  router.replace("/") │
│                      │
│  Property now visible│
│  to all users        │
└──────────────────────┘
```

---

## 🔍 Data Flow: Search with Filters

```
┌─────────────────────┐
│  Search Screen      │
│                     │
│ 1. User types query │
│ 2. Selects filters  │
└──────────┬──────────┘
           │
           │ Updates global state
           ▼
┌──────────────────────────────────┐
│  Zustand Filter Store            │
│                                  │
│  {                               │
│    search: "Mumbai",             │
│    type: "apartment",            │
│    bedrooms: 2,                  │
│    minPrice: 5000000,            │
│    maxPrice: 10000000            │
│  }                               │
└──────────┬───────────────────────┘
           │
           │ useEffect triggers fetch
           ▼
┌──────────────────────────────────┐
│  Build Query                     │
│                                  │
│  let q = supabase                │
│    .from("properties")           │
│    .select("*")                  │
│                                  │
│  if (search)                     │
│    q = q.or(`title.ilike.%...`) │
│                                  │
│  if (type)                       │
│    q = q.eq("type", type)        │
│                                  │
│  if (bedrooms)                   │
│    q = q.eq("bedrooms", bedrooms)│
│                                  │
│  if (minPrice)                   │
│    q = q.gte("price", minPrice)  │
│                                  │
│  if (maxPrice)                   │
│    q = q.lte("price", maxPrice)  │
│                                  │
│  q = q.order("created_at", desc) │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────┐
│  Execute Query       │
│                      │
│  SQL Generated:      │
│  SELECT *            │
│  FROM properties     │
│  WHERE               │
│    city ILIKE '%...' │
│    AND type = 'apt'  │
│    AND bedrooms = 2  │
│    AND price >= 5M   │
│    AND price <= 10M  │
│  ORDER BY            │
│    created_at DESC   │
└──────────┬───────────┘
           │
           │ Returns filtered results
           ▼
┌──────────────────────┐
│  Display Results     │
│                      │
│  <FlatList>          │
│    {results.map}     │
│    <PropertyCard/>   │
│  </FlatList>         │
│                      │
│  "12 properties      │
│   found"             │
└──────────────────────┘
```

---

## 📱 Navigation Flow

```
app/
├── index.tsx                  →  /
│   ├─ isSignedIn? YES → /(root)/(tabs)
│   └─ isSignedIn? NO  → /sign-in
│
├── (auth)/
│   ├── sign-in.tsx            →  /sign-in
│   └── sign-up.tsx            →  /sign-up
│
└── (root)/                    →  Requires auth
    ├── (tabs)/
    │   ├── index.tsx          →  /(root)/(tabs)  [Home]
    │   ├── search.tsx         →  /(root)/(tabs)/search
    │   ├── create.tsx         →  /(root)/(tabs)/create
    │   ├── saved.tsx          →  /(root)/(tabs)/saved
    │   └── profile.tsx        →  /(root)/(tabs)/profile
    │
    └── property/
        ├── [id].tsx           →  /(root)/property/123
        └── map.tsx            →  /(root)/property/map


Navigation Methods:
─────────────────────────────────────────────────────
router.push("/property/123")     → Go forward
router.replace("/sign-in")       → Replace current
router.back()                    → Go back
<Link href="/sign-up">           → Declarative link
```

---

## 🎨 Component Hierarchy

```
App
└── ClerkProvider
    └── Expo Router
        ├── (auth)
        │   └── Stack Navigator
        │       ├── SignInScreen
        │       └── SignUpScreen
        │
        └── (root)
            └── Auth Guard
                └── Tabs Navigator
                    ├── HomeScreen
                    │   ├── FeaturedCard (×N)
                    │   └── PropertyCard (×N)
                    │
                    ├── SearchScreen
                    │   ├── FilterModal
                    │   └── PropertyCard (×N)
                    │
                    ├── CreateScreen (Admin)
                    │   ├── ImagePicker
                    │   ├── LocationDetector
                    │   └── FormInputs
                    │
                    ├── SavedScreen
                    │   └── PropertyCard (×N)
                    │
                    └── ProfileScreen
```

---

## 🔐 Security Layers

```
┌─────────────────────────────────────────────────┐
│            Layer 1: Client-Side UI              │
│  - Hide admin tabs if !isAdmin                  │
│  - Disable buttons during loading               │
│  - Show appropriate error messages              │
└────────────────┬────────────────────────────────┘
                 │ ⚠️ Can be bypassed
                 ▼
┌─────────────────────────────────────────────────┐
│         Layer 2: Authentication (Clerk)         │
│  - JWT generation and validation                │
│  - Session management                           │
│  - Email verification                           │
└────────────────┬────────────────────────────────┘
                 │ ✓ Secure
                 ▼
┌─────────────────────────────────────────────────┐
│    Layer 3: Row Level Security (Supabase)       │
│  - Database-level access control                │
│  - Cannot be bypassed from client               │
│  - Validates JWT claims                         │
│                                                 │
│  Examples:                                      │
│  - Users can only read own saved_properties     │
│  - Only admins can insert/update properties     │
│  - Anyone can read properties (public)          │
└─────────────────────────────────────────────────┘
```

---

## 📦 State Management Strategy

```
┌────────────────────────────────────────────────┐
│           Local State (useState)                │
│  - Component-specific                           │
│  - Form inputs, loading states, modal visibility│
│                                                 │
│  Example: const [loading, setLoading] = ...    │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│          Global State (Zustand)                 │
│  - App-wide state                               │
│  - Persists across navigation                   │
│                                                 │
│  userStore:    { isAdmin }                      │
│  filterStore:  { search, type, bedrooms, ... }  │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│        Server State (Direct Fetch)              │
│  - Database data                                │
│  - Fetched on demand                            │
│  - No caching (always fresh)                    │
│                                                 │
│  Example: Properties, Saved items               │
└────────────────────────────────────────────────┘
```

---

## 🚀 Complete Tech Stack Visualization

```
┌───────────────────────────────────────────────────┐
│              Frontend (Mobile App)                │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │  React Native (UI Framework)            │     │
│  │  - Cross-platform (iOS, Android, Web)   │     │
│  └───────────┬─────────────────────────────┘     │
│              │                                    │
│  ┌───────────▼─────────────────────────────┐     │
│  │  Expo (Development Platform)            │     │
│  │  - Build, deploy, update OTA            │     │
│  └───────────┬─────────────────────────────┘     │
│              │                                    │
│  ┌───────────▼─────────────────────────────┐     │
│  │  Expo Router (Navigation)               │     │
│  │  - File-based routing                   │     │
│  └───────────┬─────────────────────────────┘     │
│              │                                    │
│  ┌───────────▼─────────────────────────────┐     │
│  │  NativeWind (Styling)                   │     │
│  │  - Tailwind CSS for React Native        │     │
│  └─────────────────────────────────────────┘     │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │  Zustand (State Management)             │     │
│  │  - Lightweight global state             │     │
│  └─────────────────────────────────────────┘     │
└───────────────────────────────────────────────────┘
                        │
                        │ HTTP/HTTPS
                        ▼
┌───────────────────────────────────────────────────┐
│             Backend Services                      │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │  Clerk (Authentication)                 │     │
│  │  - User management, JWT generation      │     │
│  └─────────────────────────────────────────┘     │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │  Supabase (Backend-as-a-Service)        │     │
│  │                                         │     │
│  │  ├─ PostgreSQL Database                │     │
│  │  │  - Users, Properties, Saved         │     │
│  │  │                                     │     │
│  │  ├─ Storage (S3-like)                  │     │
│  │  │  - Property images                  │     │
│  │  │                                     │     │
│  │  └─ Realtime (WebSocket)               │     │
│  │     - Live updates (optional)          │     │
│  └─────────────────────────────────────────┘     │
└───────────────────────────────────────────────────┘
```

---

**For detailed implementation, see the numbered documentation guides in the `docs/` folder!**
