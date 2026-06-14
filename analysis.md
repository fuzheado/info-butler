# Project Analysis: "The Info Butler"

> **Note on methodology:** Per the project's own `AGENTS.md` directive, I verified all factual claims about tool/model existence and pricing against live sources before writing this analysis. The findings may surprise you — most claimed models actually *do* exist as of June 2026.

---

## Executive Summary

This is an ambitious, genuinely well-researched project spec that assembles real, existing tools into an integrated system. The core idea — an ambient group-chat agent that captures knowledge and renders it as a visual wiki — solves a real problem ("chat decay"). However, several architectural assumptions and missing details make this more of a **vision document** than a buildable blueprint. The gaps are bridgeable, but non-trivial.

---

## ✅ What the Spec Gets Right

### 1. All referenced models and tools are real

Every single tool, model, and service name in the spec exists in production as of June 2026:

| Claim | Reality | Verdict |
|---|---|---|
| `nousresearch/hermes-agent` on Docker Hub | 1M+ pulls, active Telegram/Discord/Slack gateway | ✅ Real |
| Hermes Dashboard on port 9119 | Confirmed — dashboard docs use port 9119 | ✅ Real |
| Hermes API on port 8642 | Confirmed in Hermes Docker docs | ✅ Real |
| `Qwen 3.5 9B` on Ollama | Available at `qwen3.5:9b`, Q4_K_M ~6.6GB | ✅ Real |
| `Gemini 3.5 Flash` | Real model, $1.50/$9.00 per 1M in/out tokens | ✅ Real |
| `Gemini 3.1 Flash-Lite` | Real model, $0.25/$1.50 per 1M in/out tokens | ✅ Real |
| `gpt-image-1-mini` | Real OpenAI model, $0.005/1024² low quality | ✅ Real |
| `jackyzha0/quartz` v4/v5 | 12k+ stars, active, Obsidian-compatible | ✅ Real |

**This is surprisingly accurate for a spec of this breadth.** The model names aren't hallucinated — they're the current (2026) lineup.

### 2. The "chat decay" problem statement is excellent

The opening motivation is the strongest part of the document. The observation that valuable insights get buried under day-to-day noise in group chats is real, and the insight that "traditional knowledge management requires too much friction" is correct. An ambient agent inside the conversation is a legitimate architectural answer.

### 3. Multi-tier routing is architecturally sound

Routing chat parsing to a free local model (Qwen), heavy lifts to cheap cloud APIs (Gemini Flash-Lite), and code generation to more capable models is the right pattern. It optimizes for:
- **Latency** — local models respond instantly for routine tasks
- **Cost** — cloud spend is concentrated on high-value work
- **Privacy** — most conversation processing stays on-device

### 4. Docker isolation for "move fast and break things"

Sandboxing the agent (4 CPU cores, 6GB RAM) is the correct approach for a system where group chat participants can issue arbitrary commands. The blast-radius argument is sound.

### 5. Bi-directional linking as a knowledge paradigm

The "networked digital garden" model (WikiLinks, Index.md, emergent graph structure) is a proven approach (Obsidian, Roam Research, Foam). It fits the use case better than a hierarchical wiki.

---

## ❌ What's Wrong or Missing

### 🔴 Critical Issues

#### 1. Memory math doesn't add up

The Mac Mini M1 has **16GB unified memory**. Here's the real budget:

| Component | Memory Estimate |
|---|---|
| macOS (Ventura/Sonoma) | ~3-4 GB |
| Ollama + Qwen 3.5 9B (Q4_K_M) | ~6.6 GB |
| Docker container (6 GB reservation) | Up to 6 GB |
| **Total** | **~15.6-16.6 GB** |

There is **zero headroom**. The system will thrash swap constantly, or the kernel OOM killer will terminate processes under load. If Hermes spawns sub-processes, runs Python code, or Quartz builds the site while Ollama is loaded — something crashes.

**Worse:** The 6GB Docker container + Ollama's 6.6GB = 12.6GB before macOS has taken a single byte. This isn't "tight" — it's physically exceeding the hardware limit under moderate load.

#### 2. The "AGENTS.md system directive" is not how Hermes works

The spec describes a mechanism where Hermes "scans AGENTS.md to dictate how it structures text and matches cross-linking schemas." This doesn't exist. Hermes Agent does not have a built-in directive-scanning behavior for a file called `AGENTS.md`. The actual Hermes configuration works through:

- **Profiles** — YAML/JSON configuration files
- **Skills** — Python/TypeScript skill files the agent loads
- **System prompt** — you can set a custom system prompt

The behavior described (model routing, wiki maintenance, link management) would need to be implemented as **custom Hermes skills** and/or **a carefully crafted system prompt**. It's not automatic. The spec skips over this entirely.

#### 3. Quartz is a batch static-site generator, not a real-time graph

The spec promises "within seconds, you'll see the new node materialize on your interactive project mind-map." But Quartz is a **static site generator** — it runs `npx quartz build` to regenerate the full HTML/CSS/JS output. For a site of any size, this takes 10-60 seconds *minimum*. The graph visualization (D3.js force-directed graph) is rendered client-side from a JSON index, so:

