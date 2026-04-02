---
name: fleetcraft-backend
description: >
  FleetCraft backend development rules. Use this skill whenever working on
  server.js (Fleet API), container-sync.js, ftu-tracker.js, archive-worker.js,
  dispatcher-orchestrator.js, vessel-sync.js, dispatch-agent.js, or any PM2
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
| Container archive | archive-worker + manual archive endpoint | `archive-worker.js`, `server.js` | SQLite (archived_at + user_status) → Postgres via container-sync |
| Container exclusions | Fleet API endpoints | `server.js` | Postgres container_exclusions (tombstone table) |
| Vessel AIS positions | AIS collector | `index.js` (ais-collector-v2) | Postgres vessels table |
| Vessel cache | vessel-sync | `vessel-sync.js` | Postgres vessels_with_containers (must filter by user_status) |
| Dispatch records | Fleet API | `server.js` | Postgres dispatches table |
| Driver milestones | Fleet API (from driver app) | `server.js` | Postgres container_events table |
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

### Correct Archive Sequence (Auto)

```
archive-worker.js runs every 2 minutes:
  1. Queries Postgres for completed dispatches older than 24h
  2. Sets archived_at AND user_status = 'archived' in SQLite for matching containers
  3. INSERT INTO container_exclusions in Postgres
  4. container-sync.js picks up archived_at IS NOT NULL
  5. Moves to archived_containers in Postgres
  6. Calls FTU unregister (only for auto-archive, NOT manual — manual already did it)
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
- `archive-worker.js` sets `archived_at` AND `user_status` in SQLite, creates exclusion record
- `container-sync.js` handles Postgres sync AND moves to `archived_containers`
- `ftu-tracker.js` does cache refresh ONLY — no archive logic
- Manual archive in `server.js` handles FTU unregister inline — `syncArchivedContainers()` skips `manual`/`manual_archive` reasons to avoid duplicate API calls
- The `archived_containers` table is a permanent record — never delete from it

---

## 4. FTU Integration Rules

> **Lesson learned:** FTU is an event-based webhook API, not a polling API. The old ftu-tracker.js was calling Terminal 49 endpoints that don't exist on FTU. T49 is fully deactivated.

### Active System
- **FTU (FindTEU)** via `ftu-tracker.js` — cache refresh only
- Webhook pushes from FTU arrive at Fleet API → SQLite → Postgres
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
| 0 | dispatcher | dispatcher-orchestrator.js | online | Email/alert dispatching |
| 1 | vessel-sync | vessel-sync.js | online | Postgres vessel cache refresh (filters by user_status) |
| 2 | ais-collector-v2 | index.js | online | AIS vessel position collection |
| 3 | ftu-tracker | ftu-tracker.js | online | FTU container cache refresh (checks tombstone before upsert) |
| 4 | container-sync | container-sync.js | online | SQLite → Postgres container sync (tombstone check + upsert guard) |
| 5 | dispatch-worker | dispatch-worker.js | STOPPED | Postgres pool migration needed |
| 7 | fleet-api | server.js | online | Express API (port 3001) |
| 8 | archive-worker | archive-worker.js | online | Auto-archive completed containers (writes user_status + exclusion) |

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

## 9. Vessel Tracking & Cache Rules

> **Lesson learned:** SOG of 0.1 knots (GPS drift) divided into distance produced a 212-hour ETA for a moored vessel.

### AIS Rules
- SOG threshold: **>1 knot** to count as moving. SOG ≤1 = stationary/GPS drift.
- Moored/anchored vessels: **exclude from ETA calculations** in both ftu-tracker and dispatcher
- `nav_status_label === 'Moored'` → skip in 24H alert loop, do not clear alerted flags
- "In Processing" at terminal = container discharged, undergoing customs/terminal handling — definitively NOT on vessel. Count in `at_terminal_count`.

### vessel-sync.js Rules
- When counting containers per vessel, filter by `user_status`:
  ```sql
  SELECT vessel_name, COUNT(*) FROM containers
  WHERE vessel_name IS NOT NULL
    AND (user_status IS NULL OR user_status = 'active')
  GROUP BY vessel_name
  ```
- When a vessel's active container count drops to **zero**, DELETE its row from `vessels_with_containers`. The vessel still exists in the `vessels` table (AIS keeps tracking), but it disappears from the dashboard.
- When new active containers arrive on that vessel, it reappears in `vessels_with_containers` automatically on the next sync cycle.
- **Terminal-agnostic**: No hardcoded terminal names in any vessel-sync query. The system works identically for 2 terminals or 200.

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

---

## 11. Self-Documenting Files — Required Header

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
 * container-sync.js — CQRS Query Side Sync: SQLite → Postgres
 *
 * READS FROM: SQLite container-registry.db (containers table), Postgres container_exclusions
 * WRITES TO:  Postgres fleetcraft_db (containers, archived_containers, vessels_with_containers)
 * CALLED BY:  PM2 (every 30 seconds)
 * DEPENDS ON: ftu-tracker.js populating SQLite, archive-worker.js setting archived_at
 * DEPENDED ON BY: Fleet API reads Postgres, frontend reads via API, vessel-sync reads containers
 *
 * Single writer for container data in Postgres. Loads tombstone set from container_exclusions
 * at start of each cycle — skips excluded containers. Upsert includes WHERE guard on user_status.
 * Must never overwrite user_status. Must include findteu_shipment_id in column list.
 */
```

When Claude creates or modifies any backend file, it MUST include or update this header.
