# TAMALE — Infrastructure & Operations Runbook

**Version:** 1.0  
**Last Updated:** June 14, 2026  
**Audience:** Any developer (or AI assistant) walking in cold to troubleshoot, modify, or hand off the TAMALE production system.

---

## Table of Contents

1. [What TAMALE Is](#1-what-tamale-is)
2. [Top-Level Architecture](#2-top-level-architecture)
3. [VM & Infrastructure](#3-vm--infrastructure)
4. [Container Inventory](#4-container-inventory)
5. [Network Topology & Ports](#5-network-topology--ports)
6. [Critical File Paths](#6-critical-file-paths)
7. [Secrets & Credentials](#7-secrets--credentials)
8. [The Chatwoot Fork (Main App)](#8-the-chatwoot-fork-main-app)
9. [The Dograh Fork (Voice Studio)](#9-the-dograh-fork-voice-studio)
10. [Database](#10-database)
11. [Backups](#11-backups)
12. [Deployment Workflows](#12-deployment-workflows)
13. [Syncing From Upstream (Security Patches)](#13-syncing-from-upstream-security-patches)
14. [Common Operations Cookbook](#14-common-operations-cookbook)
15. [Disaster Recovery](#15-disaster-recovery)
16. [Troubleshooting Decision Tree](#16-troubleshooting-decision-tree)
17. [Known Gotchas](#17-known-gotchas)
18. [Glossary](#18-glossary)

---

## 1. What TAMALE Is

TAMALE is a production AI-powered customer support SaaS platform, sold to web service agencies that manage multiple business clients.

**Core platform = two integrated open-source projects forked and customized:**

| Component | Upstream | What it does |
|-----------|----------|--------------|
| **Chatwoot fork** | github.com/chatwoot/chatwoot | Main app — conversations, channels (WhatsApp/Messenger/Instagram), billing, AI agent builder, multi-tenancy |
| **Dograh fork** | github.com/dograh-hq/dograh | Voice Studio — voice AI workflows, telephony, real-time call agents |

**Operator:** Guruji Enterprises (sole proprietorship, MSME-registered), India.  
**Primary domains:** `tamaleapp.com` (main app), `voice.tamaleapp.com` (Voice Studio).

---

## 2. Top-Level Architecture

```
                          ┌──────────────────┐
   User browser ─────────►│  Cloudflare /    │
                          │  DNS             │
                          └────────┬─────────┘
                                   │
                          ┌────────▼─────────┐
                          │  Nginx (host)    │
                          │  /etc/nginx/...  │
                          └─┬──────────────┬─┘
                            │              │
        ┌───────────────────┘              └─────────────────────────┐
        │                                                            │
        │  tamaleapp.com → :3000                voice.tamaleapp.com   │
        │                                       ├─ UI    → :3010     │
        │                                       ├─ API   → :8000     │
        │                                       ├─ MinIO → :9000     │
        │                                       └─ /sso  → :3000     │
        ▼                                                            ▼
┌──────────────────────────────────┐    ┌──────────────────────────────────┐
│  TAMALE STACK (Chatwoot fork)    │    │  DOGRAH STACK (Voice Studio)     │
│  /home/tamale/docker-compose.yml │    │  /home/dograh-compose.yaml       │
│                                  │    │                                  │
│  ├─ tamale-rails-1               │    │  ├─ home-api-1   (FastAPI)       │
│  ├─ tamale-sidekiq-1             │    │  ├─ home-ui-1    (Next.js)       │
│  └─ tamale-postgres-1            │    │  ├─ home-postgres-1 (pgvector)   │
│      (DB: tamale_production)     │    │  ├─ home-redis-1                 │
│                                  │    │  ├─ minio                        │
│                                  │    │  └─ coturn (TURN server)         │
└────────────────┬─────────────────┘    └────────────────┬─────────────────┘
                 │                                       │
                 └───────────┐                ┌──────────┘
                             ▼                ▼
                      Cross-stack glue:
                      - JWT SSO (Voice Studio iframe in TAMALE)
                      - DOGRAH_WEBHOOK_SECRET for usage push-back
                      - TAMALE_BASE_URL=http://172.17.0.1:3000 (quota check)
```

**Key design point:** Two independent Docker Compose stacks on one VM. They communicate over the docker bridge (172.17.0.1) and via JWT-signed HTTP for SSO. Either stack can be rebuilt without touching the other.

---

## 3. VM & Infrastructure

| Property | Value |
|----------|-------|
| **Provider** | Google Cloud Platform |
| **Zone** | asia-south2-a |
| **External IP** | 34.131.162.197 |
| **Default user** | g1208aarav |
| **OS** | Ubuntu 24 |
| **Docker** | 29.5.0 |
| **Total RAM** | 7.8 GB (consider 16GB upgrade if Next.js builds done on VM) |
| **Disk** | 49 GB (~26 GB used as of doc date) |
| **Access** | GCP Browser SSH only (no permanent SSH keys) |

**Recent GCP snapshots (rollback points):**
- `chatwoot-pre-phase-i-20260609` — full disk snapshot before Phase I work

---

## 4. Container Inventory

### TAMALE stack containers (`/home/tamale/docker-compose.yaml`)

| Container | Image | Port | Role |
|-----------|-------|------|------|
| `tamale-rails-1` | Chatwoot fork (built local) | 3000 | Rails app server |
| `tamale-sidekiq-1` | Chatwoot fork | — | Background jobs |
| `tamale-postgres-1` | postgres:16 + pgvector | 5432 | Main DB |

DB inside container: `tamale_production` (NOT `chatwoot_production` — common point of confusion).

### Dograh stack containers (`/home/dograh-compose.yaml`)

| Container | Image | Port | Role |
|-----------|-------|------|------|
| `home-api-1` | `ghcr.io/aarixu/dograh-api:tamale-latest` | 8000 | FastAPI + Pipecat voice agents |
| `home-ui-1` | `ghcr.io/aarixu/dograh-ui:tamale-latest` | 3010 | Next.js Voice Studio |
| `home-postgres-1` | pgvector/pgvector:pg17 | 5433 | Dograh DB |
| `home-redis-1` | redis:7 | 6380 | Cache + ARQ queues |
| `minio` | minio/minio | 9000-9001 | Object storage (call recordings) |
| `coturn` | coturn/coturn:4.8.0 | TURN ports | WebRTC NAT traversal |
| (nginx, bash helpers) | nginx:alpine, bash:5.2 | — | Internal routing |

**Health-check at a glance:**
```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}" | grep -E "tamale-|home-"
```

---

## 5. Network Topology & Ports

### Docker networks
- `tamale_default` — Chatwoot stack internal network (Rails ↔ Sidekiq ↔ Postgres)
- `home_app-network` — Dograh stack internal network (API ↔ UI ↔ Postgres ↔ Redis ↔ MinIO)
- `172.17.0.1` — docker0 bridge gateway; used for **cross-stack** calls (Dograh API calling TAMALE Rails for quota checks)

### Nginx vhost routing (`/etc/nginx/sites-available/`)

| File | Domain | Routes |
|------|--------|--------|
| `tamaleapp` | tamaleapp.com | `/` → :3000 (Rails); `/landing/` → static; legal pages → file aliases |
| `voicetamale` | voice.tamaleapp.com | `/` → :3010 (UI); `/api/` → :8000; storage → :9000; `/sso` → :3000 |

**Backups of nginx configs exist as:**
- `/etc/nginx/sites-available/tamaleapp.pre-legal-20260609.bak`

Always backup before edits: `sudo cp file file.pre-$(date +%Y%m%d_%H%M%S).bak`

---

## 6. Critical File Paths

### TAMALE app (Chatwoot fork)
```
/home/tamale/                          # Rails app root — Chatwoot fork source
/home/tamale/.env                      # TAMALE secrets (ROOT-owned)
/home/tamale/docker-compose.yaml       # TAMALE stack definition
/home/tamale/public/legal/             # Legal pages (privacy, terms, refund, data-deletion)
/home/tamale/public/landing/           # Landing page (separate git repo)
```

### Dograh stack
```
/home/dograh-compose.yaml              # Dograh stack definition (ROOT-owned)
/home/dograh-compose.yaml.pre-fork-*   # Backups before each major edit
/home/.env                             # Dograh secrets (ROOT-owned)
/home/dograh-fork/                     # Dograh source fork (writable)
/home/Dockerfile.dograh-ui             # LEGACY — pre-fork sed-patch Dockerfile (obsolete after I.5)
/home/Dockerfile.dograh-api            # LEGACY — same
/home/tamale-edit-interceptor.js       # Original interceptor script (now baked into UI fork)
/home/quota_service.py.new             # Original TAMALE quota service (now merged into fork)
```

### Backups
```
/var/backups/tamale/                   # Auto 6hr DB backups (Chatwoot DB)
/var/backups/dograh/                   # Manual DB dumps (Dograh DB)
```

### Nginx
```
/etc/nginx/sites-available/tamaleapp        # Main app vhost
/etc/nginx/sites-available/voicetamale      # Voice Studio vhost
/etc/nginx/sites-enabled/                   # Symlinks to above
```

---

## 7. Secrets & Credentials

**Location:** `/home/tamale/.env` (TAMALE) and `/home/.env` (Dograh). Both root-owned, mode 600.

**View safely:**
```bash
sudo cat /home/tamale/.env | grep -v '^#' | grep -v '^$'
```

**Critical env vars:**
- `OSS_JWT_SECRET` — shared HMAC secret between Chatwoot and Dograh for SSO
- `DOGRAH_WEBHOOK_SECRET` — Dograh → TAMALE callbacks
- `SECRET_KEY_BASE` — Rails session secret (Chatwoot)
- `SENTRY_DSN` — error tracking
- `DEEPSEEK_API_KEY` — primary AI provider
- `OPENAI_API_KEY` — embeddings (text-embedding-3-small)
- `ELEVENLABS_API_KEY` — voice TTS
- `GROQ_API_KEY` — alternate LLM
- `POSTGRES_PASSWORD` — DB password (different for tamale-postgres vs home-postgres)

**External account credentials** (not on VM, kept in password manager):
- GitHub PAT — for pushing to forks
- Dodo Payments — billing
- GCP project owner login

**Rotation cadence:** Quarterly. Procedure:
1. Generate new value
2. Update `.env` file
3. Restart affected containers
4. Update any external services referencing old value

---

## 8. The Chatwoot Fork (Main App)

### Repository
- **Origin:** `github.com/aarixu/Tamale-App.git`
- **Upstream:** `github.com/chatwoot/chatwoot.git` (read-only, push DISABLED)
- **Branch we deploy from:** `tamale-production`
- **Verify branch:** `cd /home/tamale && git branch --show-current`

### Branch model
```
upstream/develop ───►  Chatwoot's active development (DO NOT auto-pull)
upstream/master  ───►  Chatwoot stable
origin/main      ───►  Frozen fork mirror (DO NOT push TAMALE work here)
origin/tamale-production ───►  ★ SOURCE OF TRUTH for production deploys
tamale-production (local) ───►  Active working branch on VM
```

### Building
Chatwoot uses Rails + Vite. Build is **inside the running container**:

```bash
# After editing any *.vue, *.js, or *.scss
docker exec tamale-rails-1 sh -c "cd /app && bin/vite build 2>&1 | tail -15"

# After Vite finishes, restart Rails to pick up the new manifest
docker restart tamale-rails-1
```

**NEVER** run `docker compose down` on the TAMALE stack — postgres volume can desync.

### Editing workflow
1. `cd /home/tamale`
2. Edit source files (surgical edits only — never full-file SCP replace; sed/str_replace preferred)
3. Run Vite build (above)
4. Test in browser at tamaleapp.com
5. Commit: `git add <files> && git commit -m "..."`
6. Push: `git push origin tamale-production`

### Where to edit common things
| Want to change | File path |
|---------------|-----------|
| Sidebar Vue components | `app/javascript/dashboard/components/layout/` |
| AI agent builder logic | `app/services/ai_*.rb`, `app/models/ai_*.rb` |
| Routes | `config/routes.rb` |
| Plan/quota logic | `app/models/plan.rb`, `app/services/voice_quota_service.rb` |
| Voice Studio iframe + SSO | `app/javascript/dashboard/components/VoiceStudio.vue`, `app/controllers/sso/dograh_controller.rb` |
| Landing page | Separate repo: `github.com/aarixu/tamale-landing` |

### Migration discipline
- Last used timestamp: `20260608120002`
- New migrations: use later timestamps to avoid collisions
- Run: `docker exec tamale-rails-1 sh -c "cd /app && bin/rails db:migrate"`

---

## 9. The Dograh Fork (Voice Studio)

### Repository
- **Origin:** `github.com/aarixu/dograh.git`
- **Upstream:** `github.com/dograh-hq/dograh.git` (read-only, push DISABLED)
- **Branch we deploy from:** `tamale-production`
- **License:** BSD 2-Clause (commercial fork allowed — keep Zansat copyright in LICENSE)

### Why we forked
Previously, we patched the upstream Dograh Docker image with 8 `sed` commands on minified JS. Fragile (broke on every upstream push). Now we own the source.

### Build pipeline — GitHub Actions
**Trigger:** any `git push origin tamale-production` in the Dograh fork.  
**Workflow file:** `.github/workflows/build-tamale.yml`

**What it does:**
1. Checks out source (with `pipecat` submodule)
2. Builds UI Dockerfile with repo root as context
3. Builds API Dockerfile with repo root as context
4. Pushes both images to GHCR (public):
   - `ghcr.io/aarixu/dograh-ui:tamale-latest` + `:tamale-<sha>`
   - `ghcr.io/aarixu/dograh-api:tamale-latest` + `:tamale-<sha>`

**Build time:** ~8 min UI, ~10 min API (parallel).

### Editing workflow
1. `cd /home/dograh-fork`
2. Edit source
3. Verify syntax: `python3 -m py_compile <file>` (for .py)
4. Commit + push to `tamale-production`
5. **Watch:** `github.com/aarixu/dograh/actions` until both jobs green (~8 min)
6. Pull new images on VM:
   ```bash
   sudo docker pull ghcr.io/aarixu/dograh-ui:tamale-latest
   sudo docker pull ghcr.io/aarixu/dograh-api:tamale-latest
   ```
7. Restart Dograh containers (see Deployment Workflows section)

### Source-level patches applied (from I.5)
| What we changed | File |
|-----------------|------|
| Delete Chatwoot widget injection | `ui/src/components/ChatwootWidget.tsx` (file deleted) + import removed from `ui/src/app/layout.tsx` |
| Delete Slack join button | Block removed from `ui/src/components/layout/AppLayout.tsx` |
| Delete GitHub star badge | `ui/src/components/layout/GitHubStarBadge.tsx` (deleted) + all 3 usages removed |
| Delete "Report Issue" GitHub link | Block removed from `ui/src/app/overview/page.tsx` |
| Rebrand title | `ui/src/app/layout.tsx`: `"Voice Studio - Tamale"` |
| Rebrand mobile header | `ui/src/components/layout/AppLayout.tsx`: `"Voice Studio"` |
| Inject TAMALE assets | `ui/public/tamale-overrides.css`, `ui/public/tamale-edit-interceptor.js` + `<link>` and `<script>` tags in layout `<head>` |
| TAMALE quota gate | `api/services/quota_service.py` — upstream `authorize_workflow_run_start` + injected TAMALE check (fail-open) |

### Key files for future edits
| Want to change | Path inside `/home/dograh-fork/` |
|---------------|----------------------------------|
| Voice Studio UI components | `ui/src/components/`, `ui/src/app/` |
| Voice agent backend logic | `api/services/`, `api/routes/` |
| Quota / billing integration | `api/services/quota_service.py` (already merged) |
| Telephony / ARI manager | `api/services/telephony/ari_manager.py` |
| Campaign orchestrator | `api/services/campaign/` |
| Dockerfiles (rarely needed) | `ui/Dockerfile`, `api/Dockerfile` |

---

## 10. Database

### Two separate databases

| Database | Container | DB Name | Why |
|----------|-----------|---------|-----|
| TAMALE main | `tamale-postgres-1` | `tamale_production` | Conversations, users, plans, AI memory (pgvector), agency data |
| Dograh | `home-postgres-1` | `postgres` | Voice workflows, call runs, telephony state, recordings metadata |

### Connecting

```bash
# TAMALE DB
docker exec -it tamale-postgres-1 psql -U postgres -d tamale_production

# Dograh DB
docker exec -it home-postgres-1 psql -U postgres -d postgres
```

### Schema management
- **TAMALE:** Rails ActiveRecord migrations. `db:migrate`, `db:rollback`.
- **Dograh:** Alembic. Migrations live at `api/alembic/versions/` inside the API image. Auto-run on container start via `start_services_docker.sh`.

### Critical alembic facts (Dograh)
- Current head as of doc date: `384be6596b36`
- If `stamp` mismatch causes startup failure (saw this during I.5), fix is:
  ```bash
  sudo docker run --rm --network home_app-network \
    -e DATABASE_URL="postgresql+asyncpg://postgres:postgres@postgres:5432/postgres" \
    -e REDIS_URL="redis://:redissecret@redis:6379" \
    --entrypoint sh ghcr.io/aarixu/dograh-api:tamale-latest -c \
    "cd /app && alembic -c api/alembic.ini stamp <revision_id>"
  ```

### Common SQL queries

```sql
-- TAMALE: check account quotas
SELECT id, name, plan_id, voice_minutes_used, credits_remaining FROM accounts;

-- TAMALE: check alembic state (for pgvector schema)
SELECT version_num FROM schema_migrations ORDER BY version_num DESC LIMIT 5;

-- Dograh: check alembic state
SELECT version_num FROM alembic_version;

-- Dograh: recent workflow runs
SELECT id, workflow_id, state, created_at FROM workflow_runs
  ORDER BY created_at DESC LIMIT 20;
```

---

## 11. Backups

### Automated (6-hourly)
- **Location:** `/var/backups/tamale/` (Chatwoot DB) and `/var/backups/dograh/` (Dograh DB)
- **Format:** `pg_dump` plain SQL
- **Retention:** Manual cleanup (no auto-prune yet — TODO J.8)
- **PRESERVE:** Until at least 2026-07 (per current ops policy)

### Manual snapshot before risky operation
```bash
# TAMALE DB
sudo docker exec tamale-postgres-1 pg_dump -U postgres tamale_production \
  | sudo tee /var/backups/tamale/manual-$(date +%Y%m%d_%H%M%S).sql > /dev/null

# Dograh DB
sudo docker exec home-postgres-1 pg_dump -U postgres postgres \
  | sudo tee /var/backups/dograh/manual-$(date +%Y%m%d_%H%M%S).sql > /dev/null
```

### Restore (DESTRUCTIVE)
```bash
# Stop Rails first (so no writes during restore)
docker stop tamale-rails-1 tamale-sidekiq-1

# Restore
sudo cat /var/backups/tamale/<file>.sql | docker exec -i tamale-postgres-1 \
  psql -U postgres -d tamale_production

# Restart
docker start tamale-postgres-1 tamale-sidekiq-1 tamale-rails-1
```

### Offsite (TODO — J.8 in roadmap)
Not yet implemented. Local backups are vulnerable to VM disk failure. Planned: encrypt with `age` + push to GCS bucket nightly.

### Full VM snapshot
- GCP Console → Compute Engine → Snapshots → Create
- Last known: `chatwoot-pre-phase-i-20260609`
- Cost: ~₹50/month per snapshot (negligible)

---

## 12. Deployment Workflows

### Deploying a TAMALE (Chatwoot fork) change

```bash
cd /home/tamale

# 1. Make + verify edits
git status
git diff <files>

# 2. Build frontend if Vue/JS changed
docker exec tamale-rails-1 sh -c "cd /app && bin/vite build 2>&1 | tail -15"

# 3. Restart Rails to load new manifest
docker restart tamale-rails-1

# 4. Verify
curl -sI https://tamaleapp.com | head -3
docker logs tamale-rails-1 --tail 30

# 5. Commit + push to GitHub
git add <files>
git commit -m "..."
git push origin tamale-production
```

### Deploying a Dograh fork change

```bash
# On VM
cd /home/dograh-fork

# 1. Make + verify edits
git status
python3 -m py_compile <changed.py>   # for Python files

# 2. Push (triggers GitHub Actions build)
git add <files>
git commit -m "..."
git push origin tamale-production

# 3. WAIT — watch github.com/aarixu/dograh/actions until both jobs green (~8 min)

# 4. On VM — pull new images
sudo docker pull ghcr.io/aarixu/dograh-ui:tamale-latest
sudo docker pull ghcr.io/aarixu/dograh-api:tamale-latest

# 5. Restart Dograh UI + API (NOT down — just stop, rm, up)
cd /home
sudo docker compose -f dograh-compose.yaml --env-file .env stop ui api
sudo docker compose -f dograh-compose.yaml --env-file .env rm -f ui api
sudo ENABLE_TELEMETRY=false docker compose -f dograh-compose.yaml --env-file .env up -d --no-build ui api

# 6. Verify
sleep 30
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}" | grep home-
docker logs home-api-1 --tail 30
curl -sI https://voice.tamaleapp.com | head -3
```

### Deploying to BOTH at once (rare)
1. Push Dograh change first → wait for build
2. Push TAMALE change → vite build → restart
3. Pull + restart Dograh
4. Test cross-stack flow (SSO, Voice Studio iframe)

---

## 13. Syncing From Upstream (Security Patches)

### Chatwoot security sync

**When:** A Chatwoot security advisory affects us (check periodically at chatwoot.com/security).

```bash
cd /home/tamale

# 1. Fetch latest upstream (read-only)
git fetch upstream

# 2. See what's new
git log origin/main..upstream/develop --oneline | head -30

# 3. Find the security patch commit SHA
git log upstream/develop --oneline | grep -i "security\|cve\|fix" | head

# 4. Create a test branch from tamale-production
git checkout -b test/security-patch-$(date +%Y%m%d) tamale-production

# 5. Cherry-pick the patch
git cherry-pick <commit-sha>

# 6. Resolve conflicts if any
# (edit files, then: git add . && git cherry-pick --continue)

# 7. Build + test in container
docker exec tamale-rails-1 sh -c "cd /app && bin/vite build && bin/rails db:migrate:status"

# 8. If happy:
git checkout tamale-production
git merge --no-ff test/security-patch-<date>
git push origin tamale-production
# Rebuild + restart as in normal deploy

# 9. If broken:
git checkout tamale-production
git branch -D test/security-patch-<date>
# Production untouched ✓
```

### Dograh upstream sync

```bash
cd /home/dograh-fork

# Fetch
git fetch upstream

# Compare
git log upstream/main..tamale-production --oneline   # our changes ahead
git log tamale-production..upstream/main --oneline   # what we're missing

# Cherry-pick the way described above for Chatwoot
# After merge: push → GitHub Actions builds → pull + restart on VM
```

**Reminder:** `push` to upstream is DISABLED. You cannot accidentally damage upstream.

---

## 14. Common Operations Cookbook

### View live logs
```bash
docker logs tamale-rails-1 -f --tail 100
docker logs home-api-1 -f --tail 100
docker logs home-ui-1 -f --tail 100
```

### Restart a service without losing data
```bash
docker restart tamale-rails-1                  # Rails only
docker restart tamale-sidekiq-1                # Sidekiq only
docker restart home-api-1                      # Dograh API only
```

### Rails console
```bash
docker exec -it tamale-rails-1 sh -c "cd /app && bin/rails console"
```

### Python shell inside Dograh API
```bash
docker exec -it home-api-1 python3
```

### Container shell
**Note:** Both stacks use `sh` not `bash` inside the container.
```bash
docker exec -it tamale-rails-1 sh
docker exec -it home-api-1 sh
```

### Multi-line scripts in Rails container
**Don't** use heredoc with `!` characters (bash history expansion breaks them silently). Instead:
```bash
# Write the script locally, then docker cp + run
echo 'Account.where(...).update_all(plan_id: 2)' > /tmp/script.rb
docker cp /tmp/script.rb tamale-rails-1:/tmp/script.rb
docker exec tamale-rails-1 sh -c "cd /app && bin/rails runner /tmp/script.rb"
```

### Check disk usage
```bash
df -h /home /var
docker system df
```

### Clean up old Docker images (frees disk)
```bash
docker image prune -af
docker volume prune -f      # WARNING: drops unused volumes
```

### Reload nginx (after config edit)
```bash
sudo nginx -t                  # syntax check first
sudo systemctl reload nginx
```

---

## 15. Disaster Recovery

### Scenario 1: TAMALE app crashed, won't restart

1. Check container status: `docker ps -a | grep tamale`
2. Check logs: `docker logs tamale-rails-1 --tail 200`
3. Common cause: bad Vite build manifest → rebuild:
   ```bash
   docker exec tamale-rails-1 sh -c "cd /app && rm -rf public/vite && bin/vite build"
   docker restart tamale-rails-1
   ```
4. If still broken — revert last git commit:
   ```bash
   cd /home/tamale
   git log --oneline -5
   git revert HEAD --no-edit
   # Rebuild + restart
   ```

### Scenario 2: Dograh API in restart loop

1. Logs: `docker logs home-api-1 --tail 200`
2. Common cause: alembic migration mismatch — see Database section
3. If image itself is broken — rollback to previous image:
   ```bash
   # Find previous tag
   docker images | grep dograh
   # Edit dograh-compose.yaml to use specific :tamale-<old-sha> tag
   sudo nano /home/dograh-compose.yaml
   # Restart
   ```

### Scenario 3: DB corrupted

1. Stop writers immediately:
   ```bash
   docker stop tamale-rails-1 tamale-sidekiq-1
   ```
2. Restore from `/var/backups/tamale/<latest>.sql` (see Backups section)
3. If backup also corrupt: GCP snapshot restore (see below)

### Scenario 4: Entire VM died

1. GCP Console → Compute Engine → VM Instances → check status
2. If unrecoverable: create new VM from snapshot `chatwoot-pre-phase-i-20260609` (or latest)
3. New VM gets new external IP — update Cloudflare DNS A records:
   - `tamaleapp.com` → new IP
   - `voice.tamaleapp.com` → new IP
4. Verify nginx + docker containers come up automatically (systemd should restart docker)
5. **Expect ~10 min downtime** in worst case

### Scenario 5: Lost SSH access to VM

- GCP Console → SSH (browser-based) always works as long as you have GCP account access
- Failsafe: GCP Console → VM → Reset (forces reboot, doesn't lose data)

### Scenario 6: Upstream Chatwoot does something terrible

Doesn't affect you. You don't auto-pull. Your `tamale-production` branch is yours forever. Continue working.

### Scenario 7: Dograh company shuts down

Doesn't affect you. Your fork at `github.com/aarixu/dograh` has full source + git history. Continue working. Build via Actions still works (GitHub-hosted runners are independent of Dograh).

---

## 16. Troubleshooting Decision Tree

```
Issue: User reports problem
│
├─ Login broken / app totally down?
│   └─ Check: docker ps | grep tamale-rails-1
│       ├─ Container missing/Exited → docker start tamale-rails-1
│       ├─ Container restarting → docker logs tamale-rails-1 --tail 200
│       └─ Container running → check nginx: sudo systemctl status nginx
│
├─ Voice Studio shows 502?
│   └─ Check: docker ps | grep home-api-1
│       ├─ Missing/restarting → check alembic, then logs
│       └─ Running → check nginx routing for voice.tamaleapp.com
│
├─ AI replies stopped working?
│   └─ Check Sidekiq queue depth + DeepSeek API status
│       docker exec tamale-rails-1 sh -c "cd /app && bin/rails runner 'puts Sidekiq::Queue.new.size'"
│
├─ DB connection errors?
│   └─ docker logs tamale-postgres-1 --tail 50
│       ├─ "FATAL: too many connections" → restart Rails (kills idle conns)
│       └─ "disk full" → df -h, clean /var/backups
│
├─ "Welcome to Chatwoot" still showing?
│   └─ Frontend cache. Rebuild: docker exec tamale-rails-1 sh -c "cd /app && bin/vite build" + restart
│
├─ New TAMALE feature deployed but not visible?
│   └─ Cleared browser cache? Vite hash file served?
│       curl https://tamaleapp.com/vite/manifest.json | head
│
├─ Voice quota check returns wrong value?
│   └─ Dograh API → TAMALE Rails call works?
│       docker exec home-api-1 curl http://172.17.0.1:3000/api/v1/voice_quota/check?dograh_org_id=2 \
│         -H "Authorization: Bearer $DOGRAH_WEBHOOK_SECRET"
│
└─ Anything else broken?
    └─ Read this doc top to bottom + search logs:
        docker logs <container> --since 1h 2>&1 | grep -i error
```

---

## 17. Known Gotchas

1. **Never `docker compose down` on production** — postgres volume can desync. Use `stop` + `rm` + `up -d` instead.

2. **Container shell is `sh`, not `bash`** — exclamation marks in heredocs get silently dropped due to bash history expansion. Use base64 or `docker cp` for multi-line scripts.

3. **Vite build INSIDE container only** — pnpm is unavailable on host. Always:
   ```bash
   docker exec tamale-rails-1 sh -c "cd /app && bin/vite build 2>&1 | tail -15"
   ```

4. **TWO Pricing.vue files exist** in Chatwoot — the router uses `dashboard/components/app/pricing/Pricing.vue`, NOT `routes/dashboard/settings/billing/Pricing.vue`. Edit the right one.

5. **DB name confusion** — TAMALE DB is `tamale_production`, NOT `chatwoot_production` (despite the upstream default).

6. **pgvector dedup threshold** — set to 0.88 cosine similarity in the AI memory layer. Paraphrased-but-identical content scores around 0.90 and was being missed at higher thresholds.

7. **Message model default_scope** — Chatwoot's `app/models/message.rb` (line ~127) applies an ascending order that silently overrides explicit `.order()` calls in services. Use `.reorder()` instead of `.order()`.

8. **AsyncDispatcher#listeners is explicit, not auto-glob** — when adding new event listeners (e.g., ShadowSessionListener), they must be manually added to the listeners array.

9. **Nginx asset path conflict** — Rails compiles to `/public/assets/`; the landing page's static files must use a `/static/` alias to avoid collision.

10. **Migration timestamp discipline** — last used migration timestamp is `20260608120002`. New migrations must use later timestamps.

11. **Sidekiq cannot reach localhost** — Sidekiq containers can't reach `localhost:8000` (the API). Use docker bridge `172.17.0.1:8000` instead.

12. **Nginx needs `underscores_in_headers on`** — for the custom `api_access_token` header.

13. **Surgical edits only** — full-file SCP replace silently overwrites prior surgical patches. Always use `sed`, `str_replace`, or merge to a single file before upload.

14. **PAT exposure** — never paste GitHub PATs in chat, screenshots, or pastebins. GitHub auto-revokes detected PATs anyway. Rotate every 90 days.

15. **Dograh image pull = manual step** — GitHub Actions builds and pushes images, but does NOT auto-deploy to your VM. You must `docker pull` + restart.

---

## 18. Glossary

| Term | Meaning |
|------|---------|
| **TAMALE** | The product. Customer support SaaS for agencies. |
| **Chatwoot fork** | The forked source code of Chatwoot, with TAMALE customizations applied. Lives at `github.com/aarixu/Tamale-App`. |
| **Dograh fork** | The forked source code of Dograh, with TAMALE customizations. Lives at `github.com/aarixu/dograh`. |
| **tamale-production** | The git branch (in both forks) that represents what's actually deployed. |
| **Voice Studio** | The Dograh-powered voice agent builder, embedded in TAMALE at `voice.tamaleapp.com`. |
| **JWT SSO** | The flow where TAMALE signs a short-lived token, and Voice Studio trusts it, so the user doesn't see a separate login. |
| **GHCR** | GitHub Container Registry — where Dograh fork images are hosted (`ghcr.io/aarixu/dograh-*`). |
| **Alembic** | The Python migration tool used by Dograh's API. Equivalent to Rails ActiveRecord migrations. |
| **Pipecat** | The voice AI framework used by Dograh. Included as a git submodule in the Dograh repo. |
| **MPS** | Managed Provider Services — Dograh's hosted billing system, which we partially integrate with. |
| **Antigravity** | The AI coding agent (Gemini-based) running on Windows, used for source code edits. |
| **VM** | The single GCP virtual machine that hosts everything (`34.131.162.197`). |
| **Stack** | A Docker Compose deployment (TAMALE stack and Dograh stack are two separate stacks). |

---

## Contact Cheat Sheet

| Need | Where to look |
|------|---------------|
| Recent changes log | `cd /home/tamale && git log --oneline -20` and `cd /home/dograh-fork && git log --oneline -20` |
| Project roadmap | `github.com/aarixu/tamale-todo` |
| Anthropic Claude session history | Previous chats with project context |
| GCP Console | console.cloud.google.com |
| GitHub repos | `github.com/aarixu/Tamale-App`, `github.com/aarixu/dograh`, `github.com/aarixu/tamale-landing` |

---

**End of Runbook.**

This is a living document. After major infrastructure changes, update the relevant section and the version number at the top.
