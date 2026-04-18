---
name: fleetcraft-driver-app
description: >
  FleetCraft driver mobile app development rules. Use this skill whenever
  working on the fleetcraft-driver Expo/React Native app, modifying
  milestone flows, photo capture, GPS tracking, geofence detection, or
  Fleet API driver endpoints. Prevents known crash causes and auth issues.
---

# FleetCraft Driver App — Development Rules

---

## 1. Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Expo SDK 52+ / React Native |
| Navigation | Expo Router (file-based) |
| State | React Context + useReducer (DispatchContext) |
| Build | EAS Build (account: `sireenmalik`) |
| API | Fleet API at `api.myfleetcraft.com` |
| Photos | DO Spaces (`fleetcraft-media`, `sfo3`) |
| Maps | HERE Technologies |
| Local DB | expo-sqlite (GPS queue, offline milestone queue) |

### Local repo: `c:\expo\fleetcraft-driver`

---

## 2. Auth — JWT Key Name

> **Lesson learned:** The JWT signing key is `fleetcraft_jwt`. Using `fleetcraft_token` causes silent auth failures — the app appears logged in but every API call returns 401.

```javascript
// CORRECT
const token = jwt.sign(payload, process.env.JWT_SECRET); // JWT_SECRET = fleetcraft_jwt

// WRONG — causes silent failure
const token = jwt.sign(payload, 'fleetcraft_token');
```

### Test driver credentials:
- Phone: `2147632305`
- PIN: `1234`
- Driver ID: `ebd8dfe8-46f8-4f77-a8ef-2b24945bc58f`
- Org ID: `f8107db3-ecaa-48e1-968d-0e89c6dd8f62`

---

## 3. Dispatch Status Flow (19 statuses)

```
pending → assigned → en_route_pickup → chassis_info_required → at_terminal →
in_queue → loaded → gate_out → in_transit_parked → en_route_delivery →
at_delivery → delivered → empty_en_route_return → in_transit_parked_return →
at_return_terminal → chassis_returned → returned → completed
(cancelled can happen from any status)
```

### Milestone Flow — Spec 0032

**AUTHORITY: fleetcraft-specs/0032-driver-app.md (Spec 0032)**
Spec 0032 supersedes Spec 0026. Step numbers match dashboard row numbers (28 columns). Read Spec 0032 before any milestone code change.

| # | Step | DB field | Type | Photos |
|---|------|----------|------|--------|
| 8 | Dispatch created | dispatches.created_at | web UI | — |
| 9 | Accept/Reject | status: assigned/rejected | manual | — |
| 10 | Start (en_route_pickup) | en_route_pickup_at | manual | — |
| 11 | Truck assigned | dispatches.truck_id | web UI | — |
| 12 | Chassis info pre-term | chassis_info_at | manual+photo (conditional: outside terminal) | 5 |
| 13 | Geofence (terminal_area_arrived) | queue_start_at | auto GPS | — |
| 14 | Chassis info post-term | chassis_info_at | manual+photo (conditional: only if step 12 not done) | 5 |
| 15 | Ingate (at_terminal) | pickup_arrived_at | manual+photo | 1 |
| 16 | Loaded (container_loaded) | loaded_at | manual+photo | 2+ |
| 17 | Outgate (gate_out) | pickup_completed_at | manual+photo | 1 |
| — | Queue | computed: outgate − min(geofence, ingate) | computed | — |
| 18 | En route delivery (gate_out_detected) | en_route_delivery_at | auto GPS | — |
| 19 | En route delivery (fallback) | en_route_delivery_at | manual (if step 18 fails) | — |
| 20 | Delivery ETA (delivery_area_arrived) | delivery_eta_triggered_at | auto GPS | — |
| 21 | At delivery | delivery_arrived_at | manual+photo | 1 |
| 22 | POD (pod_captured) | delivery_completed_at | manual+photo | 1 |
| 23 | Delivered | actual_delivery_at | manual | — |
| 24 | En route return (en_route_return_detected) | en_route_return_at | auto GPS | — |
| 25 | At return terminal | return_arrived_at | manual+photo | 3+ |
| 26 | Chassis returned | return_completed_at | manual+photo | 1 |
| 27 | Complete | completed_at | auto (fires on step 26) | — |

Chassis conditional logic (steps 12 and 14):
- Step 12 (chassis_info_pre_term): chassis yard OUTSIDE terminal polygon. Driver inspects before crossing geofence.
- Step 14 (chassis_info_post_term): chassis INSIDE terminal polygon. Only fires if chassis_info_at IS NULL after geofence (step 13).
- Never both. Both write identical data: chassis_info_at, chassis_number, 5 photos, auto-checkout side effect.
- Dashboard CHSS INFO column (Row 12) shows result regardless of path.

