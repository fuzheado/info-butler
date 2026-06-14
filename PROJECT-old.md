# Project Specification: "The Info Butler"

**Target Hardware:** Mac Mini M1 (16 GB Unified Memory)

**Deployment Objective:** A 24/7 sandboxed Telegram Group Chat Agent serving as an info-butler, code writer, and continuous curator of a non-linear, interconnected visual "Living Wiki" repository.

## 0. Design philosophy
### Motivation
Modern project mobilization often suffers from "chat decay." When a team of distributed collaborators brainstorms across various topics in a chaotic Telegram group, valuable insights, architectural decisions, and sudden sparks of inspiration are buried under hundreds of messages of day-to-day noise. Traditional knowledge management tools (like corporate wikis or linear documents) fail because they require friction—someone has to manually stop brainstorming, open a separate app, and file a rigid entry.

The **Info Butler** architecture changes this dynamic by turning the group chat itself into a live, self-documenting workspace. By placing an autonomous agent inside the conversation as an omnipresent participant, the team can crowdsource project infrastructure entirely via ambient commands.

### Open Playground & Hardened Sandbox
To maximize creative velocity and encourage bold engineering experiments, this system operates on a high-trust model: any collaborator in the group chat can command the butler to perform root-level operations.

Because the system is deployed entirely within an isolated Docker runtime environment on dedicated, local Apple Silicon hardware, this radical open access is perfectly safe. If a collaborator intentionally or accidentally triggers an optimized script that results in resource exhaustion or a critical system error, the blast radius is strictly trapped inside a ephemeral Linux sandbox. The underlying host machine remains completely untouchable, allowing the team to "move fast and break things" with absolute freedom.

---

## 1. System Architecture Overview

To allow external collaborators to run intense code experiments without blocking the 16 GB hardware footprint or exhausting system memory, the butler routes operations across three distinct infrastructure layers:

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
   (Gemini 3.5)        (GPT Image Mini) (Qwen 3.5 9B) (Quartz 5 Build)

```

### The Multi-Tier Routing Model

| Task Type | Target Model / Engine | Host Environment | Core Performance Metric |
| --- | --- | --- | --- |
| **Chat Parsing & Wiki Updates** | **Qwen 3.5 9B** (4-bit Quant) | Local (Ollama + Metal) | **$0.00** (Free, 100% Private) |
| **Massive File Ingestion (Audio/PDF)** | **Gemini 3.1 Flash-Lite** | Cloud API | **Virtually Free** ($0.25 / 1M tokens) |
| **Web Prototyping & App Building** | **Gemini 3.5 Flash** | Cloud API | **Highly Efficient** ($1.50 / 1M tokens) |
| **Asset & Image Generation** | **GPT Image 1 Mini** | Cloud API | **Fixed Cost** ($0.005 / image) |

---

## 2. Remote Workspace Access & Topology

To enable fluid collaboration from outside your home network without exposing root server credentials or administrative ports to the public internet, we utilize a bifurcated routing topology:

```
[ Team Laptop ] ───► (Secure Domain via Cloudflare Access) ───► [ Cloudflare Tunnel ] ───► [ Hermes Admin Port 9119 ]
[ Public View ] ───► (Public Web Browser URL) ───────► [ Port Forward / HTTP Tunnel ] ───► [ Quartz Wiki Port 8080 ]

```

1. **The Admin Gateway (Hermes Dashboard):** Completely locked down. Accessible only through a **Cloudflare Tunnel** wrapped in Cloudflare Access (requiring a one-time email PIN login). Teammates can connect their local native Hermes Desktop Apps directly to this remote secure endpoint.
2. **The Living Wiki (Quartz 5 Visualizer):** Rendered as a read-only, interactive web node. It is safe to expose to a standard public domain so the entire team can browse the evolving project notes, mind-maps, and backlinked graphs in real-time.

---

## 3. Step-by-Step Mac Mini Setup Script

Log onto the brand-new Mac Mini and execute the following deployment phases sequentially.

### Phase 1: Native Inference Base Setup

We run **Ollama** natively at the macOS host level to ensure zero translation loss when hitting the Apple Silicon GPU via Metal.

1. Download and open the official Ollama for Mac application.
2. In the terminal, pull down the high-context, multi-modal Qwen model:
```bash
ollama pull qwen3.5:9b

