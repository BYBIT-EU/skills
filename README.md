# Bybit EU AI Trading Skill

Trade on Bybit EU using natural language. Tell any AI assistant one sentence, and it can execute trades, check markets, manage spot and spot-margin positions, and more — zero installation required.

**Version:** 1.0.1 | **License:** MIT | **Scope:** Spot + Spot-Margin only (no derivatives, options, or earn)

## How It Works

Copy the following line and send it to your AI assistant:

```
Please read https://raw.githubusercontent.com/BYBIT-EU/skills/main/SKILL.md, save it as a skill, and help me trade on Bybit EU.
```

The AI downloads and installs the skill automatically — then you can start trading in natural language. No npm packages, no CLI tools, no config files.

## Supported AI Platforms

Works with any AI assistant that can read files or URLs:

- OpenClaw
- Claude (Code, Desktop, API)
- ChatGPT
- Gemini
- Cursor / Windsurf
- Codex

## Capabilities

| Module | What Users Can Do |
|--------|-------------------|
| **Market** | Real-time prices, klines, orderbook depth, tickers, recent trades |
| **Spot** | Market/limit orders, cancel, amend, order history, open orders |
| **Margin** | Spot-margin borrow/repay, toggle margin mode, margin coin state, max borrowable |
| **Account** | Balances, fee rates, collateral settings, account info |
| **Asset** | Deposit addresses, withdraw records, deposit records, coin info, total members assets, internal transfers |
| **Convert** | Crypto↔crypto and fiat↔crypto conversion (quote, execute, history) |
| **WebSocket** | Live orderbook, ticker, kline, and trade streams |

## Quick Start

### 1. Get an API Key

1. Log in to [Bybit EU](https://www.bybit.eu/app/user/api-management) → API Management → Create New Key
2. Enable **Read + Trade** permissions only — **never enable Withdraw for AI use**
3. Recommended: bind your IP and use a dedicated sub-account with limited balance

> **Note:** API keys cannot be created programmatically on Bybit EU (error 81007). You must create keys through the web UI at the URL above.

### 2. Configure Credentials

**Local CLI** (Claude Code, Cursor, etc.):

```bash
export BYBIT_EU_API_KEY="your_api_key"
export BYBIT_EU_API_SECRET="your_secret_key"
export BYBIT_EU_ENV="mainnet"   # or "testnet"
```

**OpenClaw** — use `.env` file:

```bash
# ~/.openclaw/.env
BYBIT_EU_API_KEY=your_api_key
BYBIT_EU_API_SECRET=your_secret_key
BYBIT_EU_ENV=mainnet
```

**Cloud AI** (ChatGPT, Gemini) — the AI will ask for credentials interactively and keep them in memory for the session only.

### 3. Start Trading

Just tell the AI what you want in natural language. The skill handles the rest.

## Scope

This skill covers **Spot and Spot-Margin trading only** on the Bybit EU Unified Trading Account.

The following product categories are **not available** on Bybit EU and are therefore outside the scope of this skill:

- Derivatives (linear/inverse perpetuals and futures)
- Options
- Earn products (flexible savings, staking, dual assets)
- Crypto loans
- Copy trading
- Trading bots
- Fiat P2P / OTC advertisement flows (note: fiat↔crypto *conversion* IS supported via the convert module)

## Security

| Feature | Description |
|---------|-------------|
| **Mainnet by default** | All operations target `api.bybit.eu` (real funds — writes require CONFIRM). A Testnet (`api-testnet.bybit.eu`) is also available for practice with no real funds. |
| **Trade confirmation** | Every write operation shows a structured summary card — user must type `CONFIRM` |
| **No Withdraw permission** | The skill never requests or exercises withdrawal authority |
| **API key masking** | Keys are displayed as first 5 + last 4 characters only |
| **Local HMAC signing** | Signatures computed locally — secrets never leave the user's device |
| **Prompt injection defense** | API response text fields are displayed but never executed |
| **Graceful degradation** | If a module fails to load, write operations are disabled (read-only fallback) |
| **IP restrictions** | Bybit EU blocks requests from US and China IP addresses (HTTP 403) |
| **HMAC only** | Bybit EU supports HMAC-SHA256 signing only; RSA keys are not supported |

## Auto Update

The skill includes a self-update mechanism. At session start it checks the manifest served at `https://api.bybit.eu/ai-manifest/skill/eu/manifest`. If a newer version is available, it downloads the updated files listed in the manifest and verifies SHA256 checksums before applying — keeping users on the latest version automatically.

## License

[MIT](LICENSE)
