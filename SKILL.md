---
name: deploy-openclaw
description: Deploy a new OpenClaw AI Bot on Cloudflare Workers + Containers. Handles full setup from scratch on macOS — installs all dependencies, creates CF resources, deploys, and configures Telegram. User only needs a Cloudflare account (Workers Paid Plan) and Telegram.
---

# Deploy OpenClaw on Cloudflare (macOS)

You are deploying an OpenClaw AI Bot on Cloudflare. Follow every step in order. The bundled `moltworker/` folder contains all source code — no git clone needed.

## Step 0: Prerequisites check and auto-fix

Check each tool. If missing, install it. Run checks sequentially — later tools depend on earlier ones.

### 0.1 Homebrew

```bash
brew --version
```

If not found:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After install, ensure brew is in PATH. On Apple Silicon:

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 0.2 Node.js (>= 18)

```bash
node --version
```

If not found or version < 18:

```bash
brew install node
```

### 0.3 wrangler CLI

```bash
wrangler --version
```

If not found:

```bash
npm install -g wrangler
```

### 0.4 Docker

```bash
docker --version
```

If not found:

```bash
brew install --cask docker
```

Then check if Docker daemon is running:

```bash
docker ps 2>&1
```

If it shows "Cannot connect" or "no such file", start Docker:

```bash
open -a Docker
```

Wait up to 60 seconds for Docker daemon to be ready. Check by retrying `docker ps` every 5 seconds.

### 0.5 wrangler login

```bash
wrangler whoami 2>&1
```

If it shows error or "not logged in":

```bash
wrangler login
```

This opens a browser. Tell the user: "A browser window will open. Please log in to your Cloudflare account and authorize wrangler."

Wait for the command to complete.

### 0.6 Telegram Bot Token

Ask the user:

> Do you already have a Telegram Bot Token? If not, follow these steps:
> 1. Open Telegram and search for **@BotFather**
> 2. Send `/newbot`
> 3. Follow the prompts to name your bot
> 4. BotFather will give you a token like `8622764702:AAGkF1CqHHrVWfpwmAP4O22BlKpZXJiVRv8`
> 5. Give me that token

Store the token as `{TELEGRAM_TOKEN}`.

### 0.7 AI Model (optional)

Ask the user:

> Which AI model do you want to use? (Press Enter for default)
> Default: `@cf/nvidia/nemotron-3-120b-a12b`

Store as `{USER_MODEL}`. Default: `@cf/nvidia/nemotron-3-120b-a12b`.

### 0.8 Instance name (optional)

Ask the user:

> What name for this bot instance? (Press Enter for default)
> Default: `openclaw`

Store as `{NAME}`. Default: `openclaw`. This affects Worker name, R2 bucket, and gateway name.

## Step 1: Derive variables

```bash
ACCOUNT_ID=$(wrangler whoami 2>&1 | grep -oE '[a-f0-9]{32}' | head -1)
OAUTH_TOKEN=$(wrangler oauth-token 2>/dev/null)
SUBDOMAIN=$(curl -s "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/workers/subdomain" \
  -H "Authorization: Bearer ${OAUTH_TOKEN}" | node -e "const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));console.log(d.result?.subdomain||'')")
GATEWAY_TOKEN=$(openssl rand -hex 32)
```

Computed values:

- `WORKER_URL` = `https://{NAME}.${SUBDOMAIN}.workers.dev`
- `R2_BUCKET` = `{NAME}-data`
- `GATEWAY_ID` = `{NAME}-gateway`
- `MODEL_ID` = `workers-ai/{USER_MODEL}`

Save `GATEWAY_TOKEN` — user needs it to access the dashboard later.

## Step 2: Prepare source code

```bash
cp -r {SKILL_DIR}/moltworker {WORK_DIR}
cd {WORK_DIR}
```

`{SKILL_DIR}` is the directory containing this SKILL.md. `{WORK_DIR}` is where to deploy from (e.g. current working directory + `{NAME}`).

## Step 3: Modify files

### Dockerfile — 2 edits

Find `ENV NODE_VERSION=22.13.1` → replace with `ENV NODE_VERSION=22.16.0`
Find `openclaw@2026.2.3` → replace with `openclaw@2026.3.13`

### wrangler.jsonc — 3 edits

Find `"name": "moltbot-sandbox"` → replace with `"name": "{NAME}"`
Find `"bucket_name": "moltbot-data"` → replace with `"bucket_name": "{R2_BUCKET}"`
Find `"instance_type": "standard-1"` → replace with `"instance_type": "standard-2"`