1. Agent writes a new `.md` file
2. Quartz detects the file change
3. Quartz rebuilds the entire site
4. User refreshes browser (or live-reload triggers)
5. Graph re-renders from fresh data

This is not "near-real-time." It's a polling loop with a multi-second build step. For a "living wiki" the spec envisions, this feels wrong.

#### 4. The bot logic itself is entirely unspecified

The spec describes the topology around the bot beautifully, but **the bot's actual behavior is undefined**:

- How does Hermes know to listen to *this specific Telegram group*?
- How does it parse "Save this connection \[\[X\]\] → \[\[Y\]\]"?
- How does it decide which model to route to? (The system directive mentions routing metrics, but these aren't configured anywhere)
- How do the wiki file writes work? (File paths, conflict resolution, atomicity)
- How does it handle concurrent requests from multiple group members?

Hermes Agent can do Telegram bridge + gateway mode, but wiring it to produce the described behavior requires significant custom engineering.

#### 5. No backup or durability story

- Docker volumes (`./wiki`, `./config`) live on the Mac's filesystem. No snapshots, no git pushes, no offsite backup.
- Hermes uses an internal SQLite database for conversations and skills. If the volume corrupts, all agent memory is lost.
- The spec is entirely stateless in its description of the wiki content.

### 🟡 Moderate Concerns

#### 6. Cloudflare Tunnel setup is hand-waved

The network topology diagram is pretty, but setting up Cloudflare Tunnel + Cloudflare Access requires:
- A registered domain (not mentioned)
- A Cloudflare account (free tier works)
- Installing and authenticating `cloudflared`
- Creating the tunnel
- Configuring Access policies (email OTP, device posture, etc.)
- DNS configuration

None of this is in the setup script. A "one-time email PIN login" requires actually configuring Cloudflare Access, which has its own learning curve.

#### 7. Cold start and reboot behavior

- `OLLAMA_KEEP_ALIVE="24h"` keeps Qwen in memory, but what happens on system restart?
- Docker's `restart: unless-stopped` handles container restarts, but the containers depend on:
  - Docker daemon being up
  - Ollama being loaded into memory (takes ~15-20s after startup)
  - Hermes needing the Ollama endpoint (`host.docker.internal:11434`) to be ready
- No health checks, no startup ordering, no graceful degradation.

#### 8. "Any collaborator can run root-level operations"

The spec presents this as a feature ("high-trust model"), but it's a double-edged sword:
- Hermes Agent's gateway supports tool execution — this means any Telegram user the bot trusts can run shell commands, file operations, etc.
- While Docker limits the blast radius *to the host*, it doesn't limit damage *to the container* — someone could `rm -rf /opt/data/wiki`, install cryptominers, exfiltrate API keys stored in environment variables, etc.
- `GATEWAY_ALLOW_ALL_USERS=true` is particularly dangerous. Hermes supports per-user authorization, and this setting bypasses it entirely.

#### 9. Ollama natively on macOS vs. in Docker

The spec runs Ollama natively on macOS (not in Docker) for "zero translation loss to Metal." This is reasonable, but:
- Mixing native and containerized runtimes adds operational complexity
- Docker networking (host.docker.internal) adds latency and potential failure modes
- If Hermes talks to Ollama over the network, each request has HTTP overhead + serialization cost

### 🟢 Minor Issues

#### 10. Hermes port mapping is incomplete

The spec maps port 9119 but not port 8642 (the API server). If someone wants to use the Hermes OpenAI-compatible API, they can't reach it.

#### 11. Quartz build caching

The docker-compose runs `npx quartz create --action copy --link content` on every container restart if `.git` doesn't exist. This re-clones Quartz from scratch. Better to use a persistent node_modules volume or a pre-built image.

#### 12. No mention of Telegram webhook vs polling

Hermes supports both. For a 24/7 agent on a home Mac, **polling** is simpler (no public endpoint needed). But the Cloudflare Tunnel setup suggests **webhooks**, which means Telegram needs to reach the bot. This requires the tunnel to be up 24/7 and introduces another dependency.

---

## 💡 Alternatives & Enhancements

### Alternative Approaches

#### A. Replace Quartz with a real-time graph tool

| Option | Why |
|---|---|
| **Obsidian + Local REST API** | Obsidian's community plugin "Local REST API" lets you create/update notes programmatically. The graph view updates in real-time. No build step. |
| **Foam + VS Code** | Similar to Obsidian but open-source and VS Code-based. Lighter weight. |
| **TiddlyWiki** | A single-file wiki that can be edited via API. Not as pretty but far simpler to integrate. |
| **Custom SPA (React + D3/Force-Graph)** | Maximum control. A lightweight server watches the filesystem and pushes updates via WebSocket to a browser graph. Eliminates the static-site rebuild bottleneck entirely. |

#### B. Use a dedicated middleware instead of shoehorning Hermes

Rather than making Hermes Agent do everything (Telegram bridge + model routing + wiki management + code execution), consider:

```
Telegram ←→ [Python Middleware] ←→ Hermes (for heavy tasks)
                    ↓
              [Wiki Writer] → Quartz/Obsidian
                    ↓
              [Local Qwen] (for chat parsing)
```

A lightweight Python or Node.js service handles:
- Receiving Telegram messages (via python-telegram-bot or Telegraf)
- Classifying intent (chat summary vs. wiki update vs. code request)
- Routing to the appropriate model (local Qwen via Ollama API, or Hermes via its API)
- Writing wiki files with proper frontmatter, WikiLinks, and Index.md updates
- Emitting WebSocket events for a live graph client

This would be 200-400 lines of code and far more maintainable than trying to bend Hermes into this shape.

#### C. Consider Obsidian Publish or GitHub Pages instead of self-hosting Quartz

- **Obsidian Publish** ($10/mo) — instant graph view, no build step, password protection
- **GitHub Pages + a simpler static generator** — MkDocs with RoamLinks plugin, or Material for MkDocs. Cheaper cognitive overhead.

### Enhancements (if keeping the current architecture)

#### 1. Solve the memory crisis

Options (in order of preference):
- **Use a smaller local model:** Qwen 3.5 4B or Qwen 3.5 0.8B for chat parsing. Reserve the 9B only for complex reasoning.
- **Reduce Docker memory reservation** from 6GB to 4GB (Hermes doesn't need 6GB for Telegram + routing)
- **Use Ollama in Docker** with memory limits, so the kernel can manage under pressure
- **Add swap-aware health checks** — if memory usage exceeds 85%, gracefully degrade to cloud-only mode

#### 2. Implement the bot logic as Hermes skills

Hermes Agent supports custom skills. You'd create skills for:
- `wiki-write` — takes a note title and content, writes markdown with frontmatter and WikiLinks
- `wiki-link` — updates Index.md and bidirectional links
- `model-router` — checks message content and decides local vs. cloud
- `code-executor` — sandboxed code execution for prototyping

These are actual Hermes skill files (Python), not markdown configuration.

#### 3. Add a proper backup strategy

```bash
# In the docker-compose or a cron job
0 */6 * * * cd ~/info-butler && git add wiki/ && git commit -m "auto-snapshot $(date)" && git push
```

Or use `rsync` to a remote backup target. The wiki content is just markdown files — there's no excuse for not having version history.

#### 4. Add startup health checks and graceful degradation

```
hermes-agent:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8642/health"]
    interval: 30s
    retries: 3
  depends_on:
    ollama:
      condition: service_healthy
```

#### 5. Use Hermes' per-user authorization instead of `GATEWAY_ALLOW_ALL_USERS`

```yaml
HERMES_AUTH_USERS: "alice:telegram_id_123,bob:telegram_id_456"
GATEWAY_ALLOW_ALL_USERS: "false"
```

This lets you control who can issue commands to the bot while still keeping the group chat open for reading.

#### 6. Replace "seconds" claims with realistic expectations

Instead of promising near-real-time graph updates, describe the actual pipeline:
- Wiki file written → ~1-2s
- Quartz rebuild detected → ~5-30s (depending on site size)
- Browser live-reload → ~1-2s
- Total: **~10-35s** for a node to appear on the graph

This is still impressive! No need to oversell it.

#### 7. Add a local-only fallback mode

What happens when the Mac is offline (home Internet outage, Cloudflare down)? The system should gracefully fall back to:
- Local Qwen for all processing
- No Quartz serving (or a cached version)
- Queue Gemini/GPT requests for retry

---

## Verdict

| Dimension | Score | Notes |
|---|---|---|
| **Problem diagnosis** | ★★★★★ | Chat decay is real, frictionless capture is the right answer |
| **Tool selection** | ★★★★☆ | Surprisingly current and accurate for mid-2026 |
| **Architecture diagram** | ★★★★☆ | Multi-tier routing is sound; network topology is good |
| **Memory planning** | ★☆☆☆☆ | 16GB is insufficient for the proposed stack under load |
| **Implementation detail** | ★★☆☆☆ | The bot behavior itself is undefined; setup script skips critical steps |
| **Security model** | ★★★☆☆ | Docker isolation is good; `ALLOW_ALL_USERS` is reckless |
| **Durability/backup** | ★☆☆☆☆ | No backup strategy; single point of failure |
| **Honesty about trade-offs** | ★★☆☆☆ | "Seconds" for graph updates is optimistic; memory math isn't shown |

**Bottom line:** This is an excellent *vision document* that could become a working system with about 2-3 weeks of engineering effort. The biggest un-addressed risks are (1) memory pressure on the M1 16GB, (2) the absence of actual bot logic/skills to make Hermes do what the spec describes, and (3) the mismatch between Quartz's batch-build model and the promised "real-time" experience.

**Who this is good for:** A small team comfortable with Docker, basic Python/Node.js scripting, and willing to iterate on the agent configuration. Not suitable as a turnkey solution for non-technical teams.

**Should you build it?** Maybe. The vision is compelling. But start with a simpler prototype — a Python middleware + local Qwen + Obsidian — before committing to the full Hermes + Quartz + Cloudflare stack. Prove the core loop works, then add complexity.
