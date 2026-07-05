# Reusable Components Deep Dive

## 🧩 Component Architecture

This app has **3 main reusable components**:

1. **FeaturedCard** - Large horizontal property card (for featured section)
2. **PropertyCard** - Compact property list item (for search results)
3. **FilterModal** - Filter bottom sheet modal

---

## 🌟 FeaturedCard Component

**File:** `components/FeaturedCard.tsx`

### Purpose:
Display featured properties in a horizontal scrolling carousel on the home screen.

### Props:
```typescript
{ property: Property }
```

### Visual Design:
```
┌─────────────────────────────────┐
│  [Image 300x176px]              │
│  Badge: "apartment"    [Sold]   │
├─────────────────────────────────┤
│  Title (truncated)              │
│  📍 Address, City               │
│  ₹50L      🛏️2  🚿2            │
└─────────────────────────────────┘
```

### Code Breakdown:

```typescript
import { useRouter } from "expo-router";
import { Image, Text, TouchableOpacity, View } from "react-native";
import { Ionicons } from "@expo/vector-icons";
import { Property } from "@/types";
import { formatPrice } from "@/lib/utils";

export default function FeaturedCard({ property }: { property: Property }) {
  const router = useRouter();

  return (
    <TouchableOpacity
      onPress={() => router.push(`/(root)/property/${property.id}`)}
      className="w-72 mr-4 rounded-3xl overflow-hidden bg-white"
      style={{
        shadowColor: "#000",
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.08,
        shadowRadius: 12,
        elevation: 4,
        opacity: property.is_sold ? 0.5 : 1,
      }}
    >
      {/* Image */}
      <Image
        source={{ uri: property.images[0] }}
        className="w-full h-44"
        resizeMode="cover"
      />

      {/* Badge */}
      <View className="absolute top-3 left-3 bg-white/90 px-3 py-1 rounded-full">
        <Text className="text-xs font-semibold text-blue-600 capitalize">
          {property.type}
        </Text>
      </View>

      {property.is_sold && (
        <View className="absolute top-3 right-3 bg-red-500 px-3 py-1 rounded-full">
          <Text className="text-xs font-semibold text-white">Sold</Text>
        </View>
      )}

      {/* Info */}
      <View className="p-4">
        <Text
          className="text-base font-bold text-gray-800 mb-1"
          numberOfLines={1}
        >
          {property.title}
        </Text>

        <View className="flex-row items-center gap-1 mb-3">
          <Ionicons name="location-outline" size={13} color="#6B7280" />
          <Text className="text-xs text-gray-500" numberOfLines={1}>
            {property.address}, {property.city}
          </Text>
        </View>

        <View className="flex-row items-center justify-between">
          <Text className="text-blue-600 font-bold text-base">
            {formatPrice(property.price)}
          </Text>
          <View className="flex-row items-center gap-3">
            <View className="flex-row items-center gap-1">
              <Ionicons name="bed-outline" size={13} color="#6B7280" />
              <Text className="text-xs text-gray-500">{property.bedrooms}</Text>
            </View>
            <View className="flex-row items-center gap-1">
              <Ionicons name="water-outline" size={13} color="#6B7280" />
              <Text className="text-xs text-gray-500">
                {property.bathrooms}
              </Text>
            </View>
          </View>
        </View>
      </View>
    </TouchableOpacity>
  );
}
```

### Key Features:

#### 1. Fixed Width for Horizontal Scroll
```typescript
className="w-72 mr-4"  // 288px wide + 16px margin-right
```

Used in FlatList:
```typescript
<FlatList
  data={featured}
  renderItem={({ item }) => <FeaturedCard property={item} />}
  horizontal
  showsHorizontalScrollIndicator={false}
/>
```

#### 2. Tap Navigation
```typescript
<TouchableOpacity
  onPress={() => router.push(`/(root)/property/${property.id}`)}
>
```

