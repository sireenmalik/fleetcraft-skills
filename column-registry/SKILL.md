---
name: fleetcraft-column-registry
description: >
  FleetCraft column registry and cross-layer data contracts. Use this skill BEFORE
  adding, renaming, or removing any database column. Maps every active field through
  all 5 layers (external API -> SQLite -> container-sync -> Postgres -> fleet-api -> UX).
  Prevents the terminal_code gap bug where a column exists in one layer but is missing
  from the sync bridge.
---

# FleetCraft Column Registry — Cross-Layer Data Contracts

Every column that moves data between layers has a contract. If a column is missing from
any layer in its contract, data silently disappears. This registry is the single source
of truth for what exists where.

> **Origin bug:** `terminal_code` was added to SQLite (layer 2d) and vwc-sync (layer 2b)
> but missing from container-sync (layer 2c). Data wrote to SQLite, never reached Postgres,
> dispatch geofence embedding broke silently. This registry prevents that class of bug.

---

## Architecture: The 5 Layers

```
Layer 1: External API (FTU webhooks, AIS stream, Tradlinx future)
    |
Layer 2a: Ingestion (server.js FTU webhook handler writes to SQLite)
Layer 2b: Enrichment (vwc-sync.js — AIS positions, terminal geofence mapping)
Layer 2c: Sync Bridge (container-sync.js — SQLite -> Postgres)
Layer 2d: SQLite schema (container-registry.db containers table)
    |
Layer 3: Postgres schema (fleetcraft_db containers table)
    |
Layer 4a: Intelligence (dispatcher-orchestrator.js reads Postgres)
Layer 4b: Fleet API (server.js endpoints read/write Postgres)
    |
Layer 5: UX (React dashboard, Expo driver app)
```

### Key rule
**container-sync.js is the bridge.** If a column exists in SQLite but is NOT in
container-sync's INSERT/UPDATE column list, the data stays in SQLite forever and
Postgres never sees it.

---

## Contract 1: Core Container Identity

These columns identify a container and never change after creation.

| Column | SQLite | container-sync | Postgres | fleet-api | UX |
|--------|--------|---------------|----------|-----------|-----|
| container_number | PK | SELECT + INSERT | PK (varchar) | read | display |
| org_id | default | SELECT + INSERT | uuid | filter all queries | -- |
| shipment_ref | write | SELECT + INSERT | varchar | read | -- |
| bill_of_lading | write | SELECT + INSERT | varchar | read | display |
| findteu_shipment_id | write | SELECT + INSERT (COALESCE) | text | FTU unregister | -- |
| data_source | default 'findteu', also 'direct' (quick-add), 'test' (E2E) | SELECT + INSERT | text | read | -- |
| tracking_request_id | write | -- | -- | -- | -- |

### Notes
- `findteu_shipment_id` uses COALESCE in container-sync to preserve existing value
- `tracking_request_id` is SQLite-only (FTU internal tracking, not synced)

---

## Contract 2: Status & Lifecycle

| Column | Writer | SQLite | container-sync | Postgres | fleet-api | UX |
|--------|--------|--------|---------------|----------|-----------|-----|
| status | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| ui_status | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read + filter | tabs |
| current_status | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| user_status | Fleet API ONLY | write | SELECT + INSERT (NEVER in UPDATE SET) | text | read + write | filter |
| archived_at | container-sync / API | write | SELECT + INSERT + UPDATE | timestamptz | read + write | filter |
| archive_reason | container-sync / API | write | SELECT + INSERT + UPDATE | text | read | display |
| data_hash | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | -- | -- |
| update_count | FTU webhook | write | SELECT + INSERT + UPDATE | integer | -- | -- |
| last_checked | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | -- | -- |
| last_modified | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | -- | -- |
| created_at | auto | auto | -- | auto | read | -- |
| updated_at | auto | SELECT + INSERT + UPDATE | timestamptz | read | sort |