Step 12/14 modes (applies to whichever chassis step fires):
- chassis_number NULL → full modal (type number, select pool, 5 photos)
- chassis_number NOT NULL + pool → streamlined (read-only, 5 photos required)
- chassis_number NOT NULL + tenant → auto-skip (no photos, auto-advance)

### Implementation notes (aligned April 2026)
- `buildMilestoneList` returns 14 items (steps 1-6, 8-15). Steps 0, 7, 16 excluded.
- Auto milestones (13, 18, 20, 24) are non-tappable — fire via BackgroundServices GPS detection
- Step 16 removed from UI — auto-fires server-side when step 15 completes
- `NEXT_MILESTONE` has no `chassis_returned` entry — prevents manual "Complete" tap
- PARKED milestones removed from `MILESTONE_ORDER` and `NEXT_MILESTONE` (`in_transit_parked`, `in_transit_parked_return`). Still in `DispatchStatus` type union for backward compat with old dispatches.
- `MILESTONE_ORDER` has 12 entries (manual tappable steps only). Auto steps fire independently.
- chassis_number NOT NULL + tenant → auto-skip (no photos, auto-advance)

### Rules:
- Each milestone tap captures `lat`, `lng`, `occurred_at` — no exceptions
- GPS + timestamp on EVERY milestone via one-shot location request
- Milestones write to `container_events` table via Fleet API
- Status updates write to `dispatches` table via Fleet API
- The Fleet API is the ONLY writer for dispatches and container_events

---

## 4. Photo Pipeline

Seven photo capture screens intercept the milestone flow BEFORE milestones fire.

| Milestone | Photo Slots | Dispatch Column | Type | Multi? |
|-----------|-------------|-----------------|------|--------|
| at_terminal (ingate) | gate booth + ticket | `ingate_photo_url` | TEXT (single) | No |
| chassis_info | tires, lights, frame, pin lock, overview | `chassis_photo_urls` | JSONB array | Yes |
| container_loaded (seal) | seal number, container face | `seal_photo_urls` | JSONB array | Yes |
| gate_out (outgate) | EIR slip | `outgate_photo_url` | TEXT (single) | No |
| at_delivery | delivery location | `delivery_arrival_photo_url` | TEXT (single) | No |
| pod_captured | delivered goods / dock | `pod_photo_url` | TEXT (single) | No |
| at_return_terminal | return receipt, final condition | `return_photo_urls` | JSONB array | Yes |

### Multi-photo support (all slots):
- Max 5 photos per slot (`MAX_PHOTOS_PER_SLOT = 5`)
- `+` button with dashed border appears when slot has < 5 photos
- `X` overlay button on each thumbnail to remove individual photos
- Counter shows `(N/5)` next to each slot label
- Alert shown when trying to add 6th photo: "Maximum 5 photos"

### Upload endpoint: `POST /api/driver/photos/upload`
- Uses `multer` for file processing (10MB limit in Express)
- Nginx allows 20MB request bodies (`client_max_body_size 20M`)
- Uploads to DO Spaces bucket `fleetcraft-media`
- Returns public HTTPS URL
- URL stored in dispatch record columns listed above

### Upload flow:
1. Driver captures photo — stored locally as URI
2. On milestone submit — photos upload sequentially to Fleet API
3. Fleet API → DO Spaces → returns URL
4. Milestone call includes `photo_urls` array
5. Server writes URLs to dispatch columns + container_events

---

## 5. GPS Tracking

### GPS Architecture — App Root Level (NOT screen level)

**CRITICAL:** `useGpsTracking` and `useEtaPolling` MUST be called at the app
root (`components/BackgroundServices.tsx`, mounted inside `DispatchProvider`
in `_layout.tsx`) — **never** inside a screen component like `[id].tsx`.

**Previous bug (April 2026):** GPS hook was inside `[id].tsx` (load detail
screen). When the driver navigated away, locked the phone, or switched
apps, React unmounted the screen → cleanup called `stopLocationUpdatesAsync`
→ the background GPS task died. The milestone tap still worked (one-shot
foreground `getCurrentPositionAsync`), but the continuous stream stopped.
Result: 3–4 hour gaps in `driver_positions` with no data — dispatcher sees
a stale truck LED for the whole shift.

**Current architecture:**
- `BackgroundServices` lives in `DispatchProvider` subtree at app root
- `gpsEnabled = loads.some(l => ACTIVE_GPS_STATUSES.includes(l.status))`
- GPS does NOT stop when the driver navigates between screens
- GPS ONLY stops when: the driver logs out (AuthProvider unmounts the tree)
  or `gpsEnabled` goes false (zero active loads)
- `stopLocationUpdatesAsync` is NEVER called on screen unmount
- `useEtaPolling` lives in the same `BackgroundServices` for the same reason
- ETA state is lifted into `DispatchContext` (`eta`, `etaForLoadId`, `setEta`)
  so the load detail screen can still display it

