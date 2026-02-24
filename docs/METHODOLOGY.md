# Methodology: Investor Behavior & Exit Risk Intelligence for Thai Stocks

> This document is fully open and intentionally written to be readable by both investors and developers.
> Last updated: 2026-02-24

---

## Why This Exists

Thai retail investors consistently lose money not because they pick bad stocks, but because they can't see what sophisticated participants — institutions, foreign funds, and proprietary traders — are doing until it's too late.

This platform surfaces that information in plain language. Every signal here is derived from publicly available data published by the Stock Exchange of Thailand (SET), processed transparently, and explained in full.

There is no black box. If you disagree with a signal, you should be able to read this document and understand exactly why it was generated.

---

## Data Sources

All data is sourced from **SETSMART** (setsmart.com), a service operated by the Stock Exchange of Thailand. The specific data feeds used are:

| Data Feed | What It Tells Us |
|---|---|
| NVDR Trading & Statistics | Foreign fund participation per stock, per day |
| Investor Type (Institutional / Foreign / Retail) | Net flow broken down by participant category |
| Short Sales | Bearish bets by sophisticated participants |
| Program Trading Value | Algorithmic / systematic trading activity |
| Trading Halts & Market Surveillance | Stocks under regulatory scrutiny |
| Historical Trading (5-year) | Baseline for detecting unusual behavior |
| Financial Statements | Fundamental context for signal interpretation |

---

## Signal 1 — Investor Accumulation Index (IAI)

### What it measures
Whether institutions and/or foreign funds are quietly building positions in a stock, relative to retail activity.

### Why it matters
Large participants cannot buy significant positions quickly without moving the price. They accumulate over days or weeks, often during periods of low retail attention. Detecting this early gives retail investors a meaningful edge.

### How it's calculated

**Step 1 — Daily net flow per participant type**

For each stock on each trading day, we record:
- `inst_net` = Institutional buy value − Institutional sell value
- `foreign_net` = Foreign buy value − Foreign sell value  
- `retail_net` = Retail buy value − Retail sell value

These are taken directly from the SET Investor Type dataset.

**Step 2 — Rolling accumulation score (5-day and 20-day)**

```
IAI_5  = sum(inst_net + foreign_net) over last 5 trading days
IAI_20 = sum(inst_net + foreign_net) over last 20 trading days
```

**Step 3 — Quiet accumulation filter**

A stock is flagged as "quietly accumulating" when all of the following are true:
- `IAI_5 > 0` (net institutional/foreign buying in the past week)
- `IAI_20 > 0` (sustained over the past month)
- Daily volume is below the stock's 60-day average volume (accumulation is happening without drawing crowd attention)
- `retail_net < 0` over the same period (retail is net selling while institutions are net buying)

**Step 4 — NVDR confirmation**

If NVDR net flow is also positive over the same window, the signal is upgraded from "Accumulation" to "Strong Accumulation." NVDR data specifically captures foreign fund activity and serves as an independent confirmation.

### Output labels

| Label | Meaning |
|---|---|
| 🟢 Strong Accumulation | IAI positive + NVDR positive + quiet volume |
| 🟡 Accumulation | IAI positive + quiet volume, NVDR neutral |
| ⚪ Neutral | No clear directional bias |
| 🟠 Distribution | Institutions/foreign net selling while retail net buying |
| 🔴 Strong Distribution | Distribution confirmed by NVDR outflows |

### What this signal does NOT mean
This is a behavioral signal, not a price prediction. A stock showing accumulation is not guaranteed to rise. It means sophisticated participants are building positions — the reasons why are outside the scope of this signal.

---

## Signal 2 — Liquidity & Exit Risk Score (LERS)

### What it measures
How easily an investor can exit a position without suffering significant loss from illiquidity, forced halts, or regulatory action.

### Why it matters
Many retail investors focus entirely on entry. The Exit Risk Score forces attention onto exit — what happens if you need to sell quickly? This signal is designed to prevent the specific situation where an investor is trapped in a stock they can no longer sell at a fair price.

### How it's calculated

The LERS is a composite of four sub-components, each scored 0–25, summed to a total of 0–100.

