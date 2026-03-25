# Deploy OpenClaw

## What is this?

An **AI Skill** that automatically deploys an [OpenClaw](https://github.com/openclaw/openclaw) AI Bot on [Cloudflare](https://cloudflare.com).

OpenClaw is an open-source AI agent that runs on Cloudflare's edge network. This skill packages everything needed to go from zero to a working AI bot — including infrastructure setup, deployment, and Telegram integration.

The key file is **`SKILL.md`** — a detailed instruction set designed for AI agents (Claude Code, Claude Desktop, OpenClaw) to read and execute autonomously.

## What can it do?

- **Deploy a Telegram AI Bot** — chat with your own AI assistant in Telegram
- **Set up a Web Dashboard** — monitor and control your bot from a browser
- **Auto-install everything** — Node.js, Docker, wrangler CLI, all from scratch
- **Auto-create cloud resources** — R2 storage, AI Gateway, Worker, Container
- **Auto-configure 14 secrets** — API keys, tokens, environment variables
- **Auto-pair Telegram** — connect your bot to Telegram with pairing approval
- **Persist data** — conversations, config, and skills survive container restarts via R2 sync
- **Run on Cloudflare's edge** — low latency, global availability, serverless scaling

## Who is this for?

Anyone who wants their own AI bot but doesn't want to deal with infrastructure. You don't need to know how to code, use a terminal, or understand cloud services. Just talk to your AI agent and it handles the rest.

## How it works

```
You:  "Deploy a new AI bot for me"
AI:   Installing tools... creating resources... deploying...
AI:   "Done! Send 'hi' to your bot on Telegram."
You:  "hi"
Bot:  "Hey! I'm your new AI assistant. How can I help?"
```

The AI reads `SKILL.md` and follows 10 steps:

| Step | What happens | Who does it |
|------|-------------|-------------|
| 0 | Check & install prerequisites | AI (auto-fix) |
| 1 | Get Cloudflare account info | AI |
| 2 | Copy source code | AI |
| 3 | Modify config (2 sed commands) | AI |
| 4 | Install dependencies | AI |
| 5 | Create cloud resources | AI + You (tokens) |
| 6 | Configure 14 secrets | AI |
| 7 | Build & deploy | AI |
| 8 | Set up Telegram webhook | AI |
| 9 | Pair Telegram + Dashboard | AI + You |

## What you need

- A **Mac** (macOS)
- A **Cloudflare account** with [Workers Paid Plan](https://dash.cloudflare.com/) ($5/month)
- **Telegram** installed on your phone or desktop

That's it. The AI installs everything else.

## Deployment flow

```mermaid
flowchart TD
    S0U1["Step 0.6 🔑 Create CF API Token<br/>(one-time setup)"]:::user --> S0A1["Step 0.1-0.4 Install tools<br/>brew, node, wrangler, docker"]:::auto
    S0U2["Step 0.7 🤖 Create Telegram Bot<br/>via BotFather"]:::user --> S1["Step 1 Get Account ID + Subdomain"]:::auto
    S0A1 --> S1
    S1 --> S2["Step 2 Copy source code"]:::auto
    S2 --> S3["Step 3 Modify files<br/>2 sed commands"]:::auto
    S3 --> S4["Step 4 Install dependencies<br/>npm install"]:::auto
    S4 --> S5AB["Step 5a-b Create R2 Bucket + AI Gateway"]:::auto
    S5AB --> S5C["Step 5c 🔑 Create AI Gateway Token<br/>in Dashboard"]:::user
    S5C --> S5D["Step 5d 🔑 Create R2 Token<br/>in Dashboard (optional)"]:::user
    S5D --> S6["Step 6 Configure 14 secrets"]:::auto
    S6 --> S7["Step 7 Build + Deploy<br/>3-5 minutes"]:::auto
    S7 --> S8["Step 8 Set Telegram webhook"]:::auto
    S8 --> S9W["Step 9 Wait for container<br/>1-3 minutes"]:::auto
    S9W --> S9U["Step 9 💬 Send 'hi' to bot<br/>share pairing code"]:::user
    S9U --> S9A["Step 9 Approve Telegram pairing"]:::auto
    S9A --> SF["Final ✅ Open Dashboard + Admin"]:::auto
    SF --> SFU["Final ✅ Click Approve<br/>in Admin panel"]:::user
    SFU --> Done["🎉 Your bot is live!"]:::done

    classDef user fill:#ffcdd2,stroke:#c62828,color:#000
    classDef auto fill:#c8e6c9,stroke:#2e7d32,color:#000
    classDef done fill:#bbdefb,stroke:#1565c0,color:#000
```

| | Count | What |
|---|---|---|
| 🔴 **You do** | 6 steps | Create tokens, send "hi", click Approve |
| 🟢 **AI does** | 11 steps | Install, build, deploy, configure, everything else |

## Architecture

```mermaid
flowchart TD
    subgraph Users["👤 Users"]
        TG["📱 Telegram"]
        WEB["🌐 Web Dashboard"]
    end

    subgraph CF["☁️ Cloudflare Global Network"]
        subgraph Worker["Worker (Moltworker)"]
            AUTH["Auth + Token check"]
            PROXY["Request proxy"]
            LIFECYCLE["Container lifecycle"]
        end

        subgraph Container["Container (OpenClaw)"]
            AGENT["🤖 AI Agent"]
            TOOLS["🔧 Tools + Skills"]
            TG_CH["📡 Telegram Channel"]
            GW["Gateway :18789"]
        end

        subgraph Storage["Storage"]
            R2["📦 R2 Bucket<br/>Config, sessions, workspace"]
            DO["💾 Durable Object<br/>Container state"]
        end

        subgraph AI["AI"]
            AIGW["🛡️ AI Gateway<br/>Logging, rate limiting, auth"]
            MODEL["🧠 Workers AI<br/>nemotron-3-120b-a12b"]
        end
    end

    TG -- "webhook POST /telegram" --> Worker
    WEB -- "WebSocket wss://" --> Worker
    Worker -- "proxy to :18789" --> GW
    GW --> AGENT
    AGENT --> TOOLS
    AGENT --> TG_CH
    TG_CH -- "reply" --> TG
    AGENT -- "AI request" --> AIGW
    AIGW -- "inference" --> MODEL
    MODEL -- "response" --> AIGW
    AIGW --> AGENT
    Container -- "rclone sync<br/>every 30s" --> R2
    R2 -- "restore on<br/>cold start" --> Container
    Worker --> DO

    classDef users fill:#e3f2fd,stroke:#1565c0,color:#000
    classDef worker fill:#fff3e0,stroke:#e65100,color:#000
    classDef container fill:#f3e5f5,stroke:#6a1b9a,color:#000
    classDef storage fill:#e8f5e9,stroke:#2e7d32,color:#000
    classDef ai fill:#fce4ec,stroke:#c62828,color:#000

    class TG,WEB users
    class AUTH,PROXY,LIFECYCLE worker
    class AGENT,TOOLS,TG_CH,GW container
    class R2,DO storage
    class AIGW,MODEL ai
```

## What gets created on Cloudflare

| Resource | Name | Purpose |
|----------|------|---------|
| Worker | `{name}.{subdomain}.workers.dev` | Entry point — handles auth, routing, container lifecycle |
| Container | (auto-created) | Runs the OpenClaw AI agent |
| AI Gateway | `{name}-gateway` | Logs AI requests, rate limiting, authentication |
| R2 Bucket | `{name}-data` | Persists config, sessions, workspace across restarts |
| Secrets | 14 total | API keys, tokens, URLs, feature flags |

## Quick start

### As an OpenClaw Skill

Copy this folder to your OpenClaw skills directory. Then ask your agent:

> "Deploy a new OpenClaw bot"

### With Claude Code or Claude Desktop

Point the AI to `SKILL.md` and say:

> "Follow the instructions in SKILL.md to deploy a new OpenClaw bot"

### Manual

Read `SKILL.md` for the complete step-by-step commands. Every command is documented — you can run them yourself if needed.

## File structure

```
deploy-openclaw/
├── SKILL.md              ← Instructions for the AI agent (the brain)
├── README.md             ← You are here
└── moltworker/           ← Pre-configured source code (no git clone needed)
    ├── Dockerfile        ← Container: Node 22.16.0 + OpenClaw 2026.3.13
    ├── start-openclaw.sh ← Container startup: onboard + config patch + gateway
    ├── wrangler.jsonc    ← Worker + Container + R2 bindings
    ├── package.json      ← Dependencies
    ├── src/              ← Worker source code (TypeScript)
    ├── skills/           ← Built-in skills (browser automation)
    └── public/           ← Dashboard UI assets (logos)
```

## Key concepts

| Term | What it is |
|------|-----------|
| **OpenClaw** | Open-source AI agent — handles conversations, tools, memory |
| **Moltworker** | Cloudflare Worker that wraps OpenClaw in a container |
| **Worker** | Lightweight JavaScript at the edge — the "front door" |
| **Container** | Full Linux environment — where OpenClaw actually runs |
| **AI Gateway** | Cloudflare proxy for AI requests — adds logging and protection |
| **R2** | Cloudflare object storage — keeps your data when container sleeps |
| **SKILL.md** | Instructions file that AI agents read and execute |

## Versions

| Component | Version |
|-----------|---------|
| OpenClaw | 2026.3.13 |
| Node.js (in container) | 22.16.0 |
| Moltworker | Snapshot 2026-03-25 |
| Container size | standard-2 (2 vCPU, 4GB RAM) |

## Updating

1. Clone latest [cloudflare/moltworker](https://github.com/cloudflare/moltworker)
2. Copy files into `moltworker/` (keep current structure)
3. Re-apply modifications to `start-openclaw.sh` (auth order + allowedOrigins)
4. Update versions in `Dockerfile` if needed
5. Test deploy, then commit + push

## License

Moltworker source code is from [cloudflare/moltworker](https://github.com/cloudflare/moltworker) under its original license.
