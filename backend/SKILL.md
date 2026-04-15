---
name: fleetcraft-backend
description: >
  FleetCraft backend development rules. Use this skill whenever working on
  server.js (Fleet API), container-sync.js, vwc-sync.js,
  dispatcher-orchestrator.js, dispatch-agent.js, or any PM2
  worker on the DigitalOcean droplet. Also use when writing SQL migrations,
  modifying database tables, or changing any data write path. This skill
  prevents the recurring bugs that have caused production regressions: sync
  writers overwriting user intent, archived containers being resurrected,
  dual-write CQRS violations, and webhook handlers bypassing the source of
  truth. Read this BEFORE writing any backend code.
---

# FleetCraft Backend — Development Rules

These rules are non-negotiable. Every one was learned from a production bug.

---

## 0. File Structure (Spec 0019, 2026-04-15)

> **Rule:** `server.js` is a 50-line bootstrap. **Never add `app.get/post/patch/delete` handlers directly to `server.js`.** Every endpoint lives under `routes/`. Every cross-cutting helper lives under `lib/`.

```
fleetcraft-api/
├── server.js                       ~50 lines: express app + middleware stack + route mounts + listen()
├── lib/
│   ├── db.js                       pg pool (single shared instance)
│   ├── constants.js                ORG_ID, MILE_TO_METERS
│   ├── middleware.js               requireAuth (driver JWT), requireCustomerRole
│   ├── here.js                     hereGeocode, computeAndStoreRoute, HERE_TRUCK_PROFILE
│   ├── sqlite.js                   containerDb + upsertContainerToSQLite (shared with container-sync via file path)
│   ├── upload.js                   multer + DO Spaces S3 client
│   ├── customerAuth.js             magic-link issue/verify, requireCustomerRole
│   ├── notifications.js            SendGrid + Twilio fire-and-forget
│   └── snapshot-builder.js         dispatch lifecycle snapshot (Spec 0009)
└── routes/
    ├── health.js                   /health, /api/health, /api/health/ftu
    ├── terminals.js                /api/terminals CRUD + propagate-to-dispatches
    ├── geofences.js                /api/geofences CRUD
    ├── subscribers.js              /api/subscribers CRUD (alert_subscribers)
    ├── customers.js                /api/customers CRUD + /api/customers/:id/locations CRUD (Spec 0018)
    ├── fleet.js                    /api/fleet/positions|routes|eta
    ├── portal.js                   /api/portal/* + /api/customer-auth/* + /api/track/:token
    ├── vessels.js                  /vessels/* + /api/vessels/at-terminals
    ├── containers.js               /api/containers/* (14 endpoints, includes FTU webhook)
    ├── containers-legacy.js        /containers/* + /archived-containers/* (legacy duplicates)
    ├── _containerHelpers.js        SHARED by containers.js + containers-legacy.js — FTU constants, trackViaFTU, mapFTUToContainer
    ├── driver.js                   /api/driver/* + /api/auth/driver-login (includes the large milestone handler)
    ├── dispatches.js               /api/dispatches/* + /api/dispatch/* + legacy /dispatches
    ├── resources.js                /api/drivers + /api/trucks + /api/chassis admin CRUD
    └── misc.js                     /analytics/log-visit, /audit-logs, /navigation-order
```

### Rules for new endpoints

1. **Find the right domain file.** If adding a customer endpoint → `routes/customers.js`. Dispatch endpoint → `routes/dispatches.js`. Don't create a new route file for a single endpoint unless the domain is genuinely new.
2. **Add a relative path if the router is prefix-mounted** (e.g., `routes/customers.js` is mounted at `/api/customers`, so add `router.get('/:id', ...)` for `/api/customers/:id`).
3. **Add an absolute path if the router is root-mounted** (e.g., `routes/portal.js` is mounted at `/` because it spans three prefixes — inside it use `router.get('/api/portal/deliveries', ...)`).
4. **Import helpers from `lib/`, never duplicate.** If a helper is used by two routers, it belongs in `lib/`.
5. **Update the self-documenting header** at the top of any route file you change — keep `READS FROM / WRITES TO / AUTH / DEPENDS ON` current.

### Rollback anchors

Every Phase 0-12 extraction has a git tag:
```
v2026.04.15-pre-route-extraction   (pre-refactor baseline)
v2026.04.15-routes-foundation      (lib/db.js + lib/middleware.js)
v2026.04.15-routes-health          (Phase 1)
v2026.04.15-routes-terminals       (Phase 2)
v2026.04.15-routes-geofences       (Phase 3)
v2026.04.15-routes-subscribers     (Phase 4)
v2026.04.15-routes-vessels         (Phase 5)
v2026.04.15-routes-customers       (Phase 6)
v2026.04.15-routes-fleet           (Phase 7)
v2026.04.15-routes-portal          (Phase 8)
v2026.04.15-routes-containers      (Phase 9)
v2026.04.15-routes-driver          (Phase 10)
v2026.04.15-routes-dispatches      (Phase 11)
v2026.04.15-routes-complete        (Phase 12 final)
```

`git reset --hard v2026.04.15-pre-route-extraction && pm2 restart fleet-api` reverts the whole refactor in <30 seconds.

---

## 1. Architecture: CQRS with Materialized Views

FleetCraft uses a strict CQRS (Command Query Responsibility Segregation) pattern.

### Source of Truth
- **DigitalOcean PostgreSQL** (`fleetcraft_db` on `178.128.64.97`) is the ONLY database
- **SQLite** (`/opt/fleetcraft-ais/container-registry.db`) is the container registry write buffer — synced to Postgres via `container-sync.js`
- **Supabase is FULLY DISCONNECTED** — never reference it as read cache, auth, or anything else. It does not exist in FleetCraft anymore.

### Write Path (who writes what)
Every data domain has exactly ONE writer. No exceptions.

| Domain | Single Writer | File | Writes To |
|--------|--------------|------|-----------|
| Container tracking data | FTU webhook handler | `server.js` | SQLite → Postgres via container-sync |
| Direct-add containers | Fleet API quick-add | `server.js` | Postgres ONLY (bypasses SQLite entirely) |
| Container user intent | Fleet API endpoints | `server.js` | Postgres containers.user_status (ONLY writer) |
| Container archive (auto) | container-sync.js | `container-sync.js` | SQLite (archived_at + user_status) + Postgres container_exclusions |
| Container archive (manual) | Fleet API | `server.js` | Postgres containers + container_exclusions + SQLite delete |
| Container exclusions | Fleet API + container-sync | `server.js`, `container-sync.js` | Postgres container_exclusions (tombstone table) |
| SQLite vessel registry | AIS collector | `index.js` (ais-collector-v2) | SQLite vessel_registry |
| Vessel cache (vessels_with_containers) | vwc-sync | `vwc-sync.js` | Postgres vessels_with_containers (ALL data columns) |
| Vessel cache (vessels_cache) | vwc-sync | `vwc-sync.js` | Postgres vessels_cache |
| Alert flags (alerted_*) | Dispatcher | `dispatcher-orchestrator.js` | Postgres vessels_with_containers (alerted_* columns ONLY) |
| Dispatch records | Fleet API | `server.js` | Postgres dispatches table |
| Driver milestones | Fleet API (from driver app) | `server.js` | Postgres container_events table |
| Auto-archive events | container-sync | `container-sync.js` | Postgres container_events |
| Driver positions | Fleet API (from driver app) | `server.js` | Postgres driver_positions table |
| Alert logs | Dispatcher | `dispatcher-orchestrator.js` | Postgres alert_logs table |

