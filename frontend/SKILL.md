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

### Dates — Terminal Time (Pacific) Formatting

> **Lesson learned (Spec 0024):** Using `toLocaleDateString()` / `toLocaleString()` with no timezone argument renders timestamps in the viewer's browser timezone. A remote user in New York sees Eastern time; a developer in London sees GMT. All NWSA operations are at West Coast terminals. All timestamps must show Pacific time.

#### The Rule

**Use `terminalTime.ts` for ALL container/dispatch timestamp rendering.** Never use `toLocaleDateString`, `toLocaleTimeString`, or `toLocaleString` without an explicit `timeZone: 'America/Los_Angeles'` argument on any container or dispatch timestamp.

#### Shared utility: `src/app/utils/terminalTime.ts`

Three exports — use the right one for the right context:

| Function | Returns | Use when |
|---|---|---|
| `formatTerminalDate(dt)` | `"M/D"` in PT | Date-only fields: LFD, discharge date, archive date, pod_eta date portion. Parses date part directly — no timezone shift on midnight UTC values. |
| `formatTerminalDateTime(dt)` | `"M/D H:MMa"` in PT | Timestamps with time: milestone `occurred_at`, dispatch `created_at`, GPS `recorded_at`, `archived_at` |
| `formatTerminalDateTimeShort(dt)` | `"M/D H:MMa"` or `null` in PT | Same as above but returns `null` instead of `"—"` — use in table cells where you need conditional rendering |

All three use `Intl.DateTimeFormat` with `timeZone: 'America/Los_Angeles'`. DST is handled automatically.

#### What NOT to replace

Two functions in `ContainerTracking.tsx` are intentional UTC renderers and must NOT be replaced:

| Function | Why kept |
|---|---|
| `formatUTCDate(dt)` | LFD and other date-only fields where we want the UTC calendar date exactly (no conversion) — e.g. `"2026-04-17"` → `"4/17"` always, regardless of timezone |
| `formatUTCDateTime(dt)` | Same — UTC-explicit datetime formatting |

The `new Date(event.timestamp).toLocaleString()` in `WebhookActivityLog` is intentional (browser-local time is acceptable for live activity logs).

#### gridUtils.ts — PT formatting required

`gridUtils.ts` exports its own date helpers used by `ContainerGrid.tsx`. These must also use Pacific Time — they are NOT covered by `terminalTime.ts`.

| Function | Status | Rule |
|---|---|---|
| `formatShortDateTime` | ✅ PT (Intl.DateTimeFormat, America/Los_Angeles) | All dispatch milestone columns — EN ROUTE, INGATE, LOADED, OUTGATE, etc. |
| `formatShortDate` | ✅ PT (Intl.DateTimeFormat, America/Los_Angeles) | ETA column and EMPTY IN column |
| `formatUTCDate` | ✅ intentional UTC | LFD column only — do not change |

**Rule:** if you modify `gridUtils.ts`, never revert `formatShortDateTime` or `formatShortDate` back to `d.getMonth()` / `d.getHours()` / `d.getDate()` patterns. Those are browser-local and will break for any viewer outside Pacific Time. Always use `Intl.DateTimeFormat` with `timeZone: 'America/Los_Angeles'`.

**The ETA cell in ContainerGrid.tsx** uses `formatShortDateTime(pod_eta || eta)` — not `formatShortDate`. This is intentional: ETA is a datetime field and dispatchers need the arrival time, not just the date. Never revert it to date-only.

#### The midnight-UTC date shift bug

`formatTerminalDate` parses the date part directly:
```typescript
const datePart = str.split('T')[0].split(' ')[0]; // "2026-04-17"
const [y, m, d] = datePart.split('-').map(Number);
return `${m}/${d}`;
```

This prevents: `"2026-04-17T00:00:00.000Z"` → `"4/16"` in Pacific time (midnight UTC = 5 PM PDT previous day). Date-only fields must never pass through timezone conversion.

#### When adding new timestamp fields to the UI

1. Is it a date-only field (no time component, stored as midnight UTC)? → `formatTerminalDate`
2. Is it a datetime that should render in PT? → `formatTerminalDateTime` or `formatTerminalDateTimeShort`
3. Is it a UTC-explicit date where you want the UTC calendar date? → `formatUTCDate` (rare)
4. Is it a live/activity log where browser-local is fine? → `toLocaleString` is acceptable

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

## 8. Supabase Fully Purged (April 2026)

**Zero Supabase references remain in live code.** Commit `d9840ab` removed 5,694 lines across 39 files. All frontend components now read/write exclusively via Fleet API (`api.myfleetcraft.com`). Bundle size dropped ~230 KB. The only remaining grep hits are `//` comments documenting the migration — historical context, not executing code.

