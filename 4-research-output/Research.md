# Research: How would we implement ToastTab API in this?

_Generated on 2026-02-06_

## Answer

Integrating the ToastTab POS API would replace the current mock Firebase data layer for menus, orders, and locations with real restaurant data. The key challenge is architectural: ToastTab API calls **must go through a backend server** (authentication tokens, credit card encryption, and API keys cannot be exposed in a client-side React Native app). This means introducing a backend proxy/Cloud Functions layer, then adapting the existing service files (`menuService.ts`, `orderService.ts`, `locationService.ts`) to call your backend instead of the mock Firestore. The existing type system (`MenuItem`, `MenuCategory`, `CustomizationGroup`, `Order`) maps reasonably well to ToastTab's data model, but will need new fields for ToastTab GUIDs and modifier handling.

## Detailed Explanation

### 1. Current Architecture (What Gets Replaced)

The app currently has a clean service layer abstraction:

```
Screens → Services (menuService, orderService, locationService) → Mock Firestore
```

- **`src/services/menuService.ts`** — Reads menu categories, items, and customization groups from mock Firestore collections (`menu_categories`, `menu_items`, `customization_groups`)
- **`src/services/orderService.ts`** — Creates orders in mock Firestore, validates promo codes locally
- **`src/services/locationService.ts`** — Reads store locations from mock Firestore
- **`src/services/firebase/index.ts`** — In-memory mock Firestore implementation
- **`src/services/firebase/seedData.ts`** — Seeds demo data (12 menu items, 2 customization groups, 3 locations, promo codes)

The good news: the service layer is already a clean abstraction. Screens don't interact with Firestore directly — they call `menuService.getCategories()`, `orderService.createOrder()`, etc. This means **you only need to change the service implementations**, not the screens.

### 2. Backend Requirement (New Infrastructure)

ToastTab API requires **server-side authentication** (OAuth 2.0 tokens that must be refreshed, plus credit card encryption with a Toast-provided key). You **cannot** call ToastTab APIs directly from the React Native app.

**Option A: Firebase Cloud Functions** (fits the existing Firebase architecture)
```
React Native App → Cloud Functions (HTTPS) → ToastTab API
```

**Option B: Standalone Express/Node server** (if you want to decouple from Firebase)

Either way, you'd create endpoints like:
- `GET /api/menus/:restaurantGuid` → proxies ToastTab Menus API v3
- `POST /api/orders` → proxies ToastTab Orders API
- `GET /api/restaurants/:guid` → proxies ToastTab Restaurants API
- `POST /api/orders/:guid/prices` → proxies ToastTab `/prices` endpoint
- `POST /api/payments/authorize` → handles credit card encryption + authorization

### 3. ToastTab API Key Concepts

**Authentication:**
- OAuth 2.0 client credentials flow
- Tokens must be refreshed before expiry (backend handles this)
- Nine required API scopes for ordering: `orders.orders:write`, `menus.channel:read`, `credit_cards.authorization:write`, `stock:read`, etc.

**Restaurant GUIDs:**
- Each Toast restaurant location has a globally unique identifier (GUID)
- This maps to the app's `Location.id` — you'd add a `toastGuid` field to the `Location` type

**Menus API (v3 required for ordering):**
- Returns fully resolved menus for a location
- Includes menu groups → items → modifier groups (nested up to 3 levels)
- Has a `/metadata` endpoint for change detection (poll every 1-5 min or use webhook)
- Must respect `Availability` objects for time-based menu visibility

**Orders API:**
- `POST` to create orders with dining option (takeout for this app), guest info, line items
- Must call `/prices` endpoint first to get Toast-calculated taxes/totals
- Order items reference Toast menu item GUIDs, modifier GUIDs

**Payment:**
- Credit card data encrypted with Toast-provided public key
- Authorization requests distinguish new cards (`END_USER`) vs saved cards (`PARTNER_VAULT`)
- Must include `tipAmount`

### 4. Type System Mapping

The existing types in `src/types/index.ts` map to ToastTab concepts:

| Z Cafe Type | ToastTab Equivalent | Changes Needed |
|---|---|---|
| `Location` | Restaurant | Add `toastGuid: string`, map hours from Toast schedule format |
| `MenuCategory` | Menu Group | Add `toastGuid: string`, handle Toast availability objects |
| `MenuItem` | Menu Item | Add `toastGuid: string`, `plu?: string`, map `imageURL` from Toast media |
| `CustomizationGroup` | Modifier Group | Add `toastGuid: string`, add `minSelections`/`maxSelections`/`allowsDuplicates`, support nested modifiers (3 levels) |
| `CustomizationOption` | Modifier Option | Add `toastGuid: string`, add pre-modifiers (extra/light/no) |
| `Order` | Toast Order | Add `toastOrderGuid?: string`, map status from Toast order states |
| `CartItem` | Toast Selection | Map to Toast selection format with modifier group GUIDs |

### 5. Service Layer Changes

#### `menuService.ts` — Before and After

**Before** (reads from mock Firestore):
```typescript
getCategories: async () => {
  const snap = await firestore.collection('menu_categories').orderBy('sortOrder', 'asc').get();
  return snap.docs.map(doc => ({ ...doc.data(), id: doc.id }));
}
```