### Read Path
- Frontend dashboard reads from Postgres via Fleet API endpoints
- Driver app reads from Fleet API endpoints
- Dispatcher reads from Postgres directly (same droplet)

### The Rule
**Writers write to their domain only. Readers never write.** If you find yourself adding a write to a reader, stop — you are violating CQRS.

---

## 2. The user_status Guard (CRITICAL)

> **Lesson learned:** Sync writers (container-sync, ftu-tracker) repeatedly overwrote user decisions. A dispatcher archives a container, then 30 seconds later container-sync resurrects it because FTU still has data for it.

### The Fix: Two-Layer Defense (Lane Separation + Tombstone)

The `user_status` column on the containers table records USER intent. Only user-initiated actions via the Fleet API can write to it.

**Valid user_status values:** `active`, `archived`, `dismissed`, `held`, `flagged`

### Layer 1: Tombstone Check (defense-in-depth)

The `container_exclusions` table is a tombstone. Before ANY sync writer attempts an upsert, it loads all excluded container_numbers into a Set and skips them entirely.

```javascript
// At the START of each sync cycle in container-sync.js and ftu-tracker.js:
const { rows: exclusions } = await pgPool.query(
  'SELECT container_number FROM container_exclusions WHERE org_id = $1',
  [orgId]
);
const excludedSet = new Set(exclusions.map(e => e.container_number));

// Then for each container before upserting:
if (excludedSet.has(container.container_number)) continue; // skip entirely
```

### Layer 2: Upsert Guard (SQL-level wall)

Every sync writer's upsert/insert MUST include this WHERE clause:

```sql
-- In container-sync.js, ftu-tracker.js, or any worker that syncs external data:
INSERT INTO containers (container_number, ui_status, vessel_name, ...)
VALUES ($1, $2, $3, ...)
ON CONFLICT (container_number) DO UPDATE SET
  ui_status = EXCLUDED.ui_status,
  vessel_name = EXCLUDED.vessel_name,
  -- ... other external fields
WHERE containers.user_status IS NULL
   OR containers.user_status = 'active';
```

**NEVER include `user_status` in the SET clause of a sync writer.** Only the Fleet API endpoints touch `user_status`.

### When each layer fires:
- Tombstone check: prevents the INSERT from even being attempted
- SQL WHERE guard: prevents the UPDATE from overwriting if tombstone was somehow missed
- Both together: belt AND suspenders

### Which user_status values create exclusion records:
- `archived` → YES (exclusion record + FTU unregister + SQLite delete)
- `dismissed` → YES (exclusion record, NO FTU unregister, NO SQLite delete)
- `held` → NO (still visible in Held tab, sync just skips via SQL guard)
- `flagged` → NO (still visible in Flagged tab, sync just skips via SQL guard)

---

## 3. Container Archive Flow

> **Lesson learned:** Archive was broken three separate times due to: frontend calling a nonexistent DELETE endpoint, SQLite not being cleaned on archive, and FTU continuing to send webhooks for archived containers.

### Correct Archive Sequence (Manual)

```
User clicks Archive → POST /containers/archive
  1. Verify container exists, is not already archived, has no active dispatch
  2. UPDATE containers SET user_status = 'archived', archived_at = NOW() in Postgres
  3. INSERT INTO container_exclusions (tombstone record)
  4. DELETE from SQLite — ONLY if data_source != 'direct' (direct-add containers were never in SQLite)
  5. HTTP DELETE to FTU — ONLY if findteu_shipment_id IS NOT NULL (direct-add containers have no FTU subscription)
  6. INSERT INTO container_events (event_type = 'archive', source = 'dispatcher')
```

### Correct Archive Sequence (Auto — handled by container-sync.js)

```
container-sync.js runs two auto-archive functions every 2 minutes:

archiveCompletedDispatches():
  1. Queries Postgres for completed dispatches older than 24h
  2. Sets archived_at + user_status = 'archived' in SQLite
  3. INSERT INTO container_exclusions in Postgres
  4. syncArchivedContainers() moves to archived_containers on next cycle
  NOTE: Only dispatches.status = 'completed' qualify. Cancelled dispatches do NOT trigger archive.

archiveStaleContainers():
  1. EMPTY_RETURNED > 24 hours → auto-archive (reason: auto_empty_returned)
  2. OUT_FOR_DELIVERY > 7 days → auto-archive (reason: auto_stale_ofd) — safety net for stuck containers
  3. Same flow: SQLite archived_at + user_status + Postgres tombstone
```

### Other user_status Transitions

```
Dismiss → POST /containers/dismiss
  1. UPDATE containers SET user_status = 'dismissed'
  2. INSERT INTO container_exclusions
  3. INSERT INTO container_events
  Note: NO FTU unregister, NO SQLite delete. Data preserved for restore.

Hold → POST /containers/hold
  1. UPDATE containers SET user_status = 'held'
  2. INSERT INTO container_events
  Note: NO exclusion record. Container visible in Held tab.

Flag → POST /containers/flag
  1. UPDATE containers SET user_status = 'flagged'
  2. INSERT INTO container_events
  Note: NO exclusion record. Container visible in Flagged tab.

Restore → POST /containers/restore
  1. UPDATE containers SET user_status = 'active', archived_at = NULL
  2. DELETE FROM container_exclusions
  3. INSERT INTO container_events
  Note: Next sync cycle resumes normal updates. May need FTU re-registration
        UNLESS data_source = 'direct' — direct-add containers were never registered with FTU.
```

### Rules
- `container-sync.js` handles ALL auto-archiving (stale + dispatch-completed) AND moves to `archived_containers`
- `archive-worker.js` is KILLED — its logic is merged into container-sync.js
- `ftu-tracker.js` is KILLED — cache refresh replaced by vwc-sync.js
- Manual archive in `server.js` handles FTU unregister inline — `syncArchivedContainers()` skips `manual`/`manual_archive` reasons to avoid duplicate API calls
- The `archived_containers` table is a permanent record — never delete from it

---

## 4. FTU Integration Rules

> **Lesson learned:** FTU is an event-based webhook API, not a polling API. The old ftu-tracker.js was calling Terminal 49 endpoints that don't exist on FTU. T49 is fully deactivated.

### Active System
- **FTU (FindTEU)** — webhook-based container tracking
- `ftu-tracker.js` is KILLED — its cache refresh logic is now in `vwc-sync.js`
- Webhooks arrive at Fleet API → SQLite → Postgres via container-sync
- FTU account #14978
- **Exception:** Direct-add containers (`data_source = 'direct'`) bypass FTU entirely — no registration, no webhooks, no unregistration. They are added via `POST /api/containers/quick-add` directly to Postgres as AT_PORT.

### Deactivated System
- **T49 (Terminal 49)** — NEVER reference `t49-container-tracker.js` as active
- Any code referencing T49 endpoints, T49 API keys, or T49 data structures is dead code

### FTU Webhook Handler Rules (in server.js)
1. Parse the webhook payload
2. Check `container_exclusions` tombstone — if excluded, drop the write
3. Write to SQLite FIRST (source of truth)
4. container-sync.js handles Postgres projection (with upsert guard)
5. When FTU sends `completed: true`, auto-archive the container

### FTU Data Gaps
FTU does NOT provide: LFD, holds detail (customs/freight/terminal), yard location, voyage number, MMSI. These fields are always null from FTU. Do not write code that depends on them being populated by FTU.

---

## 5. Event Sourcing Pattern