### ACTIVE_GPS_STATUSES (15 statuses — everything except `completed`/`cancelled`)

```
en_route_pickup, chassis_info_required, at_terminal, container_loaded,
loaded, gate_out, in_transit_parked, en_route_delivery, at_delivery,
pod_captured, delivered, empty_en_route_return, in_transit_parked_return,
at_return_terminal, chassis_returned
```

### Battery — non-negotiable operational requirement

Drivers MUST plug in phones during active loads. GPS battery drain is not
a valid concern for drayage operations. On first app launch after the
driver has an active load, `BackgroundServices` prompts the driver to
whitelist FleetCraft from Android Doze/battery-saver via
`Linking.openSettings()` — one tap to "Unrestricted" battery. Prompt is
dismissible; the "shown once" flag lives in SecureStore.

### Hook: `useGpsTracking` in `hooks/useGpsTracking.ts`

**Internal architecture:**
- Foreground `watchPositionAsync` runs every 15 seconds during active load
- Positions queued in local SQLite (`gps_queue` table) with `synced` flag
- Batch upload: flushes queue to `POST /api/driver/positions` every 30 seconds
- Background task: `TaskManager.defineTask('FLEETCRAFT_GPS')` for OS-level tracking
- Old synced records cleaned up after 24 hours
- `dispatch_id` tag published to SecureStore at `activeDispatchId` key so
  the module-level TaskManager callback can read it (React closures don't
  reach that far)

**GPS collection intervals:**
| Driver State | Interval | Rationale |
|-------------|----------|-----------|
| Active load (moving) | 15 seconds | High-resolution tracking, geofence detection |
| Active load (stopped) | 60 seconds | Reduced battery drain at terminals |
| Available (no load) | 5 minutes | Background awareness for dispatch planning |
| Off duty | No collection | Privacy protection |

**GPS status indicator:**
- Load detail screen: green pulsing dot + "GPS Active" / gray dot + "GPS Off"
- Home screen: small 7px dot next to driver name, green if any load in tracking status
- Tracking enabled during: `en_route_pickup`, `en_route_delivery`, `empty_en_route_return`

**GPS pauses during modals:**
GPS tracking is disabled when chassis modal is open (`showChassisModal`). This prevents re-renders from bouncing the Android keyboard. GPS resumes immediately when the modal closes. Geofence detection also pauses during modals — if the driver enters a geofence while typing a chassis number, it will fire on the next GPS tick after the modal closes.

### Known GPS bugs (fixed April 2026):
- accuracy/speed/heading were sent by app but dropped by server INSERT (fixed: server now inserts all 9 columns)
- Batch upload duplicated positions 5-6x due to missing concurrency lock on flush (fixed: `flushing` ref lock)
- Geofence detection had no accuracy gate — garbage GPS readings could miss or false-trigger polygons (fixed: skip if accuracy > 100m)

### GPS data fields captured:
| Field | Source | Notes |
|-------|--------|-------|
| lat, lng | coords.latitude/longitude | Always present |
| accuracy | coords.accuracy | Meters. Null on old builds. 5-15m normal, >100m = skip geofence |
| speed | coords.speed | m/s. Null when stationary |
| heading | coords.heading | Degrees. Null when stationary |
| dispatch_id | from active load | Links position to specific dispatch |

### Dispatch origin coordinates (`dispatches.origin_lat/lng`)

- Captured from the driver's phone GPS at the **"En Route to Pickup"** tap (milestone flow in `[id].tsx` → `advanceMilestone()`; migration 014 added the columns).
- Stored on the dispatch row. **Never changes after capture** — this is the anchor of the detention evidence chain: `origin → pickup_arrived_at` is the baseline trip time that HERE's predicted ETA at the same moment is compared against.
- **Each dispatch has its own independent origin.** If the same driver accepts two dispatches at different times/locations, each carries its own origin. Routes are always calculated per-dispatch; origins are never shared across dispatches even if driver/truck identical.
- Frozen by design — the whole detention model depends on "where did this specific trip start" being an immutable fact, not a moving target.

### GPS is ALWAYS ON during active dispatch (Spec 0013 v2)

`ACTIVE_GPS_STATUSES` includes all statuses from `en_route_pickup` through
`chassis_returned`. GPS only stops at `completed` or `cancelled`.
Driver must plug in phone. Battery optimization is not a concern for
drayage operations.

**Reason:** GPS was previously OFF during terminal dwell (`at_terminal`,
`container_loaded`, `gate_out`). This created a gap in breadcrumb data
during the most critical detention window. Fixed in Spec 0013.

### Defensive FG service watchdog (Spec 0021, 2026-04-15)

