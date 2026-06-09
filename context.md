# TAMALE PROJECT CONTEXT

## 🏗️ PROJECT CONTEXT

TAMALE = Chatwoot fork (Rails 7 + Vue 3) at tamaleapp.com
       + Dograh voice AI at voice.tamaleapp.com
VM:    Google Cloud 34.131.162.197, user g1208aarav
Repo:  https://github.com/aarixu/Tamale-App.git

Antigravity = primary coding agent that deploys changes to VM
Ashish runs commands manually via Google Cloud SSH browser

## 🐳 INFRASTRUCTURE

Containers:
  TAMALE:  tamale-rails-1 (port 3000), tamale-sidekiq-1, tamale-postgres-1
  Dograh:  home-api-1 (port 8000), home-ui-1 (port 3010),
           home-postgres-1 (port 5433), home-redis-1 (port 6380),
           minio (port 9000-9001)

Nginx:
  /etc/nginx/sites-available/tamaleapp    — tamaleapp.com → :3000
  /etc/nginx/sites-available/voicetamale  — voice.tamaleapp.com → :3010 UI,
                                             :8000 API, :9000 storage,
                                             :3000 for /sso

Docker compose:
  /home/tamale/docker-compose.yaml   — TAMALE stack
  /home/dograh-compose.yaml          — Dograh stack (uses /home/.env)

Postgres DB name: chatwoot_production
  Connect: docker exec tamale-postgres-1 psql -U postgres -d chatwoot_production

## 🔑 CRITICAL SECRETS (DO NOT SHARE EXTERNALLY)

OSS_JWT_SECRET (Dograh):      <REDACTED>
Dograh webhook secret:        <REDACTED>
Rails SECRET_KEY_BASE:        <REDACTED>
Account 2 dograh_user_token:  <REDACTED>
Admin:                        ashishofficial1208@gmail.com / <REDACTED>
Postgres password:            <REDACTED>
Postgres volume:              tamale_postgres_data
Sentry DSN:                   <REDACTED>
Twilio:                       <REDACTED> / <REDACTED>
DeepSeek API key:             <REDACTED>

## 🗃️ DB STATE

Account 1 (TAMALE HQ): plan=free, dograh_org=4, dograh_wf=4, inbox=6
Account 2 (test):      plan=free, dograh_org=2, dograh_wf=3, inbox=5,
                       6 active agents, 20+ conversations

Plans table seeded: Free / Starter / Pro / Agency / Enterprise
Plan::LLM_TIER_MULTIPLIERS = budget:0.5, standard:1.0, premium:5.0, pro:10.0

## 📁 CRITICAL PERMANENT FILES (DO NOT LOSE)

/home/tamale/.env                                  — TAMALE secrets
/home/.env                                         — Dograh secrets
/home/tamale/docker-compose.yaml                   — TAMALE stack
/home/dograh-compose.yaml                          — Dograh stack
/home/Dockerfile.dograh-ui                         — 8 white-label patches
/home/tamale-edit-interceptor.js                   — postMessage handler
/var/backups/tamale/                               — Auto 6hr DB backups
/var/backups/dograh/                               — Auto 6hr DB backups

JWT SSO files (Jun 5):
  /home/tamale/app/controllers/api/v1/dograh_sso_tokens_controller.rb
  /home/tamale/app/controllers/sso/dograh_controller.rb (+ .pre-jwt.bak)
  /home/tamale/config/routes.rb (+ .pre-jwt.bak)
  /home/tamale/app/javascript/dashboard/components/VoiceStudio.vue (+ .pre-jwt.bak)

B.4-Part2 files (Jun 6):
  /home/tamale/app/javascript/dashboard/components/widgets/LlmTierSelect.vue
  /home/tamale/app/javascript/dashboard/routes/dashboard/settings/inbox/FinishSetup.vue
  /home/tamale/app/javascript/dashboard/routes/dashboard/settings/inbox/settingsPage/ConfigurationPage.vue
  /home/tamale/app/javascript/dashboard/components/app/pricing/Pricing.vue  ← REAL ONE
  /home/tamale/app/controllers/api/v1/plans_controller.rb

## ⚠️ WORKING RULES — NON-NEGOTIABLE

 1. NEVER run `docker compose down` on production — causes volume mismatch
 2. SURGICAL edits only — no full-file SCP replace (silently overwrites patches)
 3. Hinglish preferred in conversation, concise responses
 4. Antigravity = primary code deploy agent; VM SSH for sensitive ops only
 5. Token-aware — don't over-explain
 6. Workflow: audit → confirm → fix → test → verify → ✅ next
 7. 1 task = 1 chat policy (context efficiency)
 8. After Vite build → Rails restart for manifest reload
 9. After Vite build → verify server serves latest hash file
10. Production-grade only — scale to 1000+ users, no demo shortcuts
11. Vite build INSIDE container:
    docker exec tamale-rails-1 sh -c "cd /app && bin/vite build 2>&1 | tail -15"
12. Container shell is `sh` NOT `bash`

## 🐛 KNOWN GOTCHAS

• Sidekiq containers cannot reach localhost:8000 → use Docker bridge 172.17.0.1:8000
• Nginx needs `underscores_in_headers on` for `api_access_token` header
• CSS `:has-text()` is NOT standard — use other selectors
• Bash history expansion (!) breaks single-quoted heredocs
• Container shell is `sh` not `bash` — use `docker exec ... sh -c`
• Dograh image rebuild:
    cd /home && sudo ENABLE_TELEMETRY=false OSS_JWT_SECRET=<REDACTED> \
    docker compose -f dograh-compose.yaml up -d --force-recreate ui
• TWO Pricing.vue files exist — router uses
    dashboard/components/app/pricing/Pricing.vue (NOT routes/dashboard/settings/billing/)
• Antigravity sometimes does full-file SCP replace and silently overwrites
  prior surgical patches → ALWAYS instruct surgical edits explicitly

## ⚖️ LEGAL IDENTITY (Current — Pre-Registration)

Operator:      Aashish (individual proprietor — Sole Proprietorship coming)
Trading as:    TAMALE
Address:       Ladrawan, Bahadurgarh, Haryana - 124507
Email:         support@tamaleapp.com
Jurisdiction:  Haryana, India (Bahadurgarh courts)
Strategy:      Apply Udyam (free, instant) → Meta App approval → GST after ₹20L

## 🎯 EXECUTION ORDER (per Ashish's call Jun 6, 2026)

 1. Phase D — Meta Channels (TOP NOW, blocked on FB OTP)
 2. Phase F — Smart AI Vision (memory/reasoning/multi-agent)
 3. Phase G — Voice Realism + Multi-Model (ElevenLabs, Gemini, OpenAI, BYOK)
 4. Phase H — Agency Multi-Account Dashboard (target market USP)
 5. Phase I — UI Overhaul + Fork Isolation (SNAPSHOT FIRST!)
 6. Phase J — Robustness & Security (Claude's best-in-class additions)
 7. V2     — WebRTC Live Handoff + Click-to-Call
 8. Phase C — Live Payments (LAST — test mode already verified)
 9. Phase E — E2E Validation (LAST — needs all upgrades first)

## 🔄 SESSION-START PROTOCOL

1. Paste this JSX file into new chat
2. Say: "We'll work on <task ID or name>"
3. Claude writes audit prompt (Antigravity-ready)
4. You run audit → paste output
5. Claude writes fix prompt (Antigravity-ready)
6. You run fix → paste output
7. Test → verify → mark ✅ in this dashboard → end chat
8. Start new chat for next task

NEVER: mix multiple tasks in one chat, skip audit, full-file SCP replace,
       run docker compose down