> **Lesson learned:** Overwriting current state loses history. Once a milestone timestamp was overwritten, it was unrecoverable.

### Architecture: Event Sourcing + State Snapshot

| Table | Role |
|-------|------|
| `container_events` | Append-only event ledger — NEVER update or delete rows |
| `containers` | Current state projection — derived from latest events |
| `dispatches` | Current state snapshot — updated by Fleet API |

### Rules
- Every state change writes an event to `container_events` with a `source` field (ftu, driver, dispatcher, system)
- Every user_status transition (archive, dismiss, hold, flag, restore) writes an event
- Current state in `containers` is rebuilt/projected from events
- `container_events` rows are immutable — INSERT only, no UPDATE, no DELETE

### Geofence Event Pattern
The `terminal_area_arrived` event is the only auto-triggered milestone. It follows a specific pattern:

- **Source:** driver app (on-device geofence detection)
- **Fields:** `auto_triggered: true`, `triggered_by: 'geofence'`, `occurred_at` from GPS reading
- **Side effect:** Sets `dispatches.queue_start_at` WHERE `queue_start_at IS NULL` (idempotent)
- **Idempotency:** Second fire does NOT overwrite — the WHERE NULL guard is the server-side backstop
- **Timestamp:** Uses client-provided `occurred_at` via `COALESCE($1::timestamptz, now())`

### Queue Time Calculation
Queue time is DERIVED, never tapped by the driver.
- **Primary source:** `dispatches.queue_start_at` (geofence polygon entry, auto-triggered) to `dispatches.queue_stop_at` (outgate EIR photo)
- **Fallback source:** `dispatches.terminal_ingate_at` (Ingate milestone tap) to `dispatches.pickup_completed_at` (Gate Out milestone tap)
- `in_queue` status exists in the CHECK constraint for backwards compatibility but is never set by the current driver app
- The driver milestone flow is: `chassis_info` → `at_terminal` (Ingate) → `container_loaded` → `gate_out`

---

## 6. PM2 Workers — Current State

| ID | Name | Script | Status | Purpose |
|----|------|--------|--------|---------|
| 0 | dispatcher | /opt/fleetcraft-alerts/Dispatchers-Live/dispatcher-orchestrator.js | online | Email/alert dispatching (SendGrid since 2026-04-14) |

---

## Customer Portal Auth Lane (Spec 0016 Phase 1, 2026-04-14)

Third auth lane alongside driver + dispatcher. Implemented in `lib/customerAuth.js`.

### Three auth lanes, zero crossover

| Lane | Middleware | JWT role field | Rejects |
|---|---|---|---|
| Driver | `requireAuth` (server.js) | absent OR anything but `'customer'` | `role='customer'` returns 403 |
| Customer | `requireCustomerRole` (lib/customerAuth.js) | must equal `'customer'` | absent, driver, or bad → 401/403 |
| Dispatcher | **unauthenticated** (pre-existing gap) | n/a | n/a — any caller can reach `/api/dispatches` etc. |

The driver-side rejection was added by updating `requireAuth` to check `payload.role === 'customer'` → 403. Existing driver JWTs are minted without a role field, so backward compat holds.

### Magic-link flow

```
POST /api/customer-auth/magic-link     { email }
  → generateMagicLink() in lib/customerAuth.js
  → customers.magic_link_token (64-hex), .magic_link_expires_at (NOW + 15m)
  → SendGrid email to customer.notification_email with
    https://myfleetcraft.com/portal/verify?token=<hex>
  → Always returns 200 (enumeration prevention — unknown emails drop silently)
  → Rate limit: 3/email/hour via magic_link_attempt_count + window_start columns

GET /api/customer-auth/verify?token=<hex>
  → verifyMagicLink() checks 64-hex format, expiry, clears token (single-use)
  → Issues jsonwebtoken with payload { customer_id, org_id, role: 'customer' }
  → TTL 7 days
  → Stamps customers.last_login_at
```

### Portal endpoints (all require role='customer')

```
GET   /api/portal/deliveries            paginated, sanitized dispatch list
GET   /api/portal/deliveries/:id        single dispatch with milestones + route_polyline
PATCH /api/portal/preferences           toggle pref_email_*/pref_sms_* only
```

**Double-scoped queries** — every SELECT includes `WHERE customer_id = $1 AND org_id = $2`. A crafted request with another customer's dispatch ID returns 404, not the dispatch. This is the tenant isolation guarantee.

**Sanitization rule** — portal endpoints strip `driver_id`, `pickup_lat/lng`, `delivery_lat/lng`, `origin_lat/lng`, `org_id` from responses. Customer sees container_number, status, milestones, timestamps, tracking_token, pod_photo_url. Enforced in the SELECT column list, not by post-filter — no risk of accidental field leak via `SELECT *`.

### PATCH /api/portal/preferences is locked-down

Customer can only toggle 6 boolean fields (`pref_email_scheduled / en_route / arriving / delivered`, `pref_sms_arriving / delivered`). Requests with any other field (e.g. `notification_email`) are rejected with 400. This prevents a customer from hijacking their own notification address to intercept someone else's delivery updates.

### Rate limit semantics

`magic_link_attempt_count` is incremented on every request (even ones we drop silently). `magic_link_attempt_window_start` resets when older than 1 hour. After 3 accepted requests in a window, further requests return 200 to the caller (enumeration-safe) but do NOT send email or mint a token — the counter just keeps climbing.

## Customer Locations (Spec 0018, 2026-04-15)

### CRUD endpoints

```
GET    /api/customers/:id/locations               active only, is_default desc, name asc
POST   /api/customers/:id/locations               body: { name, address, radius_mi?, notes? }
PATCH  /api/customers/:id/locations/:locId        partial update; re-geocodes if address changed
DELETE /api/customers/:id/locations/:locId        → { deleted: "soft"|"hard" }
```

### POST /api/dispatches — customer_location_id resolution

When request body contains `customer_location_id`, the endpoint does a **synchronous** lookup BEFORE INSERT and copies `address`, `lat`, `lng`, `radius_mi * MILE_TO_METERS` (rounded), and `notes` onto the new dispatch. `customer_location_id` itself is stored on the dispatch row so the `setImmediate` geofence block can see "this dispatch came from a location" and uses the location's radius instead of the legacy customer default.

One-off typed addresses still work — when `customer_location_id` is absent, the existing geocode-the-typed-address path in `setImmediate` runs unchanged.

### Delete semantics

- **Soft** — if any `dispatches.customer_location_id = $locId` rows exist, flip `is_active=false` (keeps the row so historical dispatches retain the FK target).
- **Hard** — if zero referencing dispatches, `DELETE FROM customer_locations`.

### Rule — auto-default first location

`POST /api/customers/:id/locations` checks active count; if zero, the new row is created with `is_default=true` regardless of request payload. Subsequent creates respect the payload. PATCH with `is_default=true` unsets any previously-default row in the same transaction (partial unique index enforces one-at-a-time).

### Rule — 20 location ceiling

Soft app-layer cap: the POST returns 400 `{ error: "Maximum 20 locations per customer" }` when active count already ≥ 20. No DB constraint — we may loosen this for large accounts later.
| 2 | ais-collector-v2 | index.js | online | AIS WebSocket → SQLite vessel_registry |
| 4 | container-sync | container-sync.js | online | SQLite → PG containers + auto-archive (stale + dispatch) |
| 9 | vwc-sync | vwc-sync.js | online | Single owner of vessels_with_containers + vessels_cache |
| 7 | fleet-api | server.js | online | Express API (port 3001) |
| 5 | dispatch-worker | here-dispatch-worker.js | STOPPED | Future HERE routing |

