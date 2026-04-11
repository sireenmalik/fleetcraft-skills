---
name: fleetcraft-integrations
description: >
  FleetCraft external API integration rules. Use this skill when working with
  FTU (FindTEU) webhooks, AIS vessel tracking, HERE Technologies (routing,
  geofencing, maps), Resend email, Expo push notifications, or any external
  API. Prevents misconfiguration of webhook handlers, AIS filtering bugs,
  and API auth issues.
---

# FleetCraft Integrations — External API Rules

---

## 1. FTU (FindTEU) — Container Tracking

### Status: ACTIVE (account #14978)
- Event-based webhook API — NOT polling
- `ftu-tracker.js` is cache-refresh only, it does NOT call FTU API directly
- Webhooks arrive at Fleet API → write to SQLite → container-sync → Postgres

### Webhook handler rules (in server.js):
1. Validate the payload
2. Check `archived_containers` — if container exists there, do NOT re-insert
3. Check `container_exclusions` — if tombstone exists, skip and return (prevents archive resurrection)
4. Write to SQLite first (source of truth) — SQLite upsert has `WHERE user_status IS NULL OR user_status = 'active'` guard
5. When FTU sends `completed: true`, do NOT auto-archive (see rule below)

### FTU API endpoints used:
- `POST /container/tracking` — register new container
- `DELETE /container/subscription/{findteu_shipment_id}` — unregister on archive
- `GET /container/subscriptions` — list active subscriptions
- `GET /settings/info` — check quota (`used_this_month`)

### Auth header:
```javascript
headers: {
  'Content-Type': 'application/json',
  'X-Authorization-ApiKey': process.env.FTU_API_KEY
}
```

> **LESSON LEARNED:** The FTU auth header is `X-Authorization-ApiKey`, NOT `x-api-key`. Using the wrong header causes silent auth rejection — FTU returns 201 with "API key is not set" instead of an error code. This was misdiagnosed as a key expiry when it was actually a wrong header name. Fixed April 2026.

### T49 (Terminal 49): DEACTIVATED
Never reference `t49-container-tracker.js`, T49 API keys, or T49 endpoints. All T49 code is dead code.

### Direct-Add Containers — FTU Not Involved
Containers added via `POST /api/containers/quick-add` (`data_source = 'direct'`) have NO FTU interaction:
- No `POST /container/tracking` registration (no subscription created)
- No webhooks received (no `findteu_shipment_id`)
- No `DELETE /container/subscription` on archive (guard: `findteu_shipment_id IS NOT NULL`)
- No FTU re-registration on restore (guard: `data_source != 'direct'`)
- FTU quota/billing unaffected — zero API calls for direct-add containers

### FTU Health Monitoring
- `GET /api/health/ftu` — checks API key validity against FTU `/v1/settings/info`
- `GET /api/health` — includes `ftu_last_event` timestamp and `ftu_stale` boolean
- If `ftu_stale = true` (no FTU events in 7 days), the API key may be expired
- E2E test 8.3 fails loudly when the FTU key is rejected
- **LESSON LEARNED:** FTU API key expired silently in April 2026. No webhook data arrived for 6+ days. Containers showed "On Ship" with no vessel name, ETA, or terminal. The system had no alert for this condition. This monitoring prevents recurrence.

### FTU completed flag — DO NOT USE AS ARCHIVE TRIGGER
FTU sends `completed=true` when it has no more tracking updates. This can happen:
- After vessel arrival (container still on vessel)
- After discharge (container in yard, not yet dispatched)
- After gate out full (container picked up, mid-delivery)
- After empty return (actual business completion)

FTU `completed` does NOT reliably indicate business completion. It means "FTU stopped tracking."
The ONLY valid archive trigger is `dispatches.completed_at` (set by driver app step 25).
server.js webhook handler lines 684-716 currently auto-archive on `completed=true` — **THIS IS A BUG** and must be removed.

---

## 2. AISStream — Vessel Tracking

### Status: ACTIVE
- WebSocket connection to AISStream via `index.js` (ais-collector-v2)
- Writes to SQLite `vessel_registry` table (NOT directly to Postgres)
- `vwc-sync.js` reads SQLite and pushes to Postgres `vessels_cache` and `vessels_with_containers`

### Scan Schedule:
- **PNW coast** (45-50°N, 135-122°W): continuous 24/7
- **Pacific sweep** (30-55°N, full Pacific): 30 minutes every 8 hours (UTC 0, 8, 16)
- Config in `index.js`: `expandScheduleHours` and `expandDurationMinutes`

