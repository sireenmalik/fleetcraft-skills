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

## 5. GPS Tracking

- Background location: always-on during active load
- 10-second fast polling on loads screen
- On-focus refresh when app returns to foreground
- GPS status indicator: green pulsing dot (active), gray (off)
- Enabled only during: `en_route_pickup`, `en_route_delivery`, `empty_en_route_return`

---

## 6. Known Crash Causes

1. **`advanceMilestone` after completion:** Navigate to `/driver/history` FIRST, then fire `advanceMilestone` in background. If milestone triggers state refresh before navigation, the load disappears from state and the app crashes.

2. **`EXPO_PUBLIC_FLEET_API_URL` missing:** This env var must be baked into the APK via `eas.json`. It cannot be changed at runtime. If the app can't reach the API, this is the first thing to check.

3. **Alert.prompt on Android:** Does not exist. Use a custom modal for text input (e.g., chassis number entry).

---

## 7. Build and Deploy

```bash
cd c:\expo\fleetcraft-driver
eas build --platform android --profile preview
```

Latest working APK: commit `9d5455c`