```

3. Pin the model memory state and set parallel limits to prevent aggressive unloading over group chat lulls:
   ```bash
   echo 'export OLLAMA_NUM_PARALLEL=2' >> ~/.zshrc
   echo 'export OLLAMA_KEEP_ALIVE="24h"' >> ~/.zshrc
   source ~/.zshrc

```

### Phase 2: Create local directory structure

Establish directories to host configuration profiles, active docker runtimes, and the Quartz workspace environment.

```bash
mkdir -p ~/info-butler/wiki ~/info-butler/config
cd ~/info-butler

```

### Phase 3: The Multi-Container Orchestration Blueprint

Create your `docker-compose.yml` file to bind the running agent to your Mac's host gateway.

We constrain the execution runtime container to exactly **4 CPU cores and 6 GB of RAM** so an unintended infinite loop or complex test script written by the AI can never lock up or freeze the host operating system.

```yaml
# Save this file to ~/info-butler/docker-compose.yml
version: '3.8'

services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-butler
    restart: unless-stopped
    cpus: "4.0"
    memory: "6g"
    ports:
      - "9119:9119" # Exposes Admin Dashboard WebSockets for the Remote Desktop App
    environment:
      - TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here
      - GEMINI_API_KEY=your_google_gemini_api_key_here
      - OPENAI_API_KEY=your_openai_api_key_here
      - GATEWAY_ALLOW_ALL_USERS=true
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
      - "8080:8080" # Exposes the browseable, interactive mind-map website
    volumes:
      - ./wiki:/usr/src/app/content # Binds Quartz content directory directly to the folder Hermes edits
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

---

## 4. The Brain Pattern: Non-Linear LLM-Wiki Config

To prime the agent to drop hierarchical filing rules in favor of a network-linked graph design, initialize your directory file tree:

```bash
touch ~/info-butler/wiki/AGENTS.md
touch ~/info-butler/wiki/Index.md

```

### The System Directive (`AGENTS.md`)

Paste this text inside `~/info-butler/wiki/AGENTS.md`. The Hermes core scans this file to dictate how it structures text and matches internal cross-linking schemas.

```markdown
# System Directive: The Networked Digital Garden Pattern

You are the designated Project Info Butler. You manage a directory of interconnected, non-linear Markdown files located inside `/opt/data/wiki`.

## Core Documentation Protocols

1. Bi-Directional Linking: Do not force files into rigid subfolders. Every time you document a new concept, decision, or user ideation, you must link it back to relevant concepts using standard Markdown WikiLinks (e.g., [[Tech_Stack]] or [[Project_Milestones]]).
2. Emergent Structure: Maintain a global visual mind-map. When processing text from the group, actively analyze links to see where clusters are forming, and summarize connections inside [[Index.md]].
3. Automated Graph Maintenance: Every time a collaborator states "Save this connection" or "Link this concept to our roadmap", you must update both nodes and write explicit contextual notes explaining *why* they link together.

## Execution Routing Metrics
- Routine conversation curation: Process via local Qwen 3.5 9B.
- Image production request (/draw): Route to `openai/gpt-image-1-mini` ($0.005/img).
- Detailed Code Generation / Web Prototype loops: Delegate execution to `gemini-3.5-flash`.
- Deep Context File Parsing (Video recordings/Audios/PDFs): Delegate processing to `gemini-3.1-flash-lite`.

```

---

## 5. Deployment & System Verification

Execute the initialization command to launch the full pipeline:

```bash
cd ~/info-butler
docker compose up -d

```

### How to verify your new setup:

1. **The Telegram Connection:** Check container outputs to ensure the Telegram Gateway is polling successfully: `docker logs hermes-butler`. Tag your bot in your group chat to verify responsiveness.
2. **The Quartz Live Graph:** Open a local web browser on your home network and navigate to `http://localhost:8080`. You will see Quartz auto-generate your visual index and render its graphical sidebar.
3. **The Living Sync Verification:** In your Telegram chat, tag the bot:
> *"@Butler create a note on [[Vibe_Coding_Explorations]] and link it back to our main index."*


Within seconds, refresh your browser tab at port 8080. You will visually see the new node materialize on your interactive project mind-map graph, linked exactly where instructed.