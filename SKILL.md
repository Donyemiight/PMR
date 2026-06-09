---
name: pmr
description: |
  Use this skill whenever the user wants to discover, query, analyze, or report on on-chain prediction markets — including fetching live market data (prices, volumes, liquidity, outcomes), tracking a wallet's prediction market positions, generating probability-shift / trend / liquidity analyses, or producing a structured research report. Trigger on phrases like "prediction market", "PMR", "market research", "what's the price of YES on…", "show my positions in…", "analyze market…", "liquidity in market…", "research report on market…". Do NOT use for plain token transfers or generic balance checks — those are separate on-chain skills.
version: 0.1.0
requires:
  anyBins:
    - cast
    - curl
    - jq
---
# Prediction Market Research (PMR)

Research toolkit for on-chain prediction markets on the Pharos network. Discovers markets, queries live state (prices, volumes, liquidity, outcomes), tracks wallet positions, runs trend / probability / liquidity analyses, and produces structured research reports — all driven by natural language.

PMR uses the `cast` CLI (Foundry) for on-chain reads and writes, and an optional off-chain aggregator for market metadata. It is self-contained — no other skill is required to run it.

## Prerequisites

1. **Foundry installed** — `cast` must be on `$PATH`. If not, install it before any other action:
   ```bash
   curl -L https://foundry.paradigm.xyz | bash
   source ~/.zshenv && foundryup
   cast --version
   ```
2. **Network config available** — `assets/networks.json` ships both `mainnet` (default) and `atlantic-testnet` (testnet). The agent uses mainnet by default and switches to testnet only when the user explicitly says so.
3. **Optional — private key**: only required for write operations (placing bets, claiming winnings, providing liquidity). Read-only research never needs it.
4. **Optional — public data source**: PMR also pulls off-chain context (market metadata, descriptions, category tags) from a public aggregator. The skill degrades gracefully if the aggregator is unreachable.

## Network Configuration

```bash
# Read target network RPC URL (default: mainnet)
RPC_URL=$(jq -r '.networks[] | select(.name=="mainnet") | .rpcUrl' assets/networks.json)

# Read chain ID for verification
CHAIN_ID=$(jq -r '.networks[] | select(.name=="mainnet") | .chainId' assets/networks.json)
```

For `atlantic-testnet`, replace the selector value. The full table is in `assets/networks.json`.

## Capability Index

Load the corresponding reference file based on the user's intent.

| User Need | Capability | Detailed Instructions |
|-----------|------------|----------------------|
| "Find active markets" / "list markets" / "what markets exist" | `pmr-discover` | → `references/discovery.md` |
| "What's the price of YES/NO" / "current odds" / "market data" | `pmr-query` | → `references/query.md` |
| "Show my positions" / "what did I bet on" / "my PnL" | `pmr-positions` | → `references/positions.md` |
| "Analyze this market" / "trend / probability shift / liquidity analysis" | `pmr-analyze` | → `references/analysis.md` |
| "Write me a research report" / "summarize this market for me" | `pmr-report` | → `references/report.md` |
| "Place a bet" / "claim winnings" / "add liquidity" (write) | `pmr-execute` | → `references/execute.md` |

## General Workflow

For every PMR request, follow this sequence:

1. **Identify intent** — match the user's phrasing to a capability in the table above.
2. **Resolve target** — confirm the market identifier (address, slug, or category) and the network (`mainnet` by default). If ambiguous, ask the user.
3. **Read-only first** — always start with read-only operations (`pmr-query`, `pmr-discover`, `pmr-positions`). Only escalate to `pmr-execute` when the user explicitly asks to act.
4. **Confirm before write** — for any `pmr-execute` action, run the standard pre-checks below.
5. **Synthesize** — present results in a human-readable form. Never dump raw `cast` output.

## Write Operation Pre-checks (Required for `pmr-execute`)

When a user asks to place a bet, claim winnings, or add liquidity:

### 1. Private Key Check
```bash
[ -n "$PRIVATE_KEY" ] && echo "PRIVATE_KEY is set" || echo "PRIVATE_KEY is not set"
```
If unset, prompt: `export PRIVATE_KEY=0x...` and stop.

### 2. Derive Sender Address
```bash
cast wallet address --private-key $PRIVATE_KEY
```

### 3. Network Confirmation
Show the user:
```
Detected private key address: 0x1234…abcd
Target network: Atlantic Testnet (atlantic-testnet)
Proceed with this account on this network?
```
For mainnet, prepend a `⚠️` warning and require explicit confirmation.

### 4. Balance + Allowance Check
```bash
cast balance $SENDER --rpc-url $RPC_URL
cast call $MARKET "allowance(address,address)(uint256)" $SENDER $MARKET --rpc-url $RPC_URL
```
Reject early if balance < stake or allowance < stake.

## Output Contract

Every PMR run should produce:

- A **summary line** (one sentence, e.g. *"Market X on Atlantic testnet: YES 62% / NO 38%, $14,230 liquidity, 1,402 holders."*)
- A **structured table** with the relevant fields (prices, volume, top holders, etc.).
- A **next-step suggestion** (e.g. *"Want a full research report? Say 'analyze'."*).

For `pmr-report`, additionally produce a markdown report with sections: Overview, Current State, Historical Trend, Liquidity & Holders, Risks, Verdict.

## Failure Handling

| Error | Likely Cause | Action |
|-------|--------------|--------|
| `invalid address` | Market address malformed | Validate 0x + 40 hex; ask user to re-paste |
| `execution reverted` | Market settled / paused / wrong method | Decode revert reason; surface to user |
| `insufficient funds` | Wallet too thin for stake | Show balance, suggest top-up |
| Aggregator 4xx/5xx | Off-chain metadata source down | Fall back to on-chain `name()` / `symbol()` / `description()` |
| `nonce too low` | Concurrent txs | Suggest waiting 1 block or bumping nonce manually |
| No markets found | Wrong category / network | Re-list markets on the current network; confirm with user |

## Security Reminders

- **Never log or print `$PRIVATE_KEY`.** Always pass via `--private-key $PRIVATE_KEY` argument.
- **Always confirm network** before a write op. Atlantic testnet is the default; mainnet requires explicit consent.
- **Always show slippage estimate** before a market order. If estimated slippage > 2%, warn the user.
- **Never auto-approve infinite allowances** by default. Default to the exact stake amount.

## License

MIT-0 — free to use, modify, and redistribute. No attribution required.