`BackgroundServices.tsx` runs a **60-second `setInterval`** that calls
`Location.hasStartedLocationUpdatesAsync(GPS_TASK_NAME)` while `gpsEnabled`
is true. If it returns `false`, the OS killed the task silently
(aggressive-battery OEM, Doze edge case, OOM kill) — we log a warning and
immediately call `startLocationUpdatesAsync` again with the same
`foregroundService` options.

Why `hasStartedLocationUpdatesAsync` and not `isTaskRegisteredAsync`: the
latter can report `true` for a zombie task whose process was killed. The
"has started" variant checks the active state, so a false reading means
the service needs restarting.

Watchdog deliberately skips the first tick — the hook's own initial
`startLocationUpdatesAsync` runs on mount; 60s later is enough time for it
to settle so the watchdog doesn't race the hook's first start.

### Battery optimization prompt (Spec 0021, 2026-04-15)

Three-state SecureStore flag at `fleetcraft_battery_opt_state`:
`'unknown'` / `'whitelisted'` / `'declined'`. Session-scoped (AuthContext
clears on logout) so every shift-start gets a fresh prompt.

On first active load (Android only, `gpsEnabled=true` for the first time):
- State `'unknown'` → non-dismissable `Alert.alert` with **two buttons**:
  - **"Open Settings"** → `Linking.sendIntent('android.settings.IGNORE_BATTERY_OPTIMIZATION_SETTINGS')`, falling back to `Linking.openSettings()` if the intent is rejected (Xiaomi/Oppo quirk). Optimistically writes `'whitelisted'` — we can't verify natively from Expo, so the flag is a best-effort signal.
  - **"I'll risk it"** → writes `'declined'`, sets `DispatchContext.batteryOptDeclined = true` which renders a persistent amber banner at the top of `app/driver/load/[id].tsx`: *"⚠ GPS may be interrupted — battery optimization is ON"*.
- State `'whitelisted'` or `'declined'` → skip the prompt, restore the banner if declined.

`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` is declared in `app.config.js` +
`app.json` Android permissions. It's required for `Linking.sendIntent`
to the ignore-optimization screen on some OEMs.

### Known OEM battery killers

Even with the Android `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` exemption,
these manufacturers run aggressive custom battery logic that can kill
foreground services regardless of standard Android rules:

| OEM | Behavior | Mitigation |
|---|---|---|
| **Xiaomi (MIUI)** | Kills FG services on swipe-dismiss from recents + "Battery saver" on by default | Driver must manually enable "Autostart" + set battery to "No restrictions" per app |
| **Oppo / Realme (ColorOS)** | "App freezing" kills background tasks after screen-off | Settings → Battery → Allow background activity |
| **Huawei (EMUI / HarmonyOS)** | "Protected apps" list — unlisted apps get killed aggressively | Settings → Apps → Launch → manual management → all three toggles on |
| **Samsung (One UI)** | "Deep sleeping apps" + "Put unused apps to sleep" kill FG services | Settings → Battery → Background usage limits → add FleetCraft to "Never sleeping apps" |

Our 60s watchdog catches these kills and re-registers the task, but there
will be a gap in the breadcrumb trail (up to 60s). If you see
`[GPS watchdog] FG service is DOWN` logs in production, the driver's OEM
is killing us — ask them to check the matching mitigation above.

### ETA polling covers pickup + delivery + return (Spec 0021, 2026-04-15)

`useEtaPolling` fires across three phases:

| Status | Destination |
|---|---|
| `en_route_pickup`, `chassis_info_required` | `terminals.lat/lng` (pickup) |
| `en_route_delivery` | `dispatches.delivery_lat/lng` |
| `empty_en_route_return` | `terminals.lat/lng` (same pickup terminal — no separate return concept today) |

Destination selection is server-side in `GET /api/dispatches/:id/eta`
based on `dispatch.status`. The client's `ETA_ACTIVE_STATUSES` must stay
in sync with `ETA_ARM_STATUSES` in `components/BackgroundServices.tsx` —
both define the same 4-status set.

**First-fire is immediate.** `useEffect` in `services/etaService.ts` calls
`fetchEta()` as soon as `dispatchStatus` enters an armed phase. Does NOT
wait for the interval timer. Variable cadence (10/5/3 min) only governs
subsequent polls.

**Detention baseline guard.** `eta_first_predicted_minutes` is the
detention math anchor and is only written on the PICKUP leg. Delivery and
return polls update the live `eta_*` columns but the baseline stays
pickup-scoped.

### Variable ETA polling (Spec 0013, not yet built)

Driver app will call `GET /api/dispatches/:id/eta` with current GPS.
Interval adjusts based on distance to terminal:
- `> 30 min` away: every 10 min
- `10–30 min`: every 5 min
- `< 10 min`: every 3 min

Uses HERE Routing v8 with `terminals.address` as destination (not lat/lng).

---

## 6. Geofence Detection — On-Device Polygon Detection (IMPLEMENTED)

