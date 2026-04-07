---
name: fleetcraft-driver-app
description: >
  FleetCraft driver mobile app development rules. Use this skill whenever
  working on the fleetcraft-driver Expo/React Native app, modifying
  milestone flows, photo capture, GPS tracking, or Fleet API driver
  endpoints. Prevents known crash causes and auth issues.
---

# FleetCraft Driver App — Development Rules

---

## 1. Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Expo SDK 52+ / React Native |
| Navigation | Expo Router (file-based) |
| State | React Context + useReducer |
| Build | EAS Build (account: `sireenmalik`) |
| API | Fleet API at `api.myfleetcraft.com` |
| Photos | DO Spaces (`fleetcraft-media`, `sfo3`) |
| Maps | HERE Technologies |

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

## 3. Milestone Flow (16 steps)

```
en_route_pickup → chassis_info → at_terminal → in_queue →
gate_out → in_transit_parked → en_route_delivery → delivered →
empty_en_route_return → in_transit_parked_return →
at_return_terminal → chassis_returned → completed
```

### Rules:
- Each milestone tap captures `lat`, `lng`, `occurred_at`
- GPS + timestamp on EVERY milestone — no exceptions
- Milestones write to `container_events` table via Fleet API
- Status updates write to `dispatches` table

---

## 4. Photo Pipeline

Seven photo capture screens intercept the milestone flow BEFORE milestones fire.

| Photo | Column | Type | When |
|-------|--------|------|------|
| Ingate | `ingate_photo_url` | TEXT | At terminal gate |
| Chassis | `chassis_photo_urls` | JSONB | After chassis pickup |
| Seal | `seal_photo_urls` | JSONB | Container seal verification |
| Outgate | `outgate_photo_url` | TEXT | Terminal gate out |
| Delivery arrival | `delivery_arrival_photo_url` | TEXT | Arriving at delivery |
| POD | `pod_photo_url` | TEXT | Proof of delivery |
| Return | `return_photo_urls` | JSONB | Empty return at terminal |

### Upload endpoint: `POST /api/driver/photos/upload`
- Uploads to DO Spaces bucket `fleetcraft-media`
- Returns public HTTPS URL
- URL stored in dispatch record

---

## 5. GPS Background Tracking

GPS tracking is confirmed working. 1,301 positions captured during testing.

### Sampling rates:
- Moving with active load: every 15 seconds
- Stopped with active load: every 60 seconds (50m distance threshold)

### Data flow:
```
Driver app (useGpsTracking hook) → local SQLite queue → batch upload every 30s
→ POST /api/driver/positions → driver_positions table
```

### Known issues:
- **dispatch_id is NULL** on all positions — server endpoint does not store load_id from request. Workaround: query by `driver_id + time range` overlapping `dispatch.created_at` to `dispatch.completed_at`. Fix needed: server must accept and INSERT dispatch_id/load_id.
- **Positions stop in background** on some devices — Android battery optimization may kill the background task. Foreground service notification is configured but OEM kill behavior varies.
- **Null lat/lng filter:** server filters positions where lat or lng is null/undefined/NaN before INSERT. Driver app should also guard before enqueuing.

### Implementation:
- Hook: `hooks/useGpsTracking.ts`
- API: `lib/fleetApi.ts` → `uploadPositions()`
- Type: `GpsPosition` uses `latitude`/`longitude` (Expo convention), server accepts both `lat/lng` and `latitude/longitude`
- Offline queue: local SQLite `gps_queue.db` — positions survive app crashes
- Background task: `expo-location` + `expo-task-manager`

---

## 5b. Test Environment (Bellevue)

- **Container:** ALPHA1234 on FLEET CRAFTERY at FLEETCRAFT TEST TERMINAL
- **Terminal:** FCTEST at 6302 119th Pl SE, Bellevue 98006 (47.5615, -122.1485)
- **Geofence:** FCTEST — Home Test Corridor (47.5462, -122.1806 to -122.1795), 50m buffer
- **ALPHA5678:** IN_TRANSIT anchor container keeping FLEET CRAFTERY in VWC
- **Archive exclusion:** `data_source = 'test'` skipped by both archive functions in container-sync
- **Test driver:** phone 2147632305, PIN 1234
- **E2E test:** uses ALPHA786 (not ALPHA1234) to avoid destroying permanent test data

---

## 5c. Geofence Detection — Real-Time On-Device Check

Geofence detection uses our own GPS stream, NOT the Android OS geofence engine.

### How it works:
- `useGpsTracking` accepts `geofences` array + `onGeofenceEnter` callback
- Foreground `watchPositionAsync` runs every 15 seconds
- Each position: haversine distance to corridor endpoints
- If within `buffer_meters * 2` of either endpoint → fires trigger
- Each trigger fires once per session (`geofenceFired` Set)
- Driver gets alert: "You entered the terminal area. Queue timer started."

### Detection latency: ~15 seconds (vs Android OS: 1-5 minutes)

- **WRONG:** `expo startGeofencingAsync` → Android OS batches checks → 1-5 min latency
- **RIGHT:** our GPS stream every 15s → haversine check → instant detection

### Corridor buffer sizing:
- **Production (Husky, PCT):** 25m buffer = 50m check radius. Terminals have clear entry points.
- **Test (FCTEST Bellevue):** 50m buffer = 100m check radius. Home testing needs wider zone.

---

## 6. Known Crash Causes

1. **`advanceMilestone` after completion:** Navigate to `/driver/history` FIRST, then fire `advanceMilestone` in background. If milestone triggers state refresh before navigation, the load disappears from state and the app crashes.

2. **`EXPO_PUBLIC_FLEET_API_URL` missing:** This env var must be baked into the APK via `eas.json`. It cannot be changed at runtime. If the app can't reach the API, this is the first thing to check.

3. **Alert.prompt on Android:** Does not exist. Use a custom modal for text input (e.g., chassis number entry).

---

## 6b. Spec 004: Background Photo Upload Queue (DEPLOYED)

Photos save locally and upload in the background. Milestones advance in <1 second.

### Flow:
```
capture → save to device → show local thumbnail → enqueue → advance immediately
Background: queue processes one photo at a time, compresses (1920px, JPEG 70%), uploads when signal available
```

### Key files:
- `app/services/upload-queue.ts`: SQLite queue + 10s processing timer + compress + upload + retry
- `app/driver/load/photo-capture.tsx`: `handleConfirm` enqueues locally, no server wait
- `server.js`: `POST /api/driver/photos` accepts late photos, appends to dispatch JSONB
- `app/_layout.tsx`: `initUploadQueue()` on app start resumes pending uploads
- `app/driver/load/[id].tsx`: amber badge "N photos uploading..." with 5s polling

### Rules:
- Milestone submit sends timestamp + GPS only, NO photo URLs
- Photos arrive separately via background queue
- One upload at a time, network-aware, max 5 retries with backoff
- Local SQLite queue survives app close/crash
- Compression: 5MB → ~500KB before upload

---

## 7. Build and Deploy

```bash
cd c:\expo\fleetcraft-driver
eas build --platform android --profile preview
```

Latest working APK: commit `9d5455c`
