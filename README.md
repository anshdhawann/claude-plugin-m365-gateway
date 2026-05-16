# Claude Plugin M365 Gateway

> **This is what I use and what worked for me.** I run this gateway on my own VPS with Tailscale routing to local models. Your setup may differ — YMMV. Contributions welcome if you get it working on something I haven't tested.

Self-hosted **LiteLLM + Caddy** gateway that lets you use the Claude for Microsoft 365 add-ins (Word, Excel, PowerPoint, Outlook) with **any OpenAI-compatible LLM provider** — not just Anthropic's API.

Instead of routing through Anthropic's servers or paying for a Claude plan, this gateway runs on your own infrastructure. The Office add-in talks to your gateway, and your gateway talks to whatever backend you want (local models, OpenRouter, vLLM, etc.).

![Architecture: Office Add-in → TLS/CORS (Caddy) → Translation (LiteLLM) → Your Provider]

---

## How It Works

The Claude for Microsoft 365 add-ins send requests in **Anthropic's message format** (`/v1/messages`). LiteLLM translates those requests into the format your chosen provider expects, and translates the response back. Caddy handles HTTPS, CORS headers (required by `pivot.claude.ai`), and TLS certificates via Let's Encrypt.

```
┌──────────────┐     ┌────────┐     ┌──────────┐     ┌──────────────────┐
│  Office Add-in│────▶│ Caddy  │────▶│ LiteLLM  │────▶│ Your Provider    │
│  (pivot.     │     │ (TLS,  │     │ (translate│     │ (OpenAI API,     │
│   claude.ai) │◀────│  CORS) │◀────│  format)  │◀────│  local model,    │
└──────────────┘     └────────┘     └──────────┘     │  etc.)           │
                                                      └──────────────────┘
```

---

## Quick Start

### 1. Prerequisites

- A **VPS** with Docker and Docker Compose installed
- A **domain** pointing to your VPS (Caddy needs it for TLS)
- A **provider backend** — anything with an OpenAI-compatible `/v1/chat/completions` endpoint

### 2. Clone & Configure

```bash
git clone https://github.com/anshdhawann/claude-plugin-m365-gateway.git
cd claude-plugin-m365-gateway
cp .env.example .env
```

Edit `.env` with your API keys and endpoints:

```env
LITELLM_MASTER_KEY=sk-generate-a-random-key-here
OPENER_BASE_URL=https://your-provider.com/v1
OPENER_API_KEY=sk-your-api-key
```

Edit `Caddyfile` and replace `YOUR_DOMAIN` with your actual domain.

Edit `litellm/config.yaml` and replace `YOUR_MODEL_ID_HERE` with the model name your provider uses (e.g. `gpt-4o`, `deepseek-v4-flash`, `claude-sonnet-4-5-20250929`).

### 3. Deploy

```bash
docker compose up -d
```

Caddy auto-provisions a Let's Encrypt TLS certificate. Your gateway is now live at `https://yourdomain.com`.

### 4. Verify

```bash
# Check model list
curl https://yourdomain.com/v1/models

# Quick test
curl https://yourdomain.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $LITELLM_MASTER_KEY" \
  -H "anthropic-version: 2024-01-01" \
  -d '{
    "model": "claude-opus-4-7-20250820",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Say hello"}]
  }'
```

---

## Connecting the Office Add-in

1. Open **Word, Excel, PowerPoint, or Outlook**
2. Go to **Home → Add-ins** and launch the Claude add-in (install from Microsoft AppSource if you haven't)
3. On the sign-in screen, select **"Enterprise gateway"**
4. Enter:
   - **Gateway URL**: `https://yourdomain.com`
   - **API token**: Your `LITELLM_MASTER_KEY` (or a user-specific token if you configure LiteLLM virtual keys)
5. The add-in will test the connection and present the main app

For detailed admin deployment (Azure admin consent, tenant-wide rollout, Outlook support), see [Anthropic's official documentation](https://support.claude.com/en/articles/13945233-use-claude-for-microsoft-365-with-third-party-platforms).

---

## Model ID Reference

The Claude add-in for Microsoft 365 requests models by specific Claude model IDs. You must map **these exact IDs** in `litellm/config.yaml` to whatever backend models you want to use.

The add-in discovers available models via `GET /v1/models` and presents them to the user. You need to define at least one — more gives users choice:

| Add-in Model ID | Typical Purpose |
|-----------------|----------------|
| `claude-opus-4-7-20250820` | Best for complex document analysis and editing |
| `claude-opus-4-6-20250820` | Strong all-purpose, slightly faster |
| `claude-sonnet-4-6-20250820` | Lightweight, fastest responses |

For the latest list of supported model IDs and connection details, see [Anthropic's official documentation](https://support.claude.com/en/articles/13945233-use-claude-for-microsoft-365-with-third-party-platforms).

Each of these can point to **any** provider — you decide which backend model handles "Opus" requests vs "Sonnet" requests.

---

## Using LM Studio as Your Backend

[LM Studio](https://lmstudio.ai) runs local models on your machine with an OpenAI-compatible API. Here's how to connect it to this gateway.

### Option A: LM Studio on the same machine as the gateway (simplest)

If you're running Docker on your local machine (not a VPS), LM Studio and the gateway share the same host:

1. In LM Studio, go to **Developer** tab and enable the local API server (default: `localhost:1234`)
2. Set `.env` to point to LM Studio:
   ```env
   OPENER_BASE_URL=http://host.docker.internal:1234/v1
   OPENER_API_KEY=not-needed
   ```
3. Run `docker compose up -d`
4. Open the Office add-in and point it at your gateway

For macOS Docker Desktop, `host.docker.internal` automatically resolves to your Mac. On Linux, use `--network=host` or `172.17.0.1` instead.

### Option B: LM Studio on your local machine, gateway on a VPS

When the gateway lives on a VPS but LM Studio runs on your laptop/desktop, you need a secure tunnel:

**Using Tailscale (recommended):**
1. Install Tailscale on both your local machine and VPS
2. In LM Studio, bind to `0.0.0.0` (Settings → Local API Server → Bind to `0.0.0.0`)
3. Set `.env` on your VPS:
   ```env
   OPENER_BASE_URL=http://100.x.x.x:1234/v1
   OPENER_API_KEY=not-needed
   ```
   Use your local machine's Tailscale IP.

**Using SSH tunnel:**
```bash
# On your local machine, forward LM Studio through SSH to the VPS
ssh -R 1234:localhost:1234 user@your-vps
```

Then set `.env` on the VPS to `OPENER_BASE_URL=http://localhost:1234/v1`.

**Using ngrok or bore:**
```bash
# Expose LM Studio publicly (no auth — be careful)
ngrok http 1234
```
Set `OPENER_BASE_URL` to the generated ngrok URL. Your gateway will route through it.

### LM Studio config tips

- **Model selection**: Pick a model that handles complex instructions well. Models like DeepSeek V4 Flash, Qwen 3.6 Plus, or any 70B+ class works great.
- **Context length**: Set to at least 32K in LM Studio — the Office add-in can send large document contents.
- **Keep alive**: Enable "Keep model loaded in memory" so the model is always ready.
- **GPU offloading**: Max out GPU offload in LM Studio for best performance.

## How to Configure Models

The `litellm/config.yaml` maps those **Claude model IDs** to **your provider's models**:

```yaml
# Map "Opus 4.7" → your best model
- model_name: claude-opus-4-7-20250820
  litellm_params:
    model: openai/deepseek-v4-flash
    api_base: ${OPENER_BASE_URL}
    api_key: ${OPENER_API_KEY}

# Map "Sonnet 4.6" → your cheaper/faster model
- model_name: claude-sonnet-4-6-20250820
  litellm_params:
    model: openai/kimi-k2.6
    api_base: ${OPENER_BASE_URL}
    api_key: ${OPENER_API_KEY}
```

LiteLLM supports [dozens of providers](https://docs.litellm.ai/docs/providers) — OpenAI, Anthropic, Bedrock, Vertex AI, OpenRouter, Together, Groq, vLLM, Ollama, LM Studio, and more. Use the prefix `provider/model-name` syntax (e.g. `openai/gpt-4o`, `bedrock/anthropic.claude-v2`, `vertex_ai/claude-sonnet`).

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `LITELLM_MASTER_KEY` | Yes | API key used to authenticate with LiteLLM (generate a random string) |
| `OPENER_BASE_URL` | Depends | Base URL of your OpenAI-compatible backend |
| `OPENER_API_KEY` | Depends | API key for your backend |
| `ANTHROPIC_API_KEY` | Optional | Needed if you route any models via native Anthropic passthrough |

---

## Deployment Tips

- **Rate limiting**: Adjust `rpm` per model in `config.yaml` to avoid provider throttling
- **Custom headers**: Add `extra_headers` in `litellm_params` for provider-specific auth
- **User-specific tokens**: Use LiteLLM's [virtual key system](https://docs.litellm.ai/docs/proxy/virtual_keys) to issue per-user API tokens instead of sharing your master key
- **Monitoring**: LiteLLM logs all requests at `--detailed_logging`; check with `docker compose logs litellm`
- **Updates**: `docker compose pull && docker compose up -d` to update both Caddy and LiteLLM

---

## Troubleshooting

| Symptom | Likely Cause |
|---------|-------------|
| Add-in shows "Connection refused" | Gateway unreachable — check DNS, firewall, and that Docker containers are running |
| 401 Unauthorized | Wrong API token — verify the token matches what the add-in entered |
| CORS error in browser console | Caddyfile isn't returning `Access-Control-Allow-Origin: https://pivot.claude.ai` |
| Streaming responses hang | LiteLLM or your provider doesn't support SSE passthrough |
| "No models available" | LiteLLM returns empty model list — check `config.yaml` syntax and provider connectivity |

---

## License

MIT
