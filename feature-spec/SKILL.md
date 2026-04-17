---
name: fleetcraft-feature-spec
description: >
  Use this skill BEFORE writing any new feature, endpoint, worker, or UI component.
  This skill provides the spec template that must be filled out before code is
  generated. The template is designed for AI-reproducible code — it eliminates
  ambiguity so that different AI sessions produce functionally identical code from
  the same spec. When the user says "let's build X" or "add feature Y" or "create
  a new endpoint/worker/page", load this skill FIRST and fill out the spec with the
  user before writing any code. Never skip the spec — it is the contract.
---

# FleetCraft Feature Spec — AI-Reproducible Template

## Why This Exists

Classical specs (user stories, epics) are written for humans who fill in assumptions
from experience. AI fills in assumptions too — but they're different every time.
This template eliminates assumptions by defining the exact contract: inputs, outputs,
side effects, error cases, and file locations. Two different AI sessions given the
same completed spec should produce code that behaves identically.

## The Rule

**No code before spec.** When the user asks to build something new, fill out the
relevant sections of this template first. Discuss with the user. Get confirmation.
Then generate code.

---

## Template

Copy the sections below for each new feature. Not every section applies to every
feature — skip sections that don't apply, but never skip sections 1-3.

---

### 1. WHAT (Required — always fill this out)

```
Feature name:     [short name, becomes the commit message prefix]
One-line purpose: [what this does, in one sentence]
Triggered by:     [user action / cron schedule / webhook / API call]
Skill rules:      [which SKILL.md files apply — e.g., backend, database, integrations]
```

**Why it matters:** This prevents scope creep mid-implementation. If you can't
describe it in one sentence, it's too big — split it.

---

### 2. WHERE (Required — always fill this out)

```
Files to CREATE:
  - /path/to/new-file.js    — [purpose]

Files to MODIFY:
  - /path/to/existing.js    — [what changes and why]

Files to DELETE:
  - /path/to/dead-code.js   — [why it's dead]

Repo:             [fleetcraft-ais | fleetcraft-api | Fleetcraft | fleetcraft-driver | fleetcraft-alerts]
Deploy target:    [droplet PM2 | frontend scp | driver app EAS | none]
```

**Why it matters:** AI that doesn't know WHERE to put code will create new files
when it should modify existing ones, or modify the wrong file. Listing files
upfront also forces you to think about impact radius.

---

### 3. DATA CONTRACT (Required — always fill this out)

#### 3a. Database Changes

```sql
-- Migration file: migrations/NNN_feature_name.sql
-- Purpose: [why this table/column exists]
-- Depends on: [previous migration number]

ALTER TABLE table_name ADD COLUMN IF NOT EXISTS column_name TYPE DEFAULT value;
CREATE INDEX IF NOT EXISTS idx_name ON table (org_id, column_name);
```

If no DB changes: write "None — uses existing schema" and list which existing
tables/columns are read and written.

#### 3b. API Endpoints

For each endpoint, define ALL of these:

```
ENDPOINT:     POST /api/resource/action
AUTH:         JWT (driver) | org_id query param (dashboard) | none (webhook)
INPUT:
  Headers:    { "Authorization": "Bearer <jwt>" }
  Body:       {
                "container_number": "string, required",
                "org_id": "uuid, required",
                "reason": "string, optional, default 'manual'"
              }
  Query:      ?org_id=uuid (for GET requests)

OUTPUT (success):
  Status:     200
  Body:       {
                "success": true,
                "data": { "archived_at": "ISO timestamp" }
              }

OUTPUT (errors):
  400:        { "error": "container_number is required" }
  404:        { "error": "Container not found" }
  409:        { "error": "Container already archived" }
  500:        { "error": "Internal server error" }

SIDE EFFECTS:
  1. UPDATE containers SET user_status = 'archived', archived_at = NOW() WHERE container_number = $1
  2. DELETE FROM sqlite containers WHERE container_number = $1
  3. HTTP DELETE to FTU /container/subscription/{findteu_shipment_id}
  4. INSERT INTO container_events (event_type, source, ...) VALUES ('archive', 'dispatcher', ...)

IDEMPOTENCY:  Yes — calling twice with same input returns 409 on second call
```

**Why this level of detail:** This is where reproducibility lives. Two AI sessions
given this exact contract will produce code with the same HTTP interface, same
database writes, same error responses, same side effects. The implementation
(variable names, loop structure) may vary but the *behavior* is identical.

#### 3c. Worker / Cron Job

For background workers:

```
PROCESS NAME:   archive-worker
SCRIPT:         archive-worker.js
SCHEDULE:       Every 2 minutes (setInterval)
READS FROM:     Postgres — dispatches (completed_at), containers (ui_status)
WRITES TO:      SQLite — containers (archived_at)
TRIGGERS:       container-sync.js picks up archived_at on next 30s cycle
DOES NOT WRITE: user_status, Postgres containers (that's container-sync's job)
```

#### 3d. Frontend Component

For UI changes:

```
COMPONENT:      ArchiveButton
LOCATION:       src/app/components/container/ContainerTracking.tsx
RENDERS WHEN:   container row is visible
CALLS:          POST /api/containers/archive
ON SUCCESS:     Remove row from local state, show toast "Container archived"
ON ERROR:       Show error toast with server message
PROPS:          { containerNumber: string, orgId: string, findteuShipmentId: string }
```

---

### 4. BEHAVIOR RULES (Fill when there's business logic)

Define the exact rules, not prose descriptions:

```
RULE 1: Only archive containers where ui_status IN ('RETURNING', 'COMPLETED') — Spec 0029 (was EMPTY_RETURNED, OUT_FOR_DELIVERY)
RULE 2: Auto-archive triggers when completed_at < NOW() - INTERVAL '24 hours'
RULE 3: Manual archive skips the 24-hour wait — archives immediately
RULE 4: FTU unregister is called by manual archive handler, NOT by auto-archive
         (auto-archive delegates to syncArchivedContainers which handles it)
RULE 5: If container has an active dispatch (status NOT IN ('completed','cancelled')),
         reject archive with 409 "Container has active dispatch"
```

**Why rules, not prose:** "Archive completed containers after a day" is ambiguous.
Does "completed" mean `dispatches.status = 'completed'` or `ui_status = 'EMPTY_RETURNED'`?
Is "a day" 24 hours or end-of-business? Rules eliminate this.

---

### 5. ERROR HANDLING (Fill for any feature that can fail)

```
FAILURE:        FTU unregister returns 404
CAUSE:          Container already unregistered, or findteu_shipment_id is wrong
RESPONSE:       Log warning, continue with archive (do not block on FTU failure)
USER SEES:      Nothing — archive succeeds, FTU cleanup is best-effort

FAILURE:        SQLite DELETE fails
CAUSE:          File locked by another process, or container_number doesn't exist in SQLite
RESPONSE:       Log error, return 500 to user, do NOT mark as archived in Postgres
USER SEES:      "Failed to archive container" error toast

FAILURE:        Postgres UPDATE fails
CAUSE:          Connection error or constraint violation
RESPONSE:       Return 500, no SQLite delete, no FTU call (fail fast before side effects)
USER SEES:      "Failed to archive container" error toast
```

**Why this matters:** AI will invent error handling if you don't specify it.
Sometimes it'll silently swallow errors. Sometimes it'll crash the process.
Defining error behavior explicitly means the code handles failures the same
way every time it's generated.

---

### 6. TEST CASES (Fill for critical features)

```
TEST 1: Happy path
  INPUT:   POST /containers/archive { container_number: "MSCU7234521", org_id: "f8107db3..." }
  EXPECT:  200, container disappears from /containers/list, appears in archived_containers
  VERIFY:  psql: SELECT * FROM archived_containers WHERE container_number = 'MSCU7234521'

TEST 2: Already archived
  INPUT:   Same request as TEST 1, called again
  EXPECT:  409, { "error": "Container already archived" }

TEST 3: Active dispatch blocks archive
  SETUP:   Create dispatch for MSCU7234521 with status = 'en_route_pickup'
  INPUT:   POST /containers/archive { container_number: "MSCU7234521" }
  EXPECT:  409, { "error": "Container has active dispatch" }

TEST 4: Container not found
  INPUT:   POST /containers/archive { container_number: "DOESNOTEXIST" }
  EXPECT:  404

TEST 5: Missing required field
  INPUT:   POST /containers/archive { org_id: "f8107db3..." }  (no container_number)
  EXPECT:  400, { "error": "container_number is required" }
```

---

### 7. DEPENDS ON / DEPENDED ON BY (Fill for features that touch multiple files)

```
THIS FEATURE DEPENDS ON:
  - container-sync.js must be running (syncs archived_at to Postgres)
  - FTU API key must be valid (for unregister call)
  - containers table must have user_status column (migration 003)

OTHER FEATURES THAT DEPEND ON THIS:
  - container-sync.js syncArchivedContainers() reads archived_at
  - ftu-tracker.js checks archived_containers before re-inserting
  - Frontend ContainerTracking.tsx filters archived containers from list
```

---

## How to Use This Template

### For the user (Malik):
When you say "let's build X", I'll start by filling out this template and walking
through it with you. Push back on anything that's wrong or missing. The 5 minutes
we spend on the spec saves hours of debugging wrong assumptions.

### For the AI (Claude):
1. Load this skill when the user requests a new feature
2. Fill out sections 1-3 minimum, more if the feature is complex
3. Present to the user for review — do NOT start coding yet
4. Get explicit confirmation ("looks good" or corrections)
5. Then generate code that implements exactly what the spec says
6. After code is written, save the completed spec as a markdown file
   in the repo at `specs/NNN_feature_name.md` — it's the permanent
   record of what was built and why

### Spec archive:
Completed specs go in:
```
fleetcraft-api/
  specs/
    001_container_archive.md
    002_geofencing.md
    003_push_notifications.md
    ...
```
These are the "source code" for features. If the code needs to be rebuilt,
the spec tells AI exactly what to produce.