Entire card is tappable → Navigates to property details.

#### 3. Conditional Opacity (Sold Properties)
```typescript
style={{ opacity: property.is_sold ? 0.5 : 1 }}
```

Sold properties appear faded out.

#### 4. Truncated Text
```typescript
<Text numberOfLines={1}>
  {property.title}
</Text>
```

Long titles are truncated with "...".

#### 5. Price Formatting
```typescript
<Text>{formatPrice(property.price)}</Text>

// formatPrice utility
// 5000000 → "₹50L"
// 15000000 → "₹1.5Cr"
```

---

## 📄 PropertyCard Component

**File:** `components/PropertyCard.tsx`

### Purpose:
Display properties in vertical lists (search results, saved properties).

### Props:
```typescript
{
  property: Property;
  onUnsave?: () => void;     // Callback when unsaved
  showSave?: boolean;        // Show save/unsave button
}
```

### Visual Design:
```
┌────┬────────────────────────────────┬──┐
│Img │  Title                        │♥ │
│128x│  📍 City                      │  │
│128 │  ₹50L  [Sold]  🛏️2  📐1200ft²│  │
└────┴────────────────────────────────┴──┘
```

### Code Breakdown:

```typescript
import { useSavedProperty } from "@/hooks/useSavedProperty";
import { formatPrice } from "@/lib/utils";
import { Property } from "@/types";
import { Ionicons } from "@expo/vector-icons";
import { useRouter } from "expo-router";
import { Image, Text, TouchableOpacity, View } from "react-native";

export default function PropertyCard({
  property,
  onUnsave,
  showSave = false,
}: {
  property: Property;
  onUnsave?: () => void;
  showSave?: boolean;
}) {
  const router = useRouter();
  const { isSaved, saveLoading, toggleSave } = useSavedProperty(
    property.id,
    onUnsave
  );

  return (
    <TouchableOpacity
      onPress={() => router.push(`/(root)/property/${property.id}`)}
      className="flex-row bg-white rounded-2xl mb-4 overflow-hidden"
      style={{
        shadowColor: "#000",
        shadowOffset: { width: 0, height: 1 },
        shadowOpacity: 0.06,
        shadowRadius: 8,
        elevation: 3,
        opacity: property.is_sold ? 0.5 : 1,
      }}
    >
      {/* Image */}
      <Image
        source={{ uri: property.images[0] }}
        className="w-28 h-28"
        resizeMode="cover"
      />

      {/* Info */}
      <View className="flex-1 p-3 justify-between">
        <View>
          <Text
            className="text-sm font-bold text-gray-800 mb-1"
            numberOfLines={1}
          >
            {property.title}
          </Text>
          <View className="flex-row items-center gap-1">
            <Ionicons name="location-outline" size={11} color="#6B7280" />
            <Text className="text-xs text-gray-500" numberOfLines={1}>
              {property.city}
            </Text>
          </View>
        </View>

        <View className="flex-row items-center justify-between">
          <Text className="text-blue-600 font-bold text-sm">
            {formatPrice(property.price)}
          </Text>
          {property.is_sold && (
            <View className="bg-red-50 px-2 py-0.5 rounded-full">
              <Text className="text-red-500 text-xs font-semibold">Sold</Text>
            </View>
          )}
          <View className="flex-row gap-3">
            <View className="flex-row items-center gap-1">
              <Ionicons name="bed-outline" size={11} color="#6B7280" />
              <Text className="text-xs text-gray-500">
                {property.bedrooms} bd
              </Text>
            </View>
            <View className="flex-row items-center gap-1">
              <Ionicons name="expand-outline" size={11} color="#6B7280" />
              <Text className="text-xs text-gray-500">
                {property.area_sqft} ft²
              </Text>
            </View>
          </View>
        </View>
      </View>

      {/* Save Button */}
      {showSave && (
        <TouchableOpacity
          onPress={toggleSave}
          disabled={saveLoading}
          className="w-10 items-center pt-3"
        >
          <Ionicons
            name={isSaved ? "heart" : "heart-outline"}
            size={18}
            color={isSaved ? "#EF4444" : "#9CA3AF"}
          />
        </TouchableOpacity>
      )}
    </TouchableOpacity>
  );
}
```