### Killed Workers — PERMANENTLY KILLED (do NOT restart)
| ID | Name | Reason killed | Replaced by |
|----|------|--------------|-------------|
| 1 | vessel-sync | PERMANENTLY KILLED — dual-writer on vessels_with_containers | vwc-sync.js |
| 3 | ftu-tracker | PERMANENTLY KILLED — dual-writer on vessels_with_containers, cache refresh redundant | vwc-sync.js |
| 8 | archive-worker | PERMANENTLY KILLED — logic merged into container-sync.js | container-sync.js |

### Adding a New Worker
1. Create the .js file in `/opt/fleetcraft-ais/` or `/opt/fleetcraft-api/`
2. Start with `pm2 start <file> --name <n>`
3. Run `pm2 save` to persist
4. Add to this table
5. Create a Dockerfile for the worker (incremental containerization rule)
6. Update `ecosystem.config.js` in the repo

---

## 7. Database Connection Rules

- **Postgres on droplet:** TCP required — use `-h localhost` flag. Peer auth fails on socket connection.
  ```bash
  psql -h localhost -U fleetcraft -d fleetcraft_db
  ```
- **Connection string in code:** `postgresql://fleetcraft:<password>@localhost:5432/fleetcraft_db`
- **SQLite path:** `/opt/fleetcraft-ais/container-registry.db`
- **SQLite library:** `better-sqlite3` (synchronous, not async)

---

## 8. API Endpoint Patterns

Fleet API runs on port 3001, served via nginx at `api.myfleetcraft.com`.

### Every endpoint MUST:
1. Filter by `org_id` — no endpoint returns data across orgs
2. Return proper HTTP status codes (200, 201, 400, 404, 500)
3. Log errors to console (PM2 captures stdout/stderr)
4. Use the Postgres pool, never direct SQLite reads from the API layer (SQLite is for workers only)

### Container list filtering
`GET /api/containers/list` accepts `?status=` query param:
- `active` (default) — `WHERE user_status IS NULL OR user_status = 'active'`
- `held` / `flagged` / `dismissed` / `archived` — exact match
- `all` — no user_status filter (admin view)

### Multi-tenancy
- Primary test org: `f8107db3-ecaa-48e1-968d-0e89c6dd8f62`
- Every table has `org_id` column
- Every query includes `WHERE org_id = $1`

### New CRUD endpoints (April 2026 Supabase purge — commit `fcbe2e7`)

These replaced direct frontend Supabase queries during the April 13 purge. The frontend now routes every CRUD through Fleet API — no exceptions. Pattern: all writes go through `allowed` array + dynamic `SET` clause builder (same shape as the pre-existing drivers PUT at line 1114).

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/chassis` | List available chassis (dispatch-creation dropdowns) |
| GET | `/api/chassis?all=true` | List ALL chassis (admin view / Chassis Tracker) |
| POST | `/api/chassis` | Create chassis |
| PUT | `/api/chassis/:id` | Update chassis (partial; uses `allowed` list) |
| DELETE | `/api/chassis/:id` | Soft-delete (`is_active = false`) — preserves audit history |
| POST | `/api/drivers` | Create driver |
| PUT | `/api/drivers/:id` | Update driver |
| DELETE | `/api/drivers/:id` | Soft-delete (`is_active = false, status = 'off_duty'`) |
| POST | `/api/trucks` | Create truck |
| PUT | `/api/trucks/:id` | Update truck |
| DELETE | `/api/trucks/:id` | Soft-delete |

Existing related endpoints already in place (listed for completeness):

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/vessels/tracking` | All cargo vessels from `vessels_cache` (drives VesselTracking page) |
| GET | `/vessels/with-containers` | VWC rows for dashboard vessel status cards |
| GET | `/containers/vessels` | Container counts grouped by vessel (dashboard "Live Updates" card) |
| PATCH | `/api/dispatches/:id` | Update dispatch fields (drives EditDispatchModal) |
| GET | `/api/subscribers` + POST/DELETE | Alert subscribers CRUD |
| GET | `/api/dispatches/:id/eta` | HERE Routing live ETA (Spec 0013 v2) |

**Soft-delete rationale:** chassis, drivers, trucks are referenced by dispatches and chassis_assignments. Hard DELETE would either cascade (dangerous) or error on FK. Setting `is_active = false` preserves history while removing the row from all UI queries (which filter on `is_active = true`).

### Terminal Address → Coordinates Pipeline (April 2026 — commit `37ad6df`)

HERE Routing v8 only accepts `lat,lng` — it will NOT resolve a free-text address. Addresses are a human input; coordinates are the machine truth. We convert once via HERE Geocoding and store the result; every downstream API call uses the stored coordinates.

| HERE API | Input | Output | When called |
|----------|-------|--------|-------------|
| **Geocoding** (`geocode.search.hereapi.com/v1/geocode`) | Free-text address | `{ lat, lng, label }` | **Once**, at terminal create/update or delivery address entry |
| **Routing** (`router.hereapi.com/v8/routes`) | `origin=lat,lng` + `destination=lat,lng` | polyline, summary (duration / baseDuration / length) | **Many times** — every ETA poll, route preview, dispatch route compute |

**Auto-geocode fires in two places** (both added in `37ad6df`):

1. `POST /api/terminals` — new terminal with non-null address and no explicit `lat/lng` → `hereGeocode(address)` → stored
2. `PATCH /api/terminals/:id` — address changed without explicit `lat/lng` → `hereGeocode(address)` → stored → `propagateTerminalCoordsToDispatches(terminal_code)` fans out to every active dispatch

**`hereGeocode()` call sites in `server.js`** (4 total):

1. `POST /api/terminals` — new terminal address → coords
2. `PATCH /api/terminals/:id` — address change → coords
3. `POST /api/dispatches` setImmediate — `delivery_address` → coords (customer delivery — free-text input)
4. `GET /api/dispatches/:id/eta` — fallback only if `terminals.lat/lng` is null (should never fire in practice now)

**Error handling:** all four sites treat geocoding as best-effort. If HERE returns no match or throws, we log a warning and save the row WITHOUT coordinates; the address text still persists. This prevents HERE outages from blocking terminal edits.

**RULE:** Address is human input. Lat/lng is machine truth. Once geocoded, all routing/ETA/distance uses coordinates only. **Zero address strings are ever sent to HERE Routing.**

### Dynamic vs Frozen Updates on Active Dispatches

Not every change flows through to an in-flight dispatch. The table below spells out what propagates and what's snapshotted at creation time:

| Change | Affects active dispatches? | Mechanism | Latency |
|--------|-----------------------------|-----------|---------|
| Terminal `address` edit | ✅ YES | Auto-geocode + `propagateTerminalCoordsToDispatches()` | < 10s (one API call) |
| Terminal `lat`/`lng` direct edit | ✅ YES | Same propagation path | < 10s |
| Geofence polygon redraw (`PUT /api/geofences/:id`) | ❌ NO — frozen | `dispatches.pickup_geofences` embedded as JSONB snapshot at dispatch creation | Only new dispatches see it |
| Delivery address edit | ✅ YES | `PATCH /api/dispatches/:id` direct write | Immediate |
| HERE live ETA refresh | ✅ YES | Endpoint re-reads `terminals.lat/lng` on each call | 3-10 min (driver-app cadence) |
| `dispatches.route_polyline` (pickup→delivery leg) | ❌ NO — frozen | Computed once via `computeAndStoreRoute()` at dispatch creation | Only new dispatches see it |
| `dispatches.origin_lat/lng` | ❌ NO — frozen | Captured from driver GPS at "En Route" tap | Never changes post-tap |