### Stubbed components (hidden from nav, render "Coming Soon" cards)

These still exist as files but don't touch any backend. Rebuild against Fleet API when needed:

- `src/app/components/fleet/FleetManagement.tsx` — driver/truck/chassis CRUD UI
- `src/app/components/dispatch/DispatchControlCenter.tsx` — dispatch management UI
- `src/app/components/vessel/CollectorControlPanel.tsx` — AIS collector admin
- `src/app/components/vessel/CollectionSettingsPanel.tsx` — collector settings

### Known Fleet API stubs (return empty, UI handles gracefully)

- **NotificationSystem** — no `GET /api/alert-logs` endpoint yet. `fetchData()` returns empty; logs table shows zero entries; subscribers tab is fully functional.
- **VehicleMap** — `dispatch_routes` polyline overlay is stubbed to `[]` until a `GET /api/dispatch-routes` endpoint exists. All other map features (truck markers, driver positions, milestone dots, breadcrumbs) work.

### RULE — zero tolerance

**No new code may import from Supabase.** Any `getSupabaseClient`, `supabase.from()`, or `@supabase/supabase-js` import is a bug. Every data read and write goes through Fleet API endpoints. If a grep for `supabase` in live code returns anything other than a comment, ship a fix.

All data fetches use:
```typescript
const response = await fetch(`${API_BASE}/api/endpoint`);
```

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

### Map Components inventory

| Component | Tile source | Used by | User-editable | Persists on save | Draggable pin |
|-----------|-------------|---------|---------------|------------------|---------------|
| FleetMap | HERE Maps JS (vector) | Fleet dashboard — live truck/route view | NO | NO | NO |
| VehicleMap | HERE Maps JS (vector) | Vehicle tracking tab | NO | NO | NO |
| ContainerTracking map | Leaflet (OpenStreetMap) | Container detail modal — milestone timeline | NO | NO | NO |
| GPS Track Map | Leaflet (OpenStreetMap) | Historical driver positions replay | NO | NO | NO |
| **Terminal modal map** | **HERE Maps JS (vector)** | **Terminal edit/create modal** | **YES** | **YES** | **YES** |
| **Geofence modal map** | **HERE Maps JS (vector)** | **Geofence edit/create modal (polygon/corridor types)** | **YES (polygon)** | **YES** | **NO — vertex markers** |

**"Terminals & Geofences" is the current user-visible label** for the page that hosts both modals. Previously called "Geofence Dashboard"; renamed in commit `6c861f2`. The component, file, and route id (`GeofenceDashboard`, `src/app/components/geofence/GeofenceDashboard.tsx`, tab id `geofenceDashboard`) kept their original identifiers — only display strings changed.

### Terminal pin-drop map (April 2026 — commit `a7c2f39`)

The Terminal edit/create modal in `GeofenceDashboard.tsx` embeds a HERE Maps instance with a draggable pin for setting the exact gate location.

**Features:**
- **Road map view** (`layers.vector.normal.map`) at **zoom 14** (~1 km across, enough to see terminal + approach streets).
- **Draggable pin** — dispatcher drops on the exact gate entrance. `dragend` writes rounded (5-decimal) coords back into `form.lat/lng`.
- **Expand/collapse button** (top-right of map, `Maximize2`/`Minimize2` icons) toggles height 250 px → 500 px with a 250 ms CSS `height` transition. `map.getViewPort().resize()` fires after the animation so HERE rescales its canvas.
- **Live lat/lng display** below the map (5-decimal = ~1 m precision).
- **Manual lat/lng inputs** kept below the map for power users with exact coords.
- **No Geocode button** — pin-drop is the primary input. Server-side auto-geocode (`POST`/`PATCH /api/terminals`) still resolves the address on save as a fallback when the dispatcher only typed an address.

**Data flow on save:**
1. Pin `lat/lng` → `PATCH /api/terminals/:id`.
2. Server writes to `terminals.lat/lng`.
3. `propagateTerminalCoordsToDispatches()` updates `pickup_lat/lng/pickup_address` on every active dispatch with that `terminal_code` (COALESCE-guarded against null).
4. `computeAndStoreRoute()` recomputes `dispatches.route_polyline` for each affected dispatch so the fleet-map blue line redraws to the new coords.
5. Fleet map and driver app pick up new coords within one polling cycle (~10 s).

**Why pin-drop instead of pure geocoding:**

