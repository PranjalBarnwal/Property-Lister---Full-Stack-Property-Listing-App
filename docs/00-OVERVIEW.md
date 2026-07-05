# Project Overview: Property Lister

## 🏠 What is Property Lister?

Property Lister is a **full-stack React Native mobile application** for browsing, searching, and listing real estate properties. Think of it like a simplified version of Zillow or MagicBricks, built for mobile-first experience.

---

## 🎯 Main Features

### For Regular Users:
- **Browse Properties** - View featured and recommended properties
- **Search & Filter** - Search by location, filter by type, price, bedrooms
- **Save Favorites** - Bookmark properties you like
- **View Details** - See photos, specs, location on map
- **Contact Agent** - WhatsApp integration to reach property sellers

### For Admin Users:
- **List Properties** - Add new properties with photos, location, details
- **Manage Listings** - Mark properties as sold or delete them
- **Feature Properties** - Promote specific properties to the featured section

---

## 🏗️ Technical Architecture

### Frontend Stack
- **React Native** (v0.81.5) - Cross-platform mobile framework
- **Expo** (~54.0) - Development toolchain and runtime
- **Expo Router** (~6.0) - File-based navigation system
- **NativeWind** (v4.2.3) - Tailwind CSS for React Native
- **TypeScript** - Type safety

### Backend & Services
- **Clerk** - Authentication provider (handles sign up, login, user management)
- **Supabase** - PostgreSQL database + file storage
  - Stores: properties, users, saved_properties
  - Storage: property images
- **OpenStreetMap** - Map visualization (via WebView)

### State Management
- **Zustand** - Lightweight global state management
  - User state (admin status)
  - Filter state (search, type, price, bedrooms)

---

## 📂 Project Structure

```
property-lister/
├── app/                          # File-based routing (Expo Router)
│   ├── (auth)/                   # Authentication flows
│   │   ├── sign-in.tsx           # Login screen
│   │   ├── sign-up.tsx           # Registration screen
│   │   └── _layout.tsx           # Auth layout wrapper
│   ├── (root)/                   # Protected routes (require login)
│   │   ├── (tabs)/               # Bottom tab navigation
│   │   │   ├── index.tsx         # Home screen
│   │   │   ├── search.tsx        # Search & filter
│   │   │   ├── create.tsx        # Add property (admin only)
│   │   │   ├── saved.tsx         # Saved properties
│   │   │   ├── profile.tsx       # User profile
│   │   │   └── _layout.tsx       # Tab bar configuration
│   │   ├── property/             # Property details
│   │   │   ├── [id].tsx          # Dynamic route for property details
│   │   │   └── map.tsx           # Full-screen map view
│   │   └── _layout.tsx           # Root layout + auth guard
│   ├── index.tsx                 # Entry point (redirect logic)
│   └── _layout.tsx               # App-level layout (ClerkProvider)
│
├── components/                   # Reusable UI components
│   ├── FeaturedCard.tsx          # Featured property card
│   ├── PropertyCard.tsx          # Regular property list item
│   └── FilterModal.tsx           # Filter bottom sheet
│
├── hooks/                        # Custom React hooks
│   ├── useSupabase.ts            # Authenticated Supabase client
│   ├── useUserSync.ts            # Sync Clerk user → Supabase
│   └── useSavedProperty.ts       # Save/unsave property logic
│
├── store/                        # Zustand global state
│   ├── userStore.ts              # User admin status
│   └── filterStore.ts            # Search filters
│
├── lib/                          # Utilities
│   ├── supabase.ts               # Supabase client setup
│   └── utils.ts                  # Helper functions (formatPrice)
│
├── types/                        # TypeScript definitions
│   └── index.ts                  # Property interface
│
└── assets/                       # Images, fonts, etc.
```

---

## 🔐 Authentication Flow

```
User opens app
    ↓
Check if signed in (Clerk)
    ↓
├─ NO  → Redirect to /sign-in
│         ↓
│     User signs in/up via Clerk
│         ↓
│     Redirect to /(root)/(tabs)
│
└─ YES → Load /(root)/(tabs)
          ↓
      useUserSync runs
          ↓
      1. Check if user exists in Supabase
      2. If not, create user record
      3. Set admin status in userStore
```

