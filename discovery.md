# Market Discovery (`pmr-discover`)

Find active prediction markets on Pharos. Two paths: **on-chain** (source of truth, via `cast` against the registry) and **off-chain** (fast path, via the public aggregator). Use on-chain when stakes are involved; use the aggregator for browsing.

## On-Chain Discovery (Default)

The `marketRegistry` address is read from `assets/networks.json`:

```bash
RPC_URL=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
REGISTRY=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .pmr.marketRegistry' assets/networks.json)
```

### 1. List all markets (paginated)
```bash
cast call $REGISTRY "allMarketsLength()(uint256)" --rpc-url $RPC_URL
cast call $REGISTRY "marketAt(uint256)(address)" 0 --rpc-url $RPC_URL
cast call $REGISTRY "marketAt(uint256)(address)" 1 --rpc-url $RPC_URL
# ... iterate up to allMarketsLength()-1
```

### 2. Filter by status
```bash
# Returns true if the market is still tradeable (not resolved, not past endTime)
cast call $REGISTRY "isActive(address)(bool)" $MARKET --rpc-url $RPC_URL
```

### 3. Filter by category
```bash
# Returns array of market addresses in a category
cast call $REGISTRY "marketsByCategory(string)(address[])" "crypto-price" --rpc-url $RPC_URL
```

Valid category IDs come from `assets/markets.json#categories[].id`.

## Off-Chain Discovery (Aggregator)

Faster for browsing, supports full-text search and rich metadata:

```bash
AGG=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .pmr.aggregatorUrl' assets/networks.json)

# Top markets by 24h volume
curl -s "$AGG/markets?network=atlantic-testnet&sort=volume24h&limit=20" | jq '.markets[] | {address, question, yesPrice, volume24h}'

# Search by keyword
curl -s "$AGG/markets?q=bitcoin+100k" | jq '.markets'

# Filter by category
curl -s "$AGG/markets?category=crypto-price&active=true" | jq '.markets'
```

If the aggregator returns 4xx/5xx or times out (>5s), fall back to the on-chain path and surface a note to the user.

## Output Format

Present results as a table with these columns:

| Address | Question | YES | NO | 24h Vol | Liquidity | Ends |
|---------|----------|-----|----|---------|-----------|------|
| `0xabc…` | Will BTC hit 100k by EOY 2026? | 62% | 38% | $14,230 | $87,500 | 2026-12-31 |

## Failure Handling

| Symptom | Cause | Action |
|---------|-------|--------|
| `execution reverted` on `allMarketsLength` | Wrong network / wrong registry | Re-read `assets/networks.json`; confirm network |
| Aggregator timeout | Public endpoint down | Fall back to on-chain `allMarketsLength + marketAt` loop |
| Empty result set | Wrong category / no active markets | Show full list (no filter); ask user to pick |
| Address not a contract | `cast call` returns empty | Skip that index, continue iterating |
