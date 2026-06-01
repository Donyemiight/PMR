# Market Execution (`pmr-execute`)

Place bets, sell positions, claim winnings. **Write operations** — always run the pre-checks from the main `SKILL.md` first.

## Place a Bet (`buy`)

### Inputs
- `market` — market address
- `outcomeIndex` — 0 for YES, 1 for NO, etc.
- `collateralIn` — amount of collateral (in PHRS/PROS, 18 decimals)
- `minSharesOut` — slippage protection (reject if you get fewer shares)

### Pre-flight
1. Confirm `outcomeIndex < outcomeCount`.
2. Estimate shares out: `cast call $MARKET "quoteBuy(uint256,uint256)(uint256)" $outcomeIndex $collateralIn`
3. Reject if `estimate < minSharesOut` (slippage > 2% by default — warn user).

### Send
```bash
cast send $MARKET \
  "buy(uint256,uint256,uint256)" \
  $outcomeIndex $collateralIn $minSharesOut \
  --private-key $PRIVATE_KEY \
  --rpc-url $RPC_URL
```

## Sell a Position (`sell`)

### Inputs
- `market`, `outcomeIndex`, `sharesIn`, `minCollateralOut`

### Pre-flight
1. Check `balanceOf($SENDER, $outcomeIndex) >= sharesIn`.
2. Quote proceeds: `cast call $MARKET "quoteSell(uint256,uint256)(uint256)" $outcomeIndex $sharesIn`.
3. Reject if `quote < minCollateralOut`.

### Send
```bash
cast send $MARKET \
  "sell(uint256,uint256,uint256)" \
  $outcomeIndex $sharesIn $minCollateralOut \
  --private-key $PRIVATE_KEY \
  --rpc-url $RPC_URL
```

## Claim Winnings (`claim`)

### Pre-flight
1. Confirm `resolved == true`.
2. Confirm `winningOutcome == outcomeIndex` (the user must hold shares in the winning outcome).
3. Confirm `balanceOf($SENDER, $outcomeIndex) > 0`.

### Send
```bash
cast send $MARKET \
  "claim(uint256)" $outcomeIndex \
  --private-key $PRIVATE_KEY \
  --rpc-url $RPC_URL
```

## Approval Flow (If Using an ERC20 Outcome Token)

If the outcome token is ERC20 (not native), the market needs allowance:

```bash
# Check current allowance
ALLOW=$(cast call $OUTCOME_TOKEN "allowance(address,address)(uint256)" $SENDER $MARKET --rpc-url $RPC_URL)

if [ "$ALLOW" = "0" ] || [ "$ALLOW" -lt "$SHARES_IN" ]; then
  cast send $OUTCOME_TOKEN "approve(address,uint256)" $MARKET $SHARES_IN \
    --private-key $PRIVATE_KEY --rpc-url $RPC_URL
fi
```

**Never** approve an infinite (`2^256 - 1`) allowance unless the user explicitly asks.

## Output Format

```
✅ Tx sent: 0x<hash>
   Market: 0x<market>
   Action: buy / sell / claim
   Outcome: <label>
   Size: <human-readable>
   Est. shares / proceeds: <value>
   View: <explorerUrl>/tx/<hash>
```

## Failure Handling

| Error | Cause | Action |
|-------|-------|--------|
| `insufficient funds` | Wallet too thin | Show balance, suggest top-up |
| `execution reverted: market resolved` | Tried to buy on a closed market | Read `resolved()`/`endTime()`; suggest `claim` instead |
| Slippage exceeded `minSharesOut` | Price moved | Re-quote, ask user to confirm with new bounds |
| Allowance missing | First-time sell on ERC20 outcome | Prompt user to approve exact amount |
| Nonce conflict | Pending tx in flight | Wait 1 block, re-send |