### start-openclaw.sh — 2 edits

**Edit 1: Reorder auth block.** Find the AUTH_ARGS if/elif/fi block that starts with:

```bash
    AUTH_ARGS=""
    if [ -n "$CLOUDFLARE_AI_GATEWAY_API_KEY" ]
```

Replace the entire AUTH_ARGS="" through fi with:

```bash
    AUTH_ARGS=""
    if [ -n "$ANTHROPIC_API_KEY" ]; then
        AUTH_ARGS="--auth-choice apiKey --anthropic-api-key $ANTHROPIC_API_KEY"
    elif [ -n "$CLOUDFLARE_AI_GATEWAY_API_KEY" ] && [ -n "$CF_AI_GATEWAY_ACCOUNT_ID" ] && [ -n "$CF_AI_GATEWAY_GATEWAY_ID" ]; then
        AUTH_ARGS="--auth-choice cloudflare-ai-gateway-api-key \
            --cloudflare-ai-gateway-account-id $CF_AI_GATEWAY_ACCOUNT_ID \
            --cloudflare-ai-gateway-gateway-id $CF_AI_GATEWAY_GATEWAY_ID \
            --cloudflare-ai-gateway-api-key $CLOUDFLARE_AI_GATEWAY_API_KEY"
    elif [ -n "$OPENAI_API_KEY" ]; then
        AUTH_ARGS="--auth-choice openai-api-key --openai-api-key $OPENAI_API_KEY"
    fi
```

**Edit 2: Add allowedOrigins.** Find this block:

```javascript
if (process.env.OPENCLAW_DEV_MODE === 'true') {
    config.gateway.controlUi = config.gateway.controlUi || {};
    config.gateway.controlUi.allowInsecureAuth = true;
}
```

Replace with:

```javascript
config.gateway.controlUi = config.gateway.controlUi || {};
config.gateway.controlUi.allowInsecureAuth = true;
const existingOrigins = config.gateway.controlUi.allowedOrigins || [];
const workerUrl = process.env.WORKER_URL;
if (workerUrl && !existingOrigins.includes(workerUrl)) {
    existingOrigins.push(workerUrl);
}
config.gateway.controlUi.allowedOrigins = existingOrigins.length > 0
    ? existingOrigins
    : ['{WORKER_URL}'];
```

## Step 4: Install dependencies

```bash
cd {WORK_DIR}
npm install
```

## Step 5: Create Cloudflare resources

Run each command. Ignore errors if resource already exists.

```bash
# Create AI Gateway
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/ai-gateway/gateways" \
  -H "Authorization: Bearer ${OAUTH_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"id":"{GATEWAY_ID}","name":"{GATEWAY_ID}"}'

# Create AI Gateway Token
AIG_TOKEN_RESP=$(curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/ai-gateway/gateways/{GATEWAY_ID}/tokens" \
  -H "Authorization: Bearer ${OAUTH_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name":"{NAME}-token"}')
AIG_TOKEN=$(echo "$AIG_TOKEN_RESP" | node -e "const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));console.log(d.result?.token||'')")

# Create R2 Bucket
CLOUDFLARE_ACCOUNT_ID=${ACCOUNT_ID} wrangler r2 bucket create {R2_BUCKET}

# Create R2 API Token
R2_TOKEN_RESP=$(curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/r2/tokens" \
  -H "Authorization: Bearer ${OAUTH_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name":"{NAME}-r2","permissions":["object-read-write"],"buckets":["{R2_BUCKET}"],"ttl":0}')
R2_ACCESS_KEY=$(echo "$R2_TOKEN_RESP" | node -e "const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));console.log(d.result?.access_key_id||d.result?.accessKeyId||'')")
R2_SECRET_KEY=$(echo "$R2_TOKEN_RESP" | node -e "const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));console.log(d.result?.secret_access_key||d.result?.secretAccessKey||'')")
```

If `AIG_TOKEN` is empty, tell the user:
> I couldn't auto-create the AI Gateway Token. Please go to CF Dashboard → AI → AI Gateway → {GATEWAY_ID} → Create Token, then give me the token value.

If `R2_ACCESS_KEY` is empty, tell the user:
> I couldn't auto-create the R2 API Token. Please go to CF Dashboard → R2 → Manage R2 API Tokens → Create Account API Token (Object Read & Write, bucket: {R2_BUCKET}), then give me the Access Key ID and Secret Access Key.

