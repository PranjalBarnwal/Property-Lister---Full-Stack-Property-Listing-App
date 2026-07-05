# Data Fetching & Supabase Deep Dive

## 🗄️ Database Overview

Your app uses **Supabase** (PostgreSQL) for data storage.

### Tables:
1. **`properties`** - Property listings (public read)
2. **`users`** - User profiles (private)
3. **`saved_properties`** - User favorites (private)

---

## 🔌 Supabase Client Setup

**File:** `lib/supabase.ts`

```typescript
import { createClient } from "@supabase/supabase-js";

const supabaseUrl = process.env.EXPO_PUBLIC_SUPABASE_URL!;
const supabaseAnonKey = process.env.EXPO_PUBLIC_SUPABASE_KEY!;

// Public client - No authentication required
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

### Two Client Types:

| Client | Use Case | Example |
|--------|----------|---------|
| **Public** (`supabase`) | Read public data | Browse properties |
| **Authenticated** (`useSupabase()`) | User-specific data | Save property, create listing |

---

## 📖 Reading Data (SELECT)

### Example 1: Fetch All Properties

**File:** `app/(root)/(tabs)/index.tsx`

```typescript
import { supabase } from "@/lib/supabase";
import { Property } from "@/types";
import { useState } from "react";

const [properties, setProperties] = useState<Property[]>([]);
const [loading, setLoading] = useState(true);

const fetchProperties = async () => {
  setLoading(true);
  
  const { data, error } = await supabase
    .from("properties")
    .select("*")
    .order("created_at", { ascending: false });
  
  if (error) {
    console.error("Error fetching properties:", error);
  }
  
  setProperties(data ?? []);
  setLoading(false);
};
```

### Breakdown:

```typescript
supabase
  .from("properties")        // Table name
  .select("*")               // SELECT * (all columns)
  .order("created_at", { ascending: false })  // ORDER BY created_at DESC
```

**Returns:**
```typescript
{
  data: Property[] | null,
  error: PostgrestError | null
}
```

---

### Example 2: Fetch Single Property

**File:** `app/(root)/property/[id].tsx`

```typescript
const { id } = useLocalSearchParams<{ id: string }>();

const fetchProperty = async () => {
  const { data, error } = await supabase
    .from("properties")
    .select("*")
    .eq("id", id)    // WHERE id = ?
    .single();       // Returns single object, not array
  
  if (error) {
    console.error("Property not found:", error);
  }
  
  setProperty(data);
};
```

**`.single()`** - Expects exactly one row. Throws error if 0 or multiple rows.

---

### Example 3: Filtered Query

**File:** `app/(root)/(tabs)/search.tsx`

```typescript
const fetchResults = async () => {
  let query = supabase.from("properties").select("*");

  // Search by title or city
  if (search) {
    query = query.or(`title.ilike.%${search}%,city.ilike.%${search}%`);
  }

  // Filter by type
  if (type) {
    query = query.eq("type", type);
  }

  // Filter by bedrooms
  if (bedrooms) {
    query = query.eq("bedrooms", bedrooms);
  }

  // Price range
  if (minPrice) {
    query = query.gte("price", minPrice);
  }

  if (maxPrice) {
    query = query.lte("price", maxPrice);
  }

  const { data } = await query.order("created_at", { ascending: false });
  
  setResults(data ?? []);
};
```

### Query Builder Methods:

| Method | SQL Equivalent | Example |
|--------|----------------|---------|
| `.eq(column, value)` | `WHERE column = value` | `.eq("type", "apartment")` |
| `.neq(column, value)` | `WHERE column != value` | `.neq("is_sold", true)` |
| `.gt(column, value)` | `WHERE column > value` | `.gt("price", 1000000)` |
| `.gte(column, value)` | `WHERE column >= value` | `.gte("bedrooms", 2)` |
| `.lt(column, value)` | `WHERE column < value` | `.lt("price", 5000000)` |
| `.lte(column, value)` | `WHERE column <= value` | `.lte("area_sqft", 2000)` |
| `.like(column, pattern)` | `WHERE column LIKE pattern` | `.like("city", "%Mumbai%")` |
| `.ilike(column, pattern)` | `WHERE column ILIKE pattern` (case-insensitive) | `.ilike("title", "%luxury%")` |
| `.or(query)` | `WHERE (condition1 OR condition2)` | `.or("title.ilike.%search%,city.ilike.%search%")` |
| `.in(column, array)` | `WHERE column IN (...)` | `.in("type", ["apartment", "villa"])` |

---

### Example 4: Related Data (Joins)

**File:** `app/(root)/(tabs)/saved.tsx`

```typescript
const fetchSaved = async () => {
  const { data } = await authSupabase
    .from("saved_properties")
    .select("id, property_id, properties(*)")  // JOIN properties table
    .eq("user_clerk_id", userId)
    .order("id", { ascending: false });

  setSaved(data ?? []);
};
```

### SQL Equivalent:
```sql
SELECT 
  saved_properties.id,
  saved_properties.property_id,
  properties.*
