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
