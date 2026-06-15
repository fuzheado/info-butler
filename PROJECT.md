# System Design Document: "The Info Butler"

**Target Hardware:** Mac Mini M1 (16 GB Unified Memory)

**System Class:** Decentralized Team Infrastructure / Hybrid Edge-Cloud Knowledge Engine

**Security Boundary:** Containerized Sandbox via Token-Linked User Validation

---

## 1. Executive Summary & Design Philosophy

### 1.1 Motivation

Modern project mobilization often suffers from "chat decay." When a team of distributed collaborators brainstorms across various channels in a Telegram group, valuable insights, architectural decisions, and sudden sparks of inspiration are buried under hundreds of messages of day-to-day noise. Traditional knowledge management tools fail because they require friction—someone has to manually stop brainstorming, open a separate app, and file a rigid entry.

The **Info Butler Architecture** turns the group chat itself into a live, self-documenting workspace. By placing an autonomous agent inside the conversation as an omnipresent participant, the team can crowdsource project infrastructure entirely via ambient commands.

### 1.2 Core Philosophy: The Open Playground & The Hardened Sandbox

To maximize creative velocity and encourage bold engineering experiments, this system operates on a high-trust model: **any authorized collaborator in the group chat can command the butler to perform root-level operations.**

Because the system is deployed entirely within an isolated Docker runtime environment on dedicated, local Apple Silicon hardware, this open access is perfectly safe. If a collaborator triggers an optimized script that results in an infinite execution loop or a critical system error, the blast radius is strictly trapped inside a 2 GB Linux sandbox. The underlying host machine remains completely untouchable, allowing the team to iterate with absolute freedom.

### 1.3 The Hybrid Strategic Balance (16GB RAM Footprint Correction)

This design deliberately optimizes for the hardware constraints of a 16 GB M1 system. Rather than overloading the Mac's unified memory with a heavy local model, the system executes a precise memory budget:

* **Local Layer:** Runs a highly compressed, tool-efficient 4-parameter model (**Qwen 3.5 4B**). It requires only **~3.4 GB of VRAM**, leaving massive system memory headroom for the macOS base layer and the container runtime.
* **Cloud Layer:** Automatically cascades out to specialized cloud APIs only when intensive processing (large document intake, advanced multi-file coding, image rendering) is triggered by explicit user commands.

---

## 2. Technical Topology & Model Routing

To give your team complete freedom to experiment without freezing your local hardware or blowing a hole in your wallet, the assistant routes operations across three distinct infrastructure layers:

```
                  [ Incoming Telegram Command ]
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
       [ Heavy Task? ]                 [ Everyday Task? ]
               │                               │
        (Route to Cloud)                (Process Locally)
               │                               │
 ┌─────────────┴─────────────┐         ┌───────┴───────┐
 ▼                           ▼         ▼               ▼
[Coding/Prototyping]    [Asset Gen]  [Chat Summary]  [Wiki Update]
   (Gemini 3.5)       (GPT Image Mini) (Qwen 3.5 4B) (Quartz 5 Build)

```

### The Multi-Tier Routing Model

| Task Type | Target Model / Engine | Host Environment | Core Performance Metric |
| --- | --- | --- | --- |
| **Chat Parsing & Wiki Updates** | **Qwen 3.5 4B** (Q4_K_M ~3.4GB) | Local (Ollama + Metal) | **$0.00** (Free, 100% Private) |
| **Massive File Ingestion (Audio/PDF)** | **Gemini 3.1 Flash-Lite** | Cloud API | **Virtually Free** ($0.25 / 1M tokens) |
| **Web Prototyping & App Building** | **Gemini 3.5 Flash** | Cloud API | **Highly Efficient** ($1.50 / 1M tokens) |
| **Asset & Image Generation** | **GPT Image 1 Mini** | Cloud API | **Fixed Cost** ($0.005 / image) |

---

## 3. Step-by-Step Mac Mini Setup Script

Log onto the brand-new Mac Mini and execute the following deployment phases sequentially.

### Phase 1: Native Inference Base Setup

We run **Ollama** natively at the macOS host level to ensure zero translation loss when hitting the Apple Silicon GPU via Metal.

1. Download and open the official Ollama for Mac application.
2. In the terminal, pull down the lightweight, high-context Qwen 3.5 model:
```bash
ollama pull qwen3.5:4b

```
3. Set the system variables to preserve memory stability and prevent aggressive unloading over group chat lulls:
   ```bash
   echo 'export OLLAMA_NUM_PARALLEL=2' >> ~/.zshrc
   echo 'export OLLAMA_KEEP_ALIVE="24h"' >> ~/.zshrc
   source ~/.zshrc

```

### Phase 2: Create Local Directory Structure

