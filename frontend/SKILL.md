---
name: fleetcraft-frontend
description: >
  FleetCraft frontend development rules. Use this skill whenever modifying
  React components, TypeScript files, Tailwind styles, or any file in the
  Fleetcraft frontend repo. Also use when adding new pages, modifying API
  calls, or debugging data display issues. This prevents the recurring bug
  of components reading from Supabase directly instead of the Fleet API.
---

# FleetCraft Frontend — Development Rules

---

## 1. API Calls — Fleet API Only

> **Lesson learned:** Four components (VehicleTracking, DispatchControlCenter, CreateDispatchModal, VehicleMap) were reading from Supabase directly instead of the Fleet API. This caused ghost dispatch rows and stale data.

### The Rule
Every data fetch goes through the Fleet API at `api.myfleetcraft.com`. No Supabase client, no Supabase realtime subscriptions, no direct Supabase queries.

```typescript
// CORRECT — Fleet API
const response = await fetch(
  `https://api.myfleetcraft.com/api/containers/list?org_id=${orgId}`
);

// WRONG — Supabase direct (this is dead code, remove if found)
const { data } = await supabase.from('containers').select('*');
```

### If you find Supabase imports in the frontend:
- `@supabase/supabase-js` → remove
- `createClient()` with Supabase URL → remove
- `.from('table').select()` → replace with Fleet API fetch
- Supabase realtime `.subscribe()` → replace with polling or remove

---

## 2. Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | React 18 + TypeScript |
| Styling | Tailwind CSS |
| Build | Vite |
| Maps | Leaflet (vessel tracking) |
| Routing | React Router |

### Frontend serves from:
- `myfleetcraft.com` → nginx → `/var/www/fleetcraft-frontend/dist/`

---

## 3. Deploy

Build locally, SCP to droplet. Run from LOCAL machine:

```bash
cd c:\Users\siree\OneDrive\FleetCraft\FleetCraft Git\Fleetcraft
npm run build
scp -r dist/* root@178.128.64.97:/var/www/fleetcraft-frontend/dist/
```

After deploy: hard-refresh browser (Ctrl+Shift+R).

---

## 4. Key Patterns

### Dates — UTC formatting
Use `getUTCMonth()` / `getUTCDate()` / `getUTCFullYear()` for all date display. Using `toLocaleDateString()` causes off-by-one day errors because the database stores UTC timestamps.

```typescript
function formatUTCDate(dateStr: string): string {
  const d = new Date(dateStr);
  return `${d.getUTCMonth() + 1}/${d.getUTCDate()}/${d.getUTCFullYear()}`;
}
```

### Container status badges
Map `ui_status` to badge colors:

| ui_status | Badge |
|-----------|-------|
| `IN_TRANSIT` | Blue — on vessel |
| `AT_PORT` | Yellow — at terminal |
| `OUT_FOR_DELIVERY` | Green — gate out |
| `EMPTY_RETURNED` | Gray — lifecycle complete |

### Dispatch button visibility
Show "Dispatch" button ONLY when `available_for_pickup = true` AND `user_status = 'active'` AND no existing dispatch for that container.

### org_id
Always pass `org_id` as query parameter on every API call. Primary test org: `f8107db3-ecaa-48e1-968d-0e89c6dd8f62`.

---

## 5. Container Filter Tabs (user_status)

ContainerTracking.tsx displays containers in tabs based on `user_status`:

```
[Active (12)] [Held (2)] [Flagged (1)] [Dismissed (3)] [Archived (8)]
```

### API calls per tab:
| Tab | API Call |
|-----|----------|
| Active (default) | `GET /api/containers/list?org_id=X&status=active` |
| Held | `GET /api/containers/list?org_id=X&status=held` |
| Flagged | `GET /api/containers/list?org_id=X&status=flagged` |
| Dismissed | `GET /api/containers/list?org_id=X&status=dismissed` |
| Archived | `GET /api/containers/list?org_id=X&status=archived` |

### Count badges
Each tab shows a count badge. Fetch counts on page load with `?status=all` or separate count endpoint, then update locally when actions are taken.

### Action buttons per tab:
| Current Tab | Available Actions |
|-------------|-------------------|
| Active | [Hold] [Flag] [Dismiss] [Archive] |
| Held | [Restore] [Archive] |
| Flagged | [Restore] [Archive] |
| Dismissed | [Restore] |
| Archived | [Restore] |

### Action endpoints:
| Action | Endpoint | On success |
|--------|----------|------------|
| Archive | `POST /api/containers/archive` | Remove from current tab, increment Archived count |
| Dismiss | `POST /api/containers/dismiss` | Remove from current tab, increment Dismissed count |
| Hold | `POST /api/containers/hold` | Remove from Active tab, increment Held count |
| Flag | `POST /api/containers/flag` | Remove from Active tab, increment Flagged count |
| Restore | `POST /api/containers/restore` | Remove from current tab, increment Active count |

### Error handling:
On any action error, show toast with the server's error message. Do NOT remove the row from the list — let the user retry.

---

## 6. Numeric Display Safety

Never call `.toFixed()` directly on API values. Postgres numeric types come as strings via JSON.

```typescript
// WRONG — crashes with "toFixed is not a function"
value.toFixed(2)

// RIGHT — coerce to number first
Number(value || 0).toFixed(2)

// Or with null guard
value ? Number(value).toFixed(2) : '—'
```

This applies to `daily_rate`, `demurrage_amount`, and any numeric column.

---

## 7. API Field Coverage

When adding a new data card or section to the UI, verify the API endpoint returns the fields you need.

Check: `curl` the endpoint, parse the JSON, confirm your fields exist in the response.
If missing: update the server.js SELECT query for that endpoint.

The geofence detention card was built but showed blank because `GET /dispatches` didn't include `queue_start_at`, `queue_stop_at`, `pickup_geofences` in its SELECT.

Also check the frontend mapper — if `fetchDispatchLookup()` or similar functions manually map fields from the API response into a typed object, new fields must be added to the mapper too.

---

## 8. Supabase Is Dead

No Supabase imports, no Supabase client, no Supabase queries. All data comes from fleet-api.

If a component imports from `supabase/client` or uses `supabase.from()`, it **will** read from a different database than what fleet-api writes to — causing FK violations, stale data, and ghost records.

All data fetches use:
```typescript
const response = await fetch(`${API_BASE}/api/endpoint`);
```

If you find a Supabase import in any component, replace it with a fleet-api fetch call immediately.

---

## 9. Download Report Pattern

The Download Report button in `ContainerDetailsModal` generates a printable HTML report using `window.print()`. No external PDF libraries needed.

- Opens new browser window with print-friendly HTML (white bg, black text, clean tables)
- Auto-triggers `window.print()` → user saves as PDF
- Data comes from `container` + `dispatch` objects already in the modal — no new API calls
- Photos load as `<img>` from DO Spaces URLs
- Includes: container details, shipment timeline, dispatch milestones, geofence & detention data, milestone photos
- Print CSS hides browser chrome, margins set to 0
- Pattern: build HTML string → `window.open` → `document.write` → `setTimeout(print, 600)`

If adding new data sections to the report, add them to `generateDispatchReport()` in `ContainerTracking.tsx`. Keep inline CSS only — no Tailwind in the print window.
