---
name: fleetcraft-database
description: >
  FleetCraft database schema and data flow rules. Use this skill whenever
  writing SQL migrations, altering tables, adding columns, modifying
  container-sync.js column mappings, debugging data that shows in one
  layer but not another, or working with the archived_containers table.
  Also use when any sync worker's INSERT or UPDATE statement is being
  modified. Read this BEFORE any schema or sync change.
---

# FleetCraft Database — Schema and Sync Rules

---

## 1. Database Topology

```
SQLite (container-registry.db)     PostgreSQL (fleetcraft_db)
  ├── containers                     ├── containers (synced from SQLite)
  └── (write buffer only)            ├── archived_containers
                                     ├── container_events (event ledger)
                                     ├── container_exclusions (tombstone)
                                     ├── vessels
                                     ├── vessels_with_containers (cache)
                                     ├── dispatches
                                     ├── drivers
                                     ├── trucks
                                     ├── chassis
                                     ├── chassis_assignments
                                     ├── driver_positions
                                     ├── alert_subscribers
                                     ├── alert_logs
                                     ├── geofences (planned)
                                     └── navigation_order
```

### Who Owns What
- **SQLite** is a write buffer for container data ONLY. FTU webhooks write here first.
- **Postgres** is the source of truth for everything. container-sync.js pushes SQLite → Postgres every 30 seconds.
- **Exception:** Direct-add containers (`data_source = 'direct'`) bypass SQLite entirely — written directly to Postgres via `POST /api/containers/quick-add`. container-sync never sees them. Direct-add containers have: `data_source='direct'`, `terminal_code` derived from `terminal_name` mapping (e.g., "PCT" → "PCT", "Husky Terminal" → "HUSKY").
- Dispatches, drivers, trucks, chassis, events, exclusions — Postgres ONLY, no SQLite copy.

---

## 2. container-sync.js Column Mapping

When adding a column to the containers table, you MUST update THREE places:

1. **SQLite schema** — `ALTER TABLE containers ADD COLUMN <n> <type>`
2. **Postgres schema** — `ALTER TABLE containers ADD COLUMN <n> <type>`
3. **container-sync.js** — add the column to BOTH the SELECT from SQLite AND the INSERT/UPDATE to Postgres

> **Lesson learned:** `findteu_shipment_id` was added to both databases but was missing from container-sync.js INSERT/UPDATE column list. It never synced. FTU unregistration broke because the ID was always null in Postgres.

### Sync query structure in container-sync.js:

```javascript
// SQLite read — every synced column must be listed here
const rows = sqliteDb.prepare(`
  SELECT container_number, org_id, ui_status, current_status,
         vessel_name, vessel_imo, terminal_name, pod_eta,
         pod_arrival, pod_discharged_at, pod_full_out_at,
         empty_terminated_at, last_free_day, customs_hold,
         freight_hold, terminal_hold, equipment_type,
         shipping_line_scac, data_source, findteu_shipment_id,
         archived_at, archive_reason, user_status, updated_at
         -- ADD NEW COLUMNS HERE
  FROM containers
  WHERE archived_at IS NULL
`).all();

// Postgres write — every synced column must ALSO be listed here
// AND in the ON CONFLICT SET clause
// EXCEPT user_status — NEVER in the SET clause
```

### The Sync Guard
container-sync.js must NEVER sync rows where `archived_at IS NOT NULL` to the active `containers` table. Those go to `archived_containers` via `syncArchivedContainers()`.

### The Tombstone Check
Before upserting, container-sync.js loads `container_exclusions` into a Set and skips any container in that Set. This happens BEFORE the SQL upsert guard — defense-in-depth.

---

## 2a. Terminal Coordinates Lifecycle (April 2026)

Terminals carry both a human-readable `address` and machine coordinates `lat/lng`. The two are kept in sync automatically so HERE Routing (which requires coordinates) and the dispatcher UI (which shows the address) never drift apart.