**Key Point:** Clerk handles authentication (password hashing, JWT, sessions). Supabase just stores a copy of user data for app features (saved properties, admin checks).

---

## 🗄️ Database Schema (Supabase)

### `users` table
```sql
id              uuid PRIMARY KEY
clerk_id        text UNIQUE         -- Links to Clerk user ID
email           text
first_name      text
last_name       text
avatar_url      text
is_admin        boolean DEFAULT false
created_at      timestamp
```

### `properties` table
```sql
id              uuid PRIMARY KEY
title           text
description     text
price           integer
type            text               -- 'apartment' | 'house' | 'villa' | 'studio'
bedrooms        integer
bathrooms       integer
area_sqft       integer
address         text
city            text
latitude        float
longitude       float
images          text[]             -- Array of image URLs
is_featured     boolean
is_sold         boolean
created_at      timestamp
```

### `saved_properties` table
```sql
id              uuid PRIMARY KEY
user_clerk_id   text              -- Foreign key to users(clerk_id)
property_id     uuid              -- Foreign key to properties(id)
created_at      timestamp
```

---

## 🚀 How It Works (Data Flow)

### 1. User Signs Up
```
1. User fills sign-up form
2. Clerk creates user account
3. User receives verification email
4. User verifies email → Clerk marks as verified
5. App redirects to /(root)/(tabs)
6. useUserSync() runs:
   - Gets Clerk JWT token
   - Queries Supabase users table
   - If user doesn't exist, creates new record
   - Sets isAdmin flag in userStore
```

### 2. Browsing Properties
```
1. Home screen loads
2. Fetch properties from Supabase (public read, no auth needed)
   - Featured: WHERE is_featured = true
   - Recommended: WHERE is_featured = false
3. Display in FlatList
4. User taps property → Navigate to /property/[id]
```

### 3. Searching Properties
```
1. User types in search box
2. useFilterStore updates search state
3. useEffect triggers fetchResults()
4. Build Supabase query:
   - search text → .or('title.ilike.%search%,city.ilike.%search%')
   - type filter → .eq('type', type)
   - bedrooms → .eq('bedrooms', bedrooms)
   - price range → .gte('price', min).lte('price', max)
5. Display results
```

### 4. Saving a Property
```
1. User taps heart icon
2. useSavedProperty hook:
   - Gets authenticated Supabase client (with Clerk JWT)
   - INSERT INTO saved_properties (user_clerk_id, property_id)
3. Icon changes to filled heart
4. Property appears in Saved tab
```

### 5. Admin Creating Property
```
1. Admin taps "Add Property" tab
2. Fills form:
   - Photos → Upload to Supabase Storage (property-images bucket)
   - Location → Uses expo-location to get GPS coordinates
   - Price, bedrooms, etc.
3. Submit → INSERT INTO properties
4. Success → Navigate back to home
```

---

## 🔑 Key Concepts

### 1. Clerk + Supabase Integration
- **Clerk JWT** contains user ID in the `sub` claim
- Supabase validates JWT using Clerk's JWKS endpoint
- Row Level Security (RLS) policies use `auth.jwt() ->> 'sub'` to identify users
- This allows secure user-specific queries without exposing sensitive data

### 2. File-Based Routing (Expo Router)
- Folders in `app/` become routes
- `(auth)` and `(root)` are **route groups** (organize without affecting URL)
- `[id].tsx` is a **dynamic route** (matches `/property/123`)
- `_layout.tsx` wraps child routes (used for navigation, auth guards)

### 3. State Management Strategy
- **Local state** (useState) → Component-specific data
- **Zustand stores** → Global app state (filters, admin status)
- **React Query-like pattern** → useFocusEffect + fetch functions

### 4. Image Handling
- User picks images → expo-image-picker
- Convert to base64 → Upload to Supabase Storage
- Get public URL → Store in database as string array
- Display → React Native `<Image source={{ uri }}>

---

## 🔒 Security Features

### Row Level Security (RLS) Policies
```sql
-- Users can only insert their own profile
CREATE POLICY "Users can insert own profile"
ON users FOR INSERT
WITH CHECK (clerk_id = auth.jwt() ->> 'sub');

