# Authentication System Deep Dive

## 🔐 How Authentication Works

This app uses **dual authentication** system:
- **Clerk** → Handles actual authentication (login, passwords, sessions)
- **Supabase** → Stores user data for app features, validates Clerk JWTs

Think of it like this:
- Clerk is the **bouncer** who checks IDs
- Supabase is the **VIP list** that knows who you are once you're inside

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         Your App                            │
│                                                             │
│  ┌──────────────┐         ┌──────────────┐                │
│  │   Sign Up    │         │   Sign In    │                │
│  │   Screen     │         │   Screen     │                │
│  └──────┬───────┘         └──────┬───────┘                │
│         │                        │                         │
│         └────────┬───────────────┘                         │
│                  │                                          │
│                  ▼                                          │
│         ┌─────────────────┐                                │
│         │  Clerk SDK      │                                │
│         │  @clerk/expo    │                                │
│         └────────┬────────┘                                │
│                  │                                          │
└──────────────────┼──────────────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   Clerk Service      │  ← Manages users, passwords, JWTs
        │  (clerk.com)         │
        └──────────┬───────────┘
                   │
                   │ JWT Token
                   │
                   ▼
        ┌──────────────────────┐
        │   Your App           │
        │   (Authenticated)    │
        └──────────┬───────────┘
                   │
                   │ Supabase requests with JWT
                   │
                   ▼
        ┌──────────────────────┐
        │   Supabase           │  ← Validates JWT, checks RLS policies
        │   Database           │     Stores user profiles
        └──────────────────────┘
```

---

## 📁 File Structure

```
app/
├── _layout.tsx                    # ClerkProvider wraps entire app
├── index.tsx                      # Auth redirect logic
├── (auth)/
│   ├── _layout.tsx                # Auth screens layout
│   ├── sign-in.tsx                # Login screen
│   └── sign-up.tsx                # Registration screen
└── (root)/
    ├── _layout.tsx                # Auth guard + useUserSync
    └── (tabs)/...                 # Protected routes

hooks/
├── useUserSync.ts                 # Syncs Clerk user → Supabase
└── useSupabase.ts                 # Authenticated Supabase client

lib/
└── supabase.ts                    # Supabase client factory

store/
└── userStore.ts                   # Global user state (isAdmin)
```

---

## 🔑 Step 1: ClerkProvider Setup

**File:** `app/_layout.tsx`

```typescript
import { ClerkProvider } from "@clerk/expo";
import { tokenCache } from "@clerk/expo/token-cache";
import { Slot } from "expo-router";

const publishableKey = process.env.EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY!;

export default function RootLayout() {
  return (
    <ClerkProvider publishableKey={publishableKey} tokenCache={tokenCache}>
      <Slot />
    </ClerkProvider>
  );
}
```

### What's Happening?
1. **ClerkProvider** wraps your entire app
2. Provides auth context to all child components
3. **tokenCache** stores JWT tokens securely (uses expo-secure-store)
4. **publishableKey** identifies your Clerk app

---

## 🚪 Step 2: Entry Point Redirect Logic

**File:** `app/index.tsx`

```typescript
import { Redirect } from "expo-router";
import { useAuth } from "@clerk/expo";

export default function Index() {
  const { isSignedIn, isLoaded } = useAuth();

  // Wait for auth state to load
  if (!isLoaded) return null;

  // Redirect based on auth state
  if (isSignedIn) return <Redirect href="/(root)/(tabs)" />;
  return <Redirect href="/sign-in" />;
}
```

### Flow:
```
User opens app
    ↓
Is auth loaded? NO → Show nothing (prevent flash)
    ↓
Is user signed in?
    ├─ YES → Go to /(root)/(tabs) (home)
    └─ NO  → Go to /sign-in