### CRITICAL: user_status sync guard
```
WHERE containers.user_status IS NULL OR containers.user_status = 'active'
```
container-sync's ON CONFLICT DO UPDATE has this WHERE clause. If user_status is
'archived', 'dismissed', 'held', or 'flagged', the sync writer skips the row.
user_status is NEVER in the UPDATE SET clause — only Fleet API writes it.

---

## Contract 3: Vessel & Voyage

| Column | Writer | SQLite | container-sync | Postgres | fleet-api | UX |
|--------|--------|--------|---------------|----------|-----------|-----|
| vessel_name | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| vessel_imo | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| vessel_mmsi | AIS enrichment | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| voyage_number | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| port_of_discharge | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| eta | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | read | display |
| pod_eta | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | read | display |
| pod_arrival | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | read | display |
| pod_discharged_at | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | read | display |

---

## Contract 4: Terminal & Location

| Column | Writer | SQLite | container-sync | Postgres | fleet-api | UX |
|--------|--------|--------|---------------|----------|-----------|-----|
| terminal_name | FTU / vwc-sync geofence | write | SELECT + INSERT + UPDATE | varchar | read | display |
| terminal_firms_code | vwc-sync geofence | write | -- | varchar | read | -- |
| terminal_code | vwc-sync geofence | write | SELECT + INSERT (COALESCE) | text | dispatch geofence lookup | -- |
| pod_terminal | FTU / vwc-sync geofence | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| pod_name | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| pol_terminal | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| pol_name | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| yard_location | Terminal-specific (future) | write | SELECT + INSERT + UPDATE | varchar | read | display |

### Terminal geofence chain (vwc-sync.js / ftu-tracker.js):
```
AIS position (lat/lng) + nav_status = 'Moored'
    -> pointInPolygon(lat, lng, terminal.polygon)
    -> matched terminal { name, code, firms_code }
    -> UPDATE containers SET terminal_name, pod_terminal, terminal_firms_code, terminal_code
       WHERE container_number = ? AND (terminal_name IS NULL OR terminal_name = '')
```

### CRITICAL: terminal_code must be in container-sync
`terminal_code` is written to SQLite by vwc-sync geofence mapping. container-sync
must include it in the INSERT column list with COALESCE to preserve existing values:
```sql
terminal_code = COALESCE(EXCLUDED.terminal_code, containers.terminal_code)
```

---

## Contract 5: Holds, Fees & Availability

| Column | Writer | SQLite | container-sync | Postgres | fleet-api | UX |
|--------|--------|--------|---------------|----------|-----------|-----|
| customs_hold | FTU / Tradlinx (future) | int 0/1 | SELECT + INSERT + UPDATE (cast to bool) | boolean | read | hold badge |
| freight_hold | FTU / Tradlinx (future) | int 0/1 | SELECT + INSERT + UPDATE (cast to bool) | boolean | read | hold badge |
| terminal_hold | FTU / Tradlinx (future) | int 0/1 | SELECT + INSERT + UPDATE (cast to bool) | boolean | read | hold badge |
| customs_released | FTU | int 0/1 | SELECT + INSERT + UPDATE (cast to bool) | boolean | read | -- |
| freight_released | FTU | int 0/1 | SELECT + INSERT + UPDATE (cast to bool) | boolean | read | -- |
| available_for_pickup | FTU | int 0/1 | SELECT + INSERT + UPDATE (cast to bool) | boolean | read | badge |
| last_free_day | Terminal (future) | text | SELECT + INSERT + UPDATE | timestamptz | read | urgency |
| ssl_lfd | FTU | text | -- | date | read | -- |
| demurrage_amount | Terminal (future) | real | SELECT + INSERT + UPDATE | numeric | read | display |
| tags | FTU | JSON text | SELECT + INSERT + UPDATE (stringify) | jsonb | read | -- |
| holds | FTU | JSON text | SELECT + INSERT + UPDATE (stringify) | jsonb | read | -- |
| fees | FTU | JSON text | SELECT + INSERT + UPDATE (stringify) | jsonb | read | -- |

