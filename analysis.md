# Market Analysis (`pmr-analyze`)

Run a structured analysis on a single market. Produces three sections: **Probability Trend**, **Liquidity Profile**, **Holder Distribution**.

## Probability Trend (24h / 7d / 30d)

PMR snapshots prices on every run and stores them under `.pmr-cache/<network>/<market>.jsonl`. If the cache has prior data, compute deltas.

For first-time analysis (cold cache), pull historical price points from the aggregator:

```bash
curl -s "$AGG/markets/$MARKET/history?interval=1h&window=24h" | \
  jq '[.points[] | {t: .timestamp, p: .yesPrice}]'
```

Compute:
- **Δ 24h**: `now - 24h_ago`
- **Δ 7d**:  `now - 7d_ago`
- **Volatility**: stdev of 1h deltas
- **Momentum**: sign of Δ24h × |Δ24h|

Verdict labels: `strong-yes-trend`, `weak-yes-trend`, `flat`, `weak-no-trend`, `strong-no-trend`.

## Liquidity Profile

```bash
RESERVE=$(cast call $MARKET "reserve()(uint256)"   --rpc-url $RPC_URL)
V24=$(cast call $MARKET "volume24h()(uint256)"      --rpc-url $RPC_URL)
TVL=$(cast call $MARKET "totalVolume()(uint256)"    --rpc-url $RPC_URL)
HOLDERS=$(cast call $MARKET "holderCount()(uint256)" --rpc-url $RPC_URL)

# Turnover ratio: V24 / reserve
TURNOVER=$(echo "scale=4; $V24 / $RESERVE" | bc)
```

Classify by `riskBands` in `assets/markets.json`:
- `low`:  turnover < 0.10 and reserve > $50k
- `med`:  turnover < 0.50 and reserve > $10k
- `high`: everything else

## Holder Distribution

The market contract doesn't expose a holder iteration API, so use the aggregator:

```bash
curl -s "$AGG/markets/$MARKET/holders?limit=20" | jq '.holders'
```

Compute Herfindahl-Hirschman Index (HHI):

```
HHI = Σ (share_i * 100)^2
```

- HHI < 1500: distributed
- 1500–2500: moderate concentration
- > 2500: highly concentrated (whale risk)

## Output Format

```
# Market Analysis — <name>

## Probability Trend
| Window | YES | Δ |
|--------|-----|---|
| Now    | 62% | — |
| 24h    | 58% | +4 |
| 7d     | 51% | +11|

Verdict: strong-yes-trend (momentum +4%, vol 1.2%)

## Liquidity Profile
Reserve: $87,500
24h Volume: $14,230
Turnover: 0.16
Band: low

## Holder Distribution
Top 20 holders: 64% of supply
HHI: 1820 (moderate)
Whale risk: medium — top holder owns 12%
```

## Failure Handling

| Error | Cause | Action |
|-------|-------|--------|
| Aggregator history endpoint 404 | New market, no history | Show current state only; mark "no history" |
| Cache file corrupted | Partial write | Delete the bad cache line, recompute |
| HHI > 5000 | Single-wallet market | Flag "extreme concentration", suggest caution |