- `terminals.address` — human-readable, editable via the Geofence Dashboard Terminal modal.
- `terminals.lat` / `terminals.lng` — geocoded from `address` via HERE Geocoding, auto-updated on address change (commit `37ad6df`).
- **On address change:** `PATCH /api/terminals/:id` calls `hereGeocode(address)`, writes the new coords, then calls `propagateTerminalCoordsToDispatches()` which fans the new `{lat, lng, address}` out to every active dispatch with that `terminal_code`.
- **On direct lat/lng edit:** same propagation path fires (coords_changed trigger).
- **On terminal create:** same auto-geocode runs if caller supplied an address but no coords.

### Dispatch-side coordinate columns

Related columns on `dispatches` have different lifetimes. Key the design:

| Column | Kind | Behaviour |
|--------|------|-----------|
| `pickup_lat` / `pickup_lng` | LIVE | Propagated from `terminals.lat/lng` whenever the terminal's coords change (active dispatches only) |
| `pickup_address` | LIVE | Propagated alongside coords — stays in sync with `terminals.address` |
| `pickup_geofences` (JSONB) | FROZEN | Snapshot of the geofence polygon(s) at dispatch creation time. Later polygon edits do NOT affect existing dispatches. Intentional — detention timers must be deterministic |
| `origin_lat` / `origin_lng` | FROZEN | Captured from driver phone GPS at the moment of the "En Route to Pickup" tap (migration 014). Never changes post-tap. Anchors the detention evidence chain |
| `route_polyline` | FROZEN | Computed once via `computeAndStoreRoute()` at dispatch creation. Redrawn live only for the bright-blue current-leg line via `/api/dispatch/route-preview` |
| `delivery_address` | LIVE via direct PATCH | Set at dispatch creation from customer request; editable via `PATCH /api/dispatches/:id`. Retroactively captured in migration 018 |
| `delivery_lat` / `delivery_lng` | LIVE via direct PATCH | Set by `hereGeocode(delivery_address)` at creation; editable later via `PATCH /api/dispatches/:id`. Retroactively captured in migration 018 |
| `delivery_geofence_id` | FROZEN | Added by migration 013, populated by `POST /api/dispatches` (Spec 0015, 2026-04-14). FK to geofences row of `type='circle'` created at dispatch creation. Soft-deleted (`active=false`) when dispatch completes or cancels (Rule 13). Only non-null when `customer_id` is set on the dispatch |
| `eta_predicted_minutes` + siblings | LIVE | Re-read from HERE on every ETA polling call (3–10 min); no propagation needed |

**Why the split:** anything driver-observable at trip start (pickup address, geofence shape, origin, base route) freezes so the detention contract is stable. Anything that's just routing metadata (current coords, delivery destination, ETA) stays live so corrections propagate.

### Delivery tracking — Spec 0015 Phase 1 shipped (2026-04-14)

Backend wiring for delivery geofences, customer notifications, and the public tracking page is **live on the droplet** (Spec 0015 Phase 1):

- `POST /api/dispatches` with `customer_id` → creates a `type='circle'` row in `geofences` (dispatch_id set, terminal_code NULL), populates `dispatches.delivery_geofence_id`, `tracking_token` (64-char hex), `delivery_geofences` JSONB snapshot, and `delivery_radius_m`. Fires 'scheduled' notification via `lib/notifications.js`.
- `POST /api/driver/loads/:id/milestone` accepts `delivery_area_arrived` (idempotent — `delivery_arrived_at IS NULL` guard) and fires 'arriving' notification.
- `en_route_delivery` → 'en_route' notification with eta_minutes. `delivered` → 'delivered' notification.
- `GET /api/track/:token` — public, sanitized milestone view (no driver PII, no GPS coords, no org IDs).
- `PATCH /api/customers/:id` — updates notification email/phone/radius with validation (E.164, radius 100-10000, email regex).

**Still pending:** Driver app delivery-geofence detection (Phase 2), frontend public tracking page (Phase 3), Twilio SMS setup (Phase 4). Until Phase 2 ships, `delivery_area_arrived` only fires via direct milestone POST; no on-device detection yet.

### Customer portal columns (migration 022, 2026-04-14)

Spec 0016 Phase 1 added auth + preference columns to `customers` and one column to `organizations`:

| Column | Purpose |
|---|---|
| `customers.magic_link_token` | 64-hex single-use token for passwordless auth. Cleared by `verifyMagicLink()`. |
| `customers.magic_link_expires_at` | 15-min TTL from generation. |
| `customers.last_login_at` | Engagement signal, updated by verify. |
| `customers.magic_link_attempt_count` | Rate-limit counter. Resets when `window_start` > 1h ago. |
| `customers.magic_link_attempt_window_start` | Rolling 1-hour rate-limit window anchor. |
| `customers.pref_email_scheduled / en_route / arriving / delivered` | Per-trigger email opt-in. Default TRUE. |
| `customers.pref_sms_arriving / delivered` | Per-trigger SMS opt-in. Default TRUE. |
| `organizations.display_name` | Shown in portal "via {org}" badge. Falls back to `organizations.name` if NULL. |

**Rule — customer-editable vs org-managed** — the portal's `PATCH /api/portal/preferences` endpoint is restricted to the 6 `pref_*` booleans. `notification_email`, `notification_phone`, `delivery_radius_m`, `notifications_enabled` are org-managed via `PATCH /api/customers/:id` (dispatcher), NOT customer-editable. This prevents a compromised customer JWT from hijacking their own email/phone to intercept another customer's notifications.

### customer_locations — multi-address delivery points (migration 023, Spec 0018, 2026-04-15)

Customers can have 1..20 named delivery locations. Dispatch picks a location → delivery_address/lat/lng/radius/notes are snapshot onto the dispatch at creation. Replaces `customers.delivery_radius_m` (now deprecated, kept for backward compat with existing dispatches).

| Column | Notes |
|---|---|
| `name` | Short label for dropdowns: "Dock A", "Main Warehouse" |
| `address` | Geocoded via HERE fire-and-forget on create/edit |
| `lat` / `lng` | DOUBLE PRECISION. Null until geocode completes; null if it fails. |
| `radius_mi` | NUMERIC(4,1). CHECK 0.1–10.0. Converted to meters (`radius_mi * 1609.34`, rounded) when snapshotted to `dispatches.delivery_radius_m`. |
| `is_default` | Pre-selected in dispatch form. Partial unique index `WHERE is_default=true AND is_active=true` enforces one default per customer. First location auto-defaults on POST. |
| `is_active` | Soft-delete flag. Locations referenced by dispatches flip to false; unreferenced locations hard-delete. |
| `notes` | Free text. Copied to `dispatches.special_instructions` at create time when request doesn't override. |

**Rule — dispatch radius/address are FROZEN** — `POST /api/dispatches` resolves `customer_location_id` synchronously BEFORE INSERT: address + lat/lng + radius (in meters) + notes are copied onto the dispatch row. Editing the location afterward does NOT mutate existing dispatches. This is intentional: detention timers and notification geofences must be deterministic even when dispatcher moves a warehouse pin next quarter.

**Rule — customer_location_id is optional** — one-off typed addresses still work (NULL `customer_location_id`). In that path, `setImmediate` geocodes the typed string as before.

**Rule — radius conversion constant** — `const MILE_TO_METERS = 1609.34` at top of server.js. Use it consistently so frontend previews (in miles) and backend persistence (in meters) agree.

**Response shape on GET /api/customers** — enriched with `location_count` (active only) and `default_location` (row_to_json of id/name/address/radius_mi, or NULL when no default).

### delivery_notifications — audit log for customer notifications

Table created by migration 021. Every attempt (sent, failed, skipped) logs a row. Used for Rule 12 deduplication: `(dispatch_id, trigger, channel)` unique per sent notification. Do NOT query this table as a source of truth for WHAT notifications customers got — query `status='sent'` rows specifically.

**Ownership gotcha:** The table was initially created by `sudo -u postgres psql` which made postgres the owner, and the fleetcraft app role got "permission denied" on INSERT. Fix: migration 021 now includes `ALTER TABLE delivery_notifications OWNER TO fleetcraft` inside a DO block. **Always alter ownership on any table you create via sudo-postgres** — see deployment/SKILL.md §9.2.