### Type casting in container-sync:
```javascript
!!c.customs_hold    // SQLite int 0/1 -> Postgres boolean
!!c.freight_hold
!!c.terminal_hold
JSON.stringify(safeParseJson(c.tags, []))   // SQLite text -> Postgres jsonb
JSON.stringify(safeParseJson(c.holds, {}))
JSON.stringify(safeParseJson(c.fees, {}))
```

---

## Contract 6: Equipment & Parties

| Column | Writer | SQLite | container-sync | Postgres | fleet-api | UX |
|--------|--------|--------|---------------|----------|-----------|-----|
| container_size | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| equipment_type | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| equipment_length | FTU webhook | write | SELECT + INSERT + UPDATE | integer | read | -- |
| equipment_height | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| origin | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| destination | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| shipper | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| consignee | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| customer | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| shipping_line | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | display |
| shipping_line_scac | FTU webhook | write | SELECT + INSERT + UPDATE | varchar | read | -- |
| pod_full_out_at | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | read | -- |
| empty_terminated_at | FTU webhook | write | SELECT + INSERT + UPDATE | timestamptz | read | -- |

---

## vessels_with_containers Cache Columns

Single writer: `vwc-sync.js` (data columns) + `dispatcher-orchestrator.js` (alert flags ONLY).

| Column | Writer | Source | Notes |
|--------|--------|--------|-------|
| vessel_name | vwc-sync | SQLite GROUP BY | PK (unique) |
| org_id | vwc-sync | SQLite | multi-tenant |
| mmsi | vwc-sync | SQLite vessel_mmsi | AIS identity |
| imo | vwc-sync | SQLite vessel_imo | AIS identity |
| container_count | vwc-sync | COUNT(*) | all active containers |
| on_vessel_count | vwc-sync | ui_status = 'IN_TRANSIT' | |
| at_yard_count | vwc-sync | ui_status = 'AT_PORT' | |
| out_for_delivery_count | vwc-sync | ui_status = 'OUT_FOR_DELIVERY' | |
| empty_returned_count | vwc-sync | ui_status = 'EMPTY_RETURNED' | |
| urgent_count | vwc-sync | LFD within 7 days | |
| holds_count | vwc-sync | any hold = true | |
| available_count | vwc-sync | available_for_pickup = 1 | |
| lat, lng, sog, cog, heading | vwc-sync | vessels_cache AIS | |
| nav_status_label | vwc-sync | vessels_cache AIS | |
| destination | vwc-sync | vessels_cache AIS | |
| callsign | vwc-sync | vessels_cache AIS | |
| vessel_type | vwc-sync | vessels_cache AIS | |
| distance_nm | vwc-sync | calculated from lat/lng | |
| eta_hours | vwc-sync | distance_nm / sog | only if SOG > 1 |
| alerted_24h | dispatcher ONLY | alert state | vwc-sync must NOT write |
| alerted_2h | dispatcher ONLY | alert state | vwc-sync must NOT write |
| alerted_30m | dispatcher ONLY | alert state | vwc-sync must NOT write |
| alerted_moored | dispatcher ONLY | alert state | vwc-sync must NOT write |
| last_alert_at | dispatcher ONLY | alert state | vwc-sync must NOT write |

### CRITICAL: vwc-sync must NEVER write alerted_* columns
The DELETE cleanup query must check `alerted_* = false` before removing a vessel.
If any alert flag is true, the vessel row is preserved even with zero containers.

---

## Postgres-Only Columns (not synced from SQLite)

These columns exist in Postgres `containers` but are written by other sources, not container-sync:

| Column | Writer | Purpose |
|--------|--------|---------|
| moored_lat | vwc-sync geofence | AIS lat when moored |
| moored_lng | vwc-sync geofence | AIS lng when moored |
| moored_terminal_code | vwc-sync geofence | terminal code from AIS match |
| moored_at | vwc-sync geofence | timestamp of moored detection |
| t49_tracking_id | legacy (dead) | Terminal49 — deactivated |
| raw_t49_container | legacy (dead) | Terminal49 — deactivated |
| raw_t49_shipment | legacy (dead) | Terminal49 — deactivated |
| raw_t49_terminal | legacy (dead) | Terminal49 — deactivated |
| tracking_source | fleet-api | 'findteu' or 'terminal49' |
| container_type | fleet-api | derived from equipment |
| container_length | fleet-api | derived from equipment |
| weight_in_lbs, weight_kg | FTU (if provided) | cargo weight |
| seal_number | FTU (if provided) | seal ID |
| normalized_number | fleet-api | cleaned container number |
| submitted_at | fleet-api | when user added container |
| availability_known | fleet-api | derived boolean |
| pickup_appointment_at | fleet-api | user-set appointment |
| delivered_at | fleet-api | delivery timestamp |
| pod_* (rail/timezone/tracking) | FTU extended | rail movement fields |
| pol_* (atd/etd/timezone) | FTU extended | port of loading fields |
| ind_* | FTU extended | intermodal fields |
| destination_* | FTU extended | final destination fields |
| import_deadlines | FTU extended | deadline JSONB |
| shipment_ref_numbers | FTU extended | reference JSONB |
| container_ref_numbers | FTU extended | reference JSONB |
| estimated_arrival | fleet-api | computed ETA |
| actual_arrival | fleet-api | actual arrival |
| discharged_at | fleet-api | discharge timestamp |

---

### Direct-Add Containers (Postgres-Only)

Containers added via `POST /api/containers/quick-add` with `data_source = 'direct'` exist in Postgres only — they never enter SQLite. This means:
- container-sync.js never sees them (reads from SQLite)
- No column sync gap risk (no bridge to cross)
- Archive must skip SQLite DELETE (`data_source != 'direct'` guard)
- Archive must skip FTU unregister (`findteu_shipment_id IS NULL`)
- Restore must skip FTU re-registration (`data_source = 'direct'` guard)
- All other Postgres operations (dispatch, milestones, auto-archive) work identically

---

## Legacy / Dead Columns

These exist in the schema but are no longer actively written:

| Column | Table | Status | Notes |
|--------|-------|--------|-------|
| t49_tracking_id | containers | DEAD | Terminal49 deactivated |
| raw_t49_container | containers | DEAD | Terminal49 deactivated |
| raw_t49_shipment | containers | DEAD | Terminal49 deactivated |
| raw_t49_terminal | containers | DEAD | Terminal49 deactivated |
| raw_data | SQLite only | DEAD | was T49 raw payload |

Do NOT delete these columns — they may contain historical data. Just don't write to them.

---

## New Column Checklist

Copy this checklist every time you add a new column. Every box must be checked or
explicitly marked N/A before deploying.

```
Column name: _______________
Data type:   _______________
Writer:      _______________

- [ ] 1. SQLite: ALTER TABLE containers ADD COLUMN <name> <type>
         (on live DB + container-registry-schema.sql)
- [ ] 2. Postgres: migration file NNN_add_<name>.sql
         ALTER TABLE containers ADD COLUMN IF NOT EXISTS <name> <type>;
- [ ] 3. container-sync.js: added to INSERT column list
- [ ] 4. container-sync.js: added to VALUES ($N) parameter list
- [ ] 5. container-sync.js: added to ON CONFLICT DO UPDATE SET clause
         (UNLESS this is a user-intent column like user_status)
- [ ] 6. container-sync.js: added to parameter array with correct type cast
         (bool: !!c.col, JSON: JSON.stringify(safeParseJson(c.col, fallback)))
- [ ] 7. vwc-sync.js: updated if vessel/terminal related
- [ ] 8. fleet-api: endpoint updated if column needs to be read or written
- [ ] 9. Frontend: component updated if user-facing
- [ ] 10. archived_containers: included if needed for archive snapshot
- [ ] 11. This registry: contract table updated with new column
- [ ] 12. validate-schema.sh: added to critical column checks if applicable
- [ ] 13. Run validate-schema.sh on droplet — all checks pass
- [ ] 14. If dispatch column: ALL dispatch read endpoints return it
         (GET /dispatches, GET /api/dispatches/:id, GET /api/driver/loads,
          GET /api/driver/loads/:id, GET /api/driver/history)
- [ ] 15. Frontend fetchDispatchLookup maps the field
- [ ] 16. Frontend DispatchInfo interface includes the field
```

---

## dispatches Table — Key Columns

The dispatches table is Postgres-only (no SQLite, no sync). Written by fleet-api.

| Column | Type | Writer | Notes |
|--------|------|--------|-------|
| container_number | text | fleet-api | FK to containers |
| terminal_code | text | fleet-api | from container's terminal_code at dispatch time |
| pickup_terminal | text | fleet-api | terminal name for pickup |
| pickup_geofences | jsonb | fleet-api | geofence polygon for terminal |
| status | text | fleet-api / driver | dispatch lifecycle |
| move_type | text | fleet-api | import_dray, export_dray, etc. |

### Dispatch geofence chain:
```
Step 1: container.terminal_code (written by vwc-sync geofence match)
Step 2: container-sync pushes terminal_code to Postgres (COALESCE)
Step 3: Dispatch creation: geofences lookup WHERE UPPER(terminal_code) = UPPER($1)
Step 4: dispatch.pickup_geofences embedded (JSONB array of geofence objects)
Step 5: Driver app detects corridor crossing → queue_start_at fires
Step 6: GET /dispatches returns geofence columns (READ PATH)
        ALL dispatch SELECT queries must include: queue_start_at, queue_stop_at,
        queue_minutes, terminal_ingate_at, road_wait_minutes, terminal_wait_minutes,
        pickup_geofences, terminal_code.
        Missing any of these from the SELECT = geofence card shows blank.
Step 7: Frontend renders geofence card
        fetchDispatchLookup maps the fields → DispatchInfo interface →
        ContainerDetailsModal renders Geofence & Detention section.
```
If `terminal_code` is missing from the container, the dispatch has no geofence
and the driver app can't detect terminal arrival.

---

## Contract 7: Dispatch Geofence & Detention Columns

These columns track detention time at terminals. Written by the milestone handler, read by the frontend geofence card.

| Field | Type | Written by | Read by | API endpoints that MUST return it |
|-------|------|-----------|---------|----------------------------------|
| queue_start_at | timestamptz | milestone handler (terminal_area_arrived) | frontend geofence card | GET /dispatches, GET /api/driver/loads, GET /api/driver/history |
| queue_stop_at | timestamptz | milestone handler (terminal_outgate) | frontend geofence card | same |
| queue_minutes | integer | milestone handler (computed from start/stop) | frontend geofence card | same |
| terminal_ingate_at | timestamptz | milestone handler (at_terminal) | frontend geofence card | same |
| road_wait_minutes | integer | milestone handler (computed: ingate - start) | frontend geofence card | same |
| terminal_wait_minutes | integer | milestone handler (computed: stop - ingate) | frontend geofence card | same |
| pickup_geofences | jsonb | dispatch creation (embedded from geofences table) | frontend geofence card + driver app | same |
| terminal_code | text | dispatch creation (from container.terminal_code) | frontend geofence card | same |

### CRITICAL: read path vs write path
These columns were written correctly by the milestone handler but NOT returned by `GET /dispatches` for the entire development cycle. The geofence detention card showed blank until the SELECT was updated. Every new dispatch column must be added to ALL read endpoints, not just the write path.

### Frontend mapping
The `fetchDispatchLookup()` function manually maps API response fields into `DispatchInfo` objects. New fields must be added to:
1. `DispatchInfo` interface (ContainerTracking.tsx)
2. The mapper in `fetchDispatchLookup()` (ContainerTracking.tsx)
3. The geofence card JSX in `ContainerDetailsModal` (ContainerTracking.tsx)