**Why geofences are frozen:** detention evidence must be deterministic. If a polygon changed mid-shift, `terminal_area_arrived` would fire at a different physical location than where the trip started, breaking queue/detention timers. Freezing at creation keeps the contract: "this dispatch entered THIS polygon at THIS time, immutable."

**Why stored route polyline is frozen:** low impact. The pickup-phase "bright blue" current-leg line on the fleet map is redrawn every 10 s via `GET /api/dispatch/route-preview` from the truck's live GPS, so the driver-facing route always reflects reality. The stored polyline is only used for the faded "next hop" preview (terminal → delivery) — slightly off if delivery or terminal coords changed, but never blocks the trip.

### Error Pattern #17: Terminal address changed but route still goes to old location

**Symptom:** dispatcher edits a terminal's address in the Terminal modal. The dashboard shows the new address on the dispatch card, but the fleet map still routes the truck to the old gate. Driver follows the line and ends up at the wrong place.

**Root cause (pre-fix):** `PATCH /api/terminals/:id` updated the `address` text field but didn't re-geocode `lat/lng`. HERE Routing uses coords, so it kept routing to the old point. Active dispatches had their `pickup_lat/lng` snapshot from creation time — also stale.

**Fix (commit `37ad6df`):** auto-geocode fires on every address change, result is stored on `terminals`, then `propagateTerminalCoordsToDispatches()` updates every active dispatch's `pickup_lat/lng/pickup_address` in the same transaction. One API call fixes everything downstream within a polling cycle (~10 s). No manual SQL, no dispatch delete/recreate needed.

**Incident:** April 2026 — HUSKY terminal address was changed from `1101 Port of Tacoma Rd` to `2325 Lincoln Ave` on the dashboard. Coordinates stayed at the old location (`47.2642, -122.4082`). New HUSKY dispatches were being routed ~740 m away from where the driver thought they were going. Diagnosed by comparing `hereGeocode()` output for both addresses against the stored `terminals.lat/lng`.

---

## 9. Vessel Tracking & vwc-sync Rules

> **Lesson learned:** SOG of 0.1 knots (GPS drift) divided into distance produced a 212-hour ETA for a moored vessel. Also: three workers (vessel-sync, ftu-tracker, container-sync) all wrote to vessels_with_containers with different aggregation logic, causing count zeroing and MMSI loss.

### AIS Rules
- SOG threshold: **>1 knot** to count as moving. SOG ≤1 = stationary/GPS drift.
- Moored/anchored vessels: **exclude from ETA calculations** in vwc-sync and dispatcher
- `nav_status_label === 'Moored'` → skip in 24H alert loop, do not clear alerted flags
- "In Processing" at terminal = container discharged, undergoing customs/terminal handling — definitively NOT on vessel. Count in `at_terminal_count`.

### vwc-sync.js — Single Owner of vessels_with_containers
- **MMSI resolution order:** `vesselDb.getByName()` first, `vesselDb.getByImo()` fallback
- **Container count aggregation:** Queries Postgres containers filtered by `user_status IS NULL OR user_status = 'active'`
- **Terminal geofence mapping:** Moored vessel inside terminal polygon → update container terminal_name in SQLite
- **Stale cleanup:** Zero IN_TRANSIT containers → auto-reset alert flags → delete from cache (immediate, no 7-day wait)
- **Writes `alerted_*` columns ONLY to reset them** when vessel lifecycle completes (all containers discharged). Dispatcher sets them to true.
- **Terminal-agnostic**: No hardcoded terminal names. Works for 2 terminals or 200.

### Vessel lifecycle completion
A vessel stays on the tracker while it has at least one IN_TRANSIT container (`user_status = 'active'`).
When the LAST IN_TRANSIT container transitions to AT_PORT:
1. Alert flags auto-reset (all four set to false)
2. Next cleanup cycle (30s) removes vessel from `vessels_with_containers`
3. No 7-day wait — immediate cleanup
4. Vessel stays in `vessels_cache` (AIS map) but removed from VWC (our tracking)

A vessel carrying 5 containers disappears only when ALL 5 are discharged. Not before.
Dismissed/held containers (`user_status != 'active'`) do not count — they don't keep the vessel on the tracker.

### ETA calculation (FTU-first priority)

> **Lesson learned:** EVER SIGMA had AIS destination `CAVAN>JPTYO` (Vancouver → Japan). AIS ETA calculated distance to Japan — 87 hours shown in dashboard. Team said ETA was April 15 (Tacoma). FTU pod_eta had the correct answer. AIS destination is the vessel's plan, not the container's plan.

```
IF vessel moored at terminal:
  eta_hours = 0, eta_source = 'MOORED'
ELSE IF FTU pod_eta exists for this vessel's containers:
  eta_hours = (pod_eta - now) in hours, eta_source = 'FTU'
ELSE IF isHeadingToWA(destination) AND SOG > 1:
  eta_hours = distance_nm / SOG, eta_source = 'AIS'
ELSE:
  eta_hours = null, eta_source = null
```

### AIS Handover — vwc-sync status override

When vwc-sync detects a vessel moored inside a terminal geofence, it checks all IN_TRANSIT containers on that vessel:
- Override `ui_status` to `'AT_PORT'` in Postgres directly (not SQLite)
- Set `terminal_name` and `terminal_code` from geofence match
- This is the ONLY case where vwc-sync writes to Postgres containers table
- The timestamp guard in container-sync ensures SQLite won't overwrite this
- Log: "AIS handover: {container} on {vessel} → AT_PORT at {terminal}"

### Container status values
- `ui_status` values are FTU-mapped: `IN_TRANSIT`, `AT_PORT`, `OUT_FOR_DELIVERY`, `EMPTY_RETURNED`
- Dispatch-ready containers: `ui_status = 'AT_PORT'` AND `user_status = 'active'` — both conditions required

---

## 10. Error Patterns to Avoid

These specific mistakes have caused production regressions:

