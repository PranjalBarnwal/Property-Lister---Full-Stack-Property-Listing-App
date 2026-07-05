# Admin Features Deep Dive

## 👑 Admin System Overview

Admins have special permissions to:
- ✅ Create new property listings
- ✅ Mark properties as sold
- ✅ Delete properties
- ✅ Feature properties on home screen

Regular users cannot access these features.

---

## 🔑 How Admin Access Works

### 1. Database Flag
```sql
-- users table
is_admin boolean DEFAULT false
```

### 2. Zustand Store
```typescript
// store/userStore.ts
const useUserStore = create((set) => ({
  isAdmin: false,
  setIsAdmin: (value) => set({ isAdmin: value }),
}));
```

### 3. User Sync Hook
```typescript
// hooks/useUserSync.ts
const syncUser = async () => {
  const { data } = await authSupabase
    .from("users")
    .select("is_admin")
    .eq("clerk_id", user!.id)
    .single();
  
  setIsAdmin(data.is_admin ?? false);
};
```

### Flow:
```
User logs in
    ↓
useUserSync() runs
    ↓
Query Supabase: SELECT is_admin FROM users
    ↓
Update Zustand store: setIsAdmin(data.is_admin)
    ↓
UI updates based on isAdmin flag
```

---

## 📝 Feature 1: Create Property

**File:** `app/(root)/(tabs)/create.tsx`

### Tab Visibility Logic

**File:** `app/(root)/(tabs)/_layout.tsx`

```typescript
const isAdmin = useUserStore((state) => state.isAdmin);

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
      href: null,  // Hide tab but keep route accessible
    }}
  />
)}
```

**Result:**
- Admin sees "Add Property" tab
- Regular user doesn't see tab (but route still exists for navigation)

---

### Form Structure

```typescript
interface FormState {
  // Basic Info
  title: string;
  description: string;
  price: string;
  type: PropertyType;  // "apartment" | "house" | "villa" | "studio"
  
  // Property Details
  bedrooms: number;
  bathrooms: number;
  areaSqft: string;
  
  // Location
  address: string;
  city: string;
  latitude: string;
  longitude: string;
  
  // Images
  images: string[];        // Supabase Storage URLs
  localImages: string[];   // Local URIs for preview
  
  // Flags
  isFeatured: boolean;
}
```

### Initial State
```typescript
const INITIAL_FORM: FormState = {
  title: "",
  description: "",
  price: "",
  type: "apartment",
  bedrooms: 1,
  bathrooms: 1,
  areaSqft: "",
  address: "",
  city: "",
  latitude: "",
  longitude: "",
  isFeatured: false,
  images: [],
  localImages: [],
};
```

---

### Image Upload Flow

#### Step 1: Pick Images
```typescript
const handlePickImages = async () => {
  // Request permission
  const permission = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (!permission.granted) {
    Alert.alert("Permission Required", "Please allow access to photos.");
    return;
  }

  // Launch image picker
  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: "images",
    allowsMultipleSelection: true,
    quality: 0.7,          // Compress to 70% quality
    base64: true,          // Get base64 for upload
    selectionLimit: 6,     // Max 6 images
  });

  if (result.canceled) return;
  
  setUploadingImages(true);
  
  // Process each image...
};
```

#### Step 2: Upload to Supabase Storage
```typescript
for (const asset of result.assets) {
  try {
    // Generate unique filename
    const filename = `property_${Date.now()}_${Math.random()
      .toString(36)
      .slice(2)}.jpg`;

    // Convert base64 to binary buffer
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
    previewUris.push(asset.uri);  // Local URI for preview
  } catch (err) {
    console.error("Upload error:", err);
    Alert.alert("Upload Failed", "One or more images failed to upload.");
  }
}

updateForm({
  images: [...form.images, ...uploadedUrls],
  localImages: [...form.localImages, ...previewUris],
});
```

#### Storage Structure:
```
Supabase Storage
└── property-images/
    ├── property_1704067200000_abc123.jpg
    ├── property_1704067200001_def456.jpg
    └── property_1704067200002_ghi789.jpg
```

#### Image Preview:
```typescript
<View className="flex-row flex-wrap gap-3">
  {form.localImages.map((uri, index) => (
    <View key={index} className="relative">
      <Image source={{ uri }} className="w-24 h-24 rounded-2xl" />
      
      {/* Cover badge for first image */}
      {index === 0 && (
        <View className="absolute top-1 left-1 bg-blue-600 px-1.5 py-0.5 rounded-full">
          <Text className="text-white text-[9px] font-bold">COVER</Text>
        </View>
      )}
      
      {/* Remove button */}
      <TouchableOpacity
        onPress={() => handleRemoveImage(index)}
        className="absolute -top-2 -right-2 w-5 h-5 bg-red-500 rounded-full"
      >
        <Ionicons name="close" size={11} color="white" />
      </TouchableOpacity>
    </View>
  ))}
</View>
```

