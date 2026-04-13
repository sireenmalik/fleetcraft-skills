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

## 5b. Vessel Tracker Data Sources

The vessel tracker map shows TWO categories:
1. **Tracked vessels** — `GET /vessels/with-containers` (VWC). Only vessels with IN_TRANSIT containers.
2. **Terminal vessels** — `GET /api/vessels/at-terminals`. Moored at our terminals (proximity match).

**WRONG:** Load all AIS vessels from vessels_cache (shows hundreds of irrelevant ships)
**RIGHT:** VWC (our containers) + terminal moored only

### Vessel card display
- Show `pod_name` (FTU destination) first, AIS `destination` as fallback
- **WRONG:** "Destination: CAVAN>JPTYO" — this is the vessel's multi-port route
- **RIGHT:** "Destination: Tacoma" — this is where the container gets off

### Vessel pills on container page
Client-side derived from `filteredContainers.map(c => c.vessel_name)`. Shows all vessels referenced by containers in the current tab. This is correct — do NOT filter by IN_TRANSIT here.

---

## 5c. Add Container Modal — Direct Terminal Pickup Toggle

The Add Container modal (`AddContainerModal` in `ContainerTracking.tsx`) has a toggle for direct terminal pickup:

### Toggle states:
| State | Background | Border | Label | Subtitle |
|-------|-----------|--------|-------|----------|
| OFF (default) | `#FAEEDA` (amber) | `#EF9F27` | "Direct terminal pickup — OFF" | "Skip ocean tracking — container is already at the terminal" |
| ON | `#FCEBEB` (red) | `#E24B4A` | "Direct terminal pickup — ON" | "No ocean tracking — container added directly as available at terminal" |

### OFF mode (FTU tracking):
- Multi-line textarea for container numbers / B/L numbers
- Carrier (SCAC) dropdown with auto-detect
- Submits to `POST /containers/track-batch` (existing FTU flow)

### ON mode (direct-add):
- Single container number input
- Terminal dropdown (required): Husky Terminal — Tacoma, PCT — Tacoma, T18 — Seattle, T5 — Tacoma, FCTEST — Bellevue (Test)
- Vessel name input (pre-filled "Direct Request", editable)
- Shipping line input (optional)
- Info banner: "No FTU charges — container added directly to your active list"
- Red submit button: "Add to terminal"
- Calls `POST /api/containers/quick-add`
- Success toast: "Container added to \<terminal\> — ready for dispatch"

### Rules:
- Toggle defaults to OFF — FTU tracking is the normal path
- Switching the toggle resets the form
- Direct-add containers appear immediately in the Active tab with AT_PORT status (yellow badge)
- Dispatch button appears immediately (`available_for_pickup = true`)
- Adding a new terminal requires 3 changes: `geofences` table row + `terminalCodeMap` in server.js + `TERMINAL_OPTIONS` in ContainerTracking.tsx

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

---

## 10. GPS Track Map (Leaflet)

The container detail modal has a 2-column layout:
- Left: info cards, timeline, geofence, photos, download report
- Right: Leaflet map showing dispatch GPS track

Map behavior:
- Always visible — shows terminal pin if no GPS data, Tacoma default if nothing
- Per-dispatch — filters events by dispatch_id, not container_number
- Milestone dots color-coded: green (pickup), red (terminal), blue (delivery), purple (return)
- Dashed polyline connecting all points
- Corridor geofence drawn as red dashed rectangle
- Sticky position — stays visible while scrolling left column
- Mobile (<768px) stacks vertically, map below content at 250px height

Data source: `container_events` with lat/lng, fetched from `GET /api/containers/{cn}/events`
Map library: Leaflet 1.9.4 from cdnjs (free, no API key)
Tiles: OpenStreetMap

---

## 11. Fleet Map Component Rules (v1.3.1–v1.3.8 lessons)

### Map viewport rules
- Initial load: auto-fit ALL trucks + ALL pickup + delivery points with padding.
- On driver selection: fit truck + pickup + delivery + 320px top padding (for InfoBubble).
- On poll refresh (every 10s): do NOT re-fit. Track selection in a ref (`fittedForDriverRef`). Only re-fit when `selectedDriverId` changes.
- User pan/zoom is sacred — never steal it on data refresh.

### Marker scale pattern
- Default: 20x20 pulsing LED dot with tiny label (`CNTR · J.Smith`, 9px).
- Selected: 40x40 full badge with name, ETA, heading arrow, route overlay.
- Only ONE truck expanded at a time. All others stay compact.
- Pulsing animation only when Live + Moving (no pulse on Stale/Stopped).

### Connection + Motion states (replaces OFFLINE)
- Connection: Live (green, < 5 min), Stale (amber, 5–30 min), No signal (gray, > 30 min).
- Motion: Moving (speed > 5 km/h), Stopped.
- Heading arrow only renders while Moving.
- Display as two pills in InfoBubble: `[Live] [Stopped]`.

### Fleet map visibility gate
- Trucks appear on map ONLY when they have an active dispatch (status NOT IN `completed`, `cancelled`, `pending`).
- No dispatch = no marker. Cancel dispatch = marker disappears within 10s.
- The fleet map shows containers on trucks, not trucks. A truck without a dispatched container has no business on the map.

### Status-to-color map
- GREEN: `assigned`, `en_route_pickup`, `chassis_info_required`
- AMBER: `at_terminal`, `in_queue`, `loaded`, `gate_out`
- BLUE: `en_route_delivery`, `at_delivery`, `delivered`
- GRAY: `in_transit_parked`, `in_transit_parked_return`
- PURPLE: `empty_en_route_return`, `at_return_terminal`, `chassis_returned`, `returned`
- HIDDEN: `pending`, `completed`, `cancelled`

### HERE Maps in frontend
- Map tiles load via HERE JS SDK script tags (client-side, no CORS issue).
- Route polylines fetched via `GET /api/dispatch/route-preview` (server proxy, not direct HERE call).
- HERE API key hardcoded in frontend for map tiles only — this is intentional (client-side tiles).
- Polyline decoded using `fromFlexiblePolyline` (HERE flexible polyline format).

### Breadcrumbs (Spec 0013, not yet built)
- Query `driver_positions WHERE dispatch_id = X ORDER BY recorded_at`
- Draw as small green dots on FleetMap over the HERE route polyline
- Green dots over blue line = completed journey
- Uncovered blue = remaining journey
- Truck LED = current live position at the head of the trail

Depends on the SecureStore dispatch_id tagging fix in the driver app —
without it, positions upload with `dispatch_id = NULL` and breadcrumbs can't be filtered per dispatch.