1. **Dual writes:** Writing to both SQLite and Postgres from the same handler. Pick one — the sync worker handles the other.
2. **Missing findteu_shipment_id:** This field MUST be in every INSERT/UPDATE column list in container-sync.js. It was missing once and broke FTU unregistration.
3. **nav_items null:** The `navigation_order` table has a NOT NULL constraint on `nav_items`. Always default to `[]` before INSERT.
4. **JWT key name:** The driver app JWT key is `fleetcraft_jwt`, NOT `fleetcraft_token`. Using the wrong key causes silent auth failures.
5. **Supabase references:** Any code importing `@supabase/supabase-js` or referencing `SUPABASE_URL` is dead code. Remove it.
6. **alert flag reset on vessel re-insert:** `container-sync.js` and `ftu-tracker.js` must NOT delete and re-insert vessel rows — this resets `alerted_*` flags. Use UPDATE for existing rows.
7. **Missing tombstone check:** Every sync writer must load `container_exclusions` at the start of each cycle and skip excluded containers BEFORE attempting upsert.
8. **user_status in SET clause:** Sync writers must NEVER include `user_status` in the ON CONFLICT SET clause. Only Fleet API endpoints write this column.
9. **SQLite .db files tracked in git:** Database files are runtime data. If tracked, `git reset --hard` overwrites the live DB and destroys any columns added via ALTER TABLE. Always add `*.db *.db-shm *.db-wal` to `.gitignore` and use `git rm --cached` to untrack.
10. **vessels_with_containers dual-writer bug:** Three workers (vessel-sync, ftu-tracker, container-sync) all wrote to this table with different aggregation logic. ftu-tracker zeroed on_vessel_count that container-sync had set. Fix: single owner (vwc-sync.js). If you ever need to write to vessels_with_containers, it MUST go through vwc-sync — no other worker touches this table's data columns.
11. **Empty string FK values:** Frontend may send "" instead of null for optional FK fields (customer_id, driver_id, truck_id, chassis_id). Postgres treats "" as non-null, fails FK check. Always validate UUID format before INSERT. Use a helper like `uuidOrNull(value)` that returns null for empty strings, "undefined", "null", non-UUID strings, and actual null/undefined. Applied to POST /api/dispatches and any endpoint that accepts optional FK references.
12. **FTU completed=true is NOT an archive trigger.** FTU sends `completed=true` when it stops tracking — this can happen at vessel arrival, discharge, or any time. It does NOT mean the business cycle is done. The business cycle ends ONLY when the driver completes step 25 (empty return) and `dispatches.completed_at` is set. The webhook handler must NEVER auto-archive on FTU `completed=true`. Auto-archive triggers ONLY from container-sync.js `archiveCompletedDispatches()` which checks `dispatches.completed_at` + 24h.
13. **Cross-layer column gap.** Adding a column to one layer (e.g., SQLite) without adding it to container-sync's column list means the data never reaches Postgres. Always trace new fields through the Cross-Layer Impact Checklist in database/SKILL.md. The `terminal_code` field was added to SQLite and vwc-sync but missing from container-sync — data stayed in SQLite, dispatch creation couldn't find it, geofences never embedded.
14. **AIS-FTU handover lag.** Vessel moored at terminal but FTU still reports IN_TRANSIT. vwc-sync overrides ui_status to AT_PORT based on AIS moored + terminal geofence. FTU discharge data arrives later and enriches the record without overriding AT_PORT.
15. **AIS ETA wrong for transit vessels.** AIS destination shows a non-WA port (transit or next voyage). AIS ETA calculates distance to wrong port. Always prefer FTU pod_eta for container ETA. AIS ETA only valid when vessel is heading directly to WA as final destination.
16. **API SELECT doesn't return new columns.** Adding a column to a table and writing to it in one endpoint doesn't mean it's returned by read endpoints. When adding new columns to dispatches (queue_start_at, pickup_geofences, etc.), you must update EVERY SELECT that reads from that table — GET /dispatches, GET /api/dispatches/:id, GET /api/driver/loads, GET /api/driver/history. The geofence detention card showed blank because the write path worked but the read path didn't return the new columns.
17. **Postgres numeric comes as string via JSON.** Postgres numeric/decimal columns return as strings in JSON API responses (e.g., "12.50" not 12.50). Frontend code calling .toFixed() on these values crashes with "toFixed is not a function". Always wrap in Number() first: `Number(value).toFixed(2)`. This crashed the entire Container Tracking page via chassis.daily_rate.
18. **Ghost skeleton containers.** When FTU returns no vessel data, do NOT create a skeleton container with ui_status='SUBMITTED'. Reject the container with an error. Ghost containers with no vessel, no ETA, no tracking data clutter the dashboard and confuse users. Both FTU registration paths now reject when no vessel data returned.
19. **Orphaned dispatch — container moved without driver.** FTU reports container picked up (OUT_FOR_DELIVERY) but there's a pending dispatch with no driver milestones. This means another carrier moved the container. Auto-cancel the dispatch. Do NOT block the container status change. The container's lifecycle continues regardless of the dispatch.
20. **Stale vessels stuck by alert flags.** Vessel has zero IN_TRANSIT containers but alert flags block DELETE. Fix: auto-reset flags when on_vessel_count = 0 (step 3a in vwc-sync). Cleanup is immediate — no 7-day wait.
21. **VWC insert/delete loop.** vwc-sync aggregation groups ALL active containers by vessel and upserts. Cleanup deletes vessels with zero IN_TRANSIT. Result: cleanup removes vessel, aggregation re-inserts it 30s later because non-IN_TRANSIT containers still reference it. Fix: aggregation query must only build rows for vessels that have at least one IN_TRANSIT container. Both aggregation and cleanup use the same visibility rule.

22. **Frontend reads vessel data from wrong source.** Vessel tracker loaded 21 AIS vessels from vessels_cache. Container page grouped by vessel_name showing done vessels. Fix: tracker reads VWC. /containers/vessels filters by IN_TRANSIT. Vessel pills are client-side from visible containers — correct behavior.

23. **AIS destination vs FTU destination.** Vessel card showed CAVAN>JPTYO (Japan) when container's destination is Tacoma. AIS destination = vessel's route. FTU pod_name = container's destination. Fix: show pod_name first, AIS destination as fallback.
24. **driver_positions dispatch_id is NULL.** GPS positions are recorded but not linked to a dispatch. Server `POST /api/driver/positions` stores driver_id, lat, lng, recorded_at but ignores load_id/dispatch_id from the request body. Workaround: match positions to dispatch by driver_id + time range. Fix: server must accept dispatch_id and INSERT it.
25. **driver_positions null lat/lng.** Driver app background tracker sends positions before GPS has a fix. Server rejects with NOT NULL constraint on lat column. Fix: server filters null/NaN positions before INSERT. Driver app should also guard: only send when `coords.latitude && coords.longitude` are valid.
26. **Lingering timestamps after dispatch cancel/delete.** When a dispatch is cancelled or deleted, container milestone timestamps (`pod_full_out_at`, `empty_terminated_at`) must be reset. All three termination paths now handle this: (a) Driver reject: `POST /api/driver/loads/:id/reject` → resets container. (b) Dashboard cancel: `PATCH /api/dispatches/:id` with `status='cancelled'` → resets container. (c) Dashboard delete: `DELETE /api/dispatches/:id` → resets container BEFORE deleting row. All paths: reset `pod_full_out_at=NULL`, `empty_terminated_at=NULL`, `ui_status` back to AT_PORT (if discharged), log event to `container_events`. The map GPS trail should filter by active dispatch only — not show events from cancelled dispatches.
27. **Silent FTU API key expiry.** The FTU API key can expire without any visible error in the app. Containers stay IN_TRANSIT with empty vessel_name/pod_eta/terminal_name. The only signal is zero rows in `container_events` with `source='ftu'`. The `/api/health` endpoint now checks `ftu_last_event` and flags `ftu_stale=true` if no FTU data in 7 days. `GET /api/health/ftu` checks the key directly against FTU. E2E test 8.3 catches this. **Incident:** April 2026 — key expired silently, no webhook data for 6+ days, containers showed "On Ship" with no vessel/ETA/terminal.
28. **MMSI not in VWC because aggregation only runs for IN_TRANSIT containers.** The vwc-sync aggregation query filters `WHERE vessel_name IN (SELECT ... WHERE ui_status = 'IN_TRANSIT')`. Once all containers move to AT_PORT or later, the vessel row stops being updated. If MMSI wasn't written during the IN_TRANSIT window (e.g., AIS saw the vessel after the container discharged), VWC has null MMSI forever. Fix: `backfillMMSI()` step in vwc-sync runs every 30s after the main aggregation, queries VWC rows with null MMSI, looks up SQLite vessel_registry by name/IMO, and fills the gap. Also resolves the chicken-and-egg problem where AIS discovers the vessel after FTU stops reporting.

