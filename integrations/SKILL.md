---
name: fleetcraft-integrations
description: >
  FleetCraft external API integration rules. Use this skill when working with
  FTU (FindTEU) webhooks, AIS vessel tracking, HERE Technologies (routing,
  geofencing, maps), SendGrid email, Twilio SMS, Expo push notifications, or
  any external API. Prevents misconfiguration of webhook handlers, AIS
  filtering bugs, and API auth issues.
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
- **FCTEST (Bellevue test):** 50m x 50m polygon at 6302 119th Pl SE, Bellevue. For bench testing geofence detection without driving to a real terminal.

### Adding a new terminal geofence:
1. INSERT into `geofences` table with `terminal_code`, polygon coordinates, `trigger_event = 'terminal_area_arrived'`
2. Add to `terminalCodeMap` in server.js quick-add endpoint (all name variants)
3. Add to frontend `TERMINAL_OPTIONS` dropdown in ContainerTracking.tsx
4. If missing from any step, dispatch creation won't resolve terminal_code and `pickup_geofences` will be empty `[]`

### Detention timing:
- Queue start: `queue_start_at` — set automatically by geofence fire (idempotent, WHERE NULL)
- Queue stop: `queue_stop_at` — outgate EIR photo timestamp
- Total queue: `queue_stop_at` minus `queue_start_at`
- Server-side: Fleet API milestone handler accepts `occurred_at`, `auto_triggered`, `triggered_by` from driver app

### Fleet Tracking architecture (Spec 0013):
- GPS source is the **driver app phone**, NOT HERE Tracking service
- This avoids HERE's "Asset Management" license exclusion from the Base Plan
- HERE is used ONLY for: map tiles (rendering), routing (truck-legal), geocoding (address→lat/lng)
- All three are Base Plan eligible at standard transaction rates

### HERE Routing v8 — truck profile:
```javascript
// Standard truck profile for all FleetCraft route calculations
const routeParams = {
  transportMode: 'truck',
  'truck[grossWeight]': 36000,   // kg — loaded container truck
  'truck[height]': 410,          // cm
  'truck[width]': 255,           // cm
  'truck[length]': 1650,         // cm
  return: 'polyline,summary'
};
// Base URL: https://router.hereapi.com/v8/routes
// Auth: apiKey query param from HERE_API_KEY env var
```

### ETA refresh rules:
- Worker runs every 5 minutes (`eta-refresh.js`, PM2)
- Max 10 concurrent HERE API calls (semaphore)
- Skip if `eta_updated_at` < 4 minutes ago (debounce)
- Skip if driver hasn't moved since last calc (same `last_position_at`)
- On 429 (rate limit): log warning, skip dispatch, retry next cycle — do NOT crash

### Route polyline rules:
- Calculated ONCE at dispatch creation via `POST /api/dispatch/route`
- Stored in `dispatches.route_polyline` (encoded flexible polyline format)
- ETA refresh does NOT update the stored polyline
- Dashboard draws from current driver position to destination, not from origin

### Delivery geofences (per-dispatch, not permanent):
- Created at dispatch time alongside route calculation
- 200m radius around delivery lat/lng
- `dispatch_id` set, `terminal_code` null
- Soft-deleted (`is_active=false`) when dispatch completes
- Terminal geofences (shared, permanent) have `terminal_code` set and `dispatch_id` null

### Status-to-color mapping (dashboard truck markers, Spec 0013 §3d):
- GREEN: `assigned`, `en_route_pickup`, `chassis_info_required`
- AMBER: `at_terminal`, `in_queue`, `loaded`, `gate_out`
- BLUE: `en_route_delivery`, `at_delivery`, `delivered`
- GRAY: `in_transit_parked`, `in_transit_parked_return`
- PURPLE: `empty_en_route_return`, `at_return_terminal`, `chassis_returned`, `returned`
- NO MARKER: `pending`, `completed`, `cancelled`

### HERE Geocoding API (one-time address → coordinates)