Establish the baseline directories on the Mac Mini to host configuration profiles, active docker runtimes, the persistent Quartz cache layer, and your raw Markdown files.

```bash
mkdir -p ~/info-butler/wiki ~/info-butler/config
cd ~/info-butler

```

### Phase 2.5: Docker Runtime (Mac-Specific)

With only 16 GB of unified memory, choosing the right Docker backend matters. Every background process competes with Ollama and your containers for RAM.

| Option | Idle RAM | Apple Silicon | Cost | Verdict |
|---|---|---|---|---|
| **Colima** (recommended) | ~300–500 MB | ✅ Native (Virtualization.framework) | Free | Best balance of perf, memory, and price |
| **OrbStack** | ~200–400 MB | ✅ Native | $10/mo | Slightly faster, has GUI, paid |
| **Docker Desktop** | ~1–2 GB | ✅ Native | Free (personal) | Heavy — 1–2 GB overhead hurts on 16GB |
| `brew install docker` alone | 0 | N/A | Free | ❌ CLI only — can't actually run containers |

#### Setup: Colima (recommended)

```bash
brew install docker docker-compose colima

# Start the VM with a tight resource budget
colima start --cpu 4 --memory 8 --disk 60 --vm-type=vz --mount-type=virtiofs
```

- `--vm-type=vz` uses Apple's native `Virtualization.framework` (faster, lower overhead than QEMU)
- `--mount-type=virtiofs` provides near-native filesystem performance between macOS and the VM
- `--memory 8` gives the VM 8 GB, leaving the remaining ~8 GB for macOS and Ollama

Verify it works:

```bash
docker info          # should show Colima as runtime
docker compose version  # should be v2+
```

> **Note:** `brew install docker` alone is just the CLI client with no runtime — don't stop there. You need Colima or OrbStack to actually run containers.

### Phase 3: The Hardened Multi-Container Orchestration Blueprint

A pre-built `docker-compose.yml` exists in the repo at `~/info-butler/docker-compose.yml`. The configuration below constrains the Hermes container to **4 CPU cores and 6 GB of RAM** — with the Colima VM capped at 8 GB total across all containers, leaving ~6 GB headroom for macOS and Ollama after the Qwen 3.5 4B model (~3.4 GB) is loaded.

> **Design note:** The model was switched from Qwen 3.5 9B (6.6 GB) to 4B (3.4 GB) after profiling showed the 9B + Docker + macOS exceeded 16 GB of unified memory, causing kernel OOM kills under load. With Colima capped at 8 GB and the Qwen 4B model at ~3.4 GB, the total system budget sits at roughly 15 GB — tight but stable.

```yaml
# File: ~/info-butler/docker-compose.yml
# (edit the file directly — this snippet tracks the repo version)
version: '3.8'

services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-butler
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 6g
    ports:
      - "9119:9119"   # Admin Dashboard
    environment:
      # Telegram bot token (get from @BotFather)
      - TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here
      # Authorized user IDs (get from @userinfobot)
      - TELEGRAM_ALLOWED_USERS=user_id_1,user_id_2
      # Restrict to specific group chat (get ID from web.telegram.org URL)
      - TELEGRAM_GROUP_ALLOWED_CHATS=-1001234567890
      # API keys
      - GEMINI_API_KEY=your_google_gemini_api_key_here
      - OPENAI_API_KEY=your_openai_api_key_here
      # Hermes config
      - HERMES_DASHBOARD=1
      - OLLAMA_HOST=http://host.docker.internal:11434
    volumes:
      - ./wiki:/opt/data/wiki
      - ./config:/opt/data/config
    extra_hosts:
      - "host.docker.internal:host-gateway"

  quartz-visualizer:
    image: node:22-alpine
    container_name: quartz-wiki
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./wiki:/usr/src/app/content
    working_dir: /usr/src/app
    command: >
      sh -c "if [ ! -d '.git' ]; then
               apk add --no-cache git &&
               git clone https://github.com/jackyzha0/quartz.git . &&
               npm ci &&
               npx quartz create --action copy --link content;
             fi &&
             npx quartz build --serve --port 8080"
```

#### Credential setup walkthrough