29. **GPS silence despite active dispatch (April 2026).** Symptom: `container_events` shows recent milestones (e.g., `en_route_pickup`) with GPS coordinates attached, but `driver_positions` has a multi-hour gap. Dispatcher sees a stale truck LED on the fleet map while the driver thinks GPS is working. Root cause: the `useGpsTracking` hook was inside a screen component (`[id].tsx` load detail). React unmounted the screen on navigation → cleanup called `stopLocationUpdatesAsync` → background GPS task killed. Milestones kept working because they use one-shot `getCurrentPositionAsync`, not the continuous `watchPositionAsync` stream. **Fix:** GPS hook moved to app root (`components/BackgroundServices.tsx` mounted in `_layout.tsx`). Never lives inside a screen component. Applies equally to `useEtaPolling`. **Diagnosis query:**
    ```sql
    SELECT max(recorded_at), now() - max(recorded_at) AS silence
      FROM driver_positions WHERE driver_id = $1;
    ```
    If `silence > 5 minutes` and the driver has an active dispatch, GPS is dead on the phone — check whether they have the latest APK, whether the app is still logged in, and whether Android battery optimization was ever whitelisted for FleetCraft.
### Dispatch Lifecycle Cleanup

| Action | Events | Positions | Audit |
|--------|--------|-----------|-------|
| Cancel (API) | Deleted (except dispatch_cancelled) | Deleted | dispatch_cancelled event preserved |
| Delete (API) | All deleted | All deleted | Nothing preserved |
| Complete | All kept | All kept | Full audit trail for detention |
| Cancel (direct SQL) | NOT cleaned — orphans left | NOT cleaned | NEVER do this in production |

All 3 API cancel paths (driver reject, dashboard PATCH cancel, dashboard DELETE) auto-clean events and positions. The `dispatch_cancelled` event is logged AFTER cleanup so it survives.

30. **Orphaned container_events after dispatch delete.** Dispatches deleted via direct SQL (psql, e2e test cleanup) bypass the API cancel handler, leaving orphaned `container_events` and `driver_positions`. These show as stale milestone markers on the dispatch map. Rules: (a) NEVER delete dispatches via direct SQL in production — always use the API. (b) E2E test cleanup MUST delete `container_events` and `driver_positions` BEFORE deleting dispatches. (c) To find orphans: `SELECT container_number, count(*) FROM container_events WHERE dispatch_id NOT IN (SELECT id FROM dispatches) GROUP BY container_number`. **Incident:** April 2026 — cancelled FCTEST dispatches left 29 orphaned events showing stale markers on the map.

29. **Archive resurrection via FTU webhook SQLite write.** The FTU webhook handler writes to SQLite without checking `container_exclusions`. If a webhook arrives after archive, it re-creates the container in SQLite. container-sync then pushes it back to Postgres (the INSERT path of ON CONFLICT bypasses the user_status guard). Fix: (a) FTU webhook checks `container_exclusions` BEFORE writing to SQLite — if excluded, skip and return. (b) SQLite upsert has `WHERE user_status IS NULL OR user_status = 'active'` guard — archived/dismissed containers bounce off. (c) Archive endpoint verifies completion — force-deletes from both Postgres and SQLite if container is still present after the transaction. **Incident:** April 2026 — HDMU5611750 and TGBU5159346 kept reappearing after archive.

31. **SYSTEMIC RULE — every container read endpoint MUST filter by user_status.** Symptom: dashboard header counts (total, on_ship, delivered, etc.) don't match the active container list shown below — because aggregates count `held`, `flagged`, and `dismissed` rows while the list hides them. **The predicate that MUST appear in every query that counts or lists containers:**
    ```sql
    AND (user_status IS NULL OR user_status = 'active')
    ```
    Verified endpoints (all must stay this way):
    | Endpoint | Status |
    |----------|--------|
    | `/containers/stats` | ✅ fixed `0dc02a8` |
    | `/containers/vessels` | ✅ fixed `f0a6782` |
    | `/api/containers/list` | ✅ already correct |
    | `/api/fleet/positions` | ✅ already correct (via dispatch join) |
    | `vwc-sync.js` aggregation | ✅ already correct |

    Any new read endpoint must include this filter. No exceptions unless it's explicitly the "all statuses" admin view. **Incident:** April 2026 — `/containers/stats` double-counted 5 held containers into the "Active Containers" tile; `/containers/vessels` hid the synthetic "Direct Request" vessel because the inner subquery also required `ui_status = 'IN_TRANSIT'` which direct-add containers never are.

32. **VWC ghost vessels after last container archived.** Symptom: "Containers — Live Updates" panel shows a vessel with "1 Unit" but every container on that vessel is archived. Root cause: `vwc-sync.js` early-returns when its aggregation finds zero active vessels, which skips the step-3b cleanup that deletes orphaned VWC rows. Archiving the last active container on a vessel leaves its VWC row permanently stuck. **Fix (commit `c35c741`):** cleanup runs regardless of whether new aggregation produced rows. **Rule:** any cleanup or DELETE logic in a sync loop must execute BEFORE early-return guards, not after them. The guard should only gate work that depends on having fresh input data.

33. **Direct-add containers missing `pod_discharged_at`.** Symptom: Discharged column blank for AT_PORT containers added via the Direct Terminal Pickup flow; detention "days-since-discharge" calculations start from null, producing nonsense. Root cause: `POST /api/containers/quick-add` INSERT never set `pod_discharged_at`. **Fix (commit `5398ef7`):** quick-add now sets `pod_discharged_at = NOW()` at creation. **Rule:** direct-add containers are already physically at the terminal; they need ALL "arrival" timestamps populated at creation time (discharged, last_seen, any other yard-phase anchor). FTU flow has these arrive via webhook — direct-add must synthesize them.

34. **`/containers/vessels` only shows IN_TRANSIT vessels.** Symptom: "Containers — Live Updates" dashboard card shows "No active vessels" even when there are 5 active containers at a terminal. Root cause: the endpoint's inner IN subquery required `ui_status = 'IN_TRANSIT'`, so any vessel whose containers are all AT_PORT (including the synthetic "Direct Request" vessel used by direct-add) got filtered out. **Fix (commit `f0a6782`):** removed the IN_TRANSIT gate — endpoint aggregates ALL active containers grouped by `vessel_name`, and the per-phase breakdown columns (`on_vessel`, `in_yard`, `picked_up`, `empty_returned`) carry the ui_status detail. Same error class as #31 (systemic rule).

---

## 11. Container Lifecycle Ownership

FTU owns steps 1-3 ONLY (vessel inbound → arrival → discharged in yard).
Once the container is in the yard, FTU is irrelevant to business logic.

| Steps | Owner | Trigger |
|-------|-------|---------|
| 1-3 | FTU (vessel inbound → arrival → discharge) | FTU webhook |
| BYPASS | Direct terminal pickup — skips steps 1-3 | POST /api/containers/quick-add |
| 4-7 | Future Tradlinx (holds → available → LFD) | Not yet integrated |
| 8 | Web UI (dispatch created) | POST /api/dispatches |
| 9-25 | Driver app (pickup → delivery → empty return → complete) | POST /api/driver/loads/:id/milestone |
| Auto-archive | container-sync.js | `dispatches.completed_at` + 24 hours |

