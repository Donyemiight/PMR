# PMR — Prediction Market Research

> An on-chain prediction market research skill for the **Pharos Agent Center**.
> Built for the [Pharos Agent Center Skill Builder Campaign](https://silken-muskox-24e.notion.site/pharos-agent-center-skill-builder-campaign).

PMR lets any Pharos-compatible AI agent (OpenClaw, Claude Code, Codex, etc.) **discover, query, analyze, and act on** on-chain prediction markets using natural language. It composes on top of the base `pharos-skill-engine` and reuses Foundry (`cast` / `forge`) for all on-chain operations.

## What it does

| Capability | Description | Reference |
|------------|-------------|-----------|
| 🔍 Discover | List active markets on Pharos, filter by category, status, or search keyword | `references/discovery.md` |
| 📊 Query | Read live state — prices, reserves, volume, holders, resolution status | `references/query.md` |
| 💼 Positions | Track a wallet's positions across markets, compute PnL, find claimable winnings | `references/positions.md` |
| 📈 Analyze | Probability trend (24h/7d/30d), liquidity profile (turnover band), holder concentration (HHI) | `references/analysis.md` |
| 📝 Report | Compose a full markdown research report from the above | `references/report.md` |
| ⚙️ Execute | Buy / sell / claim (write ops, gated by standard pre-checks) | `references/execute.md` |

## Layout

```
PMR/
├── SKILL.md                       # Main entry — frontmatter + procedure
├── assets/
│   ├── networks.json              # Pharos RPC + chain IDs + PMR contract addrs
│   ├── markets.json               # Category taxonomy + risk bands
│   └── abi/MarketView.json        # Standard market ABI used by all read calls
├── references/
│   ├── discovery.md
│   ├── query.md
│   ├── positions.md
│   ├── analysis.md
│   ├── report.md
│   └── execute.md
├── examples/
│   └── example-prompts.md
├── README.md
└── LICENSE                        # MIT-0
```

## Quick Start

### 1. Install (any Pharos-compatible agent framework)

```bash
# OpenClaw
mkdir -p ~/.openclaw/skills && cp -R . ~/.openclaw/skills/pmr/

# Claude Code
mkdir -p ~/.claude/skills && cp -R . ~/.claude/skills/pmr/

# Codex
mkdir -p ~/.codex/skills && cp -R . ~/.codex/skills/pmr/
```

Or use the Pharos skill engine installer:

```bash
npx skills add https://github.com/Donyemiight/PMR
```

### 2. Install Foundry (one-time)

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.zshenv && foundryup
cast --version
```

### 3. Try it

> "What prediction markets are live on Pharos testnet?"

> "Show me the current YES price on `0xabc…123`."

> "Analyze market `0xabc…123`."

> "Write me a research report on the top-volume market in the pharos-ecosystem category."

## How It Works

1. **Read by default.** Every PMR request starts with read-only operations (`cast call` against the market contract, plus optional aggregator lookups for metadata).
2. **Compose, don't duplicate.** PMR reuses Pharos network config from `assets/networks.json` and the standard `MarketView` ABI in `assets/abi/MarketView.json`.
3. **Write with gates.** Buy / sell / claim operations require the standard private-key + network + balance pre-checks (see `SKILL.md` and `references/execute.md`).
4. **Degrade gracefully.** If the off-chain aggregator is down, PMR falls back to on-chain reads and surfaces a note to the user.

## Supported Networks

- `atlantic-testnet` (default) — Pharos Atlantic testnet, chain ID 688689, native PHRS
- `mainnet` — Pharos mainnet, chain ID 1672, native PROS

## Security

- Never logs or prints `$PRIVATE_KEY`.
- Always confirms the target network before any write op (mainnet requires explicit `⚠️` consent).
- Default slippage cap: 2%. Markets with estimated slippage above this are surfaced and the user is asked to confirm.
- Default approval is **exact amount**, not infinite.

## License

MIT-0 — see [LICENSE](./LICENSE).
