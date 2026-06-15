# Handoff Document: Info Butler

**Date:** 2026-06-14  
**Repo:** `https://github.com/fuzheado/info-butler` (private)  
**Working directory:** `~/Documents/ai/info-butler`  
**Target hardware:** Mac Mini M1, 16 GB unified memory

---

## What This Project Is

An autonomous Telegram group chat agent ("The Info Butler") that captures knowledge from team discussions and renders it as a visual, interconnected "Living Wiki" using Quartz (a digital garden static site generator with D3 force-directed graphs). Uses Nous Research's Hermes Agent as the Telegram bot backend, with a multi-tier model routing strategy (local Qwen 3.5 4B for chat parsing, Google Gemini for heavy lifting, OpenAI GPT Image for asset generation).

---

## Current State

The project is in the **design + documentation phase**. Nothing is deployed yet. All work so far has been:

- Writing and refining the system design document (`PROJECT.md`)
- Creating the infrastructure config (`docker-compose.yml`)
- Setting up the git repo and pushing to GitHub
- Writing a critical analysis (`analysis.md`)

### What exists in the repo

| File | Purpose | Status |
|---|---|---|
| `PROJECT.md` | Full system design document — architecture, routing, setup phases, verification | ✅ Current (updated through 7 revisions) |
| `PROJECT-old.md` | Previous version of the spec (pre-analysis) | 📦 Archive |
| `analysis.md` | Honest critique of the original spec — memory math, missing bot logic, Quartz latency | ✅ Complete |
| `AGENTS.md` | Project-level directive for AI agents (verify-before-assert) | ✅ Complete |
| `docker-compose.yml` | Two-service compose: hermes-agent + quartz-visualizer | ✅ Current |
| `.gitignore` | Ignores wiki/, node_modules/, .env, macOS/IDE cruft | ✅ Complete |

### Infrastructure config (`docker-compose.yml`)

```yaml
services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    command: ["gateway", "run"]
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 6g
    ports:
      - "9119:9119"    # Hermes Dashboard
    environment:
      # Telegram
      - TELEGRAM_BOT_TOKEN          → placeholder (was exposed, needs new one)
      - TELEGRAM_ALLOWED_USERS      → placeholder
      - TELEGRAM_GROUP_ALLOWED_CHATS → placeholder
      # Primary model: DeepSeek V4 Flash
      - DEEPSEEK_API_KEY            → sk-your-deepseek-key
      # OpenAI key: for image generation / tools
      - OPENAI_API_KEY              → sk-your-openai-key
      # Cloud fallback
      - GEMINI_API_KEY              → placeholder
      # Dashboard disabled
      # - HERMES_DASHBOARD=1
    volumes:
      - ./wiki:/opt/data/wiki
      - ./config:/opt/data/config
      - ./config/config.yaml:/opt/data/config.yaml:ro

  quartz-visualizer:
    image: node:22-alpine
    ports:
      - "8080:8080"
    volumes:
      - ./wiki:/usr/src/app/content
    command: clones quartz, npm ci, quartz build --serve
```

---

## What Has Been Done

### 1. Project spec written and iterated
- Original `PROJECT.md` written as a vision document
- `analysis.md` written as a thorough critique (verified all model names/tools exist — they do)
- Multiple revisions applied: memory budget corrected (9B → 4B model), auth tightened (allowlist), Quartz latency expectations made realistic, Mac-specific Docker recommendations added, reboot survival section added

### 2. Git repo created and pushed to GitHub
- Private repo at `github.com/fuzheado/info-butler`
- 7 commits tracking the evolution of the docs and config

### 3. Real credentials obtained (and one revoked)
- Telegram bot created via @BotFather
- **Token was accidentally exposed in the chat** — needs to be revoked and replaced (see Action Items)
- Group chat ID identified (via `getUpdates` API)
- User IDs identified (via @userinfobot)

### 4. Key technical decisions made
| Decision | Rationale |
|---|---|
| Qwen 3.5 4B instead of 9B | 9B (6.6GB) + Docker + macOS exceeds 16GB |
| Colima instead of Docker Desktop | Saves ~1-1.5GB RAM overhead |
| `deploy.resources.limits` for memory/cpu | Compose v3.8 doesn't allow service-level `memory` |
| `TELEGRAM_ALLOWED_USERS` instead of `GATEWAY_ALLOW_ALL_USERS` | Per-user allowlist is safer |
| Quartz instead of Obsidian Publish | Free, open-source, self-hosted |
| Hermes Agent as the bot framework | Real, maintained, Telegram support built-in |

---

## What Is Pending / Blocked

### 🚨 Critical before deployment

