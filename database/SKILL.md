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

## 3. ui_status Values

These are FTU-derived. Do not invent new values.

| ui_status | Meaning | FTU trigger |
|-----------|---------|-------------|
| `IN_TRANSIT` | On vessel, en route | Default until discharged |
| `AT_PORT` | Discharged at terminal | FTU 'discharged from' event |
| `OUT_FOR_DELIVERY` | Gate out from terminal | FTU 'gate out full' event |
| `EMPTY_RETURNED` | Empty container returned | FTU 'gate in empty' event |

### Dispatch eligibility
Only containers where `ui_status = 'AT_PORT'` AND `user_status = 'active'` are dispatch-ready. Both conditions required.

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

Dispatch row keeps all timestamps for audit (`status='cancelled'`, `completed_at` set).
`container_events` are NEVER deleted — append-only audit ledger.
GPS positions (`driver_positions`) are kept — they record what happened.

The map in the container detail panel should only show events and GPS from the CURRENT active dispatch, not from cancelled ones. Filter by `dispatch_id` or by `occurred_at > current_dispatch.created_at`.

### Test data cleanup
For the permanent test container ALPHA1234: after cancelling a dispatch, also clean up `container_events` and `driver_positions` from the test run so the map shows clean for the next test. This is test-only — production containers keep all events.