### Trip stats + chassis ownership (migration 024, Spec 0021 Phase 1, 2026-04-15)

Driver-app-v2 introduces authoritative trip stats computed at dispatch completion and a chassis-ownership flag that controls the inspection-skip UX. All six columns live on `dispatches`.

| Column | Type | Purpose |
|---|---|---|
| `trip_moving_pct` | NUMERIC(4,1) | % of GPS ticks with speed > 2 m/s. Populated by `POST /api/dispatches/:id/compute-trip-stats`. |
| `trip_standing_minutes` | INTEGER | Minutes with speed ≤ 2 or NULL. 15s tick × standing_ticks / 60. |
| `trip_distance_miles` | NUMERIC(6,1) | Σ haversine between consecutive `driver_positions` rows, in miles. |
| `trip_total_minutes` | INTEGER | `completed_at − en_route_pickup_at` in minutes. |
| `trip_computed_at` | TIMESTAMPTZ | Latest compute timestamp. Idempotent — repeat calls overwrite. |
| `chassis_owner` | TEXT (default `'pool'`) | `'pool'` → rental chassis (DCLI/TRAC/Flexi-Van), full photo inspection required. `'tenant'` → company-owned, inspection skippable. Set at dispatch creation from truck/chassis record. |
| `delivery_eta_triggered_at` | TIMESTAMPTZ | Spec 0026 step 9. Delivery geofence trigger. Customer notification fires here. Separate from `delivery_arrived_at` (step 10). FROZEN — written once. Migration 026. |

**Rule — compute-trip-stats is idempotent and driver-authenticated.** Endpoint uses `requireAuth` (driver JWT), reads `driver_positions` between `en_route_pickup_at` and `completed_at`, and overwrites all five columns in one query. Safe to call twice (latest wins). Called fire-and-forget by the app on the final milestone — HTTP failure doesn't block completion.

**Rule — chassis_owner = 'pool' is the safe default.** Missing/unknown ownership means inspection is required; drivers only skip when `chassis_owner = 'tenant'` explicitly. Never flip the default to `'tenant'` — a compliance miss there is a real incident.

---

## 3. ui_status Values

These are FTU-derived. Do not invent new values.

| ui_status | Meaning | Set by |
|-----------|---------|--------|
| `IN_TRANSIT` | On vessel, en route | FTU webhook (default until discharged) |
| `AT_PORT` | Discharged at terminal | FTU 'discharged from' event, OR `POST /api/containers/quick-add` (direct-add) |
| `OUT_FOR_DELIVERY` | Gate out from terminal | FTU 'gate out full' event, OR driver milestone `gate_out` |
| `EMPTY_RETURNED` | Empty container returned | FTU 'gate in empty' event, OR driver milestone `completed` |

### Dispatch eligibility
Only containers where `ui_status = 'AT_PORT'` AND `user_status = 'active'` are dispatch-ready. Both conditions required.

### Direct-add containers

`POST /api/containers/quick-add` creates containers that are already at the terminal — they bypass FTU vessel tracking entirely. Full default set at INSERT time:

| Column | Value | Rationale |
|--------|-------|-----------|
| `data_source` | `'direct'` | Marks origin — skips FTU sync, skips AIS enrichment |
| `ui_status` | `'AT_PORT'` | Container is physically at the terminal at creation |
| `user_status` | `'active'` | Eligible for dispatch |
| `pod_discharged_at` | `NOW()` | Anchors detention calculations; direct-add has no FTU webhook to populate this |
| `vessel_name` | `'Direct Request'` | Synthetic vessel used for grouping in dashboards; no AIS/FTU enrichment applies |
| `terminal_code` | caller-supplied | Which terminal the container is at |
| `available_for_pickup` | `true` | Container is ready for pickup |

Because direct-add skips the normal FTU ingestion path, every timestamp that normally flows in from webhooks must be set synthetically at creation — otherwise downstream calculations (detention days, demurrage thresholds, dwell time) start from null and produce nonsense. The "Direct Request" vessel_name is intentional: it's the single grouping anchor so dashboards can render direct-add containers in vessel-grouped views alongside real vessels (see `/containers/vessels` — no IN_TRANSIT filter, so direct-add vessels show up).

