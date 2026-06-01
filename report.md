# Research Report (`pmr-report`)

Generate a full markdown research report on a market or category. Combines `pmr-query`, `pmr-analyze`, and `pmr-positions` outputs into a single shareable document.

## Report Sections

1. **Overview** — name, question, category, resolve date, current YES/NO.
2. **Current State** — prices, liquidity, 24h volume, holder count, status.
3. **Historical Trend** — Δ24h / Δ7d / Δ30d, momentum verdict, volatility.
4. **Liquidity & Holders** — turnover band, HHI, top-3 holders.
5. **Risks** — concentrated holders, thin liquidity, near-expiry, oracle risk.
6. **Verdict** — one of: `bullish-on-yes`, `bullish-on-no`, `neutral`, `illiquid-avoid`, `concentrated-caution`.

## Generation Procedure

```bash
# 1. Pull data
bash references/query.md    → current state
bash references/analysis.md → trend + liquidity
bash references/positions.md (optional) → user position overlay

# 2. Compose
cat <<EOF > $REPORT_PATH
# PMR Research Report — <name>
**Generated:** <UTC ISO-8601>
**Network:** <network>
**Market:** <address>

## Overview
...

## Current State
| Field | Value |
| --- | --- |
...
EOF
```

Default output file: `./pmr-report-<market-slug>-<YYYYMMDD>.md`.

## Optional: User Position Overlay

If `$SENDER` is provided, append a "Your Position" section showing shares held, current value, and PnL.

## Verdict Heuristics

```
if turnover > 0.5      → "illiquid-avoid"
elif HHI > 2500        → "concentrated-caution"
elif Δ24h > 0.10       → "bullish-on-yes"
elif Δ24h < -0.10      → "bullish-on-no"
else                   → "neutral"
```

Show the heuristic + inputs in the report so the user can audit it.

## Output Format

A standalone `.md` file. Should render cleanly in any markdown viewer. No HTML, no images (keeps the file portable).

## Failure Handling

| Error | Cause | Action |
|-------|-------|--------|
| Missing data fields | Partial aggregator failure | Mark "n/a" rather than abort the report |
| `cast call` revert on a section | Market contract quirk | Skip that section, note it, continue |
| Verdict heuristic conflicts | E.g. high turnover AND bullish | Surface both, let the user decide |
