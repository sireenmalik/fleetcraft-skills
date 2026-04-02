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
- Dispatches, drivers, trucks, chassis, events — Postgres ONLY, no SQLite copy.

---

## 2. container-sync.js Column Mapping

When adding a column to the containers table, you MUST update THREE places:

1. **SQLite schema** — `ALTER TABLE containers ADD COLUMN <name> <type>`
2. **Postgres schema** — `ALTER TABLE containers ADD COLUMN <name> <type>`
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
         archived_at, archive_reason, updated_at
         -- ADD NEW COLUMNS HERE
  FROM containers
  WHERE archived_at IS NULL
`).all();

// Postgres write — every synced column must ALSO be listed here
// AND in the ON CONFLICT SET clause
```

### The Sync Guard
container-sync.js must NEVER sync rows where `archived_at IS NOT NULL` to the active `containers` table. Those go to `archived_containers` via `syncArchivedContainers()`.

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
Only `ui_status = 'AT_PORT'` containers are dispatch-ready. FTU does not provide hold or availability flags — those are blind until Tradlinx is integrated.

---

## 4. FTU Data Gaps (Always Null)

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

## 5. archived_containers Table

Permanent record of all archived containers. Never delete from this table.

### Flow:
```
containers (active) → archived_at set → container-sync moves to archived_containers → deleted from containers
```

### Schema matches containers table plus:
- `archived_at` — when archived
- `archive_reason` — why (auto_completed, manual, manual_archive, stale)

---

## 6. container_events Table (Event Ledger)

Append-only. Never update or delete rows.

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | Unique event ID |
| container_number | text | FK to containers |
| org_id | uuid | Multi-tenant scope |
| event_type | text | ftu_update, milestone, status_change, archive, restore, etc. |
| source | text | ftu, driver, dispatcher, system |
| payload | jsonb | Full event data |
| occurred_at | timestamptz | When the event actually happened |
| created_at | timestamptz | When we recorded it |

### Rules:
- Every FTU webhook writes an event
- Every driver milestone writes an event
- Every archive/restore writes an event
- Events are the audit trail — if current state is wrong, rebuild from events

---

## 7. Multi-Tenancy

- Every table has `org_id` (uuid)
- Every query filters by `org_id`
- Composite indexes: always `(org_id, <primary_filter>)` — e.g. `(org_id, container_number)`, `(org_id, ui_status)`
- Primary test org: `f8107db3-ecaa-48e1-968d-0e89c6dd8f62`

---

## 8. Schema Change Checklist

Before deploying any schema change:

- [ ] Write a numbered migration file (see section 10)
- [ ] SQLite ALTER TABLE applied (if container-related)
- [ ] Postgres ALTER TABLE applied via the migration file
- [ ] container-sync.js column list updated (SELECT + INSERT + UPDATE)
- [ ] Fleet API endpoints updated if new column needs to be read/written
- [ ] Frontend updated if new column needs to be displayed
- [ ] This skill file updated with the new column

---

## 9. Postgres Connection

```bash
# From the droplet — TCP required, peer auth fails:
psql -h localhost -U fleetcraft -d fleetcraft_db

# Connection string for Node.js:
postgresql://fleetcraft:<password>@localhost:5432/fleetcraft_db
```

### navigation_order gotcha
The `nav_items` column has a NOT NULL constraint. Always default to `[]` (empty JSON array) before INSERT. Failing to do this causes 500 errors in fleet-api logs.

---

## 10. Schema as Code — Migration Files

> **Rule:** Never run ad-hoc ALTER TABLE directly in psql. Every schema change is a numbered SQL file committed to Git.

### Location:
```
fleetcraft-api/
  migrations/
    001_initial_schema.sql
    002_add_container_events.sql
    003_add_user_status.sql
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