---

### Location Detection

```typescript
const handleDetectLocation = async () => {
  setDetectingLocation(true);
  
  try {
    // Request permission
    const { status } = await Location.requestForegroundPermissionsAsync();
    if (status !== "granted") {
      Alert.alert("Permission Denied", "Location permission required.");
      return;
    }

    // Get current position
    const location = await Location.getCurrentPositionAsync({
      accuracy: Location.Accuracy.Balanced,
    });

    // Update form with coordinates
    updateForm({
      latitude: String(location.coords.latitude),
      longitude: String(location.coords.longitude),
    });
  } catch (err) {
    Alert.alert("Error", "Could not detect location. Enter manually.");
  } finally {
    setDetectingLocation(false);
  }
};
```

**UI:**
```typescript
<TouchableOpacity
  onPress={handleDetectLocation}
  disabled={detectingLocation}
  className="bg-blue-50 px-3 py-1.5 rounded-full"
>
  {detectingLocation ? (
    <ActivityIndicator size="small" color="#2563EB" />
  ) : (
    <>
      <Ionicons name="locate-outline" size={13} color="#2563EB" />
      <Text className="text-blue-600">Detect Location</Text>
    </>
  )}
</TouchableOpacity>
```

---

### Form Validation

```typescript
const handleSubmit = async () => {
  // Title required
  if (!form.title.trim()) {
    return Alert.alert("Validation", "Title is required.");
  }

  // Price required and valid
  if (!form.price.trim()) {
    return Alert.alert("Validation", "Price is required.");
  }

  const priceNum = Number(form.price);
  if (isNaN(priceNum) || priceNum < MIN_PRICE) {
    return Alert.alert("Validation", "Price must be greater than ₹0.");
  }
  
  if (priceNum > MAX_PRICE) {
    return Alert.alert(
      "Validation",
      `Price cannot exceed ₹${MAX_PRICE.toLocaleString("en-IN")}.`
    );
  }

  // Address and city required
  if (!form.address.trim()) {
    return Alert.alert("Validation", "Address is required.");
  }
  
  if (!form.city.trim()) {
    return Alert.alert("Validation", "City is required.");
  }

  // At least one image
  if (form.images.length === 0) {
    return Alert.alert("Validation", "Please upload at least one image.");
  }

  // All validation passed - proceed with submission
  submitProperty();
};
```

---

### Property Submission

```typescript
const submitProperty = async () => {
  setSubmitting(true);

  const { error } = await authSupabase
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
      is_sold: false,  // New properties are not sold
    });

  setSubmitting(false);

  if (error) {
    Alert.alert("Error", "Failed to create property. Please try again.");
    console.error(error);
    return;
  }

  // Reset form
  setForm(INITIAL_FORM);
  
  // Show success and navigate
  Alert.alert("Success! 🎉", "Property listed successfully.", [
    { text: "OK", onPress: () => router.replace("/(root)/(tabs)") },
  ]);
};
```

---

## 🏷️ Feature 2: Mark as Sold

**File:** `app/(root)/property/[id].tsx`

### UI (Admin Only)
```typescript
const isAdmin = useUserStore((state) => state.isAdmin);

{isAdmin && (
  <View className="flex-row gap-3">
    {!property.is_sold && (
      <TouchableOpacity
        onPress={handleMarkSold}
        className="flex-1 bg-amber-50 py-4 rounded-2xl"
      >
        <Ionicons name="checkmark-circle-outline" size={18} color="#D97706" />
        <Text className="text-amber-600 font-semibold">Mark Sold</Text>
      </TouchableOpacity>
    )}
  </View>
)}
```

### Logic
```typescript
const handleMarkSold = () => {
  Alert.alert("Mark as Sold", "Are you sure?", [
    { text: "Cancel", style: "cancel" },
    {
      text: "Mark Sold",
      onPress: async () => {
        const authSupabase = useSupabase();
        
        const { error } = await authSupabase
          .from("properties")
          .update({ is_sold: true })
          .eq("id", id);

        if (error) {
          Alert.alert("Error", "Failed to mark as sold.");
          return;
        }

        // Update local state for immediate UI update
        setProperty((prev) => (prev ? { ...prev, is_sold: true } : prev));
        
        Alert.alert("Success", "Property marked as sold.");
      },
    },
  ]);
};
```

### Visual Effects

#### Property Cards (Faded)
```typescript
<TouchableOpacity
  style={{ opacity: property.is_sold ? 0.5 : 1 }}
>
```