### Rules:
- **SOG threshold: >1 knot** — anything ≤1 is GPS drift, not movement
- **Moored/anchored vessels:** exclude from ALL ETA calculations
- **nav_status_label === 'Moored':** do NOT clear `alerted_*` flags, skip in alert loops
- **is_cargo filter:** only track vessel types 70-89 (cargo vessels)
- **"In Processing"** at terminal = container discharged, undergoing customs — count as `at_terminal_count`, NOT on vessel

---

## 3. HERE Technologies — Mapping & Routing

### Used for:
- Truck-legal routing (dispatch routes)
- Geocoding (address → lat/lng)
- Map tiles (driver app)
- Geofencing (terminal boundaries, detention tracking)
- Route matching

### Geofencing architecture: IMPLEMENTED — on-device polygon detection
- Geofences stored in `geofences` table keyed by `terminal_code`
- Embedded in dispatch `pickup_geofences` at dispatch creation time (terminal_code lookup)
- Detection happens ON-DEVICE in the driver app via `useGpsTracking.ts`
- Pure JavaScript ray casting point-in-polygon — no HERE SDK needed for detection
- HERE is used for map tiles and routing only, NOT for geofencing
- Driver app checks every 15 seconds via foreground `watchPositionAsync`
- Fire-once per dispatch (geofenceFired flag keyed by `${dispatchId}:${trigger_event}`)
- 100 trucks each check their own dispatch geofences locally — no server load

All geofences are polygons. Corridor type has been removed. Detection uses ray casting point-in-polygon. Polygon coordinates include a 25% buffer baked into the vertices for GPS accuracy.

### Terminal geofences configured:
- **Husky Terminal:** Polygon covering Lot F + adjacent roads (Maxwell Way to E 19th St, Thorne Rd to Port of Tacoma Rd)
- **PCT Tacoma:** Polygon covering gate booth area (25% expanded)
- **T18 Seattle:** Polygon covering gate booth area (3 geofences: 16th Ave SW, Klickitat Bridge, Gate Booth)

### Detention timing:
- Queue start: `queue_start_at` — set automatically by geofence fire (idempotent, WHERE NULL)
- Queue stop: `queue_stop_at` — outgate EIR photo timestamp
- Total queue: `queue_stop_at` minus `queue_start_at`
- Server-side: Fleet API milestone handler accepts `occurred_at`, `auto_triggered`, `triggered_by` from driver app

---

## 4. Resend — Email

### Used by: `dispatcher-orchestrator.js`
- Morning briefings, vessel arrival alerts, LFD warnings
- From address: `contact@myfleetcraft.com`

### Subscriber system:
- `alert_subscribers` table in Postgres
- `GET/POST/DELETE /api/subscribers` endpoints in `server.js`
- DST bug was fixed: `convertToUTC` now uses `Intl.DateTimeFormat` instead of hardcoded `-8` offset

---

## 5. DO Spaces — Photo Storage

| Setting | Value |
|---------|-------|
| Bucket | `fleetcraft-media` |
| Region | `sfo3` |
| Access | Public HTTPS URLs |
| Upload endpoint | `POST /api/driver/photos/upload` |

Photos upload from driver app → Fleet API → DO Spaces → URL stored in dispatch columns.

---

## 6. Expo Push Notifications

### Status: Planned, not yet deployed
- Will use Expo push notification system
- Trigger: new load assigned → notify driver
- Requires stable driver app (APK crash must be resolved first)

---

## 7. Anthropic API — Agentic Dispatch

### Status: Fully specced and coded, not yet deployed
- Model: `claude-sonnet-4-6`
- File: `dispatch-agent.js`
- Requires env vars not yet on droplet: `ANTHROPIC_API_KEY`, Twilio credentials, `DISPATCHER_PHONE`
- Three new DB tables needed: `dispatch_suggestions`, `dispatch_failures`, `agent_run_log`

---

## 8. Vessel Tracking Pipeline

### Architecture: AIS → SQLite → vwc-sync → Postgres

```
AISStream WebSocket → ais-collector-v2 → SQLite vessel_registry
                                                ↓
                                           vwc-sync.js (reads SQLite via vessel-db.js)
                                                ↓
                                    Postgres vessels_cache + vessels_with_containers
```

