# Installation

---

## Requirements

- Node.js ≥ 20.6.0
- An LLM API key (or OAuth login to a supported provider)

---

## Global npm Install (Recommended)

```bash
npm install -g @mariozechner/pi-coding-agent
pi
```

The `pi` command is immediately available after install.

---

## Binary Download

Pre-compiled standalone binaries are available on the [GitHub releases page](https://github.com/badlogic/pi-mono/releases). No Node.js installation required.

Available platforms:
- macOS arm64 (Apple Silicon)
- macOS x86_64 (Intel)
- Linux x86_64
- Linux arm64
- Linux arm64-musl (Alpine/containers)

Download and place in your `$PATH`:

```bash
chmod +x pi
sudo mv pi /usr/local/bin/pi
```

---

## First Run & Authentication

### API Key (Simplest)

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

Supported environment variables for all providers — see [LLM Providers](providers.md) for the full list.

### OAuth Login (Claude Pro/Max, GitHub Copilot, Gemini CLI)

```bash
pi
/login
```

Select your provider in the interactive picker. OAuth tokens are stored in `~/.pi/agent/auth.json`.

### API Key via `models.json`

Store API keys per-provider without environment variables:

```json
// ~/.pi/agent/models.json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-..."
    },
    "openai": {
      "apiKey": "!op read op://vault/openai/api-key"
    }
  }
}
```

Values prefixed with `!` are resolved as shell commands (useful for 1Password, Vault, etc.).

---

## Platform-Specific Setup

### Windows

Pi runs on Windows via Git Bash or WSL. See the project's `docs/windows.md` for:
- Terminal configuration (Windows Terminal recommended)
- Shell handling details
- WSL2 integration

### macOS

No special setup required. The `pi` binary supports both arm64 (Apple Silicon) and x86_64.

### Linux

Standard npm install works. For musl/Alpine systems (Docker), use the `arm64-musl` binary.

### Android / Termux

```bash
pkg install nodejs termux-api git
npm install -g @mariozechner/pi-coding-agent
mkdir -p ~/.pi/agent
echo "You are running on Android in Termux." > ~/.pi/agent/AGENTS.md
pi
```

See `docs/termux.md` in the repo for additional setup steps.

### tmux

Shift+Enter works in tmux with no configuration changes. See `docs/tmux.md` for key handling details.

---

## Nix / Guix

```bash
PI_PACKAGE_DIR=/path/to/writable/dir pi
```

The `PI_PACKAGE_DIR` environment variable overrides where pi stores its installed packages.

---

## HTTP Proxy

```bash
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080
pi
```

---

## Offline Mode

Start pi without any network operations:

```bash
PI_OFFLINE=1 pi
# or
pi --offline
```

---

## Building from Source

```bash
git clone https://github.com/badlogic/pi-mono.git
cd pi-mono
npm install
npm run build
./pi-test.sh   # Run pi from sources (must be run from repo root)
```

Build binary (requires Bun):

```bash
cd packages/coding-agent
npm run build:binary
# Binary at dist/pi
```

---

## Update

```bash
npm update -g @mariozechner/pi-coding-agent
# or
pi update
```

---

## Uninstall

```bash
pi uninstall
# or
npm uninstall -g @mariozechner/pi-coding-agent
```

---

## Configuration Paths

| Path | Purpose |
|------|---------|
| `~/.pi/settings.json` | Global settings |
| `.pi/settings.json` | Project-local settings |
| `~/.pi/agent/auth.json` | API keys and OAuth tokens |
| `~/.pi/agent/models.json` | Custom model definitions and provider API keys |
| `~/.pi/agent/sessions/` | Stored conversation sessions |
| `PI_CODING_AGENT_DIR` | Override the agent directory (replaces `~/.pi/agent`) |
