# OpenClaw — VPS Setup Guide 🐾

Complete guide to deploy OpenClaw AI agent on VPS with Telegram bot.

**GitHub:** https://github.com/nousresearch/openclaw

## Quick Start

```bash
# Clone & install
git clone https://github.com/nousresearch/openclaw.git ~/openclaw
cd ~/openclaw
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# Configure
cp config.example.yaml config.yaml
nano config.yaml  # Add API keys + Telegram token

# Run
python main.py
```

## VPS Requirements

| Spec | Minimum | Recommended |
|------|---------|-------------|
| OS | Ubuntu 22.04 | Ubuntu 22.04 LTS |
| RAM | 2 GB | 4 GB+ |
| CPU | 1 core | 2 core+ |
| Python | 3.10+ | 3.11+ |
| Disk | 10 GB | 20 GB+ |

## Installation

### Step 1: System Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git python3 python3-pip python3-venv
```

### Step 2: Clone OpenClaw

```bash
git clone https://github.com/nousresearch/openclaw.git ~/openclaw
cd ~/openclaw
```

### Step 3: Python Environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

### Step 4: Install Telegram Dependencies

```bash
pip install python-telegram-bot httpx aiohttp
```

### Step 5: Configuration

```bash
cp config.example.yaml config.yaml
nano config.yaml
```

## Configuration

### Full Config Example

```yaml
# ============================================
# Agent
# ============================================
agent:
  name: "MyAgent"
  description: "My AI agent"
  model: "claude-sonnet-4-20250514"
  provider: "anthropic"
  system_prompt: |
    You are a helpful AI agent.
    Be concise and actionable.
    Execute tasks first, explain after.

# ============================================
# Providers
# ============================================
providers:
  anthropic:
    api_key: "${ANTHROPIC_API_KEY}"
    model: "claude-sonnet-4-20250514"
    max_tokens: 4096
  openai:
    api_key: "${OPENAI_API_KEY}"
    model: "gpt-4o"
    max_tokens: 4096
  deepseek:
    api_key: "${DEEPSEEK_API_KEY}"
    model: "deepseek-chat"
    base_url: "https://api.deepseek.com/v1"
    max_tokens: 4096
  kimi:
    api_key: "${KIMI_API_KEY}"
    model: "moonshot-v1-8k"
    base_url: "https://api.moonshot.cn/v1"
    max_tokens: 4096
  openrouter:
    api_key: "${OPENROUTER_API_KEY}"
    model: "anthropic/claude-sonnet-4"

fallback:
  - anthropic
  - deepseek
  - openai

# ============================================
# Telegram
# ============================================
telegram:
  enabled: true
  token: "${TELEGRAM_BOT_TOKEN}"
  allowed_users:
    - "${TELEGRAM_USER_ID}"
  mode: "polling"

# ============================================
# Tools
# ============================================
tools:
  terminal: {enabled: true}
  file: {enabled: true}
  web_search: {enabled: true}
  browser: {enabled: false}

# ============================================
# Skills & Memory
# ============================================
skills:
  directory: "./skills"
  auto_load: true

memory:
  enabled: true
  provider: "local"
  max_entries: 1000

logging:
  level: "INFO"
  file: "./logs/agent.log"
```

### Environment Variables

```bash
# ~/.bashrc or ~/.profile
export ANTHROPIC_API_KEY=sk-ant-...port OPENAI_API_KEY=*** DEEPSEEK_API_KEY=*** KIMI_API_KEY=sk-...port OPENROUTER_API_KEY=sk-or-...port TELEGRAM_BOT_TOKEN=123456789:ABCdef...port TELEGRAM_USER_ID=123456789
```

## Telegram Bot Setup

### Step 1: Create Bot

1. Open **Telegram**
2. Search **@BotFather**
3. Send `/newbot`
4. Enter bot name: `My AI Agent` (bebas)
5. Enter username: `myaiagent_bot` (harus `_bot`)
6. **Copy token**: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`

### Step 2: Get User ID

1. Search **@userinfobot** di Telegram
2. Kirim pesan apapun
3. **Copy User ID**: `123456789`

### Step 3: Configure OpenClaw

Edit `~/openclaw/config.yaml`:

```yaml
telegram:
  enabled: true
  token: "123456789:ABCdefGHIjklMNOpqrsTUVwxyz"
  allowed_users:
    - "123456789"
  mode: "polling"
```

### Step 4: Create Telegram Agent Script

Buat file `~/openclaw/telegram_agent.py`:

```python
#!/usr/bin/env python3
"""
OpenClaw Telegram Agent
Runs AI agent connected to Telegram.
"""
import os
import asyncio
import logging
from telegram import Update
from telegram.ext import (
    Application, 
    CommandHandler, 
    MessageHandler, 
    filters,
    ContextTypes,
)

# Config
TELEGRAM_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
ALLOWED_USERS = [int(x) for x in os.getenv("TELEGRAM_USER_ID", "0").split(",")]

# Logging
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)


async def call_llm(message: str, provider: str = "anthropic") -> str:
    """Call LLM API with fallback."""
    import httpx
    
    providers = {
        "anthropic": {
            "url": "https://api.anthropic.com/v1/messages",
            "headers": {
                "x-api-key": os.getenv("ANTHROPIC_API_KEY", ""),
                "anthropic-version": "2023-06-01",
                "content-type": "application/json",
            },
            "body": {
                "model": "claude-sonnet-4-20250514",
                "max_tokens": 4096,
                "messages": [{"role": "user", "content": message}],
            },
            "extract": lambda d: d["content"][0]["text"],
        },
        "deepseek": {
            "url": "https://api.deepseek.com/v1/chat/completions",
            "headers": {
                "Authorization": f"Bearer {os.getenv('DEEPSEEK_API_KEY', '')}",
                "Content-Type": "application/json",
            },
            "body": {
                "model": "deepseek-chat",
                "messages": [{"role": "user", "content": message}],
                "max_tokens": 4096,
            },
            "extract": lambda d: d["choices"][0]["message"]["content"],
        },
    }
    
    # Try primary, then fallback
    for p in [provider] + [k for k in providers if k != provider]:
        if p not in providers:
            continue
        config = providers[p]
        if not config["headers"].get("x-api-key") and not config["headers"].get("Authorization", "").replace("Bearer ", ""):
            continue
        try:
            async with httpx.AsyncClient(timeout=60) as client:
                resp = await client.post(
                    config["url"],
                    headers=config["headers"],
                    json=config["body"],
                )
                resp.raise_for_status()
                return config["extract"](resp.json())
        except Exception as e:
            logger.warning(f"Provider {p} failed: {e}")
            continue
    
    return "Error: All providers failed"


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /start command."""
    await update.message.reply_text(
        "🤖 *OpenClaw Agent*\n\n"
        "Kirim pesan apapun dan saya akan merespon.\n\n"
        "Commands:\n"
        "/start — Mulai\n"
        "/help — Bantuan\n"
        "/model — Lihat/ganti model\n"
        "/reset — Reset session\n",
        parse_mode="Markdown",
    )


async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /help command."""
    await update.message.reply_text(
        "📖 *Commands:*\n\n"
        "/start — Mulai\n"
        "/help — Bantuan\n"
        "/model [nama] — Ganti model (anthropic/deepseek)\n"
        "/reset — Reset conversation\n\n"
        "Kirim pesan biasa untuk chat.",
        parse_mode="Markdown",
    )


async def set_model(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /model command."""
    if context.args:
        model = context.args[0].lower()
        if model in ("anthropic", "deepseek", "openai", "kimi"):
            context.user_data["provider"] = model
            await update.message.reply_text(f"✅ Model: *{model}*", parse_mode="Markdown")
        else:
            await update.message.reply_text(f"❌ Model tidak dikenal: {model}")
    else:
        current = context.user_data.get("provider", "anthropic")
        await update.message.reply_text(
            f"Model saat ini: *{current}*\n\n"
            "Ganti dengan: /model anthropic atau /model deepseek",
            parse_mode="Markdown",
        )


async def reset(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /reset command."""
    context.user_data.clear()
    await update.message.reply_text("🔄 Session reset.")


async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle incoming messages."""
    user_id = update.effective_user.id
    
    # Check authorization
    if user_id not in ALLOWED_USERS:
        await update.message.reply_text("❌ Unauthorized")
        logger.warning(f"Unauthorized user: {user_id}")
        return
    
    message = update.message.text
    if not message:
        return
    
    # Show typing indicator
    await update.message.chat.send_action("typing")
    
    # Get provider
    provider = context.user_data.get("provider", "anthropic")
    
    # Call LLM
    try:
        response = await call_llm(message, provider)
        
        # Split long messages (Telegram limit: 4096 chars)
        if len(response) > 4000:
            for i in range(0, len(response), 4000):
                await update.message.reply_text(response[i:i+4000])
        else:
            await update.message.reply_text(response)
    except Exception as e:
        logger.error(f"Error: {e}")
        await update.message.reply_text(f"❌ Error: {str(e)[:100]}")


def main():
    """Main function."""
    print("=" * 50)
    print("  OpenClaw Telegram Agent")
    print("=" * 50)
    print(f"Allowed users: {ALLOWED_USERS}")
    print()
    
    # Build application
    app = Application.builder().token(TELEGRAM_TOKEN).build()
    
    # Add handlers
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(CommandHandler("model", set_model))
    app.add_handler(CommandHandler("reset", reset))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    # Run
    print("Bot started! Press Ctrl+C to stop.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
```

### Step 5: Create .env File