**Component A — Tradability (0–25)**

Measures how liquid the stock is relative to a typical retail position size.

```
avg_daily_value = average daily traded value over 60 days (in THB)
tradability_score = min(25, (avg_daily_value / 1,000,000) * 2.5)
```

A stock trading 10M THB/day scores 25. A stock trading 1M THB/day scores 2.5.

**Component B — Halt History (0–25)**

Stocks that have been halted frequently are harder to exit in a crisis.

```
halts_12m = number of trading halts in the past 12 months
halt_score = max(0, 25 - (halts_12m * 5))
```

Zero halts = 25 points. Five or more halts = 0 points.

**Component C — Market Surveillance Status (0–25)**

Stocks under active SET surveillance measures (C sign, NPL, etc.) carry elevated forced-exit risk.

```
If stock has active surveillance measure: 0 points
If stock had surveillance measure in past 6 months: 12 points
If clean: 25 points
```

**Component D — Short Interest Pressure (0–25)**

High short interest relative to average volume indicates bearish conviction from sophisticated participants and can accelerate price declines.

```
short_ratio = short_sell_value / avg_daily_value (60-day)
short_score = max(0, 25 - (short_ratio * 50))
```

### Output

```
LERS = Component A + B + C + D   (0 to 100)
```

| Score | Label | Meaning |
|---|---|---|
| 80–100 | 🟢 Low Risk | Liquid, clean history, no red flags |
| 60–79 | 🟡 Moderate Risk | Some liquidity or history concerns |
| 40–59 | 🟠 Elevated Risk | Multiple risk factors present |
| 0–39 | 🔴 High Exit Risk | Serious liquidity or regulatory concerns |

---

## Signal 3 — Market Regime Label (MRL)

### What it measures
The overall behavioral phase of the market or a sector — whether sophisticated money is broadly entering, exiting, or speculating.

### Why it matters
Individual stock signals are more meaningful when read in the context of the broader market regime. A stock showing accumulation in a Distribution market deserves more scrutiny than the same signal in an Accumulation market.

### How it's calculated

**At market level**, we aggregate IAI across all SET/mai stocks:

```
market_IAI = sum of IAI_20 across all stocks with sufficient liquidity (avg daily value > 5M THB)
```

**At sector level**, the same aggregation is applied within each SET industry sector.

**Regime classification rules:**

| Condition | Label |
|---|---|
| market_IAI strongly positive + broad participation (>40% of stocks in accumulation) | 🟢 Accumulation Phase |
| market_IAI near zero or mixed | ⚪ Transitional |
| market_IAI negative + NVDR net outflow | 🔴 Distribution Phase |
| market_IAI positive but concentrated in <15% of stocks + high short interest | 🟡 Speculative Phase |

### Important caveats

The Speculative label is the most dangerous regime for retail investors. It looks like an Accumulation phase from the outside (prices rising) but the breadth is narrow and the underlying flow data tells a different story. This regime frequently precedes sharp corrections.

---

## Limitations & Honest Disclosures

**This platform does not predict prices.** Signals describe what has happened in the flow data. They do not guarantee future price direction.

**Data latency matters.** Real-time signals are only as fresh as the SETSMART data feed. For historical backtesting, treat all signals as end-of-day.

**Small-cap stocks are harder to read.** In thinly traded stocks, a single large trade by one participant can dominate the Investor Type data. Apply signals from small-cap stocks with extra caution.

**Signals can conflict.** A stock may show Accumulation (IAI) but High Exit Risk (LERS). These are independent dimensions. Use both together.

**This is not investment advice.** This platform is a data analysis tool. All investment decisions remain the sole responsibility of the user.

---

## Versioning

| Version | Date | Changes |
|---|---|---|
| 0.1 | 2026-02-24 | Initial methodology draft |

Signal formulas may be revised as real data is ingested and edge cases are discovered. All changes will be documented here with a date and rationale.

---

## Contributing / Feedback

If you believe a formula is flawed, a label is misleading, or a data source should be added, open an issue on GitHub. The methodology is a living document and community input is welcome.
