# AI Agent Telegram Setup 🤖

Setup AI agent di VPS dengan Telegram bot. Menggunakan Hermes Agent (open-source by Nous Research).

**Hermes Agent:** https://github.com/NousResearch/hermes-agent
**Docs:** https://hermes-agent.nousresearch.com/docs/

## Quick Start

```bash
# Install Hermes Agent
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# Setup wizard
hermes setup

# Setup Telegram
hermes gateway setup

# Start
hermes gateway start
```

## VPS Requirements

| Spec | Minimum | Recommended |
|------|---------|-------------|
| OS | Ubuntu 22.04 | Ubuntu 22.04 LTS |
| RAM | 1 GB | 2 GB+ |
| CPU | 1 core | 2 core+ |
| Disk | 10 GB | 20 GB+ |
| Node.js | 20+ | 20 LTS |

## Step 1: Install Hermes Agent

```bash
# One-click install
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# Verify
hermes --version
```

## Step 2: Setup API Key

```bash
# Interactive setup wizard
hermes setup

# Or manual
hermes model                    # Interactive model picker
hermes auth add anthropic       # Add API key
```

### Supported Providers

| Provider | Env Variable | Pricing |
|----------|-------------|---------|
| Anthropic | `ANTHROPIC_API_KEY` | $3/1M input |
| OpenAI | `OPENAI_API_KEY` | $2.50/1M input |
| DeepSeek | `DEEPSEEK_API_KEY` | $0.14/1M input |
| OpenRouter | `OPENROUTER_API_KEY` | Varies |
| Google Gemini | `GOOGLE_API_KEY` | Free tier |
| Kimi/Moonshot | `KIMI_API_KEY` | ¥12/1M input |

### Get API Keys

1. **Anthropic:** https://console.anthropic.com/ → API Keys → Create
2. **OpenAI:** https://platform.openai.com/ → API Keys → Create
3. **DeepSeek:** https://platform.deepseek.com/ → API Keys → Create
4. **OpenRouter:** https://openrouter.ai/ → Keys → Create

## Step 3: Setup Telegram Bot

### Create Bot

1. Open **Telegram**
2. Search **@BotFather**
3. Send `/newbot`
4. Enter bot name (e.g., "My AI Agent")
5. Enter username (must end with `_bot`)
6. Copy the **token**

### Get User ID

1. Search **@userinfobot** on Telegram
2. Send any message
3. Copy your **User ID**

### Configure Hermes

```bash
# Interactive gateway setup (recommended)
hermes gateway setup
# Select Telegram, enter token and user ID

# Or manual
hermes config set gateway.platforms.telegram.token YOUR_BOT_TOKEN
hermes config set gateway.platforms.telegram.allowedUsers '["YOUR_USER_ID"]'
```

## Step 4: Start Agent

```bash
# Install as background service
hermes gateway install

# Start service
hermes gateway start

# Check status
hermes gateway status
```

### Or Run Foreground (Testing)

```bash
hermes gateway run
```

## Step 5: Test

1. Open Telegram
2. Find your bot
3. Send `/start`
4. Send any message

## Telegram Commands

```
/start          — Start conversation
/help           — Show commands
/model          — View/change model
/skills         — List skills
/status         — Session info
/new            — New session
/voice on       — Voice mode
/yolo           — Toggle auto-approve
```

## Systemd Service

Hermes Gateway auto-creates systemd service. Manage with:

```bash
# Status
hermes gateway status

# Start/Stop/Restart
hermes gateway start
hermes gateway stop
hermes gateway restart

# View logs
sudo journalctl -u hermes-gateway -f
```

### Manual Systemd (if needed)

```bash
sudo tee /etc/systemd/system/hermes.service << 'EOF'
[Unit]
Description=Hermes AI Agent
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/.hermes
ExecStart=/usr/bin/hermes gateway run
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable hermes
sudo systemctl start hermes
```

## CLI Reference

```bash
hermes                        # Interactive chat
hermes chat -q "question"     # Single query
hermes setup                  # Setup wizard
hermes model                  # Change model
hermes doctor                 # Health check
hermes config                 # Show config
hermes config edit            # Edit config.yaml
hermes gateway setup          # Setup Telegram/Discord
hermes gateway start          # Start gateway
hermes gateway status         # Check status
hermes skills list            # List skills
hermes tools                  # Manage tools
hermes cron list              # List cron jobs
hermes update                 # Update Hermes
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `hermes: not found` | `npm install -g hermes-agent` |
| API key invalid | `hermes auth` or check `.env` |
| Telegram not connecting | `hermes gateway restart` |
| Model error | `hermes model` to change |
| Gateway dies on logout | `sudo loginctl enable-linger $USER` |
| Config not applying | `hermes gateway restart` |

## Config Files

```
~/.hermes/config.yaml    # Main config
~/.hermes/.env           # API keys
~/.hermes/skills/        # Installed skills
~/.hermes/logs/          # Gateway logs
```

## License

MIT
