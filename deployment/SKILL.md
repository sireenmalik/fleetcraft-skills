---
name: fleetcraft-deployment
description: >
  FleetCraft deployment and code editing rules. Use this skill whenever deploying
  code to the DigitalOcean droplet, editing files on the server, building the
  frontend, pushing to GitHub, or making any change to running production code.
  Also use when Claude Code or any AI tool is about to edit a file — this skill
  prevents the #1 cause of regressions: reconstructing files from memory instead
  of reading the actual current version. Read this BEFORE any deploy or edit.
---

# FleetCraft Deployment — Rules and Workflow

Every rule here exists because breaking it caused a production outage or regression.

---

## 1. The Golden Rule: Never Reconstruct From Memory

> **Lesson learned:** Claude reconstructed a file from memory, missed a guard clause that had been added 3 sessions ago, deployed it, and the protection was gone. This has happened multiple times.

### Before editing ANY file:
1. Pull the current version from GitHub via MCP or Claude Code
2. Read the ENTIRE file — not just the section you think needs changing
3. Only then propose changes

### Whole-file replacements only
Patch-based edits (inserting a few lines) have historically introduced bugs — wrong line numbers, duplicated imports, missing context. When editing a file, replace the entire file contents.

### If GitHub MCP returns 404
This is a known session-level OAuth token issue. Fall back to Claude Code for all file access. Do NOT attempt to reconstruct the file from conversation history or memory.

---

## 2. Backend Deploy Workflow

All backend code lives on the droplet at `178.128.64.97`.

### Standard deploy command

After every code change, deploy and test in one command:

```bash
ssh root@178.128.64.97 'bash /opt/fleetcraft-ais/scripts/deploy-and-test.sh'
```

This runs:
1. Git pull both repos (fleetcraft-ais + fleetcraft-api)
2. Restart PM2 workers (fleet-api, container-sync, vwc-sync)
3. Contract validator (110+ checks — column sync, terminal_code chain, PM2 processes)
4. E2E test suite (42+ tests — CRUD, dispatch, archive, regressions, API endpoints)

Uses `ALPHA1234` test container. Cleans up after itself.
Exit code 0 = safe deploy. Exit code 1 = something broke.

Git post-commit hook (`.githooks/post-commit`) reminds you to run this after every commit.

### Frontend-only deploy (no backend changes):

```bash
cd c:\Users\siree\OneDrive\FleetCraft\FleetCraft Git\Fleetcraft
npm run build
scp -r dist/* root@178.128.64.97:/var/www/fleetcraft-frontend/dist/
```

### Manual deploy (if you need to deploy a single worker):

```bash
ssh root@178.128.64.97
cd /opt/fleetcraft-ais    # or /opt/fleetcraft-api
git fetch origin
git reset --hard origin/main
pm2 restart <process-name>
```

### Verify after deploy:

```bash
pm2 logs <process-name> --lines 20 --nostream
```

### NEVER do this:
- `nano` or `vim` edit files directly on the server — changes will be lost on next deploy
- `npm install` on the server without checking package.json first
- `pm2 delete` and `pm2 start` when `pm2 restart` suffices
- Deploy without checking `pm2 logs` afterward

---

## 3. Frontend Deploy Workflow

Frontend is React/TypeScript/Tailwind, built locally and SCP'd to the droplet.

### Build and deploy:

```bash
# ON YOUR LOCAL MACHINE (not the droplet):
cd c:\Users\siree\OneDrive\FleetCraft\FleetCraft Git\Fleetcraft
npm run build
scp -r dist/* root@178.128.64.97:/var/www/fleetcraft-frontend/dist/
```

### Key points:
- `scp` commands run FROM the local machine, not from the droplet
- The frontend is served by nginx from `/var/www/fleetcraft-frontend/dist/`
- No PM2 process for the frontend — it's static files served by nginx
- After deploy, hard-refresh the browser (Ctrl+Shift+R) to bust cache

---

## 4. Driver App Deploy Workflow

Expo/React Native app, built via EAS Build.

### Local repo path:
```
c:\expo\fleetcraft-driver
```