#### Sold Badge
```typescript
{property.is_sold && (
  <View className="bg-red-50 px-2 py-0.5 rounded-full">
    <Text className="text-red-500 text-xs font-semibold">Sold</Text>
  </View>
)}
```

---

## 🗑️ Feature 3: Delete Property

**File:** `app/(root)/property/[id].tsx`

### UI (Admin Only)
```typescript
{isAdmin && (
  <TouchableOpacity
    onPress={handleDelete}
    className="flex-1 bg-red-50 py-4 rounded-2xl"
  >
    <Ionicons name="trash-outline" size={18} color="#EF4444" />
    <Text className="text-red-500 font-semibold">Delete</Text>
  </TouchableOpacity>
)}
```

### Logic with Confirmation
```typescript
const handleDelete = () => {
  Alert.alert(
    "Delete Property",
    "Are you sure? This action cannot be undone.",
    [
      {
        text: "Cancel",
        style: "cancel"
      },
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
            console.error(error);
            return;
          }

          // Navigate back to home
          router.replace("/(root)/(tabs)");
        },
      },
    ]
  );
};
```

### Important Notes:

#### 1. Confirmation Required
Always show confirmation dialog for destructive actions.

#### 2. Navigate After Delete
```typescript
router.replace("/(root)/(tabs)");  // Use replace, not push
```

User shouldn't be able to go back to deleted property.

#### 3. Cascade Deletion (Future Enhancement)
Currently: Only deletes property record
Future: Also delete associated data (saved_properties, images in storage)

---

## ⭐ Feature 4: Featured Properties

### Toggle Featured Status

#### In Create Form:
```typescript
<Toggle
  label="Featured Property"
  description="Show this in the Featured section on home"
  value={form.isFeatured}
  onChange={(v) => updateForm({ isFeatured: v })}
/>
```

#### Toggle Component:
```typescript
function Toggle({ label, value, onChange, description }) {
  return (
    <TouchableOpacity
      onPress={() => onChange(!value)}
      className={`flex-row items-center justify-between p-4 rounded-2xl ${
        value ? "bg-blue-50 border-blue-200" : "bg-white border-gray-200"
      }`}
    >
      <View className="flex-1">
        <Text className={`font-semibold ${value ? "text-blue-700" : "text-gray-700"}`}>
          {label}
        </Text>
        {description && (
          <Text className="text-xs text-gray-400">{description}</Text>
        )}
      </View>
      
      <View className={`w-6 h-6 rounded-full ${
        value ? "bg-blue-600" : "border-2 border-gray-300"
      }`}>
        {value && <Ionicons name="checkmark" size={14} color="white" />}
      </View>
    </TouchableOpacity>
  );
}
```

### Featured vs. Recommended

**File:** `app/(root)/(tabs)/index.tsx`

```typescript
const fetchProperties = async () => {
  // Featured properties
  const { data: featuredData } = await supabase
    .from("properties")
    .select("*")
    .eq("is_featured", true)
    .order("created_at", { ascending: false });

  // Recommended properties (not featured)
  const { data: recommendedData } = await supabase
    .from("properties")
    .select("*")
    .eq("is_featured", false)
    .order("created_at", { ascending: false });

  setFeatured(featuredData ?? []);
  setRecommended(recommendedData ?? []);
};
```

### Display:
```typescript
{/* Featured Section - Horizontal Scroll */}
<FlatList
  data={featured}
  renderItem={({ item }) => <FeaturedCard property={item} />}
  horizontal
  showsHorizontalScrollIndicator={false}
/>

{/* Recommended Section - Vertical List */}
<FlatList
  data={recommended}
  renderItem={({ item }) => <PropertyCard property={item} />}
/>
```

---

## 🔒 Security Considerations

### 1. Client-Side Protection
```typescript
// Hide UI based on isAdmin flag
{isAdmin && <CreatePropertyButton />}
```

**Limitation:** Can be bypassed by modifying client code.

### 2. Database-Level Protection (RLS)

```sql
-- Only allow admins to insert properties
CREATE POLICY "Only admins can insert properties"
ON properties FOR INSERT
TO authenticated
WITH CHECK (
  EXISTS (
    SELECT 1 FROM users
    WHERE users.clerk_id = (auth.jwt() ->> 'sub')::text
    AND users.is_admin = true
  )
);

-- Only allow admins to update properties
CREATE POLICY "Only admins can update properties"
ON properties FOR UPDATE
TO authenticated
USING (
  EXISTS (
    SELECT 1 FROM users
    WHERE users.clerk_id = (auth.jwt() ->> 'sub')::text
    AND users.is_admin = true
  )
);

-- Only allow admins to delete properties
CREATE POLICY "Only admins can delete properties"
ON properties FOR DELETE
TO authenticated
USING (
  EXISTS (
    SELECT 1 FROM users
    WHERE users.clerk_id = (auth.jwt() ->> 'sub')::text
    AND users.is_admin = true
  )
);
```