---

## 4. user_status Values

These are user-intent. Only Fleet API endpoints write them.

| user_status | Meaning | Creates exclusion? | FTU unregister? |
|-------------|---------|-------------------|-----------------|
| `active` | Normal — sync writers update freely | No | No |
| `archived` | Removed from view, FTU unregistered | Yes | Yes |
| `dismissed` | Hidden but FTU still tracking | Yes | No |
| `held` | Flagged for attention, visible in Held tab | No | No |
| `flagged` | Marked for review, visible in Flagged tab | No | No |

### CHECK constraint (Postgres):
```sql
ALTER TABLE containers ADD CONSTRAINT chk_user_status
  CHECK (user_status IN ('active', 'archived', 'dismissed', 'held', 'flagged'));
```

---

## 5. container_exclusions Table (Tombstone)

Defense-in-depth. Sync writers check this BEFORE attempting upsert.

```sql
CREATE TABLE container_exclusions (
  container_number  TEXT NOT NULL,
  org_id            UUID NOT NULL,
  excluded_at       TIMESTAMPTZ DEFAULT NOW() NOT NULL,
  reason            TEXT NOT NULL DEFAULT 'archived',
  PRIMARY KEY (container_number, org_id)
);
```

### Rules:
- Written by Fleet API archive and dismiss endpoints
- Deleted by Fleet API restore endpoint
- Read by container-sync.js at start of each cycle
- `held` and `flagged` do NOT create exclusion records (containers stay visible in their tab)
- On restore, exclusion is deleted → next sync cycle resumes normal updates

---

## 6. FTU Data Gaps (Always Null)

These columns exist in both SQLite and Postgres but FTU never populates them:

| Column | Why null | Future source |
|--------|----------|---------------|
| `last_free_day` | Terminal-specific data | Tradlinx |
| `customs_hold` | Terminal-specific data | Tradlinx |
| `freight_hold` | Terminal-specific data | Tradlinx |
| `terminal_hold` | Terminal-specific data | Tradlinx |
| `yard_location` | Terminal-specific data | Tradlinx or direct |
| `voyage_number` | FTU doesn't provide | Carrier API |
| `vessel_mmsi` | FTU doesn't provide | AIS collector fills this |

Do NOT write code that depends on these being non-null from FTU. They will only have values when a second data source is integrated.

---

## 7. archived_containers Table

Permanent record of all archived containers. Never delete from this table.

### Flow:
```
containers (active) → user_status = 'archived' + archived_at set → container-sync moves to archived_containers → deleted from active containers
```

### Schema matches containers table plus:
- `archived_at` — when archived
- `archive_reason` — why (dispatch_completed, auto_empty_returned, auto_stale_ofd, manual, manual_archive)

### Archive trigger rules
- **VALID:** `EMPTY_RETURNED` older than 24 hours (reason: auto_empty_returned — normal lifecycle end)
- **VALID:** `OUT_FOR_DELIVERY` older than 7 days (reason: auto_stale_ofd — safety net for stuck containers)
- **VALID:** `dispatches.completed_at` older than 24 hours (reason: dispatch_completed — driver finished full cycle)
- **VALID:** Manual archive via POST /containers/archive (user decision)
- **INVALID:** `dispatches.status = 'cancelled'` (cancelled dispatch does NOT trigger archive)
- **INVALID:** `OUT_FOR_DELIVERY` older than 1 day (WRONG — container actively being delivered)
- **INVALID:** FTU `completed=true` (FTU stops tracking, NOT a business signal)
- **INVALID:** FTU subscription expiry
- **INVALID:** Any external API status flag alone

---

## 8. container_events Table (Event Ledger)

Append-only. Never update or delete rows.

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | Unique event ID |
| container_number | text | FK to containers |
| org_id | uuid | Multi-tenant scope |
| event_type | text | ftu_update, milestone, archive, dismiss, hold, flag, restore, etc. |
| source | text | ftu, driver, dispatcher, system |
| payload | jsonb | Full event data |
| occurred_at | timestamptz | When the event actually happened |
| created_at | timestamptz | When we recorded it |