### Build APK:
```bash
cd c:\expo\fleetcraft-driver
eas build --platform android --profile preview
```

### Key points:
- EAS account: `sireenmalik`
- `EXPO_PUBLIC_FLEET_API_URL` must be baked into the APK via `eas.json` — it cannot be changed at runtime
- Latest working APK: commit `9d5455c`
- Test driver: phone `2147632305`, PIN `1234`, driver ID `ebd8dfe8-46f8-4f77-a8ef-2b24945bc58f`

---

## 5. GitHub Repos

All repos under `sireenmalik`, all on `main` branch:

| Repo | What | Local Path |
|------|------|------------|
| `fleetcraft-ais` | AIS collector, FTU tracker, container-sync, archive-worker, vessel-sync | `c:\Users\siree\OneDrive\FleetCraft\fleetcraft-ais` |
| `fleetcraft-api` | Fleet API (server.js) | `c:\Users\siree\OneDrive\FleetCraft\fleetcraft-api` |
| `Fleetcraft` | React frontend | `c:\Users\siree\OneDrive\FleetCraft\FleetCraft Git\Fleetcraft` |
| `fleetcraft-driver` | Expo driver app | `c:\expo\fleetcraft-driver` |
| `fleetcraft-alerts` | Dispatcher/alerts | `c:\Users\siree\OneDrive\FleetCraft\fleetcraft-alerts` |

### Workflow:
1. Make changes locally (via Claude Code in VS Code)
2. `git add . && git commit -m "description" && git push origin main`
3. SSH to droplet and pull (see section 2)

---

## 6. Debugging Workflow

### Standard diagnostic sequence:
1. `pm2 logs <process> --lines 50 --nostream` — check for errors
2. `sqlite3 /opt/fleetcraft-ais/container-registry.db "<query>"` — check SQLite state
3. `psql -h localhost -U fleetcraft -d fleetcraft_db -c "<query>"` — check Postgres state
4. Compare SQLite vs Postgres — if they differ, the bug is in container-sync.js

### Frontend debugging:
- Chrome DevTools Network tab filtered to `api.myfleetcraft.com`
- Clear network log before each test action to isolate requests
- If no request appears after a UI action: stale build (old code serving) or CORS issue

### JavaScript async testing in browser:
```javascript
// Store results in window._varName due to async timing
fetch('https://api.myfleetcraft.com/api/containers/list?org_id=f8107db3-ecaa-48e1-968d-0e89c6dd8f62')
  .then(r => r.json())
  .then(d => window._result = d);
// Then read in a follow-up console call:
window._result
```

---

## 7. Environment Variables