FROM saved_properties
LEFT JOIN properties ON saved_properties.property_id = properties.id
WHERE saved_properties.user_clerk_id = 'user_123'
ORDER BY saved_properties.id DESC;
```

### Result Structure:
```typescript
[
  {
    id: "saved_1",
    property_id: "prop_123",
    properties: {
      id: "prop_123",
      title: "Luxury Apartment",
      price: 5000000,
      // ... other property fields
    }
  }
]
```

---

## ✏️ Writing Data (INSERT, UPDATE, DELETE)

### Example 1: Insert Property (Admin)

**File:** `app/(root)/(tabs)/create.tsx`

```typescript
const handleSubmit = async () => {
  const authSupabase = useSupabase();  // Authenticated client
  
  const { data, error } = await authSupabase
    .from("properties")
    .insert({
      title: form.title.trim(),
      description: form.description.trim(),
      price: Number(form.price),
      type: form.type,
      bedrooms: form.bedrooms,
      bathrooms: form.bathrooms,
      area_sqft: form.areaSqft ? Number(form.areaSqft) : null,
      address: form.address.trim(),
      city: form.city.trim(),
      latitude: form.latitude ? Number(form.latitude) : null,
      longitude: form.longitude ? Number(form.longitude) : null,
      images: form.images,
      is_featured: form.isFeatured,
      is_sold: false,
    })
    .select()  // Return inserted row
    .single();

  if (error) {
    Alert.alert("Error", "Failed to create property.");
    console.error(error);
    return;
  }

  Alert.alert("Success", "Property created!");
  router.replace("/(root)/(tabs)");
};
```

### `.insert()` Options:

```typescript
// Insert single row
.insert({ title: "New Property" })

// Insert multiple rows
.insert([
  { title: "Property 1" },
  { title: "Property 2" }
])

// Return inserted data
.insert({ ... }).select()

// Return specific columns
.insert({ ... }).select("id, title")

// Return single object
.insert({ ... }).select().single()
```

---

### Example 2: Update Property (Mark as Sold)

**File:** `app/(root)/property/[id].tsx`

```typescript
const handleMarkSold = async () => {
  const authSupabase = useSupabase();
  
  const { error } = await authSupabase
    .from("properties")
    .update({ is_sold: true })
    .eq("id", id);

  if (error) {
    Alert.alert("Error", "Failed to mark as sold.");
    return;
  }

  setProperty(prev => prev ? { ...prev, is_sold: true } : prev);
  Alert.alert("Success", "Property marked as sold.");
};
```

### `.update()` Pattern:

```typescript
supabase
  .from("table")
  .update({ column: newValue })
  .eq("id", id)  // WHERE clause (IMPORTANT!)
```

**⚠️ Warning:** Always add `.eq()` or `.match()`. Without it, you'll update ALL rows!

---

### Example 3: Delete Property

**File:** `app/(root)/property/[id].tsx`

```typescript
const handleDelete = () => {
  Alert.alert("Delete Property", "Are you sure?", [
    { text: "Cancel", style: "cancel" },
    {
      text: "Delete",
      style: "destructive",
      onPress: async () => {
        const authSupabase = useSupabase();
        
        const { error } = await authSupabase
          .from("properties")
          .delete()
          .eq("id", id);

        if (error) {
          Alert.alert("Error", "Failed to delete property.");
          return;
        }

        router.replace("/(root)/(tabs)");
      },
    },
  ]);
};
```

---

### Example 4: Upsert (Insert or Update)

```typescript
// If property with this ID exists, update it
// Otherwise, insert new row
const { data, error } = await supabase
  .from("properties")
  .upsert({
    id: "existing-id",  // Match column
    title: "Updated Title",
    price: 6000000
  })
  .select();