### Rules:
- Every FTU webhook writes an event
- Every driver milestone writes an event
- Every user_status transition writes an event (archive, dismiss, hold, flag, restore)
- Events are the audit trail — if current state is wrong, rebuild from events

### NEVER bulk delete from container_events
> **Lesson learned:** A cleanup script deleted all container_events for archived containers,
> destroying the milestone audit trail (ingate timestamps, GPS coordinates, photos).
> The dispatches table still had the timestamps but the detailed event history was lost.
> container_events is append-only — INSERT only, no UPDATE, no DELETE, no bulk cleanup.
> If the table grows too large, archive old events to a separate table. Never delete.

---

## 9. vessels_with_containers Cache Rules

This cache table is owned by `vwc-sync.js` (single writer for data columns). `dispatcher-orchestrator.js` writes ONLY the `alerted_*` flag columns.

> **Lesson learned:** Three workers (vessel-sync, ftu-tracker, container-sync) all wrote to this table with different aggregation. ftu-tracker zeroed counts that container-sync set. Fix: single owner (vwc-sync.js).

### Container count query MUST filter by user_status:
```sql
SELECT vessel_name, COUNT(*) as container_count
FROM containers
WHERE vessel_name IS NOT NULL
  AND (user_status IS NULL OR user_status = 'active')
GROUP BY vessel_name
```

### Zero-count + stale cleanup:
```sql
DELETE FROM vessels_with_containers
WHERE vessel_name NOT IN (
  SELECT DISTINCT vessel_name FROM containers
  WHERE vessel_name IS NOT NULL
    AND (user_status IS NULL OR user_status = 'active')
)
AND (
  (alerted_moored = false AND alerted_24h = false AND alerted_2h = false AND alerted_30m = false)
  OR updated_at < NOW() - INTERVAL '7 days'
)
```

### MMSI resolution (in vwc-sync.js):
1. `vesselDb.getByName(vesselName)` — match by AIS vessel name
2. `vesselDb.getByImo(vesselImo)` — fallback match by IMO number
3. If found, write MMSI to `vessels_with_containers` → ais-collector picks it up for global tracking

### Reappearance:
When new active containers arrive on a vessel (via FTU webhook → container-sync), vwc-sync rebuilds the cache and the vessel reappears automatically.

### Terminal-agnostic:
No hardcoded terminal names. Works for 2 terminals or 200.

---

## 10. Multi-Tenancy

- Every table has `org_id` (uuid)
- Every query filters by `org_id`
- Composite indexes: always `(org_id, <primary_filter>)` — e.g. `(org_id, container_number)`, `(org_id, ui_status)`, `(org_id, user_status)`
- Primary test org: `f8107db3-ecaa-48e1-968d-0e89c6dd8f62`

---

## 11. Schema Change Checklist

Before deploying any schema change:

- [ ] Write a numbered migration file (see section 13)
- [ ] SQLite ALTER TABLE applied (if container-related)
- [ ] Postgres ALTER TABLE applied via the migration file
- [ ] container-sync.js column list updated (SELECT + INSERT + UPDATE — but NEVER user_status in SET)
- [ ] Fleet API endpoints updated if new column needs to be read/written
- [ ] Frontend updated if new column needs to be displayed
- [ ] This skill file updated with the new column

---

## 12. Postgres Connection

```bash
# From the droplet — TCP required, peer auth fails:
psql -h localhost -U fleetcraft -d fleetcraft_db

# Connection string for Node.js:
postgresql://fleetcraft:<password>@localhost:5432/fleetcraft_db
```

### navigation_order gotcha
The `nav_items` column has a NOT NULL constraint. Always default to `[]` (empty JSON array) before INSERT. Failing to do this causes 500 errors in fleet-api logs.

---

## 13. Schema as Code — Migration Files

> **Rule:** Never run ad-hoc ALTER TABLE directly in psql. Every schema change is a numbered SQL file committed to Git.