```

---

## 📝 Step 3: Sign Up Screen

**File:** `app/(auth)/sign-up.tsx`

### Key Features:
1. **Email verification required** (OTP sent via email)
2. **Multi-step process:**
   - Step 1: Enter details (name, email, password)
   - Step 2: Verify email with code
3. **Form validation** via Clerk

### Code Breakdown:

```typescript
const { signUp, errors, fetchStatus } = useSignUp();
const [email, setEmail] = useState("");
const [password, setPassword] = useState("");
const [firstName, setFirstName] = useState("");
const [lastName, setLastName] = useState("");
const [code, setCode] = useState("");
```

**Hook:** `useSignUp()` from Clerk provides:
- `signUp` - Sign-up object with methods
- `errors` - Field-specific error messages
- `fetchStatus` - Loading state

### Step 1: Submit Registration

```typescript
const onSignUpPress = async () => {
  // Call Clerk API
  const { error } = await signUp.password({
    emailAddress: email,
    password,
    firstName,
    lastName,
  });

  if (error) {
    console.error(JSON.stringify(error, null, 2));
    return;
  }

  // Send verification email
  if (!error) await signUp.verifications.sendEmailCode();
};
```

**What happens:**
1. Clerk creates user account (not verified yet)
2. Sends 6-digit code to user's email
3. UI switches to verification screen

### Step 2: Verify Email

```typescript
const onVerifyPress = async () => {
  // Submit verification code
  await signUp.verifications.verifyEmailCode({ code });

  if (signUp.status === "complete") {
    await signUp.finalize({
      navigate: ({ session, decorateUrl }) => {
        const url = decorateUrl("/");
        router.replace(url as any);
      },
    });
  }
};
```

**What happens:**
1. Clerk validates OTP code
2. If valid, marks email as verified
3. Creates session (user is now logged in)
4. Redirects to home screen

### UI Rendering Logic:

```typescript
// Show verification screen if email needs verification
if (
  signUp.status === "missing_requirements" &&
  signUp.unverifiedFields.includes("email_address")
) {
  return <OTPVerificationScreen />;
}

// Show registration form
return <SignUpForm />;
```

---

## 🔓 Step 4: Sign In Screen

**File:** `app/(auth)/sign-in.tsx`

Simpler than sign-up - just email + password.

### Basic Flow:

```typescript
const { signIn, errors, fetchStatus } = useSignIn();

const onSignInPress = async () => {
  const { error } = await signIn.password({
    emailAddress: email,
    password,
  });

  if (error) return;

  if (signIn.status === "complete") {
    await signIn.finalize({
      navigate: ({ session, decorateUrl }) => {
        const url = decorateUrl("/");
        router.replace(url as any);
      },
    });
  }
};
```

### Advanced: 2FA Support

If user has 2FA enabled, Clerk might require additional verification:

```typescript
if (signIn.status === "needs_client_trust") {
  // User needs to verify via email code (2FA)
  const emailCodeFactor = signIn.supportedSecondFactors.find(
    (factor) => factor.strategy === "email_code"
  );
  if (emailCodeFactor) {
    await signIn.mfa.sendEmailCode();
  }
}
```

Then show OTP screen similar to sign-up.

---

## 🛡️ Step 5: Auth Guard

**File:** `app/(root)/_layout.tsx`

This layout protects all routes inside `(root)/`:

```typescript
import { useAuth } from "@clerk/expo";
import { Redirect, Slot } from "expo-router";
import { useUserSync } from "@/hooks/useUserSync";

export default function RootLayout() {
  const { isSignedIn, isLoaded } = useAuth();

  // Sync user to Supabase
  useUserSync();

  // Wait for auth to load
  if (!isLoaded) return null;

  // Redirect to sign-in if not authenticated
  if (!isSignedIn) return <Redirect href="/sign-in" />;

  // Render child routes
  return <Slot />;
}
```

### What's Happening?
1. **Check authentication** before rendering child routes
2. **Sync user data** to Supabase (see next section)
3. **Redirect to sign-in** if not logged in
4. **Render children** (`<Slot />`) if authenticated

---

## 🔄 Step 6: User Sync to Supabase

**File:** `hooks/useUserSync.ts`

This is the **bridge between Clerk and Supabase**.

### The Problem:
- Clerk manages users, but doesn't store custom app data
- We need to store user profiles in Supabase for features like:
  - Checking if user is admin
  - Tracking saved properties
  - Future features (user preferences, activity logs)

### The Solution:
After login, copy Clerk user data → Supabase `users` table.

### Code:

```typescript
import { useUser } from "@clerk/expo";
import { useEffect } from "react";
import { useSupabase } from "@/hooks/useSupabase";
import { useUserStore } from "@/store/userStore";

