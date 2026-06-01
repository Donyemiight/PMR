# Market Query (`pmr-query`)

Fetch live state for a single market: prices, reserves, volume, holders, resolution status. Read-only; never requires a private key.

## Resolve the Market Address

The user may give any of:
- Full address: `0xabc…`
- Slug: `btc-100k-eoy-2026`
- Question keyword

For the latter two, resolve via the aggregator:

```bash
curl -s "$AGG/markets?q=btc+100k" | jq -r '.markets[0].address'
```

For the address form, use it directly. Validate format first.

## Read Market State

The contract exposes a standard `MarketView` ABI (see `assets/abi/MarketView.json`). Read with `cast call`:

```bash
MARKET=0xabc...
RPC_URL=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)

# Metadata
NAME=$(cast call $MARKET "name()(string)" --rpc-url $RPC_URL)
DESC=$(cast call $MARKET "description()(string)" --rpc-url $RPC_URL)
CAT=$(cast call $MARKET  "category()(string)"  --rpc-url $RPC_URL)
END=$(cast call $MARKET  "endTime()(uint256)" --rpc-url $RPC_URL)

# Outcome count and labels
N=$(cast call $MARKET "outcomeCount()(uint256)" --rpc-url $RPC_URL)
# loop 0..N-1
LABEL_0=$(cast call $MARKET "outcomeLabel(uint256)(string)" 0 --rpc-url $RPC_URL)
LABEL_1=$(cast call $MARKET "outcomeLabel(uint256)(string)" 1 --rpc-url $RPC_URL)

# Prices (scaled 1e18 → divide by 1e18 to get probability 0..1)
P0=$(cast call $MARKET "price(uint256)(uint256)" 0 --rpc-url $RPC_URL)
P1=$(cast call $MARKET "price(uint256)(uint256)" 1 --rpc-url $RPC_URL)

# Liquidity + volume
RESERVE=$(cast call $MARKET "reserve()(uint256)"   --rpc-url $RPC_URL)
V24=$(cast call $MARKET "volume24h()(uint256)"      --rpc-url $RPC_URL)
TVL=$(cast call $MARKET "totalVolume()(uint256)"    --rpc-url $RPC_URL)
HOLDERS=$(cast call $MARKET "holderCount()(uint256)" --rpc-url $RPC_URL)

# Resolution
RESOLVED=$(cast call $MARKET "resolved()(bool)" --rpc-url $RPC_URL)
if [ "$RESOLVED" = "true" ]; then
  WIN=$(cast call $MARKET "winningOutcome()(uint8)" --rpc-url $RPC_URL)
fi
```

## Convert Scaled Values

Prices and reserves are `uint256` with 18 decimals. Convert to human-readable:

```bash
# probability 0.62
P0_HUMAN=$(echo "scale=4; $P0 / 1000000000000000000" | bc)

# $14,230.55
RESERVE_HUMAN=$(echo "scale=2; $RESERVE / 1000000000000000000" | bc)
```

## Output Format

Always show **at minimum**:

```
Market: <name> (<category>)
Status: <active | resolved with outcome X | expired unresolved>
Prices: <label 0> <P0%>  |  <label 1> <P1%>
Liquidity: $<reserve>
24h Volume: $<v24>
Total Volume: $<tvl>
Holders: <N>
Resolves: <endTime ISO-8601>
```

Then suggest next steps:
- "Want position tracking? Say `show my positions in this market`."
- "Want a full research report? Say `analyze this market`."

## Failure Handling

| Error | Cause | Action |
|-------|-------|--------|
| `invalid address` | Bad market address | Re-prompt user for correct address |
| Empty return on `name()` | No contract at address | Tell user the address isn't a market contract |
| `execution reverted` on `price()` | Market settled and price() disabled | Read `resolved()` and `winningOutcome()` instead |
| Aggregator returns no match | Slug not found | List nearby markets, ask user to pick |