- **Endpoint:** `https://geocode.search.hereapi.com/v1/geocode?q=<free-text>&apiKey=<KEY>`
- **Input:** a single free-text address string (e.g. `"2325 Lincoln Ave, Tacoma, WA 98421"`)
- **Output:** `{ items: [{ position: { lat, lng }, address: { label } }, ...] }` — first item is the best match
- **Purpose:** **one-time** conversion of a human-typed address into coordinates that HERE Routing can consume. Routing v8 rejects text input; all routing destinations must be `lat,lng`.
- **Call sites in `server.js`** (via the `hereGeocode(address)` helper at the same name):
  1. `POST /api/terminals` — new terminal address → stored in `terminals.lat/lng`
  2. `PATCH /api/terminals/:id` — address change → re-geocoded and propagated to active dispatches (auto-added commit `37ad6df`)
  3. `POST /api/dispatches` setImmediate — customer `delivery_address` → stored in `dispatches.delivery_lat/lng` (only when no `customer_location_id` was provided, i.e. one-off typed address)
  4. `GET /api/dispatches/:id/eta` — null-coord fallback only (defensive)
  5. `POST /api/customers/:id/locations` setImmediate — new delivery address → `customer_locations.lat/lng` (Spec 0018)
  6. `PATCH /api/customers/:id/locations/:locId` setImmediate — address edit → re-geocodes the location. Does NOT propagate to existing dispatches (radius/coords are FROZEN per Spec 0018).
- **NEVER called at routing time.** Once an address is geocoded, every routing / ETA / distance / polyline call uses the stored coordinates directly. This caps geocode volume to "one call per terminal edit" rather than "one call per ETA poll."
- **Error handling:** best-effort. If HERE returns no match or throws, the address still saves — coordinates are left null and the caller logs a warning.

### HERE Routing for live ETA — DEPLOYED (Spec 0013 v2)
- **Endpoint:** `GET /api/dispatches/:id/eta?origin_lat=X&origin_lng=Y` (server-side proxy, no CORS)
- **Internals:** calls HERE Routing v8 with truck profile, `return=summary,polyline,typicalDuration`.
  Destination is `terminals.lat/lng` (primary) or `hereGeocode(terminals.address)` fallback —
  HERE Routing v8 requires coordinates, not free-text addresses.
- **Response:** `{ eta_minutes, base_minutes, congestion_minutes, distance_km, polyline, updated_at }`
- **Writes to `dispatches`:** `eta_predicted_minutes`, `eta_base_minutes`, `eta_traffic_minutes`,
  `eta_congestion_minutes`, `eta_predicted_at`, and **`eta_first_predicted_minutes` on the first
  call per dispatch** (detention baseline — compare against `actual_trip_minutes` for
  third-party congestion proof).
- **Driver app polling (via `useEtaPolling` in `BackgroundServices`):** variable interval keyed
  to the last-known ETA — `> 30 min`: every 10 min, `10–30 min`: every 5 min, `< 10 min`: every
  3 min. Only polls while a load is in `en_route_pickup` or `chassis_info_required`.
- **Congestion delay = `duration - baseDuration`** from the HERE response.
- **Cost:** ~43 calls per 4.5-hour trip, ~43,000 calls/month at 1,000 trips, $22–$43/month on
  HERE Base Plan.

### Spec 0017 live tracking does NOT call HERE

`lib/liveTracking.js` (customer portal live tracking) is pure computation. It reads the
`eta_predicted_minutes` that the `eta-refresh.js` worker already wrote to `dispatches` and
derives ETA windows, progress %, and banner state from that. **Zero new HERE calls per
portal API request.** If you're tempted to "just look up ETA on the fly" in a portal
handler, don't — that would 10–100× the HERE bill overnight. The 5-min worker is the
single writer of ETA data.

### Container tracking handover — Spec 0029

**AIS → FTU handover at DISCHARGED (rank 6).** AIS owns ocean statuses (IN_TRANSIT through AT_BERTH). FTU owns terminal statuses (DISCHARGED through AVAILABLE). AIS cannot tell us when a container left the vessel — only the carrier (via FTU) knows that. AT_BERTH does NOT mean DISCHARGED.

**FTU → FleetCraft handover at ASSIGNED (rank 9).** Once FTU confirms AVAILABLE, FleetCraft creates the dispatch.

**FTU webhook does NOT write ui_status during ocean phase.** AIS is authoritative for vessel position. FTU updates data fields (pod_eta, vessel_name) but vwc-sync owns ocean status writes (to SQLite, then container-sync bridges to Postgres).