### Location:
```
fleetcraft-api/
  migrations/
    001_user_status_lane_separation.sql
    ...
```

### Migration file format:
```sql
-- migrations/NNN_description.sql
-- Purpose: What this migration does and why
-- Date: YYYY-MM-DD
-- Depends on: previous migration number

-- The actual SQL
ALTER TABLE containers ADD COLUMN IF NOT EXISTS new_column TEXT;
CREATE INDEX IF NOT EXISTS idx_name ON table (org_id, new_column);
```

### Rules:
- Migrations are append-only — NEVER edit an existing migration file
- Use `IF NOT EXISTS` / `IF EXISTS` so migrations are idempotent (safe to re-run)
- Each migration has: purpose comment, date, dependency chain
- Apply with: `psql -h localhost -U fleetcraft -d fleetcraft_db -f migrations/NNN_file.sql`
- If you need to undo something, write a NEW migration that reverses it
- The migrations folder is the canonical record of the schema — if someone asks "what's the schema?", point them here

---

## 14. Cross-Layer Impact Checklist

Before adding, renaming, or removing ANY data field, trace it through all 5 layers.
If a layer is affected, the change must be made there too. Missing one layer causes silent data loss.

> **Lesson learned:** `terminal_code` was added to SQLite and vwc-sync but missing from container-sync's column list. Data stayed in SQLite forever — dispatch creation couldn't find it in Postgres, geofences never embedded.

### Example: adding terminal_code to containers

| Layer | Component | Check | Action needed |
|-------|-----------|-------|---------------|
| 1 | External API (FTU/AIS) | Does the API provide this field? | No — we derive it from AIS polygon match |
| 2a | Ingestion (fleet-api webhook) | Does the webhook handler write it? | Check server.js FTU handler |
| 2b | Ingestion (vwc-sync) | Does vwc-sync write it? | YES — add to terminal mapping UPDATE |
| 2c | Sync (container-sync) | Is it in the SQLite SELECT + Postgres INSERT column list? | YES — add to both |
| 2d | SQLite schema | Does the column exist? | YES — ALTER TABLE ADD COLUMN |
| 3 | Postgres schema | Does the column exist? | Check — may already exist |
| 4a | Intelligence (dispatcher) | Does it read this field? | No |
| 4b | Intelligence (fleet-api user actions) | Does dispatch creation use it? | YES — geofence lookup uses terminal_code |
| 5 | UX (dashboard/driver app) | Does the UI display it? | Check frontend component |

### Checklist (copy for every field change):
- [ ] SQLite column exists (ALTER TABLE if needed)
- [ ] Postgres column exists (migration if needed)
- [ ] container-sync.js includes it in SELECT + INSERT + UPDATE
- [ ] vwc-sync.js writes it (if vessel/terminal related)
- [ ] fleet-api webhook handler writes it (if FTU provides it)
- [ ] fleet-api read endpoints return it (if UX needs it)
- [ ] fleet-api read endpoint SELECT includes the column (not just the write endpoint)
- [ ] Frontend fetch mapper includes the field (fetchDispatchLookup, etc.)
- [ ] Frontend component displays it (if user-facing)
- [ ] archived_containers includes it (if needed for snapshot)
- [ ] container-registry-schema.sql updated (if SQLite column added)

### Common miss
Adding a column to SQLite + Postgres but forgetting container-sync.
Container-sync is the bridge — if it doesn't know about the column, the data stays in SQLite forever.

### Read path vs write path
Adding a column and writing to it is only half the job. If the API's SELECT query doesn't include the new column, the frontend never sees it. Always check BOTH:
1. **Write path:** Does the INSERT/UPDATE include the column? (milestone handler, archive, etc.)
2. **Read path:** Does the SELECT in every GET endpoint include the column?

The geofence columns were written correctly by the milestone handler but not returned by `GET /dispatches` — the detention card showed blank until the SELECT was updated. Also check the frontend mapper — `fetchDispatchLookup()` manually maps fields and silently drops any it doesn't list.