```bash
cat > ~/openclaw/.env << 'EOF'
ANTHROPIC_API_KEY=sk-ant-...
DEEPSEEK_API_KEY=sk-...
TELEGRAM_BOT_TOKEN=123456789:ABCdef...
TELEGRAM_USER_ID=123456789
EOF
```

### Step 6: Test Run

```bash
cd ~/openclaw
source venv/bin/activate

# Load environment
export $(cat .env | xargs)

# Run agent
python telegram_agent.py
```

Buka Telegram, cari bot kamu, kirim `/start`.

## Systemd Service (Auto-Start)

### Create Service

```bash
sudo tee /etc/systemd/system/openclaw-telegram.service << 'EOF'
[Unit]
Description=OpenClaw Telegram Agent
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/openclaw
EnvironmentFile=/root/openclaw/.env
ExecStart=/root/openclaw/venv/bin/python telegram_agent.py
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

### Start Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable auto-start on boot
sudo systemctl enable openclaw-telegram

# Start now
sudo systemctl start openclaw-telegram

# Check status
sudo systemctl status openclaw-telegram
```

### Manage Service

```bash
# Stop
sudo systemctl stop openclaw-telegram

# Restart
sudo systemctl restart openclaw-telegram

# View logs
sudo journalctl -u openclaw-telegram -f

# View last 50 lines
sudo journalctl -u openclaw-telegram -n 50
```

## Telegram Commands

After bot is running:

```
/start          — Start conversation
/help           — Show commands
/model          — View current model
/model deepseek — Switch to DeepSeek
/model anthropic — Switch to Claude
/reset          — Reset conversation
```

## API Key Setup

### Anthropic (Claude)

1. https://console.anthropic.com/
2. API Keys → Create Key
3. Copy: `sk-ant-...`

### DeepSeek (Murah!)

1. https://platform.deepseek.com/
2. API Keys → Create
3. Copy: `sk-...`

### OpenAI

1. https://platform.openai.com/
2. API Keys → Create
3. Copy: `sk-proj-...`

### Kimi (Moonshot)

1. https://platform.moonshot.cn/
2. API Key Management → Create
3. Copy: `sk-...`

### OpenRouter

1. https://openrouter.ai/
2. Keys → Create
3. Copy: `sk-or-...`

## Custom Agent Features

### Add Memory

```python
# In telegram_agent.py, add conversation history
from collections import defaultdict

conversation_history = defaultdict(list)

async def handle_message(update, context):
    user_id = update.effective_user.id
    message = update.message.text
    
    # Add to history
    conversation_history[user_id].append({"role": "user", "content": message})
    
    # Keep last 20 messages
    if len(conversation_history[user_id]) > 20:
        conversation_history[user_id] = conversation_history[user_id][-20:]
    
    # Call with history
    response = await call_llm_with_history(conversation_history[user_id])
    
    conversation_history[user_id].append({"role": "assistant", "content": response})
```

### Add Image Analysis

```python
async def handle_photo(update, context):
    """Handle photo messages."""
    photo = update.message.photo[-1]
    file = await context.bot.get_file(photo.file_id)
    
    # Download image
    import tempfile
    with tempfile.NamedTemporaryFile(suffix=".jpg", delete=False) as f:
        await file.download_to_drive(f.name)
        
        # Analyze with vision model
        response = await analyze_image(f.name)
        await update.message.reply_text(response)
```

### Add Voice Messages

```python
async def handle_voice(update, context):
    """Handle voice messages."""
    voice = update.message.voice
    file = await context.bot.get_file(voice.file_id)
    
    # Download and transcribe
    import tempfile
    with tempfile.NamedTemporaryFile(suffix=".ogg", delete=False) as f:
        await file.download_to_drive(f.name)
        text = await transcribe_audio(f.name)
        
        # Process as text
        response = await call_llm(text)
        await update.message.reply_text(response)
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `ModuleNotFoundError` | `source venv/bin/activate && pip install -r requirements.txt` |
| `API key invalid` | Check `.env` file |
| `Telegram not connecting` | Verify token, check `curl https://api.telegram.org/bot<TOKEN>/getMe` |
| `Unauthorized user` | Check `TELEGRAM_USER_ID` in `.env` |
| `Permission denied` | Run with `sudo` |
| Service won't start | `sudo journalctl -u openclaw-telegram -e` |
| High memory | Reduce `max_tokens` in config |

## Pricing Reference

| Provider | Model | Input/1M | Output/1M |
|----------|-------|----------|-----------|
| Anthropic | Claude Sonnet 4 | $3 | $15 |
| Anthropic | Claude Haiku | $0.25 | $1.25 |
| OpenAI | GPT-4o | $2.50 | $10 |
| OpenAI | GPT-4o-mini | $0.15 | $0.60 |
| DeepSeek | DeepSeek-V3 | $0.14 | $0.28 |

## License

MIT