Authority: fleetcraft-specs/0029-container-tracking-alignment.md

### Customer notification trigger — Spec 0026

Customer "arriving" notification fires on Spec 0026 Step 9 (`delivery_eta_triggered_at`), NOT Step 10 (`delivery_arrived_at`). Step 9 = geofence proximity. Step 10 = manual arrival confirmation. If you wire notifications to step 10, customers get notified AFTER the driver is already at the dock — defeats the purpose. Authority: fleetcraft-specs/0026-fleetcraft-milestones-spec.md

### Spec 0015 SendGrid "Share ETA" — `lib/notifications.js`

`sendShareEtaEmail(pool, { dispatch, recipientEmail, customerId, orgId, liveTracking, customMessage })`
is a customer-initiated one-shot forward of the current ETA snapshot. Reuses the SendGrid
singleton (same key as delivery notifications), logs to `delivery_notifications` with
`trigger='share_eta'`. Rate-limited at the handler layer (5 per dispatch per rolling hour)
— see `routes/portal.js POST /api/portal/share-eta`.

---

## 4. SendGrid — Email (replaced Resend on 2026-04-14)

> **Migration note:** Resend was swapped out after its API key silently expired and its v6 SDK's `{ data, error }` shape was being misread as success. See the skill git log + `fleetcraft-api` commit `0be47ed` for the full receipt. SendGrid's SDK throws on error — simpler error handling, no silent-failure footgun.

### Used by:
- `fleetcraft-api/lib/notifications.js` — Spec 0015 delivery notifications (scheduled / en_route / arriving / delivered)
- `fleetcraft-alerts/Dispatchers-Live/dispatcher-orchestrator.js` — vessel arrival alerts, daily briefings, LFD warnings

### Config:
- `SENDGRID_API_KEY` — env var, required. Missing → email sends log `status='skipped'` with clear reason.
- `SENDGRID_FROM_EMAIL` — defaults to `contact@myfleetcraft.com`. Dispatcher falls back to legacy `ALERT_FROM_EMAIL` for backward compat during the transition.
- Package: `@sendgrid/mail` (installed in both `fleetcraft-api` and `fleetcraft-alerts/Dispatchers-Live`).

### SDK pattern:
```javascript
const sgMail = require('@sendgrid/mail');
sgMail.setApiKey(process.env.SENDGRID_API_KEY);
const [response] = await sgMail.send({ to, from, subject, html });
// response.statusCode === 202 on accept
// response.headers['x-message-id'] is the provider id
// SDK THROWS on error — err.response.body has the JSON detail
```

### Legacy column compat
`alert_logs.resend_id` (dispatcher) and `delivery_notifications.provider_id` (fleet-api) both now hold SendGrid's `x-message-id`. The `resend_id` column name was NOT renamed — doing so would invalidate historical audit queries for zero semantic value. Comments in the INSERTs note the id source is SendGrid.

### Subscriber system (unchanged):
- `alert_subscribers` table in Postgres
- `GET/POST/DELETE /api/subscribers` endpoints in `server.js`
- DST bug fixed: `convertToUTC` uses `Intl.DateTimeFormat` instead of hardcoded `-8` offset.

### If you're reading this because emails aren't arriving:
1. `grep SENDGRID_API_KEY /opt/fleetcraft-api/.env /opt/fleetcraft-alerts/Dispatchers-Live/.env` — confirm it's set.
2. `curl -X POST "https://api.sendgrid.com/v3/mail/send" -H "Authorization: Bearer $KEY" ...` — probe the key directly.
3. Check SendGrid dashboard's "Activity" feed for delivery + bounce events — SendGrid's is much better than Resend's was.
4. `pm2 logs fleet-api --err --lines 50` — notifications.js logs failed sends with the full `err.response.body`.

---

## 4a. Twilio — SMS (deployed 2026-04-14)

> **Status:** Code deployed. Spec 0015 Phase 4 (account setup + env vars) is the manual gate. Until env vars are set, SMS sends log `status='skipped'`.

### Used by:
- `fleetcraft-api/lib/notifications.js` — Spec 0015 delivery notifications to customers via `customers.notification_phone`.

### Config:
- `TWILIO_ACCOUNT_SID` — starts with `AC...`, from Twilio console.
- `TWILIO_AUTH_TOKEN` — account-level token. Treat like a root password.
- `TWILIO_PHONE_NUMBER` — E.164 format (`+14253332328` etc.).

### Curious quirk — Twilio creds were already on the droplet
When the Resend→SendGrid swap ran, the alerts `.env` files (`/opt/fleetcraft-alerts/.env` and `/opt/fleetcraft-alerts/Dispatchers-Live/.env`) already had live Twilio credentials from a planned dispatcher-SMS feature that never shipped. The `fleetcraft-api/.env` does NOT have them — when Spec 0015 Phase 4 gets wired, either copy the alerts values over OR provision new ones (rotation is healthy anyway; anyone who saw the prior repo commit history saw these).

### SDK pattern:
```javascript
const twilio = require('twilio');
const client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);
const msg = await client.messages.create({
  from: process.env.TWILIO_PHONE_NUMBER,
  to: customer.notification_phone,   // MUST be E.164
  body: '...',
});
// msg.sid is the provider_id (stored in delivery_notifications.provider_id)
```

### Phone number validation
`PATCH /api/customers/:id` rejects non-E.164 phones with 400. Don't relax this — Twilio silently fails on malformed numbers and the cost is on you per attempt.

### Graceful degrade
Missing `TWILIO_ACCOUNT_SID` OR `TWILIO_AUTH_TOKEN` OR `TWILIO_PHONE_NUMBER` → `twilioClient` stays null, SMS sends log `status='skipped'` with reason `'Twilio not configured (SID/TOKEN/PHONE_NUMBER)'`. Email still flows independently via SendGrid.

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

### VWC cleanup rules (April 2026)

- **Cleanup MUST run every cycle, even when zero active vessels exist.** Without this, archiving the last active container on a vessel leaves its VWC row permanently orphaned (aggregation produces zero rows → any early-return skips the DELETE step → row never ages out). Fixed in commit `c35c741` — removed the `if (!vesselStats) return` guard before the cleanup step.
- **Cleanup runs BEFORE any early-return guard in the sync loop.** General rule: guards should only gate work that depends on having fresh input data. Cleanup queries the DB directly — it doesn't need aggregation output.
- **VWC only contains vessels with at least one active container.** Once every container on a vessel is `user_status != 'active'` (or `archived_at IS NOT NULL`), the VWC row is deleted on the next cycle. Dashboards reading VWC should not expect to see completed/archived vessels.
- **Stale + alert-free vessels are deleted immediately**, not after 7 days. The 7-day window only applies to vessels with alert flags still set (those need manual attention first before auto-cleanup).

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

---

## HERE API CORS rule (v1.3.5 lesson)

- **NEVER call HERE APIs from the browser.** `router.hereapi.com`, `geocode.search.hereapi.com`, and `fleet.ls.hereapi.com` all block browser-origin CORS requests when using apiKey auth.
- All HERE API calls MUST go through Fleet API server-side endpoints.
- Only HERE Map Tiles (loaded via HERE JS SDK `mapsjs-core.js` / `mapsjs-service.js`) work client-side. These are script-tag loaded, not fetch calls.
- Server-side proxy endpoint: `GET /api/dispatch/route-preview?origin=lat,lng&destination=lat,lng` — no DB writes, returns polyline + distance + duration.

## ETA display rules (v1.3.6 lesson)

- Two ETAs shown per dispatch, not one:
  - **ETA to terminal:** live during `en_route_pickup`. Shows "Arrived" when `at_terminal`. Shows "Completed" after `gate_out`.
  - **ETA to delivery:** shows "TBD" until dispatch status >= `gate_out`. Then live ETA from HERE Routing.
- ETA to delivery cannot be meaningfully calculated until the container is physically with the driver (after gate out). Before that, terminal dwell time is unknown.

## Route overlay rules (v1.3.6 lesson)

- For `en_route_pickup` phase, draw BOTH legs:
  - Bright dashed blue: truck current position to terminal (live leg)
  - Faded dotted blue: terminal to delivery address (next hop, stored polyline)
- For `en_route_delivery` phase, draw ONE leg: truck to delivery.
- For `empty_en_route_return` phase, draw ONE leg: truck to return terminal.
- Route destination must change based on dispatch status — not hardcoded to delivery.