Street addresses don't map precisely to terminal gate locations. A terminal's mail/registration address is often on a different road than where truck drivers actually enter — the gate can be 300–1000 m away. HERE Geocoding returns the address-range midpoint, which lands on the wrong road for industrial yards. The dispatcher knows which gate the trucks use; drop the pin there. This is the fix for the April 2026 HUSKY incident (address changed from `1101 Port of Tacoma Rd` to `2325 Lincoln Ave` but routing kept going to the old coords — see backend Error Pattern #17).

**HERE bootstrap pattern:**
Same as FleetMap (`ensureHereMapsLoaded`, `loadHereScript`, `loadHereCss` duplicated at top of `GeofenceDashboard.tsx`). If a third call site appears, extract to `src/utils/hereMaps.ts`. HERE API key is hardcoded (client-exposed tile key — same justification as FleetMap).

### Geofence polygon editor (April 2026 — commit `c9713cd`)

The Geofence edit/create modal in `GeofenceDashboard.tsx` embeds a second HERE Maps instance with **draggable vertex markers** for reshaping detection polygons. Only renders for `type === 'polygon'` or `type === 'corridor'` — rectangle and circle types keep their existing numeric input forms.

**Features:**
- **300 px map, expandable to 600 px** via a corner button (same pattern as Terminal modal; 280 ms CSS height transition + `getViewPort().resize()`).
- **Vector road view** at zoom 14, centred on existing vertices or falling back to parent terminal's `lat/lng`.
- **Polygon drawn** with green dashed stroke `rgba(29,158,117,0.85)` and 18% fill; corridor renders as a dashed polyline (no ring closure).
- **Green 18 px vertex markers** at each corner, draggable. HERE `dragstart` disables map pan; `drag` writes screen-to-geo coords; `dragend` rounds to 5 decimals and pushes into `vertices` state. Polygon reshapes live during drag via `setGeometry(new H.geo.Polygon(ls))`.
- **"+ Add vertex" button** toggles click-to-place mode: cursor → crosshair, a green banner appears, next `tap` on the map appends a new vertex and exits the mode. Dispatcher can then drag it into position.
- **Remove vertex** via `✕` button in the vertex list below the map. Blocked with a toast when only 3 vertices remain ("Polygon needs at least 3 vertices").
- **Vertex list** (monospace, scrollable, max-height 120 px) shows each corner as `N. lat, lng` for quick visual audit.
- **"Raw JSON (advanced)" collapsible** below the list — renders current vertices serialized; editable as a fallback when the editor has no vertices.

**Save wiring** — `buildCoordinates()` in the modal prefers editor vertices over the JSON textarea. For polygons it appends the first vertex at the end (`[...vertices, vertices[0]]`) to close the ring, matching how Postgres stores geofence polygons (first point duplicated at end). For corridors it uses the raw vertex array (polyline, no ring closure). No backend changes — `POST`/`PATCH /api/geofences` already accepts `coordinates` as JSONB.

**Critical context — polygon edits DON'T affect active dispatches.** The `dispatches.pickup_geofences` column is a JSONB snapshot of the polygon at dispatch creation time (Spec 0013 Rule 11). Edits made in this editor only affect **newly-created dispatches**. A mid-shift polygon change never reshapes what an in-flight driver sees — intentional, to keep detention timers deterministic. If you need to force a refresh on active dispatches, you'd need a separate propagation pass (not implemented).

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

---

## 12. Container Grid — TanStack Table (Spec 0026)

> **Updated:** 2026-04-16 (Spec 0026 — Milestones Lifecycle v3)

**AUTHORITY: fleetcraft-specs/0026-fleetcraft-milestones-spec.md**

The Container Tracking page uses `@tanstack/react-table` for the main data table.

### Key files
- `src/app/components/container/ContainerGrid.tsx` — TanStack Table with 28 columns
- `src/app/components/container/ColumnVisibilityMenu.tsx` — column visibility dropdown
- `src/app/components/container/gridUtils.ts` — shared column width defaults and format helpers
- `src/app/hooks/usePersistedColumnState.ts` — localStorage persistence

### Grid Column Order — Spec 0026 (28 columns)

This table is a CONTRACT. Adding, removing, renaming, or reordering columns requires updating Spec 0026 FIRST.

| Pos | Column ID | Header | DB field | Group |
|-----|-----------|--------|----------|-------|
| 1 | container | CONTAINER | container_number | OCEAN |
| 2 | status | STATUS | ui_status | OCEAN |
| 3 | vessel | VESSEL | vessel_name | OCEAN |
| 4 | eta | ETA | pod_eta | OCEAN |
| 5 | discharged | DISCHGD | pod_discharged_at | TERMINAL |
| 6 | holds | HOLDS | holds/customs/freight/terminal | TERMINAL |
| 7 | lfd | LFD | last_free_day | TERMINAL |
| 8 | dispatched | DSPTCHD | created_at | DISPATCH |
| 9 | driver | DRIVER | driver_name | DISPATCH |
| 10 | truck | TRUCK | truck_name | DISPATCH |
| 11 | en_route_term | EN ROUTE | en_route_pickup_at | DISPATCH |
| 12 | chassis_info | CHSS INFO | chassis_number (text, 3 modes) | DISPATCH |
| 13 | geofence | GEOFENCE | queue_start_at | DISPATCH |
| 14 | ingate | INGATE | pickup_arrived_at | DISPATCH |
| 15 | loaded | LOADED | loaded_at | DISPATCH |
| 16 | outgate | OUTGATE | pickup_completed_at | DISPATCH |
| 17 | queue | QUEUE | computed: outgate − min(geofence, ingate) | DISPATCH |
| 18 | en_route | EN RT DLV | en_route_delivery_at | DELIVERY |
| 19 | delivery_eta | DELIVERY ETA | delivery_eta_triggered_at | DELIVERY |
| 20 | at_deliv | AT DLVRY | delivery_arrived_at | DELIVERY |
| 21 | pod_field | POD | delivery_completed_at | DELIVERY |
| 22 | delivered | DLVRD | actual_delivery_at | DELIVERY |
| 23 | complete | COMPLETE | completed_at | DELIVERY |
| 24 | en_route_rtn | EN RT RTN | en_route_return_at | RETURN |
| 25 | at_term_rtn | AT TRM RTN | return_arrived_at | RETURN |
| 26 | chassis_rtn | CHSS RTN | return_completed_at | RETURN |
| 27 | empty_in | EMPTY IN | empty_terminated_at | RETURN |
| 28 | actions | ACTIONS | — | ACTIONS |

**CONTAINER, STATUS, ACTIONS are pinned — cannot be hidden.**
**empty_in is FTU lifecycle data, not a Spec 0026 driver milestone.**

Removed columns (do NOT re-add):
- chassis (CHASSIS) — merged into CHSS INFO detail modal
- parked (PARKED) — Spec 0026 removed this milestone
- parked_rtn (PRKD RTN) — Spec 0026 removed this milestone

localStorage key: `fc-container-grid-v3` (bumped on any column change)

### Rules
- CONTAINER and STATUS are always pinned left. ACTIONS is always pinned right. None of these can be hidden.
- RETURN group (4 columns) is hidden by default.
- Column widths and visibility persist to localStorage key `fc-container-grid-v3`.
- All cell rendering reuses existing format functions. Do not duplicate formatting logic.
- When adding a new column: update Spec 0026 FIRST, then add to ContainerGrid.tsx column defs AND update DEFAULT_VISIBILITY and DEFAULT_SIZING in usePersistedColumnState.ts AND add to COLUMN_GROUPS in gridUtils.ts.
- The old HTML `<table>` markup is gone. Do not recreate it. All table rendering goes through TanStack.

### Hard-won formatting rules

**CSS specificity:** Inline React styles lose to Tailwind preflight. All grid dimensions (font-size, line-height, padding, height, table-layout, border-collapse) must be in a scoped `<style>` block keyed by `.fc-container-grid` using `!important`. Inline styles are fallback only.

**Table width with `tableLayout: fixed`:** Table width must equal `table.getTotalSize()` exactly. Never set `minWidth: 100%` — it causes the browser to redistribute width across all columns when one is resized.

**Column width enforcement:** Always render a `<colgroup>` with one `<col style={{ width }}/>` per visible column. `<colgroup>` widths are authoritative; `<th>` widths alone are hints the browser can override.

**Sort vs resize event conflict:** Never put sort `onClick` on the whole `<th>`. Scope it to an inner `<div>` wrapping only the label and sort icon. The resize handle is an absolutely-positioned sibling with `stopPropagation`.

**Pinned column borders:** Use `border-right`/`border-left` on the last pinned cell, never `inset box-shadow` on sticky cells — shadows stack into a solid vertical bar across all rows.

**Row height control:** Set explicit `height` on `<tr>` elements via CSS `!important` with `box-sizing: border-box` on th/td. Without this, the tallest cell content wins.

**Font-size hierarchy:** Use distinct CSS rules per role, ordered from broad to specific:
1. `.fc-grid td, td *` — body text
2. `.fc-grid th, th *` — header text
3. `.fc-grid .fc-status-cell *` — status pill (smaller)
4. `.fc-grid .fc-vessel-sub` — secondary metadata line (smallest)
5. `.fc-grid .fc-hold-badge` — hold pill (smallest)

**localStorage schema versioning:** Key by `fc-container-grid-v{N}`. On read, validate every saved key against current column defs and drop unknowns. Bump version suffix on breaking column key changes to force clean defaults. Always provide "Reset to defaults" in the UI.

**Column keys are a contract per Spec 0026:** Column IDs match dispatch field names written by the driver app (en_route_term, chassis_info, ingate, loaded, outgate, etc.). Visual order can change freely. Column keys cannot be renamed without updating Spec 0026 + driver-app + backend. Renaming an ID silently drops a user's saved customization.

**Build vs deploy:** `npm run build` creates `dist/` locally. The droplet serves the previous bundle until `scp` overwrites it. Before debugging styling complaints, verify the console hash matches the latest local build.

---

## 13. Public-Facing Pages — Minimal Routing Pattern

> **Added:** 2026-04-14 (Spec 0015 Phase 3)

The dispatcher dashboard uses a single `useState<Tab>` for navigation — there is **no React Router**. When a customer-facing page needs a dedicated URL (e.g. `/track/:token` for the public delivery tracking page), introduce the route at the `main.tsx` boot branch rather than installing `react-router-dom`.

### Pattern (spec 0015 reference)

```tsx
// src/main.tsx
import { createRoot } from "react-dom/client";
import App from "./app/App.tsx";
import { PublicTrackingPage } from "./app/components/tracking/PublicTrackingPage.tsx";

const path = window.location.pathname;
const match = path.match(/^\/track\/([0-9a-fA-F]+)\/?$/);

function Root() {
  if (match) return <PublicTrackingPage token={match[1]} />;
  return <App />;
}
createRoot(document.getElementById("root")!).render(<Root />);
```

### Why this pattern
- **Zero impact on the existing app.** The dispatcher's currentPage state, auth flow, sidebar, and tab navigation stay untouched.
- **Zero new dependencies.** `react-router-dom` adds ~20 KB gzip and a coupling cost we don't need for two or three public routes.
- **Full code-splitting friendly** if growth demands it — swap the direct import for `React.lazy()` + Suspense.

### Server-side requirement
Nginx must serve `index.html` for the public path. The existing `fleetcraft-frontend` config already has `try_files $uri $uri/ /index.html` as a SPA fallback — any `/track/<anything>` URL loads the same bundle, then `main.tsx` routes client-side.

### When to escalate to a real router
If you ever need more than 3 distinct public routes, or nested routes with shared layout, or URL-driven modals, migrate to `react-router-dom` (wrap `App` in `BrowserRouter`, convert tab state to routes). Don't build a homegrown router in `main.tsx` beyond a handful of `pathname.match()` branches.

**Note (2026-04-14):** Spec 0016 added a second public route tree — `/portal/*` — and we hit the nested-routing case. Rather than importing `react-router-dom`, the `main.tsx` branch delegates `/portal/*` to a small `PortalApp` component that does its own `pathname` switch over the 5 portal sub-routes. Works cleanly for now; if a third public surface lands, collapse all three into one real router.

### Public page rules (must hold for every `/track/*`-style route)
- **No auth wrapper, no sidebar, no dispatcher state.** Page renders as a standalone card at `max-w-[480px]` on a gray background.
- **Data comes from sanitized endpoints only.** `GET /api/track/:token` must strip driver names, GPS coords, org IDs, and financial fields before responding (backend/SKILL.md enforces this server-side).
- **Polling instead of WebSockets in v1.** 30-second interval is enough for milestone status updates.
- **Time zone is Pacific explicitly.** `toLocaleString('en-US', { timeZone: 'America/Los_Angeles' })` — customers should not see raw UTC.
- **Error copy is generic.** "This tracking link is not valid or has expired." Never reveal whether a token has been seen, is truly expired, or never existed.

---

## 14. Customer Portal — `/portal/*` (Spec 0016 Phase 2, 2026-04-14)

Authenticated customer-facing portal. Magic-link auth, separate from dispatcher auth. Lives in `src/app/pages/portal/` — each page is its own file, shared layout via `PortalLayout`.

### File map

| Path | Purpose |
|---|---|
| `src/app/hooks/usePortalAuth.ts` | localStorage JWT hook, key `fc-customer-jwt`, auto-logs-out on expiry |
| `src/app/pages/portal/PortalApp.tsx` | tiny pathname router — dispatches to the 5 portal pages |
| `src/app/pages/portal/PortalLogin.tsx` | email input → `POST /api/customer-auth/magic-link` |
| `src/app/pages/portal/PortalVerify.tsx` | `?token=X` callback → JWT → redirect to `/portal/dashboard` |
| `src/app/pages/portal/PortalLayout.tsx` | header (logo + name + "via {org}" + logout) + top nav |
| `src/app/pages/portal/PortalDashboard.tsx` | delivery list, 30 s polling, active + collapsible completed |
| `src/app/pages/portal/PortalDeliveryDetail.tsx` | milestone timeline + HERE map, 15 s polling |
| `src/app/pages/portal/PortalPreferences.tsx` | 6 per-trigger toggles (4 email + 2 SMS) |

### Auth lane isolation (don't break this)

- **localStorage key for portal JWT: `fc-customer-jwt`** — distinct from the dispatcher's `fc-auth-token`. Both sessions coexist in the same browser; logging into one does NOT log out of the other. Never share these keys or fall back from one to the other.
- **JWT role must be `'customer'`** — validated server-side by `requireCustomerRole`. Client decodes the payload for display only (customer name + org name + expiry check).
- **On expiry, `usePortalAuth` calls `logout()`** which clears localStorage and redirects to `/portal`. Pages that show the layout also redirect via `PortalLayout`'s `useEffect` guard — belt and suspenders against stale state.

### Routing

Main routing decision happens once in `main.tsx` at boot:
```
pathname match /track/<hex>  → PublicTrackingPage (Spec 0015)
pathname starts /portal      → PortalApp (Spec 0016)
anything else                → dispatcher <App>
```

`PortalApp.tsx` then runs its own pathname switch over the 5 portal routes. All navigations use `window.location.assign()` for a full reload — the portal tree is tiny and this avoids any `pushState` / `popstate` complexity. Back/forward still works because `PortalApp` listens to `popstate` and re-renders.

### HERE map in PortalDeliveryDetail

Reuses the loader pattern from `VehicleMap.tsx` (not extracted to a shared helper — third call site would be the trigger per §11). Renders three layers:

1. **Route polyline** from `dispatches.route_polyline`, decoded via `H.geo.LineString.fromFlexiblePolyline`, cyan (`#0891b2`), 4 px.
2. **Delivery geofence circle**, dashed stroke, 8 % fill, centered on `delivery_lat/lng` with `delivery_radius_m` (defaults 2000 m).
3. **Delivery pin** — single cyan dot at the delivery center.

**No live driver dot in v1** — Spec 0016 Rule 13. Customers don't see the driver's real-time GPS in the portal. If the map fails to load or the delivery isn't geocoded, the detail page still renders with a "Map unavailable" placeholder.

### Polling intervals (from spec Rule 14 + Spec 0017 Rule 12)
- Dashboard (Delivering Today section): **60 s**
- Delivery detail: **15 s** (more critical as driver approaches)
- Public tracking page (`/track/:token`): **30 s**
- Upcoming + Completed sections: refresh on page load only (not polled)
- No WebSockets in v1. Polling is fine for this volume.

### Live Delivery Tracking components (Spec 0017, 2026-04-15)

Six components live under `src/app/components/portal/`. All six are pure presentation — the computed `live_tracking` object comes from the backend (`lib/liveTracking.js`). The frontend never derives ETA / progress / banner state locally.

| Component | File | Used by |
|---|---|---|
| `LiveBanner` | `portal/LiveBanner.tsx` | delivery-day cards + detail page + public tracking page (when `live_tracking.banner_type !== null`) |
| `RouteProgressBar` | `portal/RouteProgressBar.tsx` | same — amber at terminal, green en route, truck glyph at fill edge |
| `EtaWindow` | `portal/EtaWindow.tsx` | right-aligned ETA time over the narrowing bracket window |
| `MilestoneFeed` | `portal/MilestoneFeed.tsx` | vertical timeline — pulsing dot on the last-completed milestone, filled check for prior, outline circle for pending |
| `FiveDotTimeline` | `portal/FiveDotTimeline.tsx` + `deriveFiveDotStep(...)` | simplified 5-step for upcoming cards (loaded → at port → dispatched → in transit → delivered) |
| `LiveTrackingCard` | `portal/LiveTrackingCard.tsx` | composes the above four + Share-ETA modal; used for each `is_delivery_day` delivery on the dashboard |

**Dashboard sections (Spec 0017 §9):**
- **Stats bar** — 4 tiles (Delivering today / At port / On vessel / Completed) driven by `stats` on the `/api/portal/deliveries` response.
- **Delivering today** — `LiveTrackingCard` per `is_delivery_day` delivery, polls every 60 s.
- **Upcoming** — static card with `FiveDotTimeline`, no polling.
- **Completed** — collapsed by default, 0.75 opacity when expanded.

**Rules for the components:**
1. **Never compute live_tracking locally.** Backend is the source. If you need a new field, add it to `lib/liveTracking.js` + the response schema.
2. **Never render `driver_first_name` on the public tracking page.** Spec 0017 Rule 7. Server enforces it via `includePrivate: false`; don't reference the field in `PublicTrackingPage.tsx`.
3. **Share-ETA modal** lives inside `LiveTrackingCard` — it POSTs `/api/portal/share-eta` with the customer's JWT. 429 means the dispatch hit the 5-per-hour rate limit; show the user a friendly message instead of retrying.
4. **Five-dot timeline activeStep** comes from `deriveFiveDotStep({ dispatchStatus, containerUiStatus })`. Add new statuses there, not inline in cards.

### Sanitization (enforced server-side)

The backend's `GET /api/portal/deliveries` + `GET /api/portal/deliveries/:id` do NOT return: `driver_id`, `driver_name`, `driver_phone`, `pickup_lat/lng`, `delivery_lat/lng` (oops — detail DOES return delivery lat/lng for the map, that's the one exception), `org_id`, `origin_lat/lng`, financial fields. The frontend doesn't do any additional filtering — if it's in the response, it's considered safe to render.

