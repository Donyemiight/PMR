# PMR — Prediction Market Research

> A skill that lets any AI agent **discover, query, analyze, and report on** on-chain prediction markets. Composes with on-chain tools and an off-chain metadata aggregator to deliver structured market intelligence in natural language.

**Live demo:** [https://predictmr.lovable.app](https://predictmr.lovable.app)

## What it does

PMR gives an agent six capabilities, each backed by a dedicated reference doc:

| Capability | Description | Reference |
|------------|-------------|-----------|
| 🔍 Discover | List live prediction markets, filter by category / status / keyword | `references/discovery.md` |
| 📊 Query | Read live state — prices, reserves, volume, holders, resolution status | `references/query.md` |
| 💼 Positions | Track a wallet's positions across markets, compute PnL, find claimable winnings | `references/positions.md` |
| 📈 Analyze | Probability trend (24h / 7d / 30d), liquidity profile, holder concentration (HHI) | `references/analysis.md` |
| 📝 Report | Compose a markdown research report with Overview, Trend, Risks, Verdict | `references/report.md` |
| ⚙️ Execute | Buy / sell / claim — gated by private-key + network + balance pre-checks | `references/execute.md` |

## Network

- **Pharos mainnet** (default) — chain ID `1672`, native token `PROS`
- **Pharos Atlantic testnet** (optional) — chain ID `688689`, native token `PHRS`

Network config lives in `assets/networks.json`. The default is mainnet; the agent switches to testnet only when the user explicitly says so.

## Framework

- **Required binary:** `cast` (Foundry)
- **Optional binaries:** `curl`, `jq`
- **Compatible with:** any agent runtime that can execute shell commands and read markdown (e.g. Claude Code, OpenAI function-calling agents, custom orchestrators).

The skill itself is framework-agnostic — it is a markdown procedure file (`SKILL.md`) plus a set of references. No SDK or runtime lock-in.

## Installation

```bash
# Clone the repo into your skills directory
git clone https://github.com/Donyemiight/PMR.git

# Or download the ZIP from the repo and unzip it
```

Then point your agent at `SKILL.md` (or copy the directory into your framework's skills folder, e.g. `~/.claude/skills/` for Claude Code, `~/.codex/skills/` for Codex, `~/.openclaw/skills/` for OpenClaw).

### Prerequisite: Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.zshenv && foundryup
cast --version
```

## How to use

Ask the agent natural-language questions. Examples:

> "What prediction markets are live on Pharos?"

> "What's the current YES price on the BTC market?"

> "Show positions for wallet `0xa11ce42f…`."

> "Analyze the Pharos TVL market."

> "Write a research report on the Fed rate market."

The agent loads `SKILL.md`, routes the intent to the right capability, and reads the matching `references/<x>.md` to execute. Read-only requests run with no private key; write requests (buy / sell / claim) trigger standard pre-checks.

## Layout

```
PMR/
├── SKILL.md                       # Main entry — frontmatter + procedure
├── assets/
│   ├── networks.json              # RPC + chain IDs (mainnet default + testnet)
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

## How it works

1. **Read by default.** Every request starts with read-only operations (`cast call` against the market contract, plus optional aggregator lookups for metadata).
2. **Compose, don't duplicate.** PMR reuses network config from `assets/networks.json` and the standard `MarketView` ABI in `assets/abi/MarketView.json`.
3. **Write with gates.** Buy / sell / claim require the private-key + network + balance + allowance pre-checks (see `references/execute.md`).
4. **Degrade gracefully.** If the off-chain aggregator is down, PMR falls back to on-chain reads and surfaces a note to the user.

## Security

- Never logs or prints `$PRIVATE_KEY`.
- Always confirms the target network before any write op.
- Default slippage cap: 2%. Markets with estimated slippage above this are surfaced and the user is asked to confirm.
- Default approval is **exact amount**, not infinite.

## License

MIT-0 — see [LICENSE](./LICENSE).