> **CRITICAL:** Geofence detection uses `isInsidePolygon()` in `useGpsTracking.ts` — pure ray casting, no OS geofence engine, no HERE SDK. Do NOT use `startGeofencingAsync`.

> **STATUS: FULLY BUILT.** Polygon detection deployed in v1.2.0. All terminals use polygon type.

### How it works:
1. Driver accepts load → app loads `pickup_geofences` from dispatch payload into memory
2. `useGpsTracking` accepts geofences array + `onGeofenceEnter` callback
3. Foreground `watchPositionAsync` fires every 15 seconds
4. Each GPS position: `isInsidePolygon(lat, lng, vertices)` — ray casting point-in-polygon
5. If inside polygon → trigger fires (step 13)
6. Trigger fires ONCE per dispatch (`geofenceFired` Set keyed by `${dispatchId}:${trigger_event}`)
7. On fire: calls `POST /api/driver/loads/:id/milestone` with:
   - `milestone: 'terminal_area_arrived'`
   - `auto_triggered: true`
   - `triggered_by: 'geofence'`
   - `lat`, `lng`, `occurred_at` from the GPS reading
8. Driver sees alert: "You entered the terminal area. Queue timer started."
9. If chassis_info_at IS NULL when geofence fires, chassis_info_post_term (step 14) opens next.

### Detection window (statuses where geofence is armed):
- `en_route_pickup` → driver heading to terminal
- `chassis_info_required` → driver entered chassis details but still approaching

### Detection stops when:
- `geofenceFired` flag is true for this dispatch ID
- Dispatch status is `at_terminal` or any later status
- Dispatch is `completed` or `cancelled`
- `pickup_geofences` is null or empty array

### Terminal geofences (all polygon, all exact boundary):
| Terminal | Notes |
|----------|-------|
| Husky Terminal | Lot F rectangle (Maxwell Way / Thorne Rd / E 19th / Port of Tacoma Rd) |
| PCT Tacoma | Gate booth area polygon (25% expanded for GPS accuracy) |
| T18 Seattle | Gate booth area polygon (3 geofences: 16th Ave SW, Klickitat Bridge, Gate Booth) |
| FCTEST (Bellevue test) | 50m x 50m polygon at test address for bench testing |

All polygons use ray casting algorithm (`isInsidePolygon`). The 25% buffer is baked into the polygon vertices — no runtime buffer calculation needed.

### Critical dependency — pickup_geofences must be populated:
- `pickup_geofences` must be populated on the dispatch (not empty `[]`)
- Dispatch creation embeds geofences by looking up `terminal_code` from the `geofences` table
- If `pickup_geofences` is `[]`, detection silently does nothing — no error, no alert
- Direct-add containers must have `terminal_code` set (derived from `terminal_name` mapping in quick-add endpoint)

### Dead code removed:
Old OS geofencing code (`startGeofencingAsync`, `registerGeofences`, `pendingGeofenceTriggers`) was removed from `DispatchContext.tsx`. All detection is in `useGpsTracking.ts`.

### Server side (Fleet API):
- `terminal_area_arrived` milestone handler sets `dispatches.queue_start_at` WHERE `queue_start_at IS NULL`
- Idempotent — second fire does not overwrite the first timestamp
- Event logged to `container_events` with `auto_triggered: true`
- Server accepts `occurred_at`, `auto_triggered`, `triggered_by` from driver app

### Queue time calculation:
- If geofence fired: `queue_start_at` to `queue_stop_at`
- If geofence did NOT fire (GPS missed, accuracy gate, offline): fallback to `terminal_ingate_at` to `pickup_completed_at`
- Queue is NEVER a driver tap. It is always derived from timestamps.

---

## 6a. Delivery Geofence — Spec 0015 (deployed 2026-04-14)

On-device circle detection alongside the existing terminal polygon detection. Same `useGpsTracking` hook, same foreground watcher, same fire-once Set — two detection paths share one GPS tick.

### Detection primitives
- `isInsidePolygon()` — ray casting, for pickup terminal geofences
- `isInsideCircle()` — haversine distance, **exported** for unit testing — for delivery geofences

### Hook signature (breaking change, 2026-04-14)
`useGpsTracking` options replaced `geofences` with two typed fields:

```typescript
{
  pickupGeofences?: PolygonGeofence[];   // terminal polygons (Spec 0007)
  deliveryGeofences?: CircleGeofence[];  // delivery circles (Spec 0015)
  onGeofenceEnter?: (triggerEvent, lat, lng) => void;
}
```

One callback serves both — the `triggerEvent` string (`'terminal_area_arrived'` vs `'delivery_area_arrived'`) routes in the consumer.