**After** (calls your backend, which calls ToastTab):
```typescript
getCategories: async (restaurantGuid: string) => {
  const response = await fetch(`${API_BASE}/api/menus/${restaurantGuid}`);
  const toastMenu = await response.json();
  return toastMenu.menuGroups.map(mapToastGroupToCategory);
}
```

You'd add mapping functions to translate Toast's data format to your app's types.

#### `orderService.ts` — Key Changes

**`createOrder`** becomes a two-step process:
1. Call `/prices` endpoint to get Toast-calculated totals (tax, discounts)
2. Submit order with encrypted payment data

**`validatePromoCode`** — Toast has its own discount/promo system via the Configuration API. The `promoCodes` value in the Discount object (added in their recent 2026 API update) replaces the local promo code validation.

#### `locationService.ts` — Key Changes

Locations would be fetched from Toast Restaurants API instead of Firestore. Each location stores a `toastGuid` that's used for all subsequent API calls.

### 6. Config Changes

`src/constants/config.ts` would need:
```typescript
export const ToastTabConfig = {
  API_BASE_URL: 'https://your-backend.com/api', // your proxy server
  // Toast GUIDs for each Z Cafe location
  RESTAURANT_GUIDS: {
    'montrose-collective': 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
    'heights-boulevard': 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
    'downtown': 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
  },
} as const;
```

Actual Toast API credentials (`clientId`, `clientSecret`) stay **server-side only**.

### 7. Implementation Phases

**Phase 1: Backend Proxy**
- Set up Cloud Functions or Express server
- Implement Toast OAuth 2.0 authentication + token refresh
- Create proxy endpoints for menus, orders, restaurants

**Phase 2: Menu Sync**
- Update `menuService.ts` to call backend
- Add GUID-based mapping functions (Toast → app types)
- Implement menu change detection (webhook or `/metadata` polling)
- Handle stock/availability via Toast Stock API

**Phase 3: Order Submission**
- Update `orderService.ts` to call `/prices` then submit orders
- Map cart items to Toast selection format
- Handle Toast order status webhooks for real-time updates

**Phase 4: Payment**
- Integrate Toast credit card encryption
- Build payment authorization flow
- Handle tip amounts

**Phase 5: Keep Non-Toast Features on Firestore**
- Z Society, Wordle, events, rituals, memory game — these are app-specific features that stay on Firestore
- Only menu, orders, locations, and promo codes move to ToastTab

### 8. What Stays on Firestore

The mock Firestore (or real Firestore when deployed) continues to power:
- User profiles and authentication (`AuthContext`)
- Z Society membership whitelist
- Wordle game state and leaderboard
- Houston Events
- Daily Rituals
- Memory Game
- Promo banners (display-only, separate from Toast promo codes)

## Related Services

| Service | Relationship to ToastTab Integration |
|---|---|
| **menuService** | Primary: reimplemented to call Toast Menus API v3 via backend |
| **orderService** | Primary: reimplemented to call Toast Orders API + `/prices` via backend |
| **locationService** | Primary: reimplemented to call Toast Restaurants API via backend |
| **firebase/index.ts** | Partially replaced: menu/order/location data moves to Toast; other collections stay |
| **firebase/seedData.ts** | Partially replaced: no longer need to seed menu, items, customizations, locations, promo codes |
| **CartContext** | Consumer: cart-to-order mapping needs Toast GUID fields |
| **LocationContext** | Consumer: selected location needs Toast restaurant GUID |
| **AuthContext** | Unaffected: stays on Firebase Auth |
| **config.ts** | Extended: add Toast API base URL and restaurant GUIDs |
| **types/index.ts** | Extended: add Toast GUID fields to Location, MenuItem, MenuCategory, CustomizationGroup, Order |

## Key Files

```
- app/src/services/menuService.ts — Reimplemented: fetch menus from ToastTab via backend
- app/src/services/orderService.ts — Reimplemented: submit orders to ToastTab via backend
- app/src/services/locationService.ts — Reimplemented: fetch locations from ToastTab via backend
- app/src/services/firebase/index.ts — Partially replaced: keeps non-menu/order collections
- app/src/services/firebase/seedData.ts — Partially replaced: remove menu/order seed data
- app/src/types/index.ts — Extended: add Toast GUID fields
- app/src/constants/config.ts — Extended: add ToastTab config
- app/src/store/CartContext.tsx — Consumer: cart items need Toast-compatible format
- app/src/store/LocationContext.tsx — Consumer: location needs toastGuid field
- app/App.tsx — May need initialization changes (Toast menu prefetch, etc.)
- [NEW] backend/ — New backend server for Toast API proxy (Cloud Functions or Express)
- [NEW] app/src/services/toastTabClient.ts — HTTP client for calling your backend proxy
```

## Discovery Path

- **Documentation Locator** identified Menu, Order, Firebase, and Location docs as directly relevant (menu syncing, order submission, data layer)
- **Service Locator** identified 25 files across services, screens, components, and tests that would be affected
- **Web search** confirmed ToastTab uses OAuth 2.0 auth, requires server-side integration, uses GUIDs for all entities, and has a Menus v3 API required for ordering partners
- **WebFetch** of Toast developer docs confirmed: 9 required API scopes, `/prices` endpoint for tax calculation, credit card encryption requirement, modifier group constraints (min/max selections, 3-level nesting), stock webhook for availability
- **1 iteration** was sufficient — the codebase has a clean service abstraction layer, making the integration path clear