### Key Features:

#### 1. Horizontal Layout
```typescript
className="flex-row"
```

Image on left, info on right (unlike FeaturedCard which is vertical).

#### 2. Conditional Save Button
```typescript
{showSave && (
  <TouchableOpacity onPress={toggleSave}>
    <Ionicons name={isSaved ? "heart" : "heart-outline"} />
  </TouchableOpacity>
)}
```

Only shows on Saved screen (not on search results).

#### 3. Save/Unsave Hook Integration
```typescript
const { isSaved, saveLoading, toggleSave } = useSavedProperty(
  property.id,
  onUnsave
);
```

Hook handles:
- Checking if property is saved
- Toggle save/unsave
- Loading state during toggle
- Callback when unsaved (removes from list)

#### 4. Compact Info Display
```typescript
<Text>{property.bedrooms} bd</Text>  // "2 bd"
<Text>{property.area_sqft} ft²</Text>  // "1200 ft²"
```

Short labels for compact layout.

---

## 🔍 FilterModal Component

**File:** `components/FilterModal.tsx`

### Purpose:
Modal for applying search filters (type, bedrooms, price range).

### Props:
```typescript
{
  visible: boolean;
  onClose: () => void;
}
```

### Visual Design:
```
┌─────────────────────────────────────┐
│ ✕  Filters              Reset      │
├─────────────────────────────────────┤
│ Property Type                       │
│ [All] [Apartment] [House] [Villa]  │
│                                     │
│ Bedrooms                            │
│ [Any] [1] [2] [3] [4+]             │
│                                     │
│ Price Range (₹)                     │
│ [Min: ____] [Max: ____]            │
│ [Under ₹50L] [₹50L-₹1Cr] ...      │
│                                     │
│ [Apply Filters (2)]                │
└─────────────────────────────────────┘
```

### Code Structure:

```typescript
export default function FilterModal({ visible, onClose }) {
  // Get global filter state from Zustand
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

  // Local state for price inputs (apply on button press)
  const [localMin, setLocalMin] = useState(minPrice ? String(minPrice) : "");
  const [localMax, setLocalMax] = useState(maxPrice ? String(maxPrice) : "");

  // Count active filters for badge
  const activeCount = [type, bedrooms, minPrice, maxPrice].filter(
    (v) => v !== null
  ).length;

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
    <Modal visible={visible} animationType="slide" presentationStyle="pageSheet">
      <View className="flex-1 bg-gray-50">
        {/* Header */}
        <View className="flex-row items-center justify-between px-5 pt-6 pb-4">
          <TouchableOpacity onPress={onClose}>
            <Ionicons name="close" size={22} />
          </TouchableOpacity>
          <Text className="text-lg font-bold">Filters</Text>
          <TouchableOpacity onPress={handleReset}>
            <Text className="text-blue-600 font-semibold">Reset</Text>
          </TouchableOpacity>
        </View>

        <ScrollView className="flex-1">
          {/* Property Type Section */}
          <Text className="text-base font-bold mb-3">Property Type</Text>
          <View className="flex-row flex-wrap gap-2 mb-6">
            {TYPES.map((item) => (
              <TouchableOpacity
                key={String(item.value)}
                onPress={() => setType(item.value)}
                className={chip(type === item.value)}
              >
                <Text className={chipText(type === item.value)}>
                  {item.label}
                </Text>
              </TouchableOpacity>
            ))}
          </View>

          {/* Bedrooms Section */}
          <Text className="text-base font-bold mb-3">Bedrooms</Text>
          <View className="flex-row gap-2 mb-6">
            {BEDS.map((item) => (
              <TouchableOpacity
                key={String(item.value)}
                onPress={() => setBedrooms(item.value)}
                className={`flex-1 items-center py-3 rounded-2xl ${
                  bedrooms === item.value
                    ? "bg-blue-600"
                    : "bg-white"
                }`}
              >
                <Text>{item.label}</Text>
              </TouchableOpacity>
            ))}
          </View>

          {/* Price Range Section */}
          <Text className="text-base font-bold mb-3">Price Range (₹)</Text>
          <View className="flex-row gap-3 mb-3">
            <TextInput
              placeholder="Min Price"
              value={localMin}
              onChangeText={setLocalMin}
              keyboardType="numeric"
            />
            <TextInput
              placeholder="Max Price"
              value={localMax}
              onChangeText={setLocalMax}
              keyboardType="numeric"
            />
          </View>

          {/* Price Presets */}
          <View className="flex-row flex-wrap gap-2">
            {PRICE_PRESETS.map((p) => (
              <TouchableOpacity
                key={p.label}
                onPress={() => {
                  setLocalMin(p.min ? String(p.min) : "");
                  setLocalMax(p.max ? String(p.max) : "");
                  setMinPrice(p.min);
                  setMaxPrice(p.max);
                }}
              >
                <Text>{p.label}</Text>
              </TouchableOpacity>
            ))}
          </View>
        </ScrollView>

        {/* Apply Button */}
        <View className="px-5 pb-8 pt-4">
          <TouchableOpacity
            onPress={handleApply}
            className="bg-blue-600 rounded-2xl py-4"
          >
            <Text className="text-white font-bold">
              Apply Filters{activeCount > 0 ? ` (${activeCount})` : ""}
            </Text>
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );
}
```

### Key Features:

#### 1. Modal Presentation
```typescript
<Modal
  visible={visible}
  animationType="slide"          // Slide up from bottom
  presentationStyle="pageSheet"  // iOS sheet style
>
```

#### 2. Immediate Updates (Type, Bedrooms)
```typescript
// Updates Zustand store immediately
<TouchableOpacity onPress={() => setType("apartment")}>
```

User sees results update instantly.

#### 3. Delayed Updates (Price)
```typescript
// Local state first
const [localMin, setLocalMin] = useState("");

// Apply to Zustand on button press
const handleApply = () => {
  setMinPrice(Number(localMin));
  setMaxPrice(Number(localMax));
  onClose();
};
```

User can adjust range before applying.

#### 4. Active Filter Count Badge
```typescript
const activeCount = [type, bedrooms, minPrice, maxPrice]
  .filter((v) => v !== null)
  .length;

<Text>Apply Filters ({activeCount})</Text>
```

Shows how many filters are active.

#### 5. Price Presets
```typescript
const PRICE_PRESETS = [
  { label: "Under ₹50L", min: null, max: 5000000 },
  { label: "₹50L – ₹1Cr", min: 5000000, max: 10000000 },
  // ...
];
```

Quick selection buttons for common price ranges.

---

## 🎨 Component Styling Patterns

### Pattern 1: Conditional Classes
```typescript
className={`px-4 py-2 rounded-full ${
  isActive ? "bg-blue-600 text-white" : "bg-white text-gray-600"
}`}
```

### Pattern 2: Style Props for Complex Styles
```typescript
<View
  className="bg-white rounded-2xl"
  style={{
    shadowColor: "#000",
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 12,
    elevation: 4,  // Android shadow
  }}
>
```

Use `style` for properties not supported by Tailwind/NativeWind.

### Pattern 3: Reusable Style Functions
```typescript
const chip = (active: boolean) =>
  `px-4 py-2 rounded-full border ${
    active ? "bg-blue-600 border-blue-600" : "bg-white border-gray-200"
  }`;

const chipText = (active: boolean) =>
  `text-sm font-semibold ${active ? "text-white" : "text-gray-600"}`;

// Usage
<View className={chip(isActive)}>
  <Text className={chipText(isActive)}>Label</Text>
</View>
```

