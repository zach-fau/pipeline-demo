# Direct Answer: Implementing ToastTab API

> This is the conversational answer returned to the developer after the research agent completed its analysis. The full structured report is in `Research.md`.

---

## Research: Implementing ToastTab API

### Direct Answer

Integrating the **ToastTab POS API** would replace the current mock Firebase data layer for menus, orders, and locations with real restaurant data. The biggest architectural requirement: **you need a backend server** — ToastTab uses OAuth 2.0 authentication, credit card encryption, and API secrets that cannot live in a client-side React Native app. The good news is the existing service layer (`menuService`, `orderService`, `locationService`) is already a clean abstraction, so screens won't need to change — only the service implementations get swapped.

### Architecture: What Changes

```
Current:  Screens → Services → Mock Firestore
After:    Screens → Services → Your Backend (Cloud Functions or Express) → ToastTab API
```

### The 5 Phases

**Phase 1 — Backend Proxy** (new infrastructure)
- Firebase Cloud Functions or Express server
- Toast OAuth 2.0 token management
- Proxy endpoints: `/api/menus/:guid`, `/api/orders`, `/api/restaurants/:guid`, `/api/payments/authorize`

**Phase 2 — Menu Sync**
- Rewrite `menuService.ts` to call your backend instead of Firestore
- Map Toast menu groups/items/modifiers → app's `MenuCategory`/`MenuItem`/`CustomizationGroup` types
- Toast requires Menus API **v3** for ordering partners
- Poll `/metadata` every 1-5 min or subscribe to menus webhook for changes

**Phase 3 — Order Submission**
- Rewrite `orderService.ts` — orders become a 2-step process:
  1. Call `/prices` to get Toast-calculated taxes/totals (required — you must use their numbers)
  2. Submit order with dining option (takeout), guest info, line items with Toast GUIDs

**Phase 4 — Payment**
- Encrypt credit card data with Toast-provided public key
- Handle authorization requests and tip amounts

**Phase 5 — Keep non-Toast features on Firestore**
- Z Society, Wordle, events, rituals, memory game all stay on Firestore
- Only menu, orders, locations, and promo codes move to ToastTab

### Type System Impact

The existing types map well to Toast concepts but need GUID fields:

| Z Cafe Type | Toast Equivalent | New Fields |
|---|---|---|
| `Location` | Restaurant | `toastGuid` |
| `MenuCategory` | Menu Group | `toastGuid`, availability |
| `MenuItem` | Menu Item | `toastGuid`, `plu` |
| `CustomizationGroup` | Modifier Group | `toastGuid`, `minSelections`, `maxSelections`, nested modifiers (3 levels) |
| `Order` | Toast Order | `toastOrderGuid` |

### Key Requirement: Toast Integration Partner Program

To get API access, you must apply through Toast's [Integration Partner Program](https://pos.toasttab.com/partners/integration-partner-application) or request custom API access from a Toast account representative. Nine API scopes are required for ordering integrations.

Full report written to `documentation/Research.md`.

Sources:
- [Toast Platform Docs](https://doc.toasttab.com/)
- [Toast API Overview](https://doc.toasttab.com/doc/devguide/apiOverview.html)
- [Building an Online Ordering Integration](https://doc.toasttab.com/doc/cookbook/apiIntegrationChecklistOrdering.html)
- [How to Build a Toast Integration](https://doc.toasttab.com/doc/devguide/portalHowToBuildAToastIntegration.html)
- [Toast API Change Log](https://doc.toasttab.com/doc/relnotes/devPortalApiChangeLog.html)
