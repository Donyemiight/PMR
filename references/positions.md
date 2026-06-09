# Position Tracking (`pmr-positions`)

Show a wallet's prediction-market positions: outcome balances, current value, unrealized PnL, claimable winnings.

## Inputs

- `wallet` — the address to inspect (EIP-55 checksummed or lowercase 0x…40). If the user supplied `$PRIVATE_KEY`, derive via `cast wallet address` instead of asking.
- `market` (optional) — if omitted, show positions across all markets in the registry.

## Read Balances Per Market

```bash
SENDER=0x123...
MARKET=0xabc...
N=$(cast call $MARKET "outcomeCount()(uint256)" --rpc-url $RPC_URL)

# Loop 0..N-1
BAL_0=$(cast call $MARKET "balanceOf(address,uint256)(uint256)" $SENDER 0 --rpc-url $RPC_URL)
BAL_1=$(cast call $MARKET "balanceOf(address,uint256)(uint256)" $SENDER 1 --rpc-url $RPC_URL)
P0=$(cast call $MARKET "price(uint256)(uint256)" 0 --rpc-url $RPC_URL)
P1=$(cast call $MARKET "price(uint256)(uint256)" 1 --rpc-url $RPC_URL)
```

## Compute Current Value

For a CPMM-style market, the *current value* (in collateral) of `B` outcome tokens is roughly:

```
value_0 ≈ B0 * P0 / 1e18
value_1 ≈ B1 * P1 / 1e18
```

(With reserves `R`, `price_i ≈ R * shares_i / totalSupply_i`. Use the contract's own `reserve()` for the global rate.)

For full PnL, also need `costBasis`. If the user has tx history, sum each `Buy` event's cost and each `Sell` event's proceeds.

## Scan Across All Markets (When Market Not Specified)

```bash
RPC_URL=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
REGISTRY=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .pmr.marketRegistry' assets/networks.json)
TOTAL=$(cast call $REGISTRY "allMarketsLength()(uint256)" --rpc-url $RPC_URL)

for i in $(seq 0 $((TOTAL-1))); do
  M=$(cast call $REGISTRY "marketAt(uint256)(address)" $i --rpc-url $RPC_URL)
  # Read balances at M, skip if both zero
done
```

This is O(N) RPC calls. For >100 markets, batch with `cast abi-encode` + a multicall contract if available, or warn the user that this will be slow.

## Output Format

| Market | Outcome | Shares | Price | Value | PnL | Claimable? |
|--------|---------|--------|-------|-------|-----|------------|

Then a summary block:
- Total positions: N
- Total value: $X
- Total unrealized PnL: $Y
- Total claimable: $Z (from resolved markets with winning outcome)

## Failure Handling

| Error | Cause | Action |
|-------|-------|--------|
| Wallet has 0 balances everywhere | New wallet, no positions | Confirm with user; offer to query a different address |
| `balanceOf` revert | Market uses a different token standard | Surface revert reason; do not retry blindly |
| RPC rate limit on big scan | Too many calls in a short window | Add `sleep 0.2` between calls; warn user it may take a minute |