## Step 6: Set secrets

Run each command from `{WORK_DIR}`. If `wrangler secret put` fails, use `wrangler versions secret put` as fallback.

```bash
cd {WORK_DIR}
echo "sk-ant-dummy-bypass-key-not-real" | wrangler secret put ANTHROPIC_API_KEY
echo "${AIG_TOKEN}" | wrangler secret put CLOUDFLARE_AI_GATEWAY_API_KEY
echo "${ACCOUNT_ID}" | wrangler secret put CF_AI_GATEWAY_ACCOUNT_ID
echo "{GATEWAY_ID}" | wrangler secret put CF_AI_GATEWAY_GATEWAY_ID
echo "{MODEL_ID}" | wrangler secret put CF_AI_GATEWAY_MODEL
echo "${GATEWAY_TOKEN}" | wrangler secret put MOLTBOT_GATEWAY_TOKEN
echo "{TELEGRAM_TOKEN}" | wrangler secret put TELEGRAM_BOT_TOKEN
echo "{WORKER_URL}" | wrangler secret put WORKER_URL
echo "true" | wrangler secret put DEV_MODE
echo "true" | wrangler secret put DEBUG_ROUTES
echo "${ACCOUNT_ID}" | wrangler secret put CF_ACCOUNT_ID
echo "${R2_ACCESS_KEY}" | wrangler secret put R2_ACCESS_KEY_ID
echo "${R2_SECRET_KEY}" | wrangler secret put R2_SECRET_ACCESS_KEY
```

## Step 7: Deploy

```bash
cd {WORK_DIR}
npm run deploy
```

This takes 3-5 minutes on first run (Docker build + push + deploy). Ensure Docker is running.

## Step 8: Set Telegram webhook

```bash
curl -s "https://api.telegram.org/bot{TELEGRAM_TOKEN}/setWebhook?url={WORKER_URL}/telegram"
```

## Step 9: Wait for container and pair Telegram

1. Trigger container start:
```bash
curl -s --max-time 10 "{WORKER_URL}/" > /dev/null
```

2. Wait 1-3 minutes for cold start. Poll every 15 seconds:
```bash
curl -s --max-time 10 "{WORKER_URL}/debug/gateway-api?path=/health"
```
Ready when response does NOT contain "not listening".

3. Tell the user:
> Your bot is starting up. Send **"hi"** to your bot in Telegram now.

4. Poll for pending pairing code (every 10 seconds, up to 3 minutes):
```bash
curl -s --max-time 10 "{WORKER_URL}/debug/cli?cmd=cat+/root/.openclaw/devices/pending.json"
```

5. When a code appears in the JSON output, approve it:
```bash
curl -s --max-time 15 "{WORKER_URL}/debug/cli?cmd=openclaw+pairing+approve+telegram+{CODE}"
```

6. Confirm to user:
> Pairing complete! Your bot is ready. Try sending a message.

If pairing times out, give the user manual instructions:
> 1. Send "hi" to your bot in Telegram
> 2. Copy the pairing code from the bot's reply
> 3. Open: `{WORKER_URL}/debug/cli?cmd=openclaw+pairing+approve+telegram+CODE`

## Final output

Tell the user:

```
✅ OpenClaw Bot deployed successfully!

Worker URL:    {WORKER_URL}
Dashboard:     {WORKER_URL}/?token={GATEWAY_TOKEN}
Admin Panel:   {WORKER_URL}/_admin/
Gateway Token: {GATEWAY_TOKEN}

Save the Gateway Token — you need it to access the Dashboard.
Send messages to your Telegram bot to start chatting!
```

## Troubleshooting

If anything goes wrong during or after deployment:

- Container not starting → `curl -s "{WORKER_URL}/debug/processes?logs=true"`
- Check which secrets are set → `curl -s "{WORKER_URL}/debug/env"`
- View OpenClaw config → `curl -s "{WORKER_URL}/debug/container-config"`
- "origin not allowed" on Dashboard → allowedOrigins patch failed in start-openclaw.sh
- Telegram no response → `curl -s "https://api.telegram.org/bot{TELEGRAM_TOKEN}/getWebhookInfo"`
- "openclaw requires Node >= X" → Node version in Dockerfile too old
- Onboard hangs on startup → ANTHROPIC_API_KEY secret not set
- `wrangler secret put` fails → use `wrangler versions secret put` then redeploy with `npm run deploy`