**Protection:** Even if user bypasses client-side checks, database will reject unauthorized requests.

---

## 🎨 Admin UI Patterns

### Pattern 1: Admin-Only Sections
```typescript
const isAdmin = useUserStore((state) => state.isAdmin);

{isAdmin && (
  <View>
    <Text>Admin Tools</Text>
    {/* Admin actions */}
  </View>
)}
```

### Pattern 2: Conditional Tab Visibility
```typescript
{isAdmin ? (
  <Tabs.Screen name="create" options={{...}} />
) : (
  <Tabs.Screen name="create" options={{ href: null }} />
)}
```

### Pattern 3: Admin Badge
```typescript
{isAdmin && (
  <View className="bg-blue-50 px-2 py-1 rounded-full">
    <Text className="text-blue-600 text-xs font-bold">ADMIN</Text>
  </View>
)}
```

### Pattern 4: Destructive Action Confirmation
```typescript
const handleDestructiveAction = () => {
  Alert.alert(
    "Confirm Action",
    "This action cannot be undone.",
    [
      { text: "Cancel", style: "cancel" },
      { text: "Delete", style: "destructive", onPress: performDelete },
    ]
  );
};
```

---

## 📊 Admin Dashboard (Future Enhancement)

Could add:
- **Statistics**
  - Total properties
  - Total users
  - Properties by type
  - Average price
  
- **User Management**
  - View all users
  - Grant/revoke admin access
  - View user activity
  
- **Property Moderation**
  - Approve user-submitted properties
  - Edit property details
  - Bulk operations
  
- **Analytics**
  - Most viewed properties
  - Search trends
  - User engagement metrics

---

## 🐛 Common Issues

### Issue: Create tab shows for non-admin
**Cause:** `isAdmin` not updating after login
**Fix:** Ensure `useUserSync` is running in root layout

### Issue: Images fail to upload
**Cause:** Storage bucket permissions
**Fix:** 
1. Check bucket exists in Supabase
2. Verify bucket is public or has correct policies
3. Check user has authenticated client

### Issue: Property creation succeeds but doesn't appear
**Cause:** Property created in database but UI not refreshed
**Fix:**
```typescript
// After creation, refresh home screen
router.replace("/(root)/(tabs)");

// Or use useFocusEffect to auto-refresh on focus
useFocusEffect(
  useCallback(() => {
    fetchProperties();
  }, [])
);
```

### Issue: Delete button visible but action fails
**Cause:** RLS policy blocking delete
**Fix:** Verify RLS policy allows admins to delete

---

## 🎯 Best Practices

### 1. Always Validate Input
```typescript
if (!form.title.trim()) {
  return Alert.alert("Error", "Title is required.");
}
```

### 2. Show Loading States
```typescript
{submitting ? (
  <ActivityIndicator color="white" />
) : (
  <Text>Submit</Text>
)}
```

### 3. Handle Errors Gracefully
```typescript
if (error) {
  Alert.alert("Error", "Something went wrong. Please try again.");
  console.error(error);
  return;
}
```

### 4. Reset Form After Success
```typescript
setForm(INITIAL_FORM);
```

### 5. Provide Feedback
```typescript
Alert.alert("Success", "Property created successfully!");
```

### 6. Use Confirmation for Destructive Actions
```typescript
Alert.alert("Delete", "Are you sure?", [
  { text: "Cancel" },
  { text: "Delete", style: "destructive", onPress: handleDelete }
]);
```

---

## 🔐 Making a User Admin

### Manual Method (Supabase SQL Editor):
```sql
UPDATE users
SET is_admin = true
WHERE email = 'admin@example.com';
```

### Programmatic Method (Future):
Could create an admin panel route:
```typescript
// app/(root)/admin/users/[id].tsx
const handleToggleAdmin = async () => {
  await authSupabase
    .from("users")
    .update({ is_admin: !user.is_admin })
    .eq("id", userId);
};
```

**Security:** Protect this route with RLS policy that only allows existing admins to grant admin access.

---

**Congratulations!** 🎉 You now understand the entire Property Lister codebase from a junior developer's perspective!

## 📚 Documentation Index

- `00-OVERVIEW.md` - High-level architecture
- `01-AUTHENTICATION.md` - Clerk + Supabase auth
- `02-NAVIGATION.md` - Expo Router file-based routing
- `03-DATA-FETCHING.md` - Supabase queries and RLS
- `04-STATE-MANAGEMENT.md` - Zustand stores
- `05-COMPONENTS.md` - Reusable UI components
- `06-ADMIN-FEATURES.md` - Property creation and management (you are here)