---

## 🔧 Component Composition Patterns

### Pattern 1: Wrapper Components
```typescript
// Reusable card wrapper
function Card({ children, onPress }) {
  return (
    <TouchableOpacity
      onPress={onPress}
      className="bg-white rounded-2xl p-4"
      style={{ shadowOpacity: 0.08, elevation: 3 }}
    >
      {children}
    </TouchableOpacity>
  );
}

// Usage
<Card onPress={handlePress}>
  <Text>Content</Text>
</Card>
```

### Pattern 2: Render Props
```typescript
function List({ data, renderItem }) {
  return (
    <FlatList
      data={data}
      renderItem={({ item }) => renderItem(item)}
    />
  );
}

// Usage
<List
  data={properties}
  renderItem={(property) => <PropertyCard property={property} />}
/>
```

### Pattern 3: Compound Components
```typescript
function Filter({ children }) {
  const [showModal, setShowModal] = useState(false);
  
  return (
    <>
      {children({ openFilter: () => setShowModal(true) })}
      <FilterModal visible={showModal} onClose={() => setShowModal(false)} />
    </>
  );
}

// Usage
<Filter>
  {({ openFilter }) => (
    <Button onPress={openFilter}>Open Filters</Button>
  )}
</Filter>
```

---

## 🎯 Best Practices

### 1. Extract Repeated UI to Components
```typescript
// ❌ Don't repeat
<View className="flex-row items-center gap-1">
  <Ionicons name="bed-outline" size={11} />
  <Text>{bedrooms}</Text>
</View>
<View className="flex-row items-center gap-1">
  <Ionicons name="water-outline" size={11} />
  <Text>{bathrooms}</Text>
</View>

// ✅ Do: Create reusable component
function Spec({ icon, value }) {
  return (
    <View className="flex-row items-center gap-1">
      <Ionicons name={icon} size={11} />
      <Text>{value}</Text>
    </View>
  );
}

<Spec icon="bed-outline" value={bedrooms} />
<Spec icon="water-outline" value={bathrooms} />
```

### 2. Props Should Be Explicit
```typescript
// ❌ Don't pass entire objects
<PropertyCard {...property} />

// ✅ Do: Be explicit about props
<PropertyCard property={property} showSave={true} />
```

### 3. Use TypeScript Interfaces
```typescript
interface PropertyCardProps {
  property: Property;
  onUnsave?: () => void;
  showSave?: boolean;
}

export default function PropertyCard(props: PropertyCardProps) {
  // ...
}
```

### 4. Optimize Performance
```typescript
// ❌ Don't create functions in render
<TouchableOpacity onPress={() => handlePress(property.id)}>

// ✅ Do: Use useCallback or extract component
const handlePress = useCallback(() => {
  router.push(`/property/${property.id}`);
}, [property.id]);

<TouchableOpacity onPress={handlePress}>
```

### 5. Handle Loading and Error States
```typescript
function PropertyCard({ property }) {
  if (!property) {
    return <Text>Loading...</Text>;
  }
  
  if (property.images.length === 0) {
    return <Text>No images available</Text>;
  }
  
  return (
    // Render card
  );
}
```

---

## 📦 Component Organization

```
components/
├── PropertyCard.tsx       # List item
├── FeaturedCard.tsx       # Featured carousel item
├── FilterModal.tsx        # Filter modal
│
├── ui/                    # Reusable UI primitives (future)
│   ├── Button.tsx
│   ├── Input.tsx
│   └── Badge.tsx
│
└── layout/                # Layout components (future)
    ├── Header.tsx
    └── TabBar.tsx
```

---

**Next:** Read `06-ADMIN-FEATURES.md` to understand property creation and management!
