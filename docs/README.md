# Property Lister - Complete Documentation

Welcome to the comprehensive documentation for the Property Lister React Native app! This guide is written like a senior developer explaining the codebase to a junior developer.

---

## 📖 Documentation Structure

### 🌟 Start Here
**[00-OVERVIEW.md](./00-OVERVIEW.md)** - High-level architecture, tech stack, project structure
- What the app does
- How the pieces fit together
- Directory structure explained
- Development workflow

### 🔐 Core Concepts (Read in Order)

1. **[01-AUTHENTICATION.md](./01-AUTHENTICATION.md)** - Clerk + Supabase authentication
   - How login/signup works
   - JWT integration
   - User sync between Clerk and Supabase
   - Auth guards and protected routes

2. **[02-NAVIGATION.md](./02-NAVIGATION.md)** - Expo Router file-based routing
   - How folders become routes
   - Layouts and route groups
   - Dynamic routes with `[id]`
   - Navigation methods (push, replace, back)

3. **[03-DATA-FETCHING.md](./03-DATA-FETCHING.md)** - Supabase queries and storage
   - Reading data (SELECT)
   - Writing data (INSERT, UPDATE, DELETE)
   - File uploads (images)
   - Row Level Security (RLS)

4. **[04-STATE-MANAGEMENT.md](./04-STATE-MANAGEMENT.md)** - Zustand stores
   - User store (admin status)
   - Filter store (search filters)
   - Local vs. global state
   - Best practices

5. **[05-COMPONENTS.md](./05-COMPONENTS.md)** - Reusable UI components
   - FeaturedCard (horizontal carousel)
   - PropertyCard (list item)
   - FilterModal (filter sheet)
   - Component patterns

6. **[06-ADMIN-FEATURES.md](./06-ADMIN-FEATURES.md)** - Property creation and management
   - Admin access control
   - Creating properties
   - Marking as sold
   - Deleting properties

---

## 🚀 Quick Start

### Prerequisites
```bash
- Node.js (v18+)
- npm or yarn
- Expo CLI
- iOS Simulator / Android Emulator
```

### Setup
```bash
# 1. Install dependencies
npm install

# 2. Copy environment variables
cp .env.example .env

# 3. Add your credentials to .env
EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
EXPO_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
EXPO_PUBLIC_SUPABASE_KEY=eyJhbGc...

# 4. Start development server
npm start
```

### First-Time Setup Checklist
- [ ] Create Clerk account and app
- [ ] Create Supabase project
- [ ] Configure Clerk JWT template for Supabase
- [ ] Add Clerk JWKS URL to Supabase
- [ ] Run RLS policies in Supabase
- [ ] Create `property-images` storage bucket
- [ ] Set first user as admin manually

---

## 📁 Key Files Reference