### BackgroundServices wiring
- `DELIVERY_PHASES = ['en_route_delivery']` — the delivery geofence is armed ONLY during this status
- `deliveryLoad = loads.find(l => DELIVERY_PHASES.includes(l.status))` — separate from `pickupLoad`
- `handleGeofenceEnter` switches on `triggerEvent`:
  - `terminal_area_arrived` → pickupLoad, alert "Queue timer started"
  - `delivery_area_arrived` (step 20) → deliveryLoad, alert "Customer has been notified"

### Detection window
Armed while `deliveryLoad.status === 'en_route_delivery'`.
Disarmed when `delivery_arrived_at` is set (either by this geofence or manual `at_delivery` tap).
The hook's fire-once `geofenceFired` Set keyed by `${dispatchId}:delivery_area_arrived` provides re-fire protection.

### What the geofence does (and does NOT do)
- **Does:** sets `dispatches.delivery_arrived_at` (idempotent, NULL-guard); fires 'arriving' customer notification (email + SMS when configured); logs `container_events` row.
- **Does NOT:** advance dispatch status. Driver still manually taps `at_delivery` with photo. Same pattern as terminal geofence (geofence captures WHEN, driver captures WHAT).

### No customer → no geofence fire that matters
If the dispatch has `customer_id = NULL`, the backend skips creating the geofence row at dispatch creation. `pickup_geofences` / `delivery_geofences` arrive empty to the driver app. The hook iterates zero times. Zero false alarms. This is by design (Spec 0015 Rule 7).

### Circle radius (default 2km, customer-configurable)
- `customers.delivery_radius_m` default 2000, range 100-10000
- Copied to `dispatches.delivery_radius_m` at dispatch creation (frozen per Rule 3)
- Embedded in `dispatches.delivery_geofences` JSONB: `{ type: 'circle', lat, lng, radius_m, trigger_event: 'delivery_area_arrived' }`
- Driver app reads the JSONB directly — does NOT call `GET /api/geofences/terminal/:code` (which is polygon-only)

### Server wiring (Spec 0015 Phase 1 deployed)
- `POST /api/driver/loads/:id/milestone` accepts `delivery_area_arrived`, writes `delivery_arrived_at` WHERE NULL, fires 'arriving' notification fire-and-forget
- `en_route_delivery` milestone fires 'en_route' notification with `eta_minutes` from `dispatches.eta_predicted_minutes`
- `delivered` milestone fires 'delivered' notification

See backend/SKILL.md + database/SKILL.md for the server side.

---

## 6b. Auto-Detect Engine — Spec 0021 Phase 2 (2026-04-15)

One GPS watcher, four detection paths, one staged-event pipeline, one 8-second undo toast. Every auto-triggered milestone (entry or exit) lands on the same rails.

### The four detection paths

| Trigger event | Step | Kind | Guard | Armed during |
|---|---|---|---|---|
| `terminal_area_arrived` | 13 | polygon entry | 2-tick dwell (~30s) | pickup phases |
| `delivery_area_arrived` | 20 | circle entry | 2-tick dwell (~30s) | `en_route_delivery` |
| `gate_out_detected` | 18 | distance exit | 0.5 mi past polygon bounding circle for 4 ticks (~60s) | `gate_out` / `loaded` / `container_loaded` / `in_transit_parked` |
| `en_route_return_detected` | 24 | distance exit | 0.5 mi from `delivery_lat/lng` for 4 ticks | `delivered` / `pod_captured` / `at_delivery` |

**Rule — distance, not speed, for gate-out.** Terminal yard speeds vary (trucks crawl at 5 mph, stop at intersections). 0.5 mi past the polygon's bounding circle for 60s is definitive. Same logic for return-leg.

**Entry dwell blocks GPS bounce.** A single-tick reading that happens to fall inside a geofence no longer fires. Two consecutive inside-ticks are required — exactly one watcher interval of latency.

### The 8-second undo toast (`components/AutoDetectToast.tsx`)

- The milestone API call happens in `onConfirm`, **not** when the toast appears. Nothing is sent to the server during the 8s window.
- Tap body → confirm immediately (shortcut past the 8s wait).
- Tap "Undo" → `dismissAutoDetect(firedKey)` clears the fire-once + dwell counters so the detector re-arms.
- Timer expires → auto-confirm.
- Single knob: `AUTO_DETECT_TIMEOUT_MS = 8000` in `constants/index.ts`. All four auto-triggers share it.
- Max 1 active toast + 1-deep queue (second detection while one is pending gets staged with last-wins).

### Hook API (`useGpsTracking`)

```ts
useGpsTracking(enabled, {
  dispatchId,
  pickupGeofences,      // polygons (Spec 0007)
  deliveryGeofences,    // circles (Spec 0015)
  gateOutDetection,     // { dispatchId, terminalVertices } (Spec 0021)
  returnDetection,      // { dispatchId, deliveryLat, deliveryLng } (Spec 0021)
  onAutoDetect,         // (event) => void — staged, consumer owns undo
}) => { flush, dismissAutoDetect, getTripStats, resetTripStats };
```