-- Users can only read their saved properties
CREATE POLICY "Users can read own saved properties"
ON saved_properties FOR SELECT
USING (user_clerk_id = auth.jwt() ->> 'sub');

-- Anyone can read properties (public)
CREATE POLICY "Anyone can read properties"
ON properties FOR SELECT
TO anon, authenticated
USING (true);

-- Only admins can create properties
-- (Handled by app logic checking is_admin flag)
```

### Environment Variables (.env)
```bash
EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
EXPO_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
EXPO_PUBLIC_SUPABASE_KEY=eyJhbGc...
```
**Never commit `.env` to git!** Always use `.env.example` as template.

---

## 📱 User Experience Flow

### New User Journey
```
1. Open app → See sign-in screen
2. Tap "Sign Up" → Fill registration form
3. Verify email → Automatic login
4. See home screen with properties
5. Browse, search, save favorites
6. Tap property → View details
7. Tap "Contact Agent" → Opens WhatsApp
```

### Admin User Journey
```
1. Login as admin
2. See "Add Property" tab in bottom navigation
3. Fill property form:
   - Upload 1-6 photos
   - Enter details (title, price, bedrooms, etc.)
   - Detect location or enter manually
   - Mark as featured if desired
4. Submit → Property appears in listings
5. View property details → See admin actions
6. Mark as sold or delete property
```

---

## 🎨 Design System

### Color Palette
- **Primary Blue:** #2563EB (buttons, icons, accents)
- **Gray Scale:** #F9FAFB (bg) → #111827 (text)
- **Red:** #EF4444 (hearts, sold badges)
- **Amber:** #D97706 (featured badges)

### Typography
- **Headers:** Bold, Gray-900
- **Body:** Regular, Gray-500
- **Labels:** Semibold, Gray-700

### Component Patterns
- **Cards:** White background, subtle shadow, rounded-2xl
- **Buttons:** Blue-600, bold text, rounded-2xl
- **Inputs:** White, border-gray-200, rounded-2xl
- **Badges:** Colored background (50), colored text (600), rounded-full

---

## 🛠️ Development Workflow

### Running the App
```bash
# Install dependencies
npm install

# Start development server
npm start

# Run on specific platform
npm run ios
npm run android
npm run web
```

### Environment Setup
1. Copy `.env.example` → `.env`
2. Add Clerk publishable key
3. Add Supabase URL and anon key
4. Configure Clerk JWT template for Supabase
5. Run RLS policies in Supabase SQL Editor

### Testing Features
- **Auth:** Test sign-up, login, logout
- **Browse:** Check featured/recommended sections
- **Search:** Verify filters work correctly
- **Save:** Test save/unsave functionality
- **Admin:** Create, mark sold, delete properties
- **Maps:** Check location display and navigation

---

## 🚧 Common Issues & Solutions

### Issue: Users not syncing to Supabase
**Solution:** Check Clerk JWT configuration in Supabase settings. Run RLS policies.

### Issue: Images not uploading
**Solution:** Check Supabase Storage bucket permissions. Ensure `property-images` bucket exists.

### Issue: Saved properties not showing
**Solution:** Check RLS policies on `saved_properties` table. Ensure user has authenticated client.

### Issue: Admin tab not visible
**Solution:** Set `is_admin = true` for user in Supabase `users` table.

---

## 📚 Next Steps

To understand specific areas in depth, read:
- `01-AUTHENTICATION.md` - Clerk + Supabase integration
- `02-NAVIGATION.md` - Expo Router file-based routing
- `03-DATA-FETCHING.md` - Supabase queries and RLS
- `04-STATE-MANAGEMENT.md` - Zustand stores and hooks
- `05-COMPONENTS.md` - Reusable UI components
- `06-ADMIN-FEATURES.md` - Property creation and management

---

**Built with ❤️ using React Native, Expo, Clerk, and Supabase**