### Configuration
- `app.json` - Expo configuration
- `package.json` - Dependencies and scripts
- `tsconfig.json` - TypeScript configuration
- `tailwind.config.js` - Tailwind/NativeWind styling
- `.env` - Environment variables (don't commit!)

### Entry Points
- `app/_layout.tsx` - Root layout (ClerkProvider)
- `app/index.tsx` - Entry point (auth redirect)

### Authentication
- `app/(auth)/sign-in.tsx` - Login screen
- `app/(auth)/sign-up.tsx` - Registration screen
- `hooks/useUserSync.ts` - Sync Clerk → Supabase
- `hooks/useSupabase.ts` - Authenticated Supabase client

### Main Screens
- `app/(root)/(tabs)/index.tsx` - Home (browse properties)
- `app/(root)/(tabs)/search.tsx` - Search & filter
- `app/(root)/(tabs)/create.tsx` - Add property (admin)
- `app/(root)/(tabs)/saved.tsx` - Saved properties
- `app/(root)/(tabs)/profile.tsx` - User profile
- `app/(root)/property/[id].tsx` - Property details

### State & Data
- `store/userStore.ts` - User state (isAdmin)
- `store/filterStore.ts` - Search filters
- `lib/supabase.ts` - Supabase client factory
- `types/index.ts` - TypeScript interfaces

### Components
- `components/FeaturedCard.tsx` - Featured property card
- `components/PropertyCard.tsx` - Property list item
- `components/FilterModal.tsx` - Filter modal

---

## 🎯 Common Tasks

### How do I...

#### Add a new screen?
Create a new file in `app/` folder:
```
app/
└── (root)/
    └── settings.tsx  → Route: /(root)/settings
```

#### Add a new API call?
```typescript
import { supabase } from "@/lib/supabase";

const fetchData = async () => {
  const { data, error } = await supabase
    .from("table_name")
    .select("*");
  
  if (error) console.error(error);
  return data;
};
```

#### Add a new Zustand store?
```typescript
// store/myStore.ts
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

#### Make a user admin?
Run in Supabase SQL Editor:
```sql
UPDATE users
SET is_admin = true
WHERE email = 'user@example.com';
```

#### Add a new database table?
1. Create table in Supabase SQL Editor
2. Add RLS policies
3. Create TypeScript interface in `types/index.ts`
4. Write queries in your components

---

## 🧪 Testing Checklist

### Authentication
- [ ] Sign up with new email
- [ ] Verify email (OTP)
- [ ] Sign out
- [ ] Sign in with existing account
- [ ] Auth guard prevents access when logged out

### Browse Properties
- [ ] Featured properties show on home
- [ ] Recommended properties show on home
- [ ] Tap property → View details
- [ ] View all images in carousel
- [ ] View location on map

### Search & Filter
- [ ] Search by title works
- [ ] Search by city works
- [ ] Filter by type works
- [ ] Filter by bedrooms works
- [ ] Filter by price range works
- [ ] Active filters show as chips
- [ ] Reset filters works

### Save Properties
- [ ] Tap heart → Property saves
- [ ] Saved tab shows saved properties
- [ ] Tap heart again → Property unsaves
- [ ] Unsaved property removes from list

### Admin Features
- [ ] Admin sees "Add Property" tab
- [ ] Can upload images
- [ ] Can detect location
- [ ] Can create property
- [ ] Property appears in home
- [ ] Can mark property as sold
- [ ] Can delete property

### Contact
- [ ] Contact button opens WhatsApp
- [ ] Pre-filled message includes property title

---

## 🐛 Troubleshooting

### App won't start
```bash
# Clear cache and restart
npx expo start -c
```

### "No suitable key was found to decode JWT"
- Check Clerk JWT template is created
- Verify JWKS URL is added to Supabase
- Wait 1-2 minutes for cache refresh

### Users not syncing to Supabase
- Check RLS policies are created
- Verify Clerk JWT template name matches `getToken({ template: "supabase" })`
- Check console for error messages

### Images not uploading
- Verify `property-images` bucket exists in Supabase
- Check bucket permissions (public or proper RLS)
- Ensure using authenticated Supabase client

### Tab bar not showing correctly
- Check `isAdmin` flag is set in userStore
- Verify `useUserSync` is running in root layout
- Check user has `is_admin = true` in database

---

## 📚 Learning Resources

### React Native
- [React Native Docs](https://reactnative.dev/docs/getting-started)
- [Expo Docs](https://docs.expo.dev/)

### Expo Router
- [Expo Router Docs](https://expo.github.io/router/docs/)
- [File-based routing guide](https://expo.github.io/router/docs/features/routing/)

### Supabase
- [Supabase Docs](https://supabase.com/docs)
- [Row Level Security](https://supabase.com/docs/guides/auth/row-level-security)
- [Storage](https://supabase.com/docs/guides/storage)

### Clerk
- [Clerk Docs](https://clerk.com/docs)
- [Expo integration](https://clerk.com/docs/quickstarts/expo)

### Zustand
- [Zustand Docs](https://docs.pmnd.rs/zustand/getting-started/introduction)

### NativeWind (Tailwind for RN)
- [NativeWind Docs](https://www.nativewind.dev/)

---

## 🎨 Code Style Guide

### File Naming
- Components: `PascalCase.tsx` (e.g., `PropertyCard.tsx`)
- Hooks: `camelCase.ts` with `use` prefix (e.g., `useSupabase.ts`)
- Utilities: `camelCase.ts` (e.g., `utils.ts`)
- Types: `index.ts` in `types/` folder

### Component Structure
```typescript
// 1. Imports
import { View, Text } from "react-native";
import { useRouter } from "expo-router";
import { Property } from "@/types";

// 2. Types
interface Props {
  property: Property;
}

// 3. Component
export default function PropertyCard({ property }: Props) {
  // 4. Hooks
  const router = useRouter();
  
  // 5. State
  const [loading, setLoading] = useState(false);
  
  // 6. Effects
  useEffect(() => {
    // ...
  }, []);
  
  // 7. Handlers
  const handlePress = () => {
    // ...
  };
  
  // 8. Render
  return (
    <View>
      <Text>{property.title}</Text>
    </View>
  );
}
```

### Styling
```typescript
// Use NativeWind classes
<View className="bg-white rounded-2xl p-4" />

// Use style prop for complex styles
<View
  className="bg-white"
  style={{
    shadowColor: "#000",
    shadowOpacity: 0.08,
    elevation: 3,
  }}
/>
```

---

## 🔮 Future Enhancements

### Features to Add
- [ ] User reviews and ratings
- [ ] Property comparison
- [ ] Mortgage calculator
- [ ] Virtual tours (360° images)
- [ ] Chat with agent (in-app messaging)
- [ ] Property alerts (price drops, new listings)
- [ ] Advanced filters (parking, amenities, etc.)
- [ ] Map view for browsing
- [ ] Share property via social media

### Technical Improvements
- [ ] Add unit tests (Jest)
- [ ] Add E2E tests (Detox)
- [ ] Implement pagination for property lists
- [ ] Add caching for offline support
- [ ] Optimize image loading (lazy loading, placeholders)
- [ ] Add analytics (Mixpanel, Amplitude)
- [ ] Implement push notifications
- [ ] Add error boundary components
- [ ] Set up CI/CD pipeline
- [ ] Add Sentry for error tracking

### Admin Dashboard
- [ ] User management
- [ ] Property analytics
- [ ] Bulk operations
- [ ] Content moderation
- [ ] Performance metrics

---

## 💡 Pro Tips

### Development
1. **Use Expo Go** for quick testing on real devices
2. **Enable Fast Refresh** for instant updates
3. **Use TypeScript** for better type safety
4. **Check console logs** for debugging
5. **Use React DevTools** for component inspection

### Performance
1. **Optimize images** before uploading (compress, resize)
2. **Use FlatList** for long lists (virtualization)
3. **Memoize expensive computations** with useMemo
4. **Avoid inline functions** in FlatList renderItem
5. **Use useCallback** for event handlers

### Security
1. **Never commit .env** to git
2. **Always use RLS policies** for sensitive data
3. **Validate input** on both client and server
4. **Use HTTPS** for all API calls
5. **Keep dependencies updated** for security patches

### Code Quality
1. **Write descriptive variable names**
2. **Keep components small and focused**
3. **Extract reusable logic to hooks**
4. **Add comments for complex logic**
5. **Use TypeScript interfaces** for props

---

## 🤝 Contributing

### Making Changes
1. Create a new branch: `git checkout -b feature/my-feature`
2. Make your changes
3. Test thoroughly
4. Commit: `git commit -m "Add my feature"`
5. Push: `git push origin feature/my-feature`
6. Create pull request

### Code Review Checklist
- [ ] Code follows style guide
- [ ] TypeScript types are defined
- [ ] No console.logs in production code
- [ ] Error handling is implemented
- [ ] Loading states are handled
- [ ] UI is responsive
- [ ] No warnings in console
- [ ] Tested on iOS and Android

---

## 📞 Support

### Getting Help
1. Check this documentation first
2. Search GitHub issues
3. Check Stack Overflow
4. Ask in project Discord/Slack
5. Open a new issue with details

### Reporting Bugs
Include:
- Steps to reproduce
- Expected behavior
- Actual behavior
- Screenshots/videos
- Device/OS information
- Console logs

---

## 📄 License

This project is for educational purposes.

---

**Happy Coding! 🚀**

Need help? Start with `00-OVERVIEW.md` and work your way through the docs in order. Each guide builds on the previous one to give you a complete understanding of the codebase.