1. **Revoke the exposed bot token** — The token `8973681953:AAHAzy23zpD6gysevB5gwIgOvYc2-qRH73s` was posted in plain text. It must be revoked via @BotFather → `/mybots` → your bot → API Token → Revoke. Then update `TELEGRAM_BOT_TOKEN` in `docker-compose.yml`.

2. **Fill in all placeholders in `docker-compose.yml`**:
   - `TELEGRAM_BOT_TOKEN` — new token after revocation
   - `TELEGRAM_ALLOWED_USERS` — actual numeric user IDs (from @userinfobot)
   - `TELEGRAM_GROUP_ALLOWED_CHATS` — actual group chat ID (from web.telegram.org URL)
   - `DEEPSEEK_API_KEY` — from [DeepSeek platform](https://platform.deepseek.com) — **required** (primary model)
   - `OPENAI_API_KEY` — from [platform.openai.com](https://platform.openai.com) — optional, for image generation
   - `GEMINI_API_KEY` — from Google AI Studio — optional, for document parsing fallback

   The `config/config.yaml` is pre-configured for DeepSeek V4 Flash. No changes needed there.

### 🔧 Setup steps not yet done

3. **Install and configure Docker runtime** (see Phase 2.5 in PROJECT.md):
   ```bash
   brew install docker docker-compose colima
   colima start --cpu 4 --memory 8 --disk 60 --vm-type=vz --mount-type=virtiofs
   ```

4. **Disable Telegram privacy mode** for the bot (it defaults to on):
   - @BotFather → `/mybots` → bot → Bot Settings → Group Privacy → Turn off
   - Remove and re-add bot to the group

5. **Activate the Hermes profile** (`~/info-butler/config/profile.yaml`):
   ```bash
   docker exec hermes-butler hermes profile activate info_butler
   ```

6. **Set up `brew services start colima`** for auto-start on reboot

### 🧪 Testing needed

9. Start the stack: `docker compose up -d`
10. Verify Telegram gateway connected — **use the log file inside the container**, not `docker logs`:
    ```bash
    docker exec hermes-butler tail -20 /opt/data/logs/gateway.log
    # Look for: ✓ telegram connected
    ```
11. Send a test message — `/help` should work immediately; `@bot_name` only works after privacy mode is disabled
12. Verify Quartz is serving: `http://localhost:8080`

### 📝 Future work (from analysis.md)

13. **Implement Hermes skills** — The bot logic (parsing "Save this connection", routing to models, writing wiki files) isn't automatic. Needs custom Python skills written for Hermes.
14. **Backup strategy** — Hourly cron `git add -A && git commit && git push` (described in PROJECT.md §4.2, not yet configured)
15. **Cloudflare Tunnel** — For remote admin dashboard access (optional, needs domain)
16. **Monitor memory pressure** — 16GB is tight. Consider adding a memory monitoring script or Docker health checks.

---

## Known Issues & Gotchas

- **Quartz is not real-time** — It's a static site generator. Graph updates take 15–30 seconds, not "seconds" as originally claimed. This is by design and acceptable.
- **Bot token exposure** — The old token is compromised. Don't deploy with it. Revoke first.
- **Privacy mode confusion** — The bot will respond to `/commands` but silently ignore `@mentions` until privacy mode is off in BotFather AND the bot is removed/re-added to the group. This is a Telegram API limitation, not a Hermes config issue.
- **Gateway logs go to a file, not stdout** — `docker logs` only shows the s6 init banner. Always use `docker exec hermes-butler tail /opt/data/logs/gateway.log` to see real activity.
- **Primary model is DeepSeek V4 Flash (cloud API)** — $0.14/1M input, $0.28/1M output. Fast tool calling. No local model needed — frees up all 16 GB RAM for Docker + macOS.
- **Hermes has a built-in `deepseek` provider** — no base URL needed. Just set `DEEPSEEK_API_KEY` and `model.provider: deepseek` in config.
- **No Ollama management** — no local model to babysit, no context window tuning, no API timeout tweaks.
- **Colima vs Docker autostart** — `brew services start colima` handles auto-start, but only after GUI login. If running headless, macOS auto-login must be enabled.

---

## Key Commands Reference

```bash
# Start everything
cd ~/info-butler && docker compose up -d

# Check logs
docker logs hermes-butler
docker logs quartz-wiki

# Verify Telegram received messages
curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates" | python3 -m json.tool

# Check Quartz site
open http://localhost:8080

# Stop everything
docker compose down

# Check Colima status
colima status

# Fully rebuild after changes
docker compose down && docker compose up -d
```

---

## Contact / Context

- Repo owner: `fuzheado` on GitHub
- All design decisions are documented in `PROJECT.md` and `analysis.md`
- The `AGENTS.md` file at the project root contains a "verify-before-assert" directive — any AI agent working on this project must search the web to verify factual claims about tools, models, or pricing before asserting them.
