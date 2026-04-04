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
  4. DELETE from SQLite immediately
  5. HTTP DELETE to FTU /container/subscription/{findteu_shipment_id} (best-effort)
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

archiveStaleContainers():
  1. Queries Postgres for stale containers (EMPTY_RETURNED >24h, OUT_FOR_DELIVERY >7d)
  2. Same flow: SQLite archived_at + user_status + Postgres tombstone
  3. archive_reason: 'auto_empty_returned' or 'auto_stale_ofd'
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
  Note: Next sync cycle resumes normal updates. May need FTU re-registration.
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

---

## 6. PM2 Workers — Current State

| ID | Name | Script | Status | Purpose |
|----|------|--------|--------|---------|
| 0 | dispatcher | dispatcher-orchestrator.js | online | Email/alert dispatching (Resend) |
| 2 | ais-collector-v2 | index.js | online | AIS WebSocket → SQLite vessel_registry |
| 4 | container-sync | container-sync.js | online | SQLite → PG containers + auto-archive (stale + dispatch) |
| — | vwc-sync | vwc-sync.js | online | Single owner of vessels_with_containers + vessels_cache |
| 7 | fleet-api | server.js | online | Express API (port 3001) |
| 5 | dispatch-worker | here-dispatch-worker.js | STOPPED | Future HERE routing |

### Killed Workers (do NOT restart)
| ID | Name | Reason killed | Replaced by |
|----|------|--------------|-------------|
| 1 | vessel-sync | Dual-writer on vessels_with_containers | vwc-sync.js |
| 3 | ftu-tracker | Dual-writer on vessels_with_containers, cache refresh redundant | vwc-sync.js |
| 8 | archive-worker | Logic merged into container-sync.js | container-sync.js |

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
- **ETA logic:** AIS ETA for WA-bound vessels (SOG > 1), FTU pod_eta for others
- **Terminal geofence mapping:** Moored vessel inside terminal polygon → update container terminal_name in SQLite
- **Stale cleanup:** Zero active containers AND (no alert flags OR updated > 7 days ago) → delete from cache
- **Never writes `alerted_*` columns** — dispatcher owns those
- **Terminal-agnostic**: No hardcoded terminal names. Works for 2 terminals or 200.

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

---

## 11. Container Lifecycle Ownership

FTU owns steps 1-3 ONLY (vessel inbound → arrival → discharged in yard).
Once the container is in the yard, FTU is irrelevant to business logic.

| Steps | Owner | Trigger |
|-------|-------|---------|
| 1-3 | FTU (vessel inbound → arrival → discharge) | FTU webhook |
| 4-7 | Future Tradlinx (holds → available → LFD) | Not yet integrated |
| 8 | Web UI (dispatch created) | POST /api/dispatches |
| 9-25 | Driver app (pickup → delivery → empty return → complete) | POST /api/driver/loads/:id/milestone |
| Auto-archive | container-sync.js | `dispatches.completed_at` + 24 hours |

**Auto-archive trigger: `dispatches.completed_at` + 24 hours. NOT FTU completed flag.**

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
