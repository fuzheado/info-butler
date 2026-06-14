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

### Phase 3: The Hardened Multi-Container Orchestration Blueprint

Create your `docker-compose.yml` file to bind the running agent to your Mac's host gateway.

We constrain the Hermes container to exactly **2 CPU cores and 2 GB of RAM**—freeing up a massive 10 GB of system memory headroom to guarantee the host OS never experiences kernel out-of-memory crashes.

```yaml
# Save this file to ~/info-butler/docker-compose.yml
version: '3.8'

services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-butler
    restart: unless-stopped
    cpus: "2.0"       # Resource constraint to guarantee macOS system stability
    memory: "2g"      # Lowered memory footprint tailored exactly for 16GB Mac Mini
    ports:
      - "9119:9119"   # Admin Dashboard WebSocket Connection
      - "8642:8642"   # OpenAI-Compatible API Server Endpoint
    environment:
      - TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here
      - GEMINI_API_KEY=your_google_gemini_api_key_here
      - OPENAI_API_KEY=your_openai_api_key_here
      - GATEWAY_ALLOW_ALL_USERS=false   # Hardened security configuration
      - HERMES_AUTH_USERS=your_tg_id_1,partner_tg_id_2,partner_tg_id_3 # Comma-separated list of authorized IDs
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
      - "8080:8080"  # Publicly exposed portal to view the visual mind-map graph
    volumes:
      - ./wiki:/usr/src/app/content
      - quartz_node_modules:/usr/src/app/node_modules # Persistent volume prevents re-cloning dependencies on reboot
    working_dir: /usr/src/app
    command: >
      sh -c "if [ ! -d '.git' ]; then
               apk add --no-cache git &&
               git clone https://github.com/jackyzha0/quartz.git . &&
               npm ci &&
               npx quartz create --action copy --link content;
             fi &&
             npx quartz build --serve --port 8080"

volumes:
  quartz_node_modules: # Native caching persistence layer

```

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

To prevent volume corruption or database loss, use the native macOS `cron` engine to automatically take snapshots of your team's visual wiki data on your Mac Mini host.

1. Open your host terminal and run `crontab -e`.
2. Add the following entry to commit and push changes silently every single hour:
```text
0 * * * * cd ~/info-butler/wiki && git init && git add . && git commit -m "Automated Butler Snapshot: $(date)"

```



```

---

## 5. Remote Workspace Access & Topology

To enable fluid collaboration from outside your home network without exposing root server credentials or administrative ports to the public internet, we utilize a bifurcated routing topology:

1. **The Admin Gateway (Hermes Dashboard):** Completely locked down. Accessible only through an encrypted **Cloudflare Tunnel** wrapped in Cloudflare Access (requiring a one-time email PIN login). Teammates can connect their local native Hermes Desktop Apps directly to this remote secure endpoint by referencing your custom secure domain (`[https://admin.yourdomain.com](https://admin.yourdomain.com)`).
2. **The Living Wiki (Quartz 5 Visualizer):** Rendered as a read-only, interactive web node. It is safe to expose port `8080` to a standard public network route or standard router forward so the entire team can browse the evolving project notes, mind-maps, and bi-directional backlinked graphs.

---

## 6. System Verification & Expected Latencies

Execute the initialization command to launch the full pipeline:

```bash
cd ~/info-butler
docker compose up -d

```

### Verification Steps:

1. Confirm logs show the Telegram Bot Gateway initialized and authenticated successfully: `docker logs hermes-butler`.
2. Test tool authorization constraints by having an unauthorized group member attempt to message the bot. It should gracefully drop the prompt.
3. Test your visual linking: Drop a message in your chat: `@YourBotName update our technical requirements to link [[Database_Architecture]] to [[M1_Mac_Hosting]].`
4. **Latency Expectation Note:** Refresh your browser window at `http://localhost:8080`. Because Quartz uses a static-batch file builder engine to index connections, the new node will materialize on your D3 force-directed network graph within **15 to 30 seconds**—allowing your team to seamlessly trace the project's evolution as your notes continuously grow.