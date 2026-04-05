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