The callback is `onAutoDetect`, **not** the old `onGeofenceEnter`. Old signature was `(trigger, lat, lng) => void`; new is `(event: { type, lat, lng, occurredAt, firedKey }) => void`. `firedKey` must be passed back to `dismissAutoDetect` on Undo.

### BackgroundServices wiring

Detection configs are derived from `loads` by matching status to arm-status lists (e.g. `GATE_OUT_ARM_STATUSES = ['gate_out','loaded','container_loaded','in_transit_parked']`). These mirror the server-side WHERE-IN guards in `routes/driver.js` — client and server agree on when each detection should fire.

`handleAutoDetect(event)` stages a `{ ...event, loadId }` into `pending` state. Toast renders. On confirm, `recordMilestone(loadId, type, coords, undefined, { auto_triggered: true, triggered_by: type.endsWith('_detected') ? 'distance_sustained' : 'geofence' })` fires.

The `<AutoDetectToast>` is rendered as the **last child** of `<DispatchProvider>` in `app/_layout.tsx` so it draws on top of every screen (RN z-index without a portal library is fragile; render order wins).

---

## 6c. Trip Stats — Spec 0021 Phase 4 (2026-04-15)

Approximate counters run on-device during the trip; authoritative recompute runs server-side at completion.

### Local counters (every GPS tick)

Inside the foreground watcher in `useGpsTracking`:

```ts
if (speed > 2) tripMovingTicks++; else tripStandingTicks++;
tripDistanceMeters += haversineMeters(prev, current);
```

Counters **run before the accuracy gate** — a garbage GPS tick still accrues trip time. Only the detection logic is accuracy-gated. Counters reset when `options.dispatchId` changes (new load).

### Exposed via `getTripStats()` → `{ movingPct, standingMinutes, totalMinutes, distanceMiles }`

`BackgroundServices` polls on a **30-second interval** (not every tick) and pushes to `DispatchContext.tripStats` + `tripStatsForLoadId`. Any screen reads directly; `TripStatsBar` renders null until `totalMinutes > 0`.

### Watcher runs whenever GPS is enabled

Not just when a detection config is active — so counters accrue continuously, even during `at_terminal` / `loaded` statuses when no detector is armed. Detection logic inside the callback is guarded; the watcher isn't.

### Authoritative recompute at completion

When `advanceMilestone` advances to `completed`, `DispatchContext` fires `POST /api/dispatches/:id/compute-trip-stats` fire-and-forget. The server reads every `driver_positions` row between `en_route_pickup_at` and `completed_at`, computes the same four metrics from authoritative data, and writes `trip_*` columns. The completed-banner summary reads server values first and falls back to the last live counters if the response hasn't propagated.

---

## 6d. v2 9-milestone flow — Spec 0021 (2026-04-15)

The driver-facing flow collapses from 15 tappable steps to 9 (4 auto + 5 photo, 1 skippable), while the DB CHECK constraint keeps all 19 statuses for backward compat (Rule 8).

| # | Milestone | Kind | Skip |
|---|---|---|---|
| 1 | Start pickup | auto | — |
| 2 | At terminal (ingate) | photo | — |
| 3 | Chassis inspection | photo | **YES** when `chassis_owner='tenant'` |
| 4 | Gate out | photo | — |
| 5 | En route delivery | auto | — |
| 6 | Arriving at delivery | auto | — |
| 7 | Delivered (POD) | photo | — |
| 8 | En route return | auto | — |
| 9 | Return complete | photo | — |

Eliminated taps (merged into auto-detect via GPS speed): `container_loaded` (merged into gate_out), `in_transit_parked` / `in_transit_parked_return` (replaced by moving/standing counters), `pod_captured` (merged into delivered), `at_return_terminal` / `chassis_returned` (merged into return complete).

**Chassis skip UX.** When `load.chassis_owner === 'tenant'` and the next milestone would be `chassis_info`, `[id].tsx` renders a blue "Owner chassis" skip card, the advance button becomes **Continue**, and `advanceMilestone` fires with a skip note — no chassis modal, no photo capture screen. Default `'pool'` keeps the full inspection flow.

**Components mounted in `[id].tsx` when `isActive`:**
- `EtaHero` — big glanceable ETA (active / arriving / arrived states)
- `DispatchRouteMap` — placeholder rendering until we pick a map dep (react-native-maps vs HERE in WebView); prop shape matches the full spec so the swap is internal
- `TripStatsBar` — moving% / time / distance, null until counters start
- `MilestoneList` — replaces old MilestoneTimeline. `buildMilestoneList(dispatch)` exported

**Completed banner** (status = 'completed') shows a 3-cell summary grid (Total time / Distance / Moving %) sourced from server `trip_*` columns with live-counter fallback.

---

## 7. Offline Mode

Terminal areas and rural delivery locations often have poor cellular coverage. The app must function offline with transparent sync on reconnect.