```

**Use cases:**
- Syncing data from external API
- User preferences (insert if new, update if exists)

---

## ❤️ Example: Save/Unsave Property

**File:** `hooks/useSavedProperty.ts`

```typescript
export function useSavedProperty(propertyId: string, onUnsave?: () => void) {
  const { userId } = useAuth();
  const authSupabase = useSupabase();

  const [isSaved, setIsSaved] = useState(false);
  const [saveLoading, setSaveLoading] = useState(false);

  // Check if property is saved
  useEffect(() => {
    checkIfSaved();
  }, [propertyId, userId]);

  const checkIfSaved = async () => {
    if (!userId) return;
    
    const { data } = await authSupabase
      .from("saved_properties")
      .select("id")
      .eq("user_clerk_id", userId)
      .eq("property_id", propertyId)
      .single();
    
    setIsSaved(!!data);  // !! converts to boolean
  };

  const toggleSave = async () => {
    if (!userId || saveLoading) return;
    
    setSaveLoading(true);
    
    if (isSaved) {
      // Unsave (delete)
      await authSupabase
        .from("saved_properties")
        .delete()
        .eq("user_clerk_id", userId)
        .eq("property_id", propertyId);
      
      setIsSaved(false);
      onUnsave?.();  // Callback to remove from list
    } else {
      // Save (insert)
      await authSupabase
        .from("saved_properties")
        .insert({ 
          user_clerk_id: userId, 
          property_id: propertyId 
        });
      
      setIsSaved(true);
    }
    
    setSaveLoading(false);
  };

  return { isSaved, saveLoading, toggleSave };
}
```

### Usage:

```typescript
// In PropertyCard component
const { isSaved, saveLoading, toggleSave } = useSavedProperty(property.id);

<TouchableOpacity onPress={toggleSave} disabled={saveLoading}>
  <Ionicons
    name={isSaved ? "heart" : "heart-outline"}
    size={18}
    color={isSaved ? "#EF4444" : "#9CA3AF"}
  />
</TouchableOpacity>
```

---

## 📁 File Storage (Images)

### Uploading Images

**File:** `app/(root)/(tabs)/create.tsx`

```typescript
const handlePickImages = async () => {
  // Pick images from library
  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: "images",
    allowsMultipleSelection: true,
    quality: 0.7,
    base64: true,  // Get base64 encoding
    selectionLimit: 6,
  });

  if (result.canceled) return;

  setUploadingImages(true);

  for (const asset of result.assets) {
    try {
      // Generate unique filename
      const filename = `property_${Date.now()}_${Math.random()
        .toString(36)
        .slice(2)}.jpg`;

      // Convert base64 to binary
      const base64 = asset.base64!;
      const buffer = Uint8Array.from(atob(base64), (c) => c.charCodeAt(0));

      // Upload to Supabase Storage
      const { error } = await authSupabase.storage
        .from("property-images")  // Bucket name
        .upload(filename, buffer, {
          contentType: "image/jpeg",
          upsert: false,
        });

      if (error) throw error;

      // Get public URL
      const { data: urlData } = authSupabase.storage
        .from("property-images")
        .getPublicUrl(filename);

      uploadedUrls.push(urlData.publicUrl);
    } catch (err) {
      console.error("Upload error:", err);
      Alert.alert("Upload Failed", "One or more images failed to upload.");
    }
  }

  updateForm({ images: [...form.images, ...uploadedUrls] });
  setUploadingImages(false);
};
```

### Storage Structure:

```
Supabase Storage
└── property-images/  (bucket)
    ├── property_1234_abc.jpg
    ├── property_1234_def.jpg
    └── property_5678_xyz.jpg
```

### Public URLs:
```
https://xxx.supabase.co/storage/v1/object/public/property-images/property_1234_abc.jpg
```

Store these URLs in database, then display with `<Image source={{ uri }}>`.

---

## 🔒 Row Level Security (RLS)

### What is RLS?

**Row Level Security** = Database-level access control.

Rules that determine **who can access which rows**.

### Example RLS Policies:

```sql
-- Anyone can read properties (public)
CREATE POLICY "Anyone can read properties"
ON properties FOR SELECT
TO anon, authenticated
USING (true);

-- Only authenticated users can insert properties
CREATE POLICY "Authenticated users can insert properties"
ON properties FOR INSERT
TO authenticated
WITH CHECK (true);

