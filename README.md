# Deploy OpenClaw

Deploy [OpenClaw](https://github.com/openclaw/openclaw) AI Bot on Cloudflare — fully automated by AI.

> Tell your AI agent _"Deploy a new AI bot for me"_ and it handles everything.

## How it works

This is an **AI Skill** — a set of instructions that AI agents (Claude Code, Claude Desktop, OpenClaw) read and execute. The AI does the heavy lifting; you just answer a few questions.

```
You:  "Deploy a new AI bot for me"
AI:   Installing tools... creating resources... deploying...
AI:   "Done! Send 'hi' to your bot on Telegram."
You:  "hi"
Bot:  "Hey! I'm your new AI assistant. How can I help?"
```

## What you need

- A Mac
- A Cloudflare account ([Workers Paid Plan](https://dash.cloudflare.com/) — $5/month)
- [Telegram](https://telegram.org/) installed

That's it. The AI installs everything else (Node.js, Docker, wrangler CLI).

## What you'll do vs what the AI does

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

| | Count | Examples |
|---|---|---|
| 🔴 **You do** | 6 steps | Create tokens, send "hi", click Approve |
| 🟢 **AI does** | 11 steps | Install, build, deploy, configure |

## What gets created

```
Telegram / Web Dashboard
        ↓
┌─────────────────────────┐
│  Worker (Moltworker)    │  ← Cloudflare Workers
│  Routing + Auth         │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│  Container (OpenClaw)   │  ← Cloudflare Containers
│  AI Agent + Tools       │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│  AI Gateway             │  ← Cloudflare AI Gateway
│  Logging + Protection   │
└───────────┬─────────────┘
            ↓
      Workers AI Model
   (nemotron-3-120b-a12b)
```

| Resource | Name | Purpose |
|----------|------|---------|
| Worker | `{name}.{subdomain}.workers.dev` | Entry point, auth, routing |
| Container | (auto-created) | Runs the AI agent |
| AI Gateway | `{name}-gateway` | Logs, rate limits AI requests |
| R2 Bucket | `{name}-data` | Persists data across restarts |
| Secrets | 14 total | API keys, tokens, config |

## Quick start

### Option 1: As an OpenClaw Skill

Copy this folder to your OpenClaw skills directory. Then ask your agent:

> "Deploy a new OpenClaw bot"

### Option 2: With Claude Code or Claude Desktop

Point the AI to `SKILL.md` and say:

> "Follow the instructions in SKILL.md to deploy a new OpenClaw bot"

### Option 3: Manual

Read `SKILL.md` for the complete step-by-step commands.

## File structure

```
deploy-openclaw/
├── SKILL.md              ← Instructions for the AI agent
├── README.md             ← You are here
└── moltworker/           ← Pre-configured source code
    ├── Dockerfile        ← Container image definition
    ├── start-openclaw.sh ← Startup script + config
    ├── wrangler.jsonc    ← Worker + Container settings
    ├── package.json      ← Dependencies
    ├── src/              ← Worker code (TypeScript)
    ├── skills/           ← Built-in skills
    └── public/           ← Dashboard UI assets
```

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