### Offline-capable (queued locally):
- GPS position collection (SQLite `gps_queue` table)
- Milestone advancement (queued with timestamp and GPS, synced on reconnect)
- Photo capture (stored as local URIs, uploaded on reconnect)
- Geofence detection (fires locally, milestone call queued for sync)

### Requires connectivity:
- Accepting new load assignments
- Turn-by-turn navigation
- Viewing updated container status from FTU
- Photo uploads to DO Spaces

---

## 8. Known Crash Causes

1. **`advanceMilestone` after completion:** Navigate to `/driver/history` FIRST, then fire `advanceMilestone` in background. If milestone triggers state refresh before navigation, the load disappears from state and the app crashes.

2. **`EXPO_PUBLIC_FLEET_API_URL` missing:** This env var must be baked into the APK via `eas.json`. It cannot be changed at runtime. If the app can't reach the API, check this first.

3. **`Alert.prompt` on Android:** Does not exist. Use a custom modal for text input (e.g., chassis number entry).

4. **Photo upload 413 error:** Nginx must have `client_max_body_size 20M`. Without this, uploads over 1MB are rejected by nginx before reaching Express. Express/multer limit is 10MB.

5. **Background GPS null lat/lng:** The background task can lose driver context. Milestone one-shot GPS always works. Background continuous tracking may send null positions — check `driver_positions` table for nulls.

6. **GPS position duplicates:** The batch flush had no concurrency lock. If flush took longer than 30 seconds (slow network), the next interval fired and uploaded the same batch again. Fixed with a `flushing` ref lock in `useGpsTracking.ts`.

7. **Chassis modal keyboard bounce (Android):** GPS tracking and context polling cause component re-renders every 10-15 seconds. On Android, re-renders while a Modal with TextInput is open cause the keyboard to scroll up and down. Fixed by pausing GPS when any modal is open: `gpsEnabled = isTrackingStatus && !showChassisModal`. If keyboard bounce persists, extract the modal into a React.memo component to isolate it from parent re-renders.

8. **GPS stops when driver leaves load screen:** The #1 production GPS bug (April 2026). If `useGpsTracking` is called inside a screen component (`[id].tsx`), React cleanup calls `stopLocationUpdatesAsync` on unmount. GPS dies silently — milestones still work (one-shot `getCurrentPositionAsync`) but continuous tracking stops. Fix: GPS MUST live at app root level (`components/BackgroundServices.tsx`), never inside a screen. This was the root cause of 3–4 hour gaps in `driver_positions` traced during the Spec 0013 v2 rollout. Detect with: `SELECT now() - max(recorded_at) FROM driver_positions WHERE driver_id = $1` — if > 5 min with an active dispatch, the phone is silent.

9. **ETA polling stops when driver leaves load screen:** Same root cause as #8. `useEtaPolling` was inside `[id].tsx`. Screen unmount killed the timer. Fix: ETA polling also lives at app root (`BackgroundServices`), ETA state lifted into `DispatchContext`.

10. **"Complete" button fires after auto-complete (Spec 0026).** If the app still has a manual "Complete" button AND the backend auto-chains step 16 on step 15, double-firing can cause a 409 or state confusion. Fix: remove the Complete button entirely. `NEXT_MILESTONE` has no `chassis_returned` entry. Step 16 is server-side only. The app detects `status='completed'` via poll/refresh and shows the completion summary card.

---

## 9. Build and Deploy

```bash
cd c:\expo\fleetcraft-driver
eas build --platform android --profile preview
```

- EAS account: `sireenmalik`
- `EXPO_PUBLIC_FLEET_API_URL` baked in via `eas.json`
- APK download link provided after build completes (5-10 min)
- Install APK on test phone, login with test driver credentials

### Local build alternative (faster, no EAS quota):
```cmd
cd C:\expo\fleetcraft-driver\android
set JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-17.0.18.8-hotspot
gradlew.bat assembleRelease
```
APK at: `android\app\build\outputs\apk\release\app-release.apk`

---

## 10. Fleet API Driver Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/auth/driver-login` | PIN-based auth, returns JWT |
| GET | `/api/driver/me` | Driver profile |
| PUT | `/api/driver/me` | Update profile, push token |
| GET | `/api/driver/loads` | Assigned loads (pending + active) |
| GET | `/api/driver/loads/:id` | Single load with full details |
| POST | `/api/driver/loads/:id/milestone` | Advance milestone with GPS + photos |
| POST | `/api/driver/photos/upload` | Upload photo to DO Spaces |
| POST | `/api/driver/positions` | Batch GPS position upload |
| GET | `/api/geofences/terminal/:code` | Get geofence coordinates for terminal |

All driver endpoints require JWT auth (`Authorization: Bearer <token>`).
All endpoints filter by `org_id` from the JWT payload.