### Dispatch cancel/delete cleanup
When a dispatch is cancelled or deleted, the cancel endpoint must reset these container columns:
- `pod_full_out_at` → NULL
- `empty_terminated_at` → NULL
- `ui_status` → `'AT_PORT'` (if `pod_discharged_at` exists) or previous state
- `status` / `current_status` → `'discharged'` (if `pod_discharged_at` exists) or previous state
- `available_for_pickup` → `true` (if discharged)
- `updated_at` → NOW()

The container reverts to its pre-dispatch state based on FTU data that still exists (`pod_discharged_at`, `pod_arrival`).

**DELETE must clean up all four FK child tables before deleting the dispatch row:**
```
1. DELETE FROM delivery_notifications WHERE dispatch_id = $1
2. DELETE FROM container_events WHERE dispatch_id = $1
3. DELETE FROM driver_positions WHERE dispatch_id = $1
4. DELETE FROM geofences WHERE dispatch_id = $1
5. DELETE FROM dispatches WHERE id = $1
```

Both `delivery_notifications.dispatch_id` and `geofences.dispatch_id` have FK constraints to `dispatches.id` with NO `ON DELETE CASCADE`. Missing either cleanup causes the FK error reported in April 2026.

For CANCEL (PATCH status='cancelled'): dispatch row is NOT deleted, so FK is not an issue. Events and positions are cleaned up; delivery_notifications and geofences are kept.

The map in the container detail panel should only show events and GPS from the CURRENT active dispatch, not from cancelled ones. Filter by `dispatch_id` or by `occurred_at > current_dispatch.created_at`.

### Test data cleanup
For the permanent test container ALPHA1234: after cancelling a dispatch, also clean up `container_events` and `driver_positions` from the test run so the map shows clean for the next test. This is test-only — production containers keep all events.

---

## NUMERIC/DECIMAL String Coercion (v1.3.5 lesson)

> **Lesson learned:** Postgres NUMERIC columns serialize as strings in JSON responses via the pg driver (e.g. `"47.282000"` instead of `47.282`). This silently breaks any JavaScript arithmetic on the frontend — `"47.28" + 0.1` becomes `"47.280.1"` (string concatenation), not `47.38`.

**Rule:** All lat/lng columns stored as NUMERIC or DECIMAL MUST be cast to `::float` in any SQL query that feeds a JSON API response:

```sql
-- CORRECT
SELECT pickup_lat::float, pickup_lng::float, delivery_lat::float, delivery_lng::float FROM dispatches;

-- WRONG — returns strings in JSON
SELECT pickup_lat, pickup_lng FROM dispatches;
```

**Frontend defense:** Always `Number()` coerce lat/lng values from API responses before any math:

```javascript
const lat = Number(driver.lat);  // never raw driver.lat in arithmetic
```

This applies to every table with NUMERIC coordinates: `dispatches`, `geofences`, `drivers`, `driver_positions`.

---

## Terminal vs Geofence Tables

`terminals` = source of truth for terminal identity, address, gate coordinates.
`geofences` = pure polygon definitions for on-device queue detection only.

| What | Table | Column |
|------|-------|--------|
| Terminal name | terminals | terminal_name |
| Terminal address | terminals | address |
| Gate coordinates (for HERE routing) | terminals | lat, lng |
| Detection polygon (for driver app) | geofences | coordinates (JSONB) |
| Polygon type | geofences | type (always 'polygon') |
| Trigger event | geofences | trigger_event |

**DEPRECATED columns in `geofences` — do not read:**

- `terminal_address` → use `terminals.address`
- `routing_lat` / `routing_lng` → use `terminals.lat` / `terminals.lng`
- `dispatch_id` → geofences are terminal-level, embedded in dispatch at creation

UNIQUE constraint: `(terminal_code, name)` — prevents duplicate geofence rows (migration 005).

**When creating a new terminal:**

1. Add row to `terminals` table (address, lat/lng for gate).
2. Add row to `geofences` table (polygon coordinates for detection zone).
3. Add `terminal_code` to `terminalCodeMap` in `server.js`.
4. Add terminal to frontend direct-add dropdown.
