# Deploy OpenClaw

An AI Skill that deploys [OpenClaw](https://github.com/openclaw/openclaw) on Cloudflare Workers + Containers via [Moltworker](https://github.com/cloudflare/moltworker).

Designed for AI agents (Claude Code, Claude Desktop, OpenClaw) to execute — not for humans to run manually.

## What it does

Deploys a fully configured OpenClaw AI Bot with one command. The AI agent reads `SKILL.md` and executes all steps automatically.

```
User: "Deploy a new AI bot for me"
  → AI installs dependencies (brew, node, wrangler, docker)
  → AI creates CF resources (R2, AI Gateway)
  → AI sets 14 secrets
  → AI builds + deploys Worker + Container
  → AI configures Telegram webhook + pairing
  → User gets a working Telegram bot + Dashboard
```

## Architecture

```
Telegram / Dashboard
        ↓
Worker (Moltworker)     ← Cloudflare Workers
  - Auth, routing, container lifecycle
        ↓
Container (OpenClaw)    ← Cloudflare Containers
  - AI Agent, conversation, tools
        ↓
AI Gateway              ← Cloudflare AI Gateway
  - Logging, rate limiting
        ↓
Workers AI (nemotron-3-120b-a12b)
```

## Prerequisites

- macOS
- Cloudflare account with Workers Paid Plan ($5/month)
- Telegram account

## User input required

| Input | Required | One-time? |
|-------|----------|-----------|
| Cloudflare API Token | Yes | Yes (reusable) |
| Telegram Bot Token | Yes | Per bot |
| AI Gateway Auth Token | Yes | Per bot |
| R2 API Token | Optional | Per bot |
| AI Model | Optional | Default: nemotron-3-120b-a12b |
| Instance name | Optional | Default: openclaw |
| Pairing code | Yes | Per bot |

## What gets created

| Resource | Name |
|----------|------|
| Worker | `{name}.{subdomain}.workers.dev` |
| Container | Runs OpenClaw inside |
| AI Gateway | `{name}-gateway` |
| R2 Bucket | `{name}-data` |
| 14 Secrets | Auto-configured |

## File structure

```
deploy-openclaw/
├── SKILL.md              ← AI agent instructions (the "script")
├── README.md             ← This file
└── moltworker/           ← Bundled source code snapshot
    ├── Dockerfile        ← Node 22.16.0 + OpenClaw 2026.3.13
    ├── start-openclaw.sh ← Container startup + config patch
    ├── wrangler.jsonc    ← Worker + Container config
    ├── package.json
    ├── src/              ← Worker source (TypeScript)
    ├── skills/           ← Built-in skills (browser)
    └── public/           ← Dashboard assets
```

## Usage

### As an OpenClaw Skill

Copy this folder into your OpenClaw skills directory. The agent will use `SKILL.md` when asked to deploy a new bot.

### With Claude Code / Claude Desktop

Point the AI to `SKILL.md` and ask it to deploy:

```
"Deploy a new OpenClaw bot using the deploy-openclaw skill"
```

### Manual reference

See `SKILL.md` for the complete step-by-step process. All commands are documented there.

## Versions

| Component | Version |
|-----------|---------|
| OpenClaw | 2026.3.13 |
| Node.js (container) | 22.16.0 |
| Moltworker | Snapshot from 2026-03-25 |
| Container instance | standard-2 (2 vCPU, 4GB RAM) |

## Updating

To update OpenClaw or Moltworker:

1. Clone latest [moltworker](https://github.com/cloudflare/moltworker)
2. Copy essential files into `moltworker/` (see current structure)
3. Apply the same modifications to `start-openclaw.sh` (auth order + allowedOrigins)
4. Update versions in `Dockerfile` if needed
5. Test deploy
6. Commit + push

## License

Moltworker source code is from [cloudflare/moltworker](https://github.com/cloudflare/moltworker) under its original license.