### vwc-sync.js — Single owner of vessels_with_containers
- Replaces both `vessel-sync.js` and `ftu-tracker.js` (both KILLED)
- MMSI resolution: `vesselDb.getByName()` first, then `vesselDb.getByImo()` fallback
- Container count: queries Postgres containers WHERE `user_status IS NULL OR user_status = 'active'`
- ETA: AIS-based for WA-bound vessels (SOG > 1), FTU pod_eta fallback for others
- Terminal geofence: moored vessels matched against terminal polygons → updates container terminal_name in SQLite
- Stale cleanup: vessels with zero active containers + no alert flags + updated > 7 days → deleted from cache
- **Never writes `alerted_*` columns** — dispatcher owns those

### MMSI Chicken-and-Egg Problem
FTU provides vessel IMO but not MMSI. AIS provides MMSI but only if the vessel is in the scan area. Resolution:
1. FTU webhook provides vessel_name + vessel_imo → stored in containers table
2. AIS Pacific sweep sees the vessel → stores MMSI + IMO in SQLite vessel_registry
3. vwc-sync reads vessel_registry by name/IMO → writes MMSI to vessels_with_containers
4. ais-collector reads vessels_with_containers MMSIs → tracks vessel globally (not just in bounding box)

### AIS-to-FTU Handover Protocol

AIS and FTU own different phases of the container lifecycle. The handover point is when the vessel moors at a terminal.

**AIS owns:** Vessel position tracking — ocean transit → approach → moored at terminal.
AIS job is DONE when: nav_status = Moored AND vessel is inside a terminal geofence polygon.

**FTU owns:** Container status — discharge → holds → available → gate out → empty return.
FTU job STARTS when: vessel moored OR FTU sends discharge webhook, whichever comes first.

**Handover rules:**

1. When AIS confirms vessel moored at terminal:
   - Container ui_status overrides to AT_PORT (even if FTU still says IN_TRANSIT)
   - terminal_name and terminal_code set from geofence match
   - ETA = 0 (vessel is already here)
   - Log: "AIS handover: {vessel} moored at {terminal}, {container} → AT_PORT"

2. When FTU sends discharge before vessel moored (FTU is faster):
   - Trust FTU — container is AT_PORT
   - AIS catches up later when vessel moors

3. When FTU is silent 48h+ after vessel moored:
   - Container stays AT_PORT (from AIS override)
   - Log warning: "FTU lag: {vessel} moored at {terminal} but no discharge data after 48h"

4. When AIS and FTU disagree on terminal:
   - FTU terminal wins for the container
   - AIS terminal used only when FTU has no terminal data

**Failure boundaries:**

| AIS | FTU | Result |
|-----|-----|--------|
| Moored at terminal | Discharged | Both agree. AT_PORT. Normal. |
| Moored at terminal | Still IN_TRANSIT | Override to AT_PORT. Wait for FTU. |
| No position | Discharged | Trust FTU. AT_PORT. |
| No position | No data | Stay at last known status. |
| Moored at terminal A | Discharged at terminal B | FTU terminal wins. |

### AIS destination vs FTU destination
- AIS `destination` = vessel's multi-port route (e.g., `CAVAN>JPTYO` = Vancouver > Japan)
- FTU `pod_name` = container's specific discharge port (e.g., `Tacoma`)
- These can be DIFFERENT — vessel visits multiple ports, container gets off at one
- Always prefer FTU `pod_name` for container-facing displays (vessel cards, dashboard)
- AIS `destination` useful only for vessel routing analysis, not container UX

### ETA Priority — FTU First, AIS Backup Only

AIS destination is the VESSEL's plan. FTU pod_eta is the CONTAINER's plan. They can be different.

Example: Vessel route Shanghai → Tacoma → Vancouver → Japan.
AIS destination: CAVAN>JPTYO (final port is Japan).
Container POD: Tacoma (gets off at Tacoma).
AIS ETA would calculate distance to Japan — WRONG for this container.
FTU ETA says April 15 (Tacoma arrival) — CORRECT.

**ETA priority (NEW — replaces old logic):**
1. FTU pod_eta exists → USE FTU. Always preferred. Source: 'FTU'
2. FTU has no ETA AND AIS destination last segment is WA terminal AND SOG > 1 → AIS ETA. Source: 'AIS'
3. Neither available → null. Show "Awaiting ETA" in dashboard.

**NEVER use AIS ETA when:**
- AIS destination is NOT a WA terminal (transit port, round trip, next voyage)
- Vessel is stationary (SOG < 1) — already arrived or anchored
- Vessel is moored — ETA = 0, vessel is here

**Round trip / next rotation scenario:**
Vessel finished a round, heading away from WA. Container stays at terminal waiting for next rotation.
FTU knows the next voyage ETA. AIS shows vessel heading to Asia. AIS ETA is meaningless.
Only FTU ETA is correct here.