### On the droplet (`/opt/fleetcraft-ais/.env` and `/opt/fleetcraft-api/.env`):
- `FTU_API_KEY` — FindTEU API key (account #14978)
- `RESEND_API_KEY` — Email sending
- `HERE_API_KEY` — Maps/routing/geofencing
- `AISSTREAM_API_KEY` — AIS vessel data
- Database credentials for Postgres

### NOT on the droplet yet (needed for agentic dispatch):
- `ANTHROPIC_API_KEY`
- Twilio credentials
- `DISPATCHER_PHONE`

### Dead env vars (remove if found):
- `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` — Supabase is disconnected
- `T49_API_KEY`, `TERMINAL49_API_KEY` — T49 is deactivated

---

## 8. Infrastructure

| Resource | Details |
|----------|---------|
| Droplet | `178.128.64.97` (DigitalOcean) |
| Database | PostgreSQL 16, `fleetcraft_db`, localhost:5432 |
| SQLite | `/opt/fleetcraft-ais/container-registry.db` |
| Frontend | `myfleetcraft.com` → nginx → `/var/www/fleetcraft-frontend/dist/` |
| API | `api.myfleetcraft.com` → nginx → port 3001 |
| Photo storage | DO Spaces, bucket `fleetcraft-media`, region `sfo3` |
| SSL | Let's Encrypt, auto-renewed |
| Process manager | PM2 |

---

## 9. Reproducibility Discipline — Config as Code

> **Goal:** If the droplet dies or code files are deleted, everything needed to rebuild the system is in Git. No tribal knowledge, no manual steps that only exist in someone's head.

### 9.1 Everything in Git — No Exceptions

These files MUST exist in the repo and stay current:

| File | Repo | Purpose |
|------|------|---------|
| `ecosystem.config.js` | fleetcraft-ais | PM2 process definitions — names, scripts, env vars, cron |
| `nginx.conf` | fleetcraft-api | Full nginx config for api.myfleetcraft.com and myfleetcraft.com |
| `.env.example` | every repo | Every env var with descriptions, no real values |
| `package.json` + `package-lock.json` | every repo | Exact dependency versions |

### Rule: If you change a PM2 process, update ecosystem.config.js
When adding, renaming, or removing a worker:
```javascript
// ecosystem.config.js — this IS the PM2 truth, not `pm2 start` commands
module.exports = {
  apps: [
    { name: 'fleet-api', script: 'server.js', cwd: '/opt/fleetcraft-api' },
    { name: 'ais-collector-v2', script: 'index.js', cwd: '/opt/fleetcraft-ais' },
    { name: 'container-sync', script: 'container-sync.js', cwd: '/opt/fleetcraft-ais' },
    { name: 'vwc-sync', script: 'vwc-sync.js', cwd: '/opt/fleetcraft-ais' },
    { name: 'dispatcher', script: 'dispatcher-orchestrator.js', cwd: '/opt/fleetcraft-alerts/Dispatchers-Live' },
    // KILLED: vessel-sync (replaced by vwc-sync), ftu-tracker (replaced by vwc-sync), archive-worker (merged into container-sync)
  ]
};
```

### Rule: If you change nginx, commit the config
After any nginx change:
```bash
cp /etc/nginx/sites-available/fleetcraft /opt/fleetcraft-api/nginx.conf
cd /opt/fleetcraft-api && git add nginx.conf && git commit -m "update nginx config" && git push
```

### Rule: .env.example stays current
When adding a new env var:
1. Add it to the actual `.env` on the droplet
2. Add a placeholder line to `.env.example` in the repo with a comment explaining what it's for
3. Commit `.env.example`

```bash
# .env.example — commit this, never commit .env
FTU_API_KEY=           # FindTEU container tracking API key (account #14978)
RESEND_API_KEY=        # Resend email service
HERE_API_KEY=          # HERE Technologies maps/routing/geofencing
AISSTREAM_API_KEY=     # AISStream vessel AIS data
PG_USER=fleetcraft
PG_PASSWORD=           # Postgres password
PG_DATABASE=fleetcraft_db
PG_HOST=localhost
PG_PORT=5432
JWT_SECRET=            # Driver app JWT signing key (must be 'fleetcraft_jwt')
```

### 9.2 Schema as Code — Numbered Migrations

> **Rule:** Never run ad-hoc ALTER TABLE in psql. Every schema change is a numbered migration file in Git.

```
fleetcraft-api/
  migrations/
    001_initial_schema.sql
    002_add_container_events.sql
    003_add_user_status.sql
    004_add_geofences.sql
    ...
```

Each migration file:
```sql
-- migrations/003_add_user_status.sql
-- Purpose: Lane separation — sync writers cannot overwrite user intent
-- Date: 2026-04-01
-- Depends on: 002

ALTER TABLE containers ADD COLUMN IF NOT EXISTS user_status TEXT DEFAULT 'active';
CREATE INDEX IF NOT EXISTS idx_containers_user_status ON containers (org_id, user_status);

-- Guard: sync writers must check this before upserting
COMMENT ON COLUMN containers.user_status IS 'Only Fleet API writes this. Sync workers must exclude from SET clause.';
```

### Applying migrations:
```bash
psql -h localhost -U fleetcraft -d fleetcraft_db -f migrations/003_add_user_status.sql
```

### The discipline:
- Migrations are append-only — never edit an existing migration file
- Each migration has a comment block: purpose, date, dependencies
- Migration files are the ONLY way to change the schema
- If you need to undo, write a new migration that reverses it

### Retroactive migrations — when production drifts from Git

> **Rule:** If you discover a column, table, index, or constraint on the live DB that is NOT represented in `migrations/`, write a retroactive migration the same day. Never leave production schema undocumented.

**Lesson learned:** In April 2026, three schema artifacts were found on the live droplet but missing from Git: `dispatches.delivery_address/lat/lng`, the `customers` table, and the dangling `dispatches.delivery_geofence_id` column added by migration 013 but never populated. This meant the DB could not be rebuilt from Git alone — a violation of reproducibility discipline (section 9). Fixed via migrations 018–020.

**Retroactive migration pattern:**

1. Dump the authoritative schema from the live droplet:
   ```bash
   ssh root@178.128.64.97 'sudo -u postgres pg_dump -s -t <table> fleetcraft_db'
   ```
2. Capture the `CREATE TABLE` / `ALTER TABLE` output verbatim into a new numbered migration.
3. Wrap EVERY statement in idempotency guards:
   - `CREATE TABLE IF NOT EXISTS`
   - `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`
   - `CREATE INDEX IF NOT EXISTS`
   - For constraints (no `IF NOT EXISTS` in Postgres), use a `DO $$ ... pg_constraint` lookup block.
4. Open the migration comment with `Purpose: Retroactive capture of <X>. Schema captured from pg_dump on <date>.` so future readers know it documents existing state rather than introducing new state.
5. Apply on the droplet — should produce only `NOTICE: ... already exists, skipping` lines and `COMMENT` acknowledgments. Zero data movement is the expected signal.

**Drift detection (future work):** A weekly cron that runs `pg_dump -s fleetcraft_db | diff <canonical-dump-in-git>` would flag drift automatically. Until that exists, audit schema quarterly by diffing `pg_dump -s` against a fresh `psql -f migrations/*.sql` on a throwaway DB.

### Sudo-postgres ownership gotcha

> **Lesson learned:** Migration 021 (Spec 0015) created `delivery_notifications` via `sudo -u postgres psql`. The table ended up owned by `postgres` and the `fleetcraft` app role got "permission denied" on INSERT. The error surfaced as silent notification failures. Fix: add `ALTER TABLE <t> OWNER TO fleetcraft` inside a `DO $$ ... END$$` idempotency block at the end of the migration.

**Rule:** Any new table defined by a migration must include an OWNER transfer if the migration is applied via `sudo -u postgres`. Prefer running migrations via the fleetcraft role directly when possible:

```bash
# Preferred — migration is owned by the app role from creation:
PGPASSWORD="$PG_PASSWORD" psql -h localhost -U fleetcraft -d fleetcraft_db -f migrations/NNN.sql

# Fallback (if app role lacks CREATE privilege on the schema) — include owner transfer:
sudo -u postgres psql -d fleetcraft_db -f migrations/NNN.sql
# migration must end with:
# DO $$ BEGIN ALTER TABLE <new_table> OWNER TO fleetcraft; END$$;
```

### package.json drift — `npm install` will prune unlisted deps

> **Lesson learned:** During Spec 0015 Phase 1 deploy, `npm install` on the droplet pruned `@aws-sdk/client-s3` and `multer` because they were `require()`d by `server.js` but not listed in `package.json`. fleet-api crashed with MODULE_NOT_FOUND. Both deps had been installed manually at some point in the past and nobody committed the package.json change.

**Rule:** Before any `npm install` on a live server, audit that every `require('X')` in the codebase corresponds to a line in `package.json` dependencies. A quick one-liner:

```bash
cd /opt/<service>
node -e "
const fs = require('fs');
const deps = new Set(Object.keys(require('./package.json').dependencies || {}));
const src = fs.readFileSync('server.js', 'utf8');
const required = [...src.matchAll(/require\\(['\"]([^./'\"][^'\"]*)['\"]/g)].map(m => m[1]);
const missing = required.filter(r => !deps.has(r.split('/').slice(0, r.startsWith('@') ? 2 : 1).join('/')));
if (missing.length) console.log('MISSING from package.json:', missing);
else console.log('OK — every require() has a matching dependency');
"
```

If the audit flags anything, run `npm install <name>` locally (which adds to package.json), commit, push, and only then pull on the droplet.

### 9.3 Self-Documenting Files

> **Rule:** Every .js file must have a header comment that explains what it does, what it reads, what it writes, and what depends on it.

```javascript
/**
 * container-sync.js — CQRS Query Side Sync
 *
 * READS FROM: SQLite container-registry.db (containers table)
 * WRITES TO:  Postgres fleetcraft_db (containers, archived_containers, vessels_with_containers)
 * CALLED BY:  PM2 (every 30 seconds)
 * DEPENDS ON: ftu-tracker.js populating SQLite, archive-worker.js setting archived_at
 *
 * This is the ONLY process that writes container data from SQLite to Postgres.
 * It must never write to user_status. It must include findteu_shipment_id in column list.
 */
```

This header means that if the file is deleted and AI has to recreate it, the header from NEIGHBORING files (which reference this one) plus the skill files give enough context to rebuild it correctly.

### 9.4 Disaster Recovery — Rebuild Sequence

If the droplet needs to be rebuilt from scratch, this is the order:

```
1. Provision new droplet (Ubuntu 24, same region)
2. Install: Node.js 22, PostgreSQL 16, nginx, PM2, better-sqlite3
3. Clone all repos to /opt/
4. Copy .env files (from secure backup — NOT in Git)
5. Run migrations in order: psql -f migrations/001... 002... 003...
6. pm2 start ecosystem.config.js
7. Configure nginx from committed nginx.conf
8. certbot for SSL
9. Update DNS A record to new IP
10. Verify: pm2 status, pm2 logs, curl api endpoint
```

This sequence should be testable — meaning you should be able to spin up a fresh droplet and follow these steps without any tribal knowledge.

---

## 10. Docker — Incremental Containerization

> **Rule:** When creating a NEW worker or service, also create a Dockerfile for it.

We're not Dockerizing the whole system at once. But every new component should be container-ready from day one.

```dockerfile
# Dockerfile for a new worker — template
FROM node:22-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
CMD ["node", "worker-name.js"]
```

And add it to `docker-compose.yml`:

```yaml
services:
  worker-name:
    build: ./workers/worker-name
    env_file: .env
    depends_on:
      - postgres
    restart: unless-stopped
```

### When Docker becomes mandatory:
- Before onboarding a second developer
- Before deploying to a second droplet
- Before launching to paying customers

Until then, PM2 + ecosystem.config.js is the process manager. But every new worker ships with a Dockerfile so the migration is incremental, not a cliff.

---

## 11. SQLite Database Files — NEVER Track in Git

> **Lesson learned:** `.db`, `.db-shm`, and `.db-wal` files were tracked in git. Every `git reset --hard origin/main` on the droplet overwrote the live database with the stale committed copy. The `user_status` column added via ALTER TABLE kept disappearing on every deploy because git replaced the file. Caused a full day of debugging during the user_status lane separation build.

### The Rule
SQLite database files are runtime data, not source code. They must NEVER be tracked in git.

### .gitignore (required in every repo that uses SQLite):
```
*.db
*.db-shm
*.db-wal
```

### If you find .db files tracked in a repo:
1. `git rm --cached *.db *.db-shm *.db-wal`
2. Add the patterns to `.gitignore`
3. Commit and push
4. On the droplet: backup the live .db BEFORE pulling, then restore after `git reset --hard`

### Schema changes to SQLite:
- Update `container-registry-schema.sql` (the schema definition file) so future DB recreations include the column
- Run `ALTER TABLE` on the live .db file on the droplet
- Both steps are required — schema file for reproducibility, ALTER TABLE for the live DB

### WAL maintenance:
- All SQLite connections must set `PRAGMA wal_autocheckpoint = 1000` on open
- container-sync.js runs `PRAGMA wal_checkpoint(PASSIVE)` at end of each cycle
- If WAL grows unbounded: `sqlite3 <db> "PRAGMA wal_checkpoint(TRUNCATE);"`
- Never use `git reset --hard` to fix WAL issues — that was the old workaround that caused the column loss bug

---

## 12. Deploy Versioning

Every multi-repo release is tagged in git with a date-based version across ALL 6 repos simultaneously.

### Release tagging convention (April 2026+)

**Format:** `v{YYYY.MM.DD}-{feature-name}`

**Example:** `v2026.04.13-dispatch-tracking`

**Rules:**
- Tag ALL 6 repos at the same version: `fleetcraft-api`, `fleetcraft-ais`, `Fleetcraft` (frontend), `fleetcraft-driver`, `fleetcraft-skills`, `fleetcraft-specs`.
- Tag **after** a feature ships and passes e2e — not before. The tag records a known-good state.
- `feature-name` is kebab-case, describes the headline change of that release (the thing you'd put in a PR title).
- `YYYY.MM.DD` is the release date, not the feature-start date.
- One release per day maximum. If multiple features ship the same day, use the last/umbrella one's name — the tag points to the combined end-of-day state.

**Command pattern (run in each of the 6 repos):**
```bash
git tag v2026.04.13-dispatch-tracking
git push origin v2026.04.13-dispatch-tracking
```

**Rollback:** `git reset --hard v2026.04.13-dispatch-tracking` in each repo, then restart services:
```bash
# api
ssh root@178.128.64.97 "cd /opt/fleetcraft-api && git fetch --tags && git reset --hard v2026.04.13-dispatch-tracking && pm2 restart fleet-api"
# ais (if workers need restart)
ssh root@178.128.64.97 "cd /opt/fleetcraft-ais && git fetch --tags && git reset --hard v2026.04.13-dispatch-tracking && pm2 restart all"
# frontend — rebuild from tagged state, then scp dist
```

### Prior convention (legacy, kept for historical tags)

Earlier tags used `v{major}.{minor}-{feature-slug}` — e.g., `v1.0-baseline`, `v1.2-push-notifications`, `v1.3-fleet-tracking`, `v2.0-agentic-dispatch`. Those existing tags remain valid for rollback; new releases use the date-based format above. Don't delete old tags — they're rollback anchors.

### Version manifest
Location: `fleetcraft-api/versions/`

Each deploy creates a JSON manifest with:
- Commit hashes for every repo (or `"unchanged"`)
- Migration file (if any, or `"none"`)
- Rollback SQL
- Rollback steps (ordered list)
- Dependency chain (`depends_on` — previous version)

### Health endpoint
`GET /api/health` returns current version, uptime, and timestamp.
Use this to verify what's deployed without SSH.

### Pre-deploy checklist
1. Tag current state with a version manifest (`versions/vX.Y-slug.json`)
2. Backup frontend: `cp -r dist dist-backup`
3. Snapshot relevant DB state (e.g., `SELECT count(*), data_source FROM containers GROUP BY data_source`)
4. Deploy (git pull + pm2 restart + scp frontend)
5. Verify via `GET /api/health` — must return new version string
6. Run E2E tests (`deploy-and-test.sh`)

### Rollback
1. Read the current version manifest in `versions/`
2. Follow `rollback_steps` in order
3. Run `rollback_sql` if data cleanup needed
4. Verify via health endpoint or test suite

---

## 13. Database Backups

Three backup layers protect Postgres data:

| Layer | Frequency | Destination | Retention | Trigger |
|-------|-----------|-------------|-----------|---------|
| Daily | 2am PT (9am UTC) | DO Spaces backups/postgres/ | 30 days | Cron |
| Weekly | Saturday 9am | OneDrive FleetCraft/backups/postgres/ | Manual cleanup | Windows Task Scheduler |
| Pre-deploy | Before every release | DO Spaces backups/postgres/ | 30 days | deploy-and-test.sh Step 0 |

Backup script: /opt/fleetcraft-ais/scripts/pg-backup.sh
Log: /var/log/fleetcraft-backup.log
Cron: 0 9 * * * (verify with crontab -l)

Restore: gunzip the .sql.gz file, pipe to psql. Always dump current state before restoring.