### Preferences are locked-down

`PortalPreferences` PATCHes only the 6 `pref_*` booleans. Attempting to send `notification_email`, `notification_phone`, `delivery_radius_m`, etc. in that PATCH returns 400 server-side. If a customer wants to change their email/phone/address, the footer instructs them to contact their trucking company.

### When adding a new portal route
1. Add the page component in `src/app/pages/portal/YourPage.tsx`.
2. Add a `pathname` match branch in `PortalApp.tsx`.
3. Wrap it in `<PortalLayout>` if it needs auth + shared chrome; otherwise render raw (like `PortalLogin` / `PortalVerify`).
4. If you need a new backend endpoint, wire it under `/api/portal/*` and protect with `requireCustomerRole` middleware.
5. Respect double-scoping: any SQL you write on the backend must include `WHERE customer_id = $1 AND org_id = $2` — never trust frontend-supplied IDs.

---

## 15. Customers Page (dispatcher dashboard) — 2026-04-14

Admin-facing list + detail view for customers, wired into the dispatcher sidebar at `activeTab === 'customers'`.

### File: `src/app/components/customer/CustomersPage.tsx`

Renders:
- Table of all customers from `GET /api/customers` — shows name, contact, notification email/phone, radius, notifications toggle, active delivery count, last login.
- Row click → slide-out detail drawer with editable notification fields + radius slider + master toggle.
- **"View as customer"** button — calls `POST /api/customers/:id/admin-view-link` and opens the returned `/portal/verify?token=...` URL in a new tab. Dispatcher is signed into the portal AS that customer. Uses the same single-use magic-link flow customers self-service with.
- **"Send login link"** button — triggers the public magic-link email to the customer's `notification_email`.
- **"Add Customer"** modal — POSTs `/api/customers`, then PATCHes portal-specific fields (`delivery_radius_m`, `notification_email`, `notification_phone`).

