# OpenClaw — VPS Setup Guide 🐾

Complete guide to deploy OpenClaw AI agent on VPS.

**GitHub:** https://github.com/nousresearch/openclaw

## What is OpenClaw?

OpenClaw is an open-source AI agent framework by Nous Research. Modular, customizable, runs locally or on VPS.

## VPS Requirements

| Spec | Minimum | Recommended |
|------|---------|-------------|
| OS | Ubuntu 22.04 | Ubuntu 22.04 LTS |
| RAM | 2 GB | 4 GB+ |
| CPU | 1 core | 2 core+ |
| Python | 3.10+ | 3.11+ |
| Disk | 10 GB | 20 GB+ |

## Installation

### Quick Install

```bash
# Clone repository
git clone https://github.com/nousresearch/openclaw.git ~/openclaw
cd ~/openclaw

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Copy config template
cp config.example.yaml config.yaml

# Edit config
nano config.yaml
```

### Systemd Service

```bash
sudo tee /etc/systemd/system/openclaw.service << 'EOF'
[Unit]
Description=OpenClaw AI Agent
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/openclaw
ExecStart=/root/openclaw/venv/bin/python main.py
Restart=always
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable openclaw
sudo systemctl start openclaw
```

## Configuration

### Config File

Edit `~/openclaw/config.yaml`:

```yaml
# ============================================
# Agent Configuration
# ============================================
agent:
  name: "MyAgent"
  description: "My AI agent"
  
  # Primary model
  model: "claude-sonnet-4-20250514"
  provider: "anthropic"
  
  # System prompt
  system_prompt: |
    You are a helpful AI agent.
    Be concise and actionable.
    Execute tasks first, explain after.

# ============================================
# Providers
# ============================================
providers:
  anthropic:
    api_key: "sk-ant-..."
    model: "claude-sonnet-4-20250514"
    max_tokens: 4096
    
  openai:
    api_key: "sk-..."
    model: "gpt-4o"
    max_tokens: 4096
    
  deepseek:
    api_key: "sk-..."
    model: "deepseek-chat"
    base_url: "https://api.deepseek.com/v1"
    max_tokens: 4096
    
  kimi:
    api_key: "sk-..."
    model: "moonshot-v1-8k"
    base_url: "https://api.moonshot.cn/v1"
    max_tokens: 4096
    
  openrouter:
    api_key: "sk-or-..."
    model: "anthropic/claude-sonnet-4"
    
  ollama:  # Local models
    api_key: "dummy"
    model: "llama3.1"
    base_url: "http://localhost:11434/v1"

# Provider fallback chain
fallback:
  - anthropic
  - deepseek
  - openai

# ============================================
# Telegram Integration
# ============================================
telegram:
  enabled: true
  token: "YOUR_BOT_TOKEN"
  allowed_users:
    - "YOUR_USER_ID"
  mode: "polling"  # "polling" or "webhook"
  # webhook_url: "https://your-domain.com/webhook"

# ============================================
# Tools
# ============================================
tools:
  terminal:
    enabled: true
    allowed_commands:
      - "ls"
      - "cat"
      - "grep"
      - "find"
      - "git"
      - "python3"
      - "pip"
      - "curl"
    blocked_commands:
      - "rm -rf /"
      - "sudo rm"
      
  file:
    enabled: true
    read: true
    write: true
    allowed_paths:
      - "~/"
      - "/tmp/"
      
  browser:
    enabled: false
    
  web_search:
    enabled: true
    provider: "duckduckgo"

# ============================================
# Skills
# ============================================
skills:
  directory: "./skills"
  auto_load: true

# ============================================
# Memory
# ============================================
memory:
  enabled: true
  provider: "local"
  max_entries: 1000

# ============================================
# Logging
# ============================================
logging:
  level: "INFO"
  file: "./logs/agent.log"
  max_size_mb: 100
  backup_count: 5
```

### Environment Variables

Instead of hardcoding API keys in config, use environment variables:

```bash
# ~/.bashrc or ~/.profile
export ANTHROPIC_API_KEY=sk-a...port OPENAI_API_KEY=sk-...
export DEEPSEEK_API_KEY=sk-...
export TELEGRAM_BOT_TOKEN=123456789:ABC...
export TELEGRAM_USER_ID=123456789
```