export const useUserSync = () => {
  const { user } = useUser();  // Clerk user object
  const setIsAdmin = useUserStore((state) => state.setIsAdmin);
  const authSupabase = useSupabase();  // Authenticated Supabase client

  useEffect(() => {
    if (!user) return;
    syncUser();
  }, [user]);

  const syncUser = async () => {
    try {
      // Check if user already exists in Supabase
      const { data, error: selectError } = await authSupabase
        .from("users")
        .select("clerk_id, is_admin")
        .eq("clerk_id", user!.id)
        .single();

      if (selectError && selectError.code !== "PGRST116") {
        console.error("Error checking existing user:", selectError);
        return;
      }

      // User exists - update admin status
      if (data) {
        setIsAdmin(data.is_admin ?? false);
        return;
      }

      // User doesn't exist - create new record
      const { data: newUser, error: insertError } = await authSupabase
        .from("users")
        .insert({
          clerk_id: user!.id,
          email: user!.emailAddresses[0].emailAddress,
          first_name: user!.firstName,
          last_name: user!.lastName,
          avatar_url: user!.imageUrl,
        })
        .select("is_admin")
        .single();

      if (insertError) {
        console.error("Error creating user in Supabase:", insertError);
        return;
      }

      setIsAdmin(newUser?.is_admin ?? false);
      console.log("User successfully synced to Supabase");
    } catch (error) {
      console.error("Unexpected error in syncUser:", error);
    }
  };
};
```

### Flow:
```
User logs in
    ↓
useUserSync() runs
    ↓
Query Supabase: Does user exist?
    ├─ YES → Update isAdmin state
    │        User ready to use app
    │
    └─ NO  → Create user record
             Copy data from Clerk
             Set isAdmin = false (default)
             User ready to use app
```

---

## 🔐 Step 7: Authenticated Supabase Client

**File:** `hooks/useSupabase.ts`

This hook creates a Supabase client that **includes the Clerk JWT** in every request.

```typescript
import { useAuth } from "@clerk/expo";
import { useMemo } from "react";
import { createClerkSupabaseClient } from "@/lib/supabase";

export function useSupabase() {
  const { getToken } = useAuth();

  const client = useMemo(
    () => createClerkSupabaseClient(() => getToken({ template: "supabase" })),
    [getToken]
  );

  return client;
}
```

### What's the `template` parameter?

Clerk can have multiple JWT templates (for different services). You need to:
1. Create a JWT template in Clerk dashboard named "supabase"
2. Specify which template to use when getting token

### Factory Function:

**File:** `lib/supabase.ts`

```typescript
import { createClient } from "@supabase/supabase-js";

const supabaseUrl = process.env.EXPO_PUBLIC_SUPABASE_URL!;
const supabaseAnonKey = process.env.EXPO_PUBLIC_SUPABASE_KEY!;

// Public client - no authentication
export const supabase = createClient(supabaseUrl, supabaseAnonKey);

// Authenticated client factory
export function createClerkSupabaseClient(
  getToken: () => Promise<string | null>
) {
  return createClient(supabaseUrl, supabaseAnonKey, {
    async accessToken() {
      return getToken();
    },
  });
}
```

### Usage:

```typescript
// Public data (anyone can read)
const { data } = await supabase.from("properties").select("*");

// User-specific data (requires authentication)
const authSupabase = useSupabase();
const { data } = await authSupabase
  .from("saved_properties")
  .select("*")
  .eq("user_clerk_id", userId);
```

---

## 🗄️ Step 8: Database Setup (Supabase)

### Users Table Schema:

```sql
CREATE TABLE users (
  id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_id text UNIQUE NOT NULL,
  email text NOT NULL,
  first_name text,
  last_name text,
  avatar_url text,
  is_admin boolean DEFAULT false,
  created_at timestamp DEFAULT now()
);
```

### Row Level Security Policies:

```sql
-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Allow authenticated users to insert their own profile
CREATE POLICY "Users can insert own profile"
ON users FOR INSERT
TO authenticated
WITH CHECK (true);

-- Allow users to read their own data
CREATE POLICY "Users can read own data"
ON users FOR SELECT
TO authenticated
USING (clerk_id = (auth.jwt() ->> 'sub')::text);

-- Allow users to update their own data
CREATE POLICY "Users can update own data"
ON users FOR UPDATE
TO authenticated
USING (clerk_id = (auth.jwt() ->> 'sub')::text);
```

### How RLS Works:

```
User makes request with JWT
    ↓
Supabase extracts JWT claims
    ↓
Check RLS policy:
    auth.jwt() ->> 'sub'  →  Clerk user ID from JWT
    clerk_id              →  Column in users table
    ↓