### Endpoint surface used by this page

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/customers` | List — now returns `active_delivery_count` + `last_login_at` |
| POST | `/api/customers` | Create (existing, unchanged) |
| PATCH | `/api/customers/:id` | Update notification fields + radius (existing, unchanged) |
| POST | `/api/customers/:id/admin-view-link` | **NEW** — mints magic link, returns `/portal/verify?token=...` URL |
| POST | `/api/customer-auth/magic-link` | Public email send (reused for "Send login link") |

### "View as customer" = portal testing tool

This is the feature that lets a dispatcher walk through every customer's real portal view across all their dispatches without emailing magic links to themselves. One-click onboarding verification and support. Because the URL contains a single-use token that expires in 15 min, it's safe to share (e.g., paste into a support ticket) without exposing the customer's account permanently.

### Rule: don't let this become a customer-accessible endpoint
`POST /api/customers/:id/admin-view-link` MUST stay on the dispatcher-only path of the API. It's currently unauthenticated because all dispatcher endpoints are — when dispatcher auth is added (follow-up spec), this endpoint goes under it. A customer who could hit this would be able to impersonate other customers. Protect it alongside the rest when that happens.

---

## 16. Terminal Time — All Container Timestamps in Pacific Time (Spec 0024)

> **Added:** 2026-04-16

All container and dispatch timestamps in the FleetCraft dashboard are rendered in `America/Los_Angeles` (Pacific Time) regardless of the viewer's browser locale. This is enforced via `src/app/utils/terminalTime.ts`.

### Components using terminalTime

| Component | File | Functions used |
|---|---|---|
| Container Tracking (main table + modals) | `ContainerTracking.tsx` | `formatTerminalDate`, `formatTerminalDateTimeShort` |
| Archived Container Detail modal | `ArchivedContainerDetail.tsx` | `formatTerminalDate`, `formatTerminalDateTime` |
| Container Grid | `ContainerGrid.tsx` | `formatTerminalDate`, `formatTerminalDateTimeShort` (if date columns present) |
| Dispatch report (print) | `generateDispatchReport()` in ContainerTracking.tsx | `formatTerminalDateTimeShort` |

### Rule — no new browser-local timestamps

Any new component that renders a container timestamp, dispatch timestamp, or milestone timestamp must import from `terminalTime.ts`. Do NOT use `new Date(x).toLocaleString()` without explicit `timeZone: 'America/Los_Angeles'`.

### Why not a global Intl config

`Intl.DateTimeFormat` doesn't have a global default timezone. Every format call must be explicit. `terminalTime.ts` is the centralization point — add new helpers there rather than inlining timezone strings throughout the codebase.

### Multi-tenant future

When FleetCraft supports terminals in other time zones, `terminalTime.ts` should accept an optional `timeZone` parameter that defaults to `'America/Los_Angeles'`. The display layer passes the org's terminal timezone; the utility handles conversion. All call sites already use the shared helper, so the migration is a one-file change.

---

## 17. Dashboard Dispatch Columns (Spec 0025)

> **Added:** 2026-04-16

### Pickup section column order

```
EN ROUTE | CHASSIS | CHSS INFO | GEOFENCE | INGATE | QUEUE | LOADED | OUTGATE | PARKED
```

Column definitions live in `src/app/components/container/ContainerGrid.tsx`. Widths and visibility defaults live in `src/app/hooks/usePersistedColumnState.ts`. Group membership (for the DISPATCH-orange header band) lives in `src/app/components/container/gridUtils.ts` `COLUMN_GROUPS`.

### Key computed columns

- **GEOFENCE** — raw timestamp from `dispatches.queue_start_at` (auto, geofence fire). Rendered as `HH:MM` in 24h PT via `toLocaleTimeString('en-US', { timeZone: 'America/Los_Angeles', hour: '2-digit', minute: '2-digit', hour12: false })`. Shows `—` if the dispatch has no geofence configured or the polygon hasn't fired yet.
- **QUEUE** — `outgate − min(geofence, ingate)` = total terminal dwell. Formula:
  ```
  anchor = min(queue_start_at, pickup_arrived_at)  (whichever came first)
  duration = pickup_completed_at − anchor           (trip complete)
           | Date.now() − anchor                    (in-flight, live tick)
  ```
  In-flight state renders in amber. Completed state renders in DISPATCH color.
  Shows `—` if neither geofence nor ingate has fired yet.

All other columns show formatted timestamps from the dispatch record via `formatShortDateTime` (or the inline helpers in ContainerGrid) — all in PT per §4.

### The QUEUE formula — frontend-computed, not server-computed

The QUEUE column is **NOT** `dispatches.queue_minutes` from the server. It is a frontend-computed duration:

```
queue = pickup_completed_at − earlier(queue_start_at, pickup_arrived_at)
```

This measures total terminal presence from first contact (geofence fire OR ingate tap, whichever came first) to departure (outgate). It works for both geofence-capable terminals and terminals without a configured polygon (falls back to `pickup_arrived_at`).

**Why not `queue_minutes`?** The server-computed `queue_minutes` uses only `queue_stop_at − queue_start_at`, which returns NULL whenever `queue_start_at` is missing (no geofence). The frontend formula degrades gracefully — always returns a useful number as long as either anchor is present.

### Rule — never revert the QUEUE formula

Do not revert the QUEUE cell to `loaded_at − pickup_arrived_at`. That was the original buggy formula — it measures terminal dwell (ingate-to-seal), not queue. If the "seal-to-ingate" metric is needed, add a new column called TERMINAL DWELL; don't overwrite QUEUE.

### Rule — GEOFENCE column must be PT-aware

`toLocaleTimeString` without `timeZone: 'America/Los_Angeles'` will render in browser-local time and violate Spec 0024. Any new time-only column must follow this pattern.

### Rule — new column checklist

Adding a new column to the grid requires THREE coordinated edits:

1. `ContainerGrid.tsx` — add the column def in `buildColumns()` at the desired position
2. `usePersistedColumnState.ts` — add `<id>: <width>` to `DEFAULT_COLUMN_SIZING` and `<id>: true` (or false) to `DEFAULT_COLUMN_VISIBILITY`
3. `gridUtils.ts` — add `<id>: '<GROUP>'` to `COLUMN_GROUPS` (OCEAN, TERMINAL, DISPATCH, DELIVERY, RETURN)

Missing any of these causes: wrong default width, column not persisted in localStorage, or column falls out of the DISPATCH orange header band.
