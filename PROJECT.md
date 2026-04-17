---
name: fleetcraft-project
description: >
  Master context file for FleetCraft. Read this FIRST in every session before
  doing any work. It tells you what FleetCraft is, where things live, and
  which skill to load for the task at hand.
---

# FleetCraft — Project Context

**What:** Drayage TMS platform for container trucking at NWSA terminals (PCT Tacoma, T18 Seattle)
**Who:** Malik, founder and lead developer, Seattle WA
**Goal:** Make dispatch so guided that an untrained person can operate without a skilled dispatch manager

---

## RULE 0 — SPEC 0026 IS THE MILESTONE AUTHORITY (non-negotiable)

Spec 0026 (fleetcraft-specs/0026-fleetcraft-milestones-spec.md) is the SINGLE SOURCE OF TRUTH for the dispatch milestone lifecycle. It defines:

- The 17 steps (0-16), their sequence, their DB columns, their triggers
- The dashboard grid column order (28 columns, 16 milestone + 12 metadata)
- Step 3 three-mode chassis logic
- Auto-detect triggers (steps 2, 8, 9, 13)
- Auto-complete rule (step 16 fires on step 15)
- Customer notification trigger (step 9, NOT step 10)

### Enforcement:

1. **Before ANY dispatch/milestone/grid code change:** read Spec 0026 from GitHub MCP. Not from memory. Not from conversation context. From the file.

2. **If a Code prompt references milestones, grid columns, dispatch statuses, or timestamp columns:** the prompt MUST say "per Spec 0026" and the implementation MUST match the spec table exactly.

3. **If the implementation contradicts Spec 0026:** the implementation is wrong. Fix the code, not the spec. If the spec needs updating, update the spec FIRST, get Malik's approval, THEN change code.

4. **Previous specs are SUPERSEDED for milestone logic:** Specs 0009, 0010, 0013a, 0013b, 0014, 0021 defined earlier versions of the milestone sequence. They are historical context only. Spec 0026 is the canonical reference. If any older spec contradicts 0026, ignore the older spec.

5. **The grid column table (28 columns) is a CONTRACT:**
   - Column IDs cannot be renamed without updating Spec 0026 + driver app + backend
   - Columns cannot be added or removed without updating Spec 0026 first
   - Column order in the grid must match the spec
   - localStorage key must be bumped on any column change

6. **E2E tests enforce the spec.** The e2e test file for 0026 validates every DB column exists, every milestone handler writes the correct timestamp, auto-complete fires on step 15, and customer notification fires on step 9. If e2e fails, the deploy is blocked.

---

## IMMUTABLE RULE: Read skills before every action

Before writing ANY prompt, code change, architecture decision, or troubleshooting step:

1. Read ALL relevant skill files FIRST
2. Check if the problem is already documented in error patterns
3. Check if the solution violates any existing rule
4. Check if a skill needs updating after the fix

This is NOT optional. This is NOT "when you remember." This is EVERY TIME.

Skills to check by task type:
- Backend code change → `backend/SKILL.md` + `database/SKILL.md` + `column-registry/SKILL.md`
- Frontend code change → `frontend/SKILL.md`
- Driver app change → `driver-app/SKILL.md`
- Deployment → `deployment/SKILL.md`
- External API work → `integrations/SKILL.md`
- New feature → `feature-spec/SKILL.md` + ALL skills above
- Troubleshooting → ALL skills (the answer is probably already documented)

Every prompt to Claude Code must start with:
  "Read the skills first:" followed by the relevant skill file paths.

If a prompt doesn't reference skills, it's wrong. Rewrite it.

The 30 seconds spent reading skills saves 30 minutes of debugging.
The skills exist because we already made the mistake. Don't make it again.

---

## Post-troubleshooting checklist (MANDATORY)

After every successful fix, bug resolution, or feature deployment, STOP and check:

1. **Skills update needed?**
   - Did we learn a new rule? (e.g., "never call .toFixed() on API values")
   - Did we discover a new error pattern? (e.g., "FTU completed=true is not an archive trigger")
   - Did a write path or table ownership change?
   - Add to the relevant skill file: backend, frontend, database, integrations, deployment, driver-app, column-registry

2. **Spec update needed?**
   - Did the fix change behavior defined in an existing spec? (e.g., spec 001 archive flow)
   - Does a new spec need to be written? (e.g., orphaned dispatch auto-cancellation)
   - Update the spec in the `fleetcraft-api/specs/` directory

3. **Column registry update needed?**
   - Did we add, rename, or remove a column?
   - Did we change who writes or reads a column?
   - Did we add a new API endpoint that returns data?
   - Update `column-registry/SKILL.md`

4. **Contract validator update needed?**
   - Did we add a new cross-layer check?
   - Did we fix a bug the validator should have caught?
   - Update `scripts/validate-schema.sh` and run it

This is NOT optional. Every fix that doesn't update skills will be rediscovered as a bug in a future session. The 30 seconds spent updating skills saves 30 minutes of debugging later.

---

## Architecture (one sentence)

CQRS with Materialized Views — DigitalOcean PostgreSQL is the single source of truth, SQLite is a write buffer for containers, Supabase is FULLY DISCONNECTED.

### Data Architecture (April 2026)

- **ONLY database:** DigitalOcean PostgreSQL (`fleetcraft_db` on the droplet)
- **ONLY API:** Fleet API (`api.myfleetcraft.com` / `server.js`)
- **Supabase:** FULLY DISCONNECTED. Zero code references. No read cache, no auth, no direct queries. Purged in commit `d9840ab` (April 13, 2026) — 5,694 lines removed across 39 frontend files.
- **Frontend:** ALL data via Fleet API `fetch()` calls. No exceptions.
- **Driver app:** ALL data via Fleet API `fetch()` calls (through `lib/fleetApi.ts` helper).
- **Sync workers** (container-sync, vwc-sync, ais-collector-v2, eta-refresh): ALL writes to PostgreSQL via `pg` pool. No intermediate caches.

---

## When to Load Which Skill

| You're working on... | Load this skill |
|----------------------|-----------------|
| Building a NEW feature (before writing code) | `feature-spec/SKILL.md` |
| server.js, any PM2 worker, data write paths | `backend/SKILL.md` |
| Deploying code, editing files, git workflow | `deployment/SKILL.md` |
| SQL migrations, table changes, sync column mapping | `database/SKILL.md` |
| React components, dashboard, Tailwind, API calls | `frontend/SKILL.md` |
| Expo driver app, milestones, photos, GPS | `driver-app/SKILL.md` |
| FTU, AIS, HERE, Resend, external APIs | `integrations/SKILL.md` |

**If the task touches multiple domains, load multiple skills.**
**If the task is a new feature, ALWAYS load feature-spec first.**

---

## The Three Rules That Prevent 80% of Bugs

1. **Never reconstruct a file from memory.** Pull current version from GitHub before editing.
2. **Sync writers never touch user_status.** Only Fleet API endpoints write user intent columns.
3. **Supabase does not exist.** If you find code referencing it, that code is dead. Remove it.
4. **Direct-add containers bypass SQLite.** `data_source = 'direct'` goes straight to Postgres. No FTU, no webhooks, no sync workers.