### Direct Terminal Pickup (FTU Bypass)
When a container is already at the terminal, dispatchers can skip FTU entirely:
- `POST /api/containers/quick-add` writes directly to Postgres with `data_source = 'direct'`, `ui_status = 'AT_PORT'`, `available_for_pickup = true`
- No SQLite write, no FTU registration, no vessel tracking
- `vessel_name` defaults to `'Direct Request'` (overridable)
- `terminal_name` is required (dispatcher selects from dropdown)
- `terminal_code` is derived from `terminal_name` using a mapping table (e.g., "PCT Tacoma" → "PCT", "Husky Terminal" → "HUSKY")
- Container enters the lifecycle at step 8 (dispatch creation) — identical from that point forward
- container-sync never sees these containers (not in SQLite) — natural isolation
- vwc-sync ignores them ("Direct Request" has no MMSI)
- Auto-archive rules still apply (dispatch completion + 24h, EMPTY_RETURNED + 24h)

**Auto-archive trigger: `dispatches.completed_at` + 24 hours. NOT FTU completed flag.**

### Geofence Embedding in Dispatches
Dispatch creation embeds geofences from the `geofences` table into `pickup_geofences` JSONB column. The driver app reads this at load time and arms on-device detection via `useGpsTracking.ts`. If `pickup_geofences` is empty `[]`, on-device detection silently does nothing.

### terminal_code Resolution
The quick-add endpoint derives `terminal_code` from `terminal_name` using a mapping table (e.g., "PCT Tacoma" → "PCT", "Husky Terminal" → "HUSKY"). This is critical because dispatch creation embeds geofences by looking up `terminal_code`. Without `terminal_code`, `pickup_geofences` will be empty `[]`.

Dispatch geofence lookup has three fallbacks:
1. `terminal_code` from request body
2. `container.terminal_code` or `container.moored_terminal_code`
3. `container.terminal_name` matched against geofences table

If all three fail, `pickup_geofences = []` and on-device detection will not fire. Direct-add containers must have `terminal_code` set (derived from `terminal_name` mapping in quick-add endpoint).

**Rule:** `terminalCodeMap` in server.js must include every `terminal_code` that appears in the `geofences` table. If a new terminal is added to geofences, add it to `terminalCodeMap` too — otherwise dispatch creation won't resolve the code and `pickup_geofences` will be empty. Current terminals: HUSKY, PCT, T18, T5, FCTEST.

### Orphaned dispatch auto-cancellation

When a container's `ui_status` changes to `OUT_FOR_DELIVERY` or `EMPTY_RETURNED` via FTU webhook (not via driver app milestone), check for orphaned dispatches:

```sql
SELECT id FROM dispatches
WHERE container_number = $1
  AND status IN ('pending', 'assigned', 'en_route_pickup')
```

Verify no driver milestones exist:
```sql
SELECT count(*) FROM container_events
WHERE container_number = $1
  AND source = 'driver_app'
  AND dispatch_id = $dispatch_id
```

If dispatch exists AND zero driver milestones:
1. `UPDATE dispatches SET status = 'cancelled', completed_at = NOW() WHERE id = $dispatch_id`
2. `INSERT INTO container_events (event_type='dispatch_cancelled', source='system', payload='{"reason":"container_moved_without_driver"}')`
3. Log: "Dispatch {dispatch_number} auto-cancelled — container {container_number} moved without driver"

The container then follows normal lifecycle:
- `OUT_FOR_DELIVERY`: stays active, may get a new dispatch
- `EMPTY_RETURNED`: auto-archives after 24h via `archiveStaleContainers()`
- Both paths build `lifecycle_snapshot` at archive time

This check runs in `container-sync` when `ui_status` changes, NOT in the FTU webhook handler. The webhook handler only writes data. Business logic belongs in `container-sync`.

---

## 12. Self-Documenting Files — Required Header

> **Goal:** If this file gets deleted, the header comments in neighboring files plus the skill files give AI enough context to rebuild it correctly.

Every .js worker file MUST have this header block:

```javascript
/**
 * <filename> — <one-line purpose>
 *
 * READS FROM: <databases/APIs this file reads>
 * WRITES TO:  <databases/APIs this file writes>
 * CALLED BY:  <PM2 cron interval, or HTTP request, or other trigger>
 * DEPENDS ON: <other workers/files that must run for this to work>
 * DEPENDED ON BY: <other workers/files that read this file's output>
 *
 * <2-3 sentences explaining the critical rules this file follows>
 */
```

### Example:
```javascript
/**
 * container-sync.js — CQRS Query Side Sync: SQLite → Postgres + Auto-Archive
 *
 * READS FROM: SQLite container-registry.db (containers table), Postgres container_exclusions, dispatches
 * WRITES TO:  Postgres fleetcraft_db (containers, archived_containers, container_exclusions, container_events)
 * CALLED BY:  PM2 (sync every 30s, archive every 2min)
 * DEPENDS ON: Fleet API webhook handler populating SQLite
 * DEPENDED ON BY: Fleet API reads Postgres, frontend reads via API, vwc-sync reads containers
 *
 * Single writer for container data in Postgres. Also handles auto-archiving (stale + dispatch-completed).
 * Loads tombstone set from container_exclusions at start of each cycle — skips excluded containers.
 * Upsert includes WHERE guard on user_status. Must never overwrite user_status.
 */
```

When Claude creates or modifies any backend file, it MUST include or update this header.

---

## Dispatch creation — coordinate sources

- **Gate coordinates for HERE routing:** READ FROM `terminals.lat` / `terminals.lng` (primary source of truth).
- **Detection polygon:** READ FROM `geofences.coordinates` → embedded in `dispatches.pickup_geofences` JSONB at creation time.
- **NEVER** read `routing_lat` / `routing_lng` from `geofences` — deprecated (migration 005).
- Fallback order in `POST /api/dispatches` setImmediate when `pickup_lat` / `pickup_lng` are missing from the request:
  1. `terminals.lat/lng` where `terminal_code` matches.
  2. Centroid of `geofences.coordinates` (approximate — warn in logs so operator sets a precise terminal lat/lng).
  3. Leave null; auto-route skips.

## Dispatch ETA columns (Spec 0013, not yet built)

- `eta_predicted_minutes`: latest HERE ETA with traffic
- `eta_base_minutes`: HERE ETA without traffic
- `eta_congestion_minutes`: derived (`eta_traffic_minutes - eta_base_minutes`)
- `eta_first_predicted_minutes`: baseline at trip start (detention evidence)
- `actual_trip_minutes`: real elapsed time (detention evidence)
- `eta_last_updated`: freshness indicator

Migration: adds columns to `dispatches` table via `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`.
Written by new `GET /api/dispatches/:id/eta` endpoint (server-side HERE Routing proxy).

## Detention dispute evidence chain (Spec 0013)

Nine timestamps / derived values that together constitute defensible detention evidence:

1. `eta_first_predicted_minutes` — what HERE said at trip start
2. `actual_trip_minutes` — how long it actually took
3. Difference between 1 and 2 = independent congestion proof (third-party)
4. `queue_start_at` — geofence polygon entry (on-device ray casting fire-once)
5. `pickup_arrived_at` — driver ingate tap
6. `pickup_completed_at` — driver gate-out tap
7. **Road queue** = `pickup_arrived_at - queue_start_at` (time crawling to gate)
8. **Terminal dwell** = `pickup_completed_at - pickup_arrived_at` (time inside the yard)
9. **Total detention** = `pickup_completed_at - queue_start_at`

This chain is the commercial justification for Spec 0013 v2: drayage operators
bill detention per hour; auditable third-party evidence (HERE) is required for dispute resolution.