If they match → Allow query
If they don't → Deny query
```

---

## 🔧 Step 9: Clerk + Supabase Integration

### Setup Steps:

#### 1. Create JWT Template in Clerk

1. Go to Clerk Dashboard
2. Navigate to: **JWT Templates**
3. Click **"New template"**
4. Choose **"Supabase"** from presets
5. Name it: `supabase`
6. Configure claims:
```json
{
  "sub": "{{user.id}}",
  "email": "{{user.primary_email_address}}",
  "role": "authenticated"
}
```
7. Copy the **JWKS URL** (looks like: `https://your-app.clerk.accounts.dev/.well-known/jwks.json`)

#### 2. Add JWKS to Supabase

1. Go to Supabase Dashboard
2. **Project Settings** → **Auth** → **JWT Settings**
3. Find **"Additional JWT Secrets"**
4. Click **"Add"**
5. Paste the JWKS URL from Clerk
6. Save

### What This Does:

```
User logs in via Clerk
    ↓
Clerk generates JWT with claims:
{
  "sub": "user_abc123",           ← Clerk user ID
  "email": "user@example.com",
  "role": "authenticated"
}
    ↓
App sends request to Supabase with JWT in header
    ↓
Supabase validates JWT:
  1. Fetch public keys from Clerk JWKS URL
  2. Verify JWT signature
  3. Extract claims
    ↓
RLS policies can access JWT claims:
  auth.jwt() ->> 'sub'  →  "user_abc123"
    ↓
Query allowed if policy matches
```

---

## 🎭 Step 10: Admin System

### How Admin Status Works:

1. **Database flag:** `users.is_admin` boolean column
2. **Global state:** Zustand store keeps admin status
3. **UI changes:** Admin users see "Add Property" tab

### Setting Admin Status:

Manually in Supabase SQL Editor:
```sql
UPDATE users
SET is_admin = true
WHERE email = 'admin@example.com';
```

### Admin-Only UI:

**File:** `app/(root)/(tabs)/_layout.tsx`

```typescript
const isAdmin = useUserStore((state) => state.isAdmin);

return (
  <Tabs>
    {/* ... other tabs ... */}
    
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
          href: null,  // Hide tab if not admin
        }}
      />
    )}
  </Tabs>
);
```

---

## 🚀 Complete Auth Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│              User Opens App                             │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
         app/index.tsx
         Check isSignedIn
                  │
        ┌─────────┴─────────┐
        │                   │
     NO │                   │ YES
        ▼                   ▼
  /sign-in              /(root)/(tabs)
        │                   │
        │                   ├─ useUserSync()
        │                   │  ├─ Check if user exists in Supabase
        │                   │  ├─ Create user if needed
        │                   │  └─ Set isAdmin state
        │                   │
        ▼                   ▼
  Sign up with Clerk    Render home screen
        │
        ├─ Enter email, password
        ├─ Clerk creates account
        ├─ Send verification email
        ├─ User enters OTP
        ├─ Clerk verifies email
        ├─ Create session (JWT)
        │
        ▼
  Redirect to /(root)/(tabs)
        │
        └─ useUserSync() runs
           └─ Sync user to Supabase
              └─ App ready!
```

---

## 🔑 Key Takeaways

### Clerk Responsibilities:
- ✅ User registration
- ✅ Password hashing & storage
- ✅ Email verification
- ✅ Session management
- ✅ JWT generation
- ✅ 2FA support

### Supabase Responsibilities:
- ✅ Store user profiles
- ✅ Store app-specific data (saved properties)
- ✅ Validate Clerk JWTs
- ✅ Enforce RLS policies

### Why This Architecture?
- **Separation of concerns** - Auth vs. data storage
- **Security** - Clerk handles sensitive auth, Supabase handles data
- **Scalability** - Each service does what it's best at
- **Flexibility** - Can swap out either service independently

---

## 🐛 Troubleshooting

### "No suitable key was found to decode the JWT"
**Cause:** Supabase doesn't have Clerk JWKS URL configured
**Fix:** Add JWKS URL to Supabase JWT settings

### Users not syncing to Supabase
**Cause:** RLS policies blocking INSERT
**Fix:** Run the RLS policy SQL scripts

### "Invalid JWT"
**Cause:** Wrong JWT template name
**Fix:** Ensure `getToken({ template: "supabase" })` matches Clerk template name

### Admin tab not showing
**Cause:** `is_admin` is false in database
**Fix:** Manually set to true in Supabase

---

**Next:** Read `02-NAVIGATION.md` to understand how file-based routing works!
