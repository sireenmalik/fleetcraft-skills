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

### Milestone flow the driver walks through:

```
en_route_pickup → chassis_info → at_terminal → in_queue →
gate_out → in_transit_parked → en_route_delivery → delivered →
empty_en_route_return → in_transit_parked_return →
at_return_terminal → chassis_returned → completed
```

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

### Hook: `useGpsTracking` in `hooks/useGpsTracking.ts`

**Architecture:**
- Foreground `watchPositionAsync` runs every 15 seconds during active load
- Positions queued in local SQLite (`gps_queue` table) with `synced` flag
- Batch upload: flushes queue to `POST /api/driver/positions` every 30 seconds
- Background task: `TaskManager.defineTask('FLEETCRAFT_GPS')` for OS-level tracking
- Old synced records cleaned up after 24 hours

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

---

## 6. Geofence Detection — On-Device Polygon Detection (IMPLEMENTED)

> **CRITICAL:** Geofence detection uses `isInsidePolygon()` in `useGpsTracking.ts` — pure ray casting, no OS geofence engine, no HERE SDK. Do NOT use `startGeofencingAsync`.

> **STATUS: FULLY BUILT.** Polygon detection deployed in v1.2.0. All terminals use polygon type.

### How it works:
1. Driver accepts load → app loads `pickup_geofences` from dispatch payload into memory
2. `useGpsTracking` accepts geofences array + `onGeofenceEnter` callback
3. Foreground `watchPositionAsync` fires every 15 seconds
4. Each GPS position: `isInsidePolygon(lat, lng, vertices)` — ray casting point-in-polygon
5. If inside polygon → trigger fires
6. Trigger fires ONCE per dispatch (`geofenceFired` Set keyed by `${dispatchId}:${trigger_event}`)
7. On fire: calls `POST /api/driver/loads/:id/milestone` with:
   - `milestone: 'terminal_area_arrived'`
   - `auto_triggered: true`
   - `triggered_by: 'geofence'`
   - `lat`, `lng`, `occurred_at` from the GPS reading
8. Driver sees alert: "You entered the terminal area. Queue timer started."

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