1. **Create a bot:** Message [@BotFather](https://t.me/BotFather), send `/newbot`, pick a name and username. Save the API token — that's `TELEGRAM_BOT_TOKEN`.
2. **Get user IDs:** Each authorized person messages [@userinfobot](https://t.me/userinfobot) — it replies with their numeric ID. Add them to `TELEGRAM_ALLOWED_USERS`.
3. **Get the group chat ID:** Open your group in [web.telegram.org](https://web.telegram.org). The URL contains `#-1001234567890` — that number is `TELEGRAM_GROUP_ALLOWED_CHATS`.
4. **Disable privacy mode (required for group listening):** In @BotFather, go to `/mybots` → your bot → Bot Settings → Group Privacy → Turn off. Then **remove and re-add** the bot to your group (Telegram caches privacy on join).
5. **Set API keys:** `GEMINI_API_KEY` from [Google AI Studio](https://aistudio.google.com), `OPENAI_API_KEY` from [platform.openai.com](https://platform.openai.com).

> ⚠️ If you accidentally expose your bot token (as happened during setup), revoke it immediately via @BotFather → `/mybots` → your bot → API Token → Revoke. Then update `TELEGRAM_BOT_TOKEN` in `docker-compose.yml` with the new one.

---

## 4. The Brain Pattern: Custom Skill & Profile Wiring

> ⚠️ **Implementation Note on Agent Logic:** Hermes Agent does not automatically scan raw Markdown text files to discover model routing architectures. To implement the "Living Document" design, you must map these parameters directly into the **Hermes System Prompt Profile YAML** and code custom actions using the Python core.

### 4.1 Step 1: The Master Profile Setup

Create a custom configuration profile file inside `~/info-butler/config/profile.yaml`. This injects your strategic routing rules straight into Hermes' system prompt instructions:

```yaml
# Save this to ~/info-butler/config/profile.yaml
profile_name: "info_butler"
system_prompt: |
  You are the designated Project Info Butler. You manage a directory of interconnected, non-linear Markdown files located inside `/opt/data/wiki`.
  
  CORE TARGETS:
  - You possess native access to the local file engine tools (read_file, write_file).
  - Every time you document an idea or process a chat command, you must link it using double-bracket WikiLinks (e.g., [[Tech_Stack]] or [[Project_Milestones]]).
  - When asked to create an asset or a logo, you MUST call the external image generation tool (`openai/gpt-image-1-mini`).
  - When asked to run multi-file software engineering or prototype code, you MUST escalate execution to the cloud-linked `gemini-3.5-flash` engine.

```

### 4.2 Step 2: The Automatic Backup Blueprint

The entire `~/info-butler` directory is already tracked as a git repository pushed to GitHub. For hourly automatic snapshots:

1. Open your host terminal and run `crontab -e`.
2. Add the following entry:
```text
0 * * * * cd ~/info-butler && git add -A && git commit -m "Auto-snapshot $(date)" && git push
```

> The `.gitignore` excludes auto-generated runtime content under `wiki/` and `node_modules/`. Only source files (config, docs, compose files) are versioned.



```

---

## 5. Remote Workspace Access & Topology

To enable fluid collaboration from outside your home network without exposing root server credentials or administrative ports to the public internet, we utilize a bifurcated routing topology:

1. **The Admin Gateway (Hermes Dashboard):** Completely locked down. Accessible only through an encrypted **Cloudflare Tunnel** wrapped in Cloudflare Access (requiring a one-time email PIN login). Teammates can connect their local native Hermes Desktop Apps directly to this remote secure endpoint by referencing your custom secure domain (`https://admin.yourdomain.com`).
2. **The Living Wiki (Quartz 5 Visualizer):** Rendered as a read-only, interactive web node. It is safe to expose port `8080` to a standard public network route or standard router forward so the entire team can browse the evolving project notes, mind-maps, and bi-directional backlinked graphs.

> **Setup note:** Cloudflare Tunnel + Access require a registered domain, a Cloudflare account, and `cloudflared` installed on the Mac Mini. See the [Cloudflare Tunnel docs](https://developers.cloudflare.com/tunnel/setup/). This step is optional — for local-only access, ports `8080` and `9119` work fine on your home network.

---

## 6. System Verification & Expected Latencies

Execute the initialization command to launch the full pipeline:

```bash
cd ~/info-butler
docker compose up -d

```

### Verification Steps:

1. Confirm the Telegram gateway initialized:
   ```bash
   docker logs hermes-butler 2>&1 | grep -i "telegram"
   ```
2. Send a message to the bot in your group (e.g. `/help`). Check delivery:
   ```bash
   curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates" | python3 -m json.tool
   ```
3. Test authorization — an unlisted user's messages should be silently dropped.
4. Test visual linking: `@YourBotName link [[Database_Architecture]] to [[M1_Mac_Hosting]]`
5. **Latency Expectation:** Quartz is a static-batch site generator. A new node typically appears on the visual graph within **15–30 seconds** after the agent writes the file.

### Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `getUpdates` returns `[]` | Bot hasn't received any messages | Send `/start` in the group, retry |
| Bot works in DMs but silent in group | Privacy mode is on | Disable in @BotFather, remove & re-add bot |
| `Additional property memory is not allowed` | `memory` at service level (v3.8) | Nest under `deploy.resources.limits` |
| Container can't reach Ollama | `host.docker.internal` unreachable | Verify Ollama is running; check `OLLAMA_HOST` |