# opencode-free-proxy

Free AI models from [OpenCode](https://opencode.ai) exposed as standard OpenAI and Anthropic APIs.

One server — works with any tool that speaks OpenAI or Anthropic format: Cursor, Continue, Cline, Claude Code, aider, opencode CLI, raw `curl`, whatever.

## 30-second setup

```bash
git clone https://github.com/bigdata2211it-web/opencode-free-proxy.git
cd opencode-free-proxy
pip install -r requirements.txt
python server.py
```

Done. Server is at `http://localhost:6446`. API keys are in `api-keys.json` (auto-generated on first run).

## What you get

| Model | What it is | Reliability |
|-------|-----------|-------------|
| `mimo-v2.5-free` | Xiaomi MiMo v2.5 | Solid |
| `deepseek-v4-flash-free` | DeepSeek V4 Flash | Solid |
| `north-mini-code-free` | North Mini Code | Solid |
| `nemotron-3-ultra-free` | NVIDIA Nemotron 3 Ultra | Solid |
| `big-pickle` | DeepSeek V4 Flash (alias) | Solid |

All models support streaming, tool calls, and system messages.

## API

### OpenAI format — `POST /v1/chat/completions`

```bash
curl http://localhost:6446/v1/chat/completions \
  -H "Authorization: Bearer YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-v4-flash-free",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": true
  }'
```

### Anthropic format — `POST /v1/messages`

```bash
curl http://localhost:6446/v1/messages \
  -H "x-api-key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-v4-flash-free",
    "system": "You are helpful.",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 1024,
    "stream": true
  }'
```

### Other endpoints

| Method | Path | What |
|--------|------|------|
| `GET` | `/v1/models` | List models |
| `GET` | `/health` | Health + version |

### Auth

Both `Authorization: Bearer KEY` and `x-api-key: KEY` work on all endpoints.

## Use with tools

### opencode CLI

Add to `~/.config/opencode/opencode.json`:

```json
{
  "provider": {
    "free": {
      "name": "free",
      "type": "openai",
      "apiKey": "YOUR_KEY",
      "baseURL": "http://localhost:6446/v1",
      "models": {
        "free/deepseek-v4-flash-free": {
          "id": "deepseek-v4-flash-free",
          "name": "free/deepseek-v4-flash-free",
          "attachment": true,
          "reasoning": true
        }
      }
    }
  }
}
```

### Cursor / Continue / Cline

- Base URL: `http://YOUR_HOST:6446/v1`
- API Key: your key from `api-keys.json`
- Model: `deepseek-v4-flash-free`

### Claude Code (Anthropic format)

- Base URL: `http://YOUR_HOST:6446`
- API Key: your key from `api-keys.json`
- Works with `/v1/messages` endpoint

## Deploy on a VPS

```bash
# On your VPS
git clone https://github.com/bigdata2211it-web/opencode-free-proxy.git
cd opencode-free-proxy
pip install -r requirements.txt
python server.py          # foreground
# or
nohup python server.py > proxy.log 2>&1 &   # background
```

If your VPS doesn't expose port 6446, use an SSH tunnel:

```bash
ssh -L 6446:127.0.0.1:6446 user@your-vps
# Now http://localhost:6446 works locally
```

### systemd service (optional)

```ini
# /etc/systemd/system/opencode-proxy.service
[Unit]
Description=OpenCode Free Proxy
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/opencode-proxy
ExecStart=/usr/bin/python3 server.py
Restart=always
RestartSec=5
Environment=PROXY_PORT=6446

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now opencode-proxy
```

## Environment variables

| Variable | Default | What |
|----------|---------|------|
| `PROXY_PORT` | `6446` | Server port |
| `KEYS_FILE` | `./api-keys.json` | API keys file path |
| `SOCKS5_PROXY` | _(none)_ | SOCKS5 proxy address for upstream requests |

## SOCKS5 Proxy

Route all upstream requests to opencode.ai through a SOCKS5 proxy.

### CLI argument

```bash
python server.py --proxy 127.0.0.1:9150
python server.py --proxy socks5://user:pass@10.0.0.1:1080
```

### Environment variable

```bash
SOCKS5_PROXY=127.0.0.1:9150 python server.py
SOCKS5_PROXY=socks5://user:pass@10.0.0.1:1080 python server.py
```

CLI `--proxy` takes priority over `SOCKS5_PROXY`. If neither is set, requests go directly.

### systemd with proxy

```ini
Environment=SOCKS5_PROXY=127.0.0.1:9150
```

## How it works

```
Your tool (Cursor, CLI, curl, etc.)
        │
        ▼
  opencode-free-proxy        ← this server, translates formats
        │
        ▼  HTTPS
  opencode.ai/zen/v1/       ← free tier API
```

The proxy adds `x-opencode-*` authentication headers that the Zen API requires. These were discovered by reverse engineering the opencode binary — without them, even `Authorization: Bearer public` gets rejected with `AuthError`.

### Zen API auth headers (for the curious)

```
Authorization: Bearer public
User-Agent: opencode/1.15.0 ai-sdk/provider-utils/4.0.23 runtime/bun/1.3.13
x-opencode-client: cli
x-opencode-project: global
x-opencode-request: msg_<unique_id>
x-opencode-session: ses_<unique_id>
```

## License

MIT