-- Users can only read their own saved properties
CREATE POLICY "Users can read own saved properties"
ON saved_properties FOR SELECT
TO authenticated
USING (user_clerk_id = (auth.jwt() ->> 'sub')::text);

-- Users can only insert their own saved properties
CREATE POLICY "Users can insert own saved properties"
ON saved_properties FOR INSERT
TO authenticated
WITH CHECK (user_clerk_id = (auth.jwt() ->> 'sub')::text);

-- Users can only delete their own saved properties
CREATE POLICY "Users can delete own saved properties"
ON saved_properties FOR DELETE
TO authenticated
USING (user_clerk_id = (auth.jwt() ->> 'sub')::text);
```

### How `auth.jwt()` Works:

```typescript
// User makes request with JWT
const authSupabase = useSupabase();  // Attaches JWT to requests

// Supabase extracts JWT claims
{
  "sub": "user_abc123",  ← Clerk user ID
  "email": "user@example.com",
  "role": "authenticated"
}

// RLS policy checks:
auth.jwt() ->> 'sub'  →  "user_abc123"
user_clerk_id         →  "user_abc123"  (from table)

// If they match → Allow query
// If they don't → Deny query
```

---

## 🔄 Data Refresh Patterns

### Pattern 1: Manual Refresh

```typescript
const [refreshing, setRefreshing] = useState(false);

const onRefresh = async () => {
  setRefreshing(true);
  await fetchProperties();
  setRefreshing(false);
};

<FlatList
  data={properties}
  refreshControl={
    <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
  }
/>
```

### Pattern 2: Refresh on Focus

```typescript
import { useFocusEffect } from "expo-router";

useFocusEffect(
  useCallback(() => {
    fetchProperties();
  }, [])
);
```

**Use case:** User edits property, returns to list → List auto-refreshes.

### Pattern 3: Realtime Subscriptions

```typescript
useEffect(() => {
  const subscription = supabase
    .channel("properties")
    .on(
      "postgres_changes",
      {
        event: "*",  // INSERT, UPDATE, DELETE
        schema: "public",
        table: "properties",
      },
      (payload) => {
        console.log("Change detected:", payload);
        fetchProperties();  // Refresh data
      }
    )
    .subscribe();

  return () => {
    subscription.unsubscribe();
  };
}, []);
```

**Use case:** Multiple users editing data, need live updates.

---

## 🎯 Best Practices

### 1. Handle Errors
```typescript
const { data, error } = await supabase.from("properties").select("*");

if (error) {
  console.error("Database error:", error);
  Alert.alert("Error", "Failed to load properties.");
  return;
}

setProperties(data ?? []);  // Fallback to empty array
```

### 2. Use Loading States
```typescript
const [loading, setLoading] = useState(true);

const fetchData = async () => {
  setLoading(true);
  const { data } = await supabase.from("properties").select("*");
  setProperties(data ?? []);
  setLoading(false);
};

if (loading) return <ActivityIndicator />;
```

### 3. Optimize Queries
```typescript
// ❌ Don't fetch all columns if not needed
.select("*")

// ✅ Fetch only what you need
.select("id, title, price, images")

// ❌ Don't fetch all rows
.select("*")

// ✅ Limit results
.select("*").limit(20)
```

### 4. Type Your Data
```typescript
// types/index.ts
export interface Property {
  id: string;
  title: string;
  price: number;
  // ...
}

// Use type
const [properties, setProperties] = useState<Property[]>([]);
```

### 5. Use Authenticated Client for User Data
```typescript
// ❌ Don't use public client for private data
const { data } = await supabase
  .from("saved_properties")
  .select("*");

// ✅ Use authenticated client
const authSupabase = useSupabase();
const { data } = await authSupabase
  .from("saved_properties")
  .select("*");
```

---

## 🐛 Common Issues

### Issue: "Row Level Security policy violation"
**Cause:** Trying to access data without proper RLS policies
**Fix:** Add RLS policies in Supabase SQL Editor

### Issue: Data returns empty array
**Cause:** Query doesn't match any rows
**Fix:** Check WHERE conditions, verify data exists

### Issue: "JWT expired"
**Cause:** Clerk JWT token expired
**Fix:** Clerk auto-refreshes tokens, but check token cache is working

### Issue: Images not loading
**Cause:** Storage bucket not public or wrong URL
**Fix:** Check bucket permissions in Supabase Storage settings

---

**Next:** Read `04-STATE-MANAGEMENT.md` to learn about Zustand stores!