Then reference in config:

```yaml
providers:
  anthropic:
    api_key: "${ANTHROPIC_API_KEY}"
```

## API Key Setup

### Anthropic (Claude)

1. Go to https://console.anthropic.com/
2. Sign up / Login
3. Go to API Keys
4. Click Create Key
5. Copy key (format: `sk-ant-...`)

### OpenAI (GPT)

1. Go to https://platform.openai.com/
2. Sign up / Login
3. Go to API Keys
4. Click Create new secret key
5. Copy key (format: `sk-proj-...`)

### DeepSeek (Cheap!)

1. Go to https://platform.deepseek.com/
2. Sign up / Login
3. Go to API Keys
4. Click Create API Key
5. Copy key (format: `sk-...`)

### Kimi (Moonshot)

1. Go to https://platform.moonshot.cn/
2. Sign up / Login
3. Go to API Key Management
4. Click Create
5. Copy key (format: `sk-...`)

### OpenRouter (100+ Models)

1. Go to https://openrouter.ai/
2. Sign up / Login
3. Go to Keys
4. Click Create Key
5. Copy key (format: `sk-or-...`)

## Telegram Bot Setup

### Step 1: Create Bot

1. Open Telegram
2. Search **@BotFather**
3. Send `/newbot`
4. Enter bot name (e.g., "My OpenClaw Agent")
5. Enter username (must end with `_bot`)
6. Copy the token

### Step 2: Get User ID

1. Search **@userinfobot** on Telegram
2. Send any message
3. Copy your User ID

### Step 3: Configure

Edit `~/openclaw/config.yaml`:

```yaml
telegram:
  enabled: true
  token: "123456789:ABCdefGHIjklMNOpqrsTUVwxyz"
  allowed_users:
    - "123456789"
  mode: "polling"
```

### Step 4: Start

```bash
cd ~/openclaw
source venv/bin/activate
python main.py
```

## Custom Tools

### Add Terminal Tool

```yaml
tools:
  terminal:
    enabled: true
    allowed_commands: ["ls", "cat", "grep", "git", "python3"]
```

### Add File Tool

```yaml
tools:
  file:
    enabled: true
    read: true
    write: true
    allowed_paths: ["~/", "/tmp/"]
```

### Add Web Search

```yaml
tools:
  web_search:
    enabled: true
    provider: "duckduckgo"  # or "google", "bing"
```

## Skills

Create skills in `~/openclaw/skills/`:

```markdown
---
name: my-skill
description: My custom skill
---

# My Skill

When user asks about X, do Y.

## Steps
1. Check Z
2. Run W
3. Report result
```

## Running

### Interactive Mode

```bash
cd ~/openclaw
source venv/bin/activate
python main.py
```

### With Telegram

```bash
python main.py --telegram
```

### Custom Config

```bash
python main.py --config /path/to/config.yaml
```

### Background (Production)

```bash
# Start as service
sudo systemctl start openclaw

# Check status
sudo systemctl status openclaw

# View logs
sudo journalctl -u openclaw -f
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `ModuleNotFoundError` | `source venv/bin/activate && pip install -r requirements.txt` |
| `API key invalid` | Check key in config.yaml or env var |
| `Telegram not connecting` | Verify token and user ID |
| `Permission denied` | Run with `sudo` or fix permissions |
| `Config not loading` | Check YAML syntax with `python -c "import yaml; yaml.safe_load(open('config.yaml'))"` |
| High memory | Reduce `max_tokens` in provider config |

## Pricing Reference

| Provider | Model | Input/1M | Output/1M |
|----------|-------|----------|-----------|
| Anthropic | Claude Sonnet 4 | $3 | $15 |
| Anthropic | Claude Haiku | $0.25 | $1.25 |
| OpenAI | GPT-4o | $2.50 | $10 |
| OpenAI | GPT-4o-mini | $0.15 | $0.60 |
| DeepSeek | DeepSeek-V3 | $0.14 | $0.28 |
| Kimi | Moonshot v1 | ¥12 | ¥12 |

## License

MIT
