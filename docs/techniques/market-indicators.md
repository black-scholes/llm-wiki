# Trading Indicators Documentation

> Source: migrated from `reference/hft_indicators/docs/INDICATORS.md` on
> 2026-07-14. Verify formulas against the implementation before relying on
> them for a production decision.

This document provides mathematical formulas, inputs, outputs, and interpretations for all indicators implemented in the system.

## Table of Contents
1. [Price-Based Indicators](#price-based-indicators)
2. [Order Book Indicators](#order-book-indicators)
3. [Trade-Based Indicators](#trade-based-indicators)
4. [Statistical Indicators](#statistical-indicators)
5. [Beta/Correlation Indicators](#betacorrelation-indicators)
6. [Utility Indicators](#utility-indicators)

---

## Price-Based Indicators

### 1. Price
**File:** `Price.cpp`

**Formula:**
$$\text{Value} = \text{Price}_{\text{type}}$$

**Inputs:**
- `aInstrument`: Instrument pointer
- `aPriceType`: Price type (MID_PX, BID_PX, ASK_PX, etc.)

**Output:**
- Raw price value based on selected price type

**Interpretation:**
Simple price extractor that returns the current price of an instrument based on the specified price type.

---

### 2. Spread
**File:** `Spread.cpp`

**Formula:**
$$\text{Spread} = \text{Ask}_0 - \text{Bid}_0$$

**Inputs:**
- `aInstrument`: Instrument pointer
- `aInterval`: Sampling interval in milliseconds

**Output:**
- Bid-ask spread sampled at specified intervals

**Interpretation:**
Measures the difference between best ask and best bid prices. High spreads indicate lower liquidity or higher market uncertainty.

---

### 3. PriceDiff
**File:** `PriceDiff.cpp`

**Formula:**

For PRICE mode:
$$\Delta_{\text{lag}} = P_{\text{lag}} - P_{\text{lag,prev}}$$
$$\Delta_{\text{lead}} = P_{\text{lead}} - P_{\text{lead,prev}}$$

For RETURN mode:
$$\Delta_{\text{lag}} = \ln\left(\frac{P_{\text{lag}}}{P_{\text{lag,prev}}}\right)$$
$$\Delta_{\text{lead}} = \ln\left(\frac{P_{\text{lead}}}{P_{\text{lead,prev}}}\right)$$

**Inputs:**
- `aLagger`: Lagging instrument
- `aLeader`: Leading instrument
- `aCoeff`: Coefficient multiplier
- `aInterval`: Sampling interval (milliseconds)
- `aHalfLife`: Half-life for exponential smoothing
- `aPriceType`: Price type to use
- `aMode`: PRICE or RETURN mode

**Output:**
- Price difference or return differential between two instruments

**Interpretation:**
Captures the relationship between two instruments by measuring their relative price movements. Used for pair trading and spread analysis.

---

### 4. PriceLag
**File:** `PriceLag.cpp`

**Formula:**
$$\text{Value} = \text{EWMA}_{\text{lag}}(P_{\text{lag}}) - \text{EWMA}_{\text{lead}}(P_{\text{lead}})$$

**Inputs:**
- `aInst1`: First instrument (lagger)
- `aInst2`: Second instrument (leader)
- `aCoeff`: Coefficient
- `aHalfLife`: Half-life for EWMA
- `aPriceType`: Price type

**Output:**
- Difference between smoothed prices

**Interpretation:**
Measures the lagged relationship between two instruments using exponentially weighted moving averages.

---

### 5. TrendLag
**File:** `TrendLag.cpp`

**Formula:**
$$\text{Value} = \frac{\text{EWMA}_{\text{lag}}(P_{\text{lag}})}{\text{EWMA}_{\text{lead}}(P_{\text{lead}})} - 1$$

**Inputs:**
- `aInst1`: First instrument
- `aInst2`: Second instrument (leader)
- `aCoeff`: Coefficient
- `aHalfLife`: Half-life for EWMA
- `aPriceType`: Price type

**Output:**
- Ratio-based trend lag indicator

**Interpretation:**
Captures the relative trend between two instruments using ratio of their smoothed prices.

---

## Order Book Indicators

### 6. OrderBookImbalance
**File:** `OrderBookImbalance.cpp`

**Formula:**
$$\text{WeightedBid} = \sum_{i=0}^{L-1} m^i \cdot Q_{\text{bid},i}$$
$$\text{WeightedAsk} = \sum_{i=0}^{L-1} m^i \cdot Q_{\text{ask},i}$$
$$\text{Imbalance} = \text{EWMA}(\text{WeightedBid} - \text{WeightedAsk})$$

**Inputs:**
- `aInstrument`: Instrument pointer
- `aCoeff`: Coefficient
- `aLevel`: Number of order book levels
- `aMultiplier`: Geometric decay multiplier $m$
- `aHalfLife`: Half-life for EWMA

**Output:**
- Smoothed order book imbalance value

**Interpretation:**
Measures the imbalance between bid and ask sides of the order book. Positive values indicate more buying pressure, negative values indicate selling pressure.

---

### 7. OrderBookImbalanceTrend
**File:** `OrderBookImbalanceTrend.cpp`

**Formula:**
$$\text{Value} = \text{Imbalance}_{\text{current}} - \text{EWMA}(\text{Imbalance})$$

**Inputs:**
- Same as OrderBookImbalance

**Output:**
- De-trended imbalance value

**Interpretation:**
Captures sudden changes in order book imbalance by removing the smoothed trend. Useful for detecting abrupt shifts in market sentiment.

---

### 8. OrderBookSkew
**File:** `OrderBookSkew.cpp`

**Formula:**
$$\text{WeightedBidPx} = \sum_{i=0}^{L-1} m^i \cdot P_{\text{bid},i} \cdot Q_{\text{bid},i}$$
$$\text{WeightedAskPx} = \sum_{i=0}^{L-1} m^i \cdot P_{\text{ask},i} \cdot Q_{\text{ask},i}$$
$$\text{Skew} = \frac{Q_{\text{ask}} \cdot \text{WeightedBidPx} + Q_{\text{bid}} \cdot \text{WeightedAskPx}}{Q_{\text{ask}} + Q_{\text{bid}}} - P_{\text{mid}}$$

**Inputs:**
- `aInstrument`: Instrument pointer
- `aCoeff`: Coefficient
- `aLevel`: Number of levels
- `aMultiplier`: Decay multiplier
- `aHalfLife`: EWMA half-life
- `aMode`: NORM or LOG mode
- `aPriceType`: Price type

**Output:**
- Smoothed order book skew

**Interpretation:**
Estimates the fair price based on weighted order book quantities. Positive skew suggests price should move up, negative suggests down.

---

### 9. OrderBookSkewTrend
**File:** `OrderBookSkewTrend.cpp`

**Formula:**
$$\text{Value} = \text{Skew}_{\text{current}} - \text{EWMA}(\text{Skew})$$

**Inputs:**
- Same as OrderBookSkew

**Output:**
- De-trended skew value

**Interpretation:**
Captures sudden changes in order book skew, filtering out the baseline trend.

---

### 10. OrderBookSlope
**File:** `OrderBookSlope.cpp`

**Formula:**
$$\text{BidVolume} = \sum_{i=0}^{L-1} Q_{\text{bid},i}$$
$$\text{AskVolume} = \sum_{i=0}^{L-1} Q_{\text{ask},i}$$
$$\text{SlopeBid} = \frac{\text{BidVolume}/(\text{BidVolume} + \text{AskVolume})}{P_{\text{mid}} - P_{\text{bid},L-1}}$$
$$\text{SlopeAsk} = \frac{\text{AskVolume}/(\text{BidVolume} + \text{AskVolume})}{P_{\text{ask},L-1} - P_{\text{mid}}}$$
$$\text{Value} = \text{SlopeBid} - \text{SlopeAsk}$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aLevel`: Number of levels
- `aPriceType`: Price type

**Output:**
- Order book slope differential

**Interpretation:**
Measures the density of liquidity on each side of the order book. Higher slope indicates steeper liquidity profile.

---

### 11. OrderBookSlopeTrend
**File:** `OrderBookSlopeTrend.cpp`

**Formula:**
$$\text{Value} = \text{SlopeDiff}_{\text{current}} - \text{EWMA}(\text{SlopeDiff})$$

**Inputs:**
- Same as OrderBookSlope plus `aHalfLife`

**Output:**
- De-trended slope value

**Interpretation:**
Detects changes in order book slope relative to recent history.

---

### 12. OrderBookGap
**File:** `OrderBookGap.cpp`

**Formula:**
$$\text{Gap}_{i} = \frac{P_{\text{top}} - P_i}{\text{TickSize}}$$
$$\text{WeightedPx} = \sum_{i=0}^{L-1} f[\text{Gap}_i] \cdot P_i \cdot Q_i$$
$$\text{WeightedQty} = \sum_{i=0}^{L-1} f[\text{Gap}_i] \cdot Q_i$$
$$\text{Value} = \frac{\text{WgtBidPx} \cdot \text{WgtAskQty}/\text{WgtBidQty} + \text{WgtAskPx} \cdot \text{WgtBidQty}/\text{WgtAskQty}}{\text{WgtAskQty} + \text{WgtBidQty}} - P_{\text{mid}}$$

where $f[i] = m^i$ are the weighting factors.

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aLevel`: Number of levels
- `aMultiplier`: Decay multiplier
- `aMode`: NORM or LOG
- `aPriceType`: Price type

**Output:**
- Gap-weighted fair price deviation

**Interpretation:**
Similar to skew but weights by tick distance from top of book. Accounts for price gaps in the order book.

---

### 13. OrderBookGapTrend
**File:** `OrderBookGapTrend.cpp`

**Formula:**
$$\text{Value} = \text{OrderBookVal}_{\text{current}} - \text{EWMA}(\text{OrderBookVal})$$

**Inputs:**
- Same as OrderBookGap plus `aHalfLife`

**Output:**
- De-trended gap value

**Interpretation:**
Captures sudden changes in gap-weighted order book metrics.

---

### 14. OrderBookElasticity
**File:** `OrderBookElasticity.cpp`

**Formula:**
$$\text{TopBidElas} = \frac{Q_{\text{bid},0}}{|P_{\text{bid},0}/P_{\text{mid}} - 1|}$$
$$\text{LowerBidElas} = \sum_{i=1}^{L-1} \frac{(\text{SumBid}_{i+1}/\text{SumBid}_i - 1)}{|P_{\text{bid},i+1}/P_{\text{bid},i} - 1|}$$
$$\text{Value} = \frac{\text{BidElas} - \text{AskElas}}{L}$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aLevel`: Number of levels
- `aPriceType`: Price type

**Output:**
- Order book elasticity differential

**Interpretation:**
Measures how order book depth changes with price. High elasticity indicates resilient liquidity.

---

### 15. BookImbalance
**File:** `BookImbalance.cpp`

**Formula:**
Similar to OrderBookGap, but with a cutoff price filter:
$$P_{\text{bid,cutoff}} = P_{\text{bid},0} - \frac{P_{\text{mid}}}{10000} \cdot \text{CutOff}$$
$$P_{\text{ask,cutoff}} = P_{\text{ask},0} + \frac{P_{\text{mid}}}{10000} \cdot \text{CutOff}$$

Only levels within cutoff are included in calculation.

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aLevel`: Number of levels
- `aMultiplier`: Decay multiplier
- `aMode`: NORM or LOG
- `aPriceType`: Price type
- `cutoff`: Price cutoff parameter

**Output:**
- Filtered fair price deviation

**Interpretation:**
Like OrderBookGap but ignores levels beyond a certain price distance, focusing on near-market liquidity.

---

### 16. BookDeltaTop
**File:** `BookDeltaTop.cpp`

**Formula:**

For QUOTE updates:
$$\Delta_{\text{bid}} = \begin{cases}
Q_{\text{bid},0} & \text{if } P_{\text{bid},0} > P_{\text{bid,prev}} \\
-Q_{\text{bid,prev}} & \text{if } P_{\text{bid},0} < P_{\text{bid,prev}} \\
Q_{\text{bid},0} - Q_{\text{bid,prev}} & \text{otherwise}
\end{cases}$$

For TRADE updates:
$$\Delta_{\text{bid}} = -Q_{\text{trade}}$$

$$\text{BookDelta} = \Delta_{\text{bid}} - \Delta_{\text{ask}}$$
$$\text{Value} = \text{EWMA}(\text{BookDelta})$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aHalfLife`: EWMA half-life
- `aSizeCap`: Maximum delta size
- `aInverse`: Reverse sign flag

**Output:**
- Smoothed top-of-book delta

**Interpretation:**
Tracks changes in top-of-book liquidity. Positive values indicate aggressive buying, negative indicates selling.

---

### 17. BookDeltaTopTrend
**File:** `BookDeltaTopTrend.cpp`

**Formula:**
$$\text{Value} = \text{BookDelta}_{\text{current}} - \text{EWMA}(\text{BookDelta})$$

**Inputs:**
- Same as BookDeltaTop (without inverse flag)

**Output:**
- De-trended book delta

**Interpretation:**
Captures sudden changes in top-of-book delta, removing baseline trend.

---

### 18. BookDeltaLevel
**File:** `BookDeltaLevel.cpp`

**Formula:**
$$\text{SumBid}_L = \sum_{i=0}^{L} Q_{\text{bid},i}$$

Adjusts level based on price movement:
- If best bid up: use $L+1$
- If best bid down: use $L-1$
- Otherwise: use $L$

$$\Delta_{\text{bid}} = \text{SumBid}_{\text{adjusted}} - \text{SumBid}_{\text{prev}}$$
$$\text{BookDelta} = \Delta_{\text{bid}} - \Delta_{\text{ask}}$$
$$\text{Value} = \text{EWMA}(\text{BookDelta})$$

**Inputs:**
- `instrument`: Instrument
- `coeff`: Coefficient
- `level`: Order book level
- `halflife`: EWMA half-life
- `sizecap`: Size cap

**Output:**
- Smoothed multi-level book delta

**Interpretation:**
Like BookDeltaTop but tracks deeper levels, capturing order flow at specific depths.

---

### 19. BookSize
**File:** `BookSize.cpp`

**Formula:**
$$w_i = 0.5^{\frac{\text{diff}_i}{\text{HalfLife}}}$$

For PXVOL mode:
$$\text{Value} = \sum_{i=0}^{L-1} w_i \cdot Q_i \cdot P_i$$

For VOL mode:
$$\text{Value} = \sum_{i=0}^{L-1} w_i \cdot Q_i$$

where $\text{diff}_i = |P_i - P_0| / P_{\text{mid}}$

**Inputs:**
- `aInstrument`: Instrument
- `aLevel`: Number of levels
- `aHalfLife`: Decay half-life (in bps)
- `aMode`: PXVOL or VOL
- `aSide`: BUY or SELL

**Output:**
- Weighted book size (volume or notional)

**Interpretation:**
Measures liquidity available on one side of the book with exponential decay by price distance.

---

## Trade-Based Indicators

### 20. TradeImbalance
**File:** `TradeImbalance.cpp`

**Formula:**

For NORM mode:
$$\text{SignedVal} = \begin{cases}
+Q_{\text{trade}} & \text{if ASK trade} \\
-Q_{\text{trade}} & \text{if BID trade}
\end{cases}$$

For LOG mode:
$$\text{SignedVal} = \begin{cases}
+\ln(Q_{\text{trade}} + 1) & \text{if ASK trade} \\
-\ln(Q_{\text{trade}} + 1) & \text{if BID trade}
\end{cases}$$

$$\text{Imbalance} = \frac{\text{SignedVal}}{\text{TotalVal}}$$
$$\text{Value} = \text{EWMA}(\text{Imbalance})$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aHalfLife`: EWMA half-life
- `aMode`: NORM or LOG
- `aPriceType`: Price type
- `aInverse`: Reverse sign flag

**Output:**
- Smoothed trade imbalance ratio

**Interpretation:**
Measures the balance of aggressive buying vs selling. Positive indicates more buying, negative more selling.

---

### 21. TradeImbalanceTrend
**File:** `TradeImbalanceTrend.cpp`

**Formula:**
$$\text{Value} = \text{Imbalance}_{\text{current}} - \text{EWMA}(\text{Imbalance})$$

**Inputs:**
- Same as TradeImbalance

**Output:**
- De-trended trade imbalance

**Interpretation:**
Detects sudden shifts in trade flow relative to recent baseline.

---

### 22. TradeTrend
**File:** `TradeTrend.cpp`

**Formula:**

For NORM mode:
$$\text{Imbalance} = \begin{cases}
+Q_{\text{trade}} & \text{if } P_{\text{trade}} > P_{\text{prev}} \\
-Q_{\text{trade}} & \text{if } P_{\text{trade}} < P_{\text{prev}} \\
0 & \text{otherwise}
\end{cases}$$

For LOG mode:
$$\text{Imbalance} = \begin{cases}
+\ln(Q_{\text{trade}} + 1) & \text{if } P_{\text{trade}} > P_{\text{prev}} \\
-\ln(Q_{\text{trade}} + 1) & \text{if } P_{\text{trade}} < P_{\text{prev}} \\
0 & \text{otherwise}
\end{cases}$$

$$\text{Value} = \text{Imbalance}_{\text{current}} - \text{EWMA}(\text{Imbalance})$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aHalfLife`: EWMA half-life
- `aMode`: NORM or LOG

**Output:**
- De-trended trade direction signal

**Interpretation:**
Measures whether trades are pushing price up or down relative to recent trend.

---

### 23. TradeBias
**File:** `TradeBias.cpp`

**Formula:**
$$\text{SignedQty} = \begin{cases}
+Q_{\text{trade}} & \text{if ASK trade} \\
-Q_{\text{trade}} & \text{if BID trade}
\end{cases}$$
$$\text{Value} = \sum_{t \in \text{window}} \text{SignedQty}_t$$

**Inputs:**
- `aInstrument`: Instrument
- `aInterval`: Time window (milliseconds)

**Output:**
- Net signed trade quantity over window

**Interpretation:**
Cumulative trade flow over a sliding window. Positive indicates net buying pressure.

---

### 24. TradeVolume
**File:** `TradeVolume.cpp`

**Formula:**
$$\text{Value} = \sum_{t \in \text{window}} Q_{t}$$

**Inputs:**
- `aInstrument`: Instrument
- `aInterval`: Time window (milliseconds)

**Output:**
- Total trade volume over window

**Interpretation:**
Measures trading activity level. High volume indicates increased market participation.

---

### 25. TradeMA
**File:** `TradeMA.cpp`

**Formula:**
$$\alpha = \frac{2}{2.8854 \cdot \text{HalfLife} + 1}$$
$$\text{IgnoreThreshold} = \text{Spread} \cdot \text{IgnoreRatio}$$

If $|P_{\text{trade}} - P_{\text{mid}}| > \text{IgnoreThreshold}$:
$$\text{MA} = \alpha \cdot P_{\text{trade}} + (1-\alpha) \cdot \text{MA}_{\text{prev}}$$

**Inputs:**
- `aInstrument`: Instrument
- `aHalfLife`: EWMA half-life
- `aIgnoreRatio`: Ratio to ignore outlier trades
- `aPriceType`: Price type

**Output:**
- Moving average of trade prices (filtered)

**Interpretation:**
Smoothed average of trade prices, excluding trades far from mid (likely prints or errors).

---

### 26. CTrade
**File:** `CTrade.cpp`

**Formula:**
$$w_t = \frac{P_{\text{trade}} - P_{\text{mid}}}{P_{\text{spread}}/2}$$
$$\lambda_t = 0.5^{\frac{\Delta t}{\text{HalfLife}}}$$
$$\text{DecayedTrades} = \lambda_t \cdot \text{DecayedTrades}_{\text{prev}} + w_t \cdot Q_{\text{trade}}$$
$$\text{SmoothedVolume} = \lambda_t \cdot \text{SmoothedVolume}_{\text{prev}} + |w_t| \cdot Q_{\text{trade}}$$
$$\text{Value} = \frac{\text{DecayedTrades}}{\text{SmoothedVolume}}$$

**Inputs:**
- `aInstrument`: Instrument
- `aHalfLife`: Time decay half-life (milliseconds)

**Output:**
- Normalized trade flow indicator

**Interpretation:**
Measures trade aggressiveness weighted by distance from mid. Values range approximately [-1, +1].

---

### 27. TradeCount, TradeQty, UpdateCount
**Files:** `TradeCount.cpp`, `TradeQty.cpp`, `UpdateCount.cpp`

**Formula:**
Simple counters incremented on each event.

**Output:**
- Count of trades/updates

**Interpretation:**
Basic activity metrics for monitoring market microstructure events.

---

### 28. TradeSweep, TradeSweepQty
**Files:** `TradeSweep.cpp`, `TradeSweepQty.cpp`

**Formula:**
Tracks when trades "sweep" through multiple price levels.

**Output:**
- Sweep count or swept quantity

**Interpretation:**
Identifies aggressive orders that consume multiple levels of liquidity, indicating strong directional intent.

---

## Statistical Indicators

### 29. Bollinger
**File:** `Bollinger.cpp`

**Formula:**
$$\alpha = \min\left(1, \frac{\Delta t}{\text{Duration}}\right)$$
$$\mu = \alpha \cdot P + (1-\alpha) \cdot \mu_{\text{prev}}$$
$$\alpha_{\sigma} = \min\left(1, \frac{\Delta t}{\text{StdDuration}}\right)$$
$$\sigma^2 = \alpha_{\sigma} \cdot P^2 + (1-\alpha_{\sigma}) \cdot \sigma^2_{\text{prev}} - \mu_{\sigma}^2$$

For NORM mode:
$$\text{Value} = \frac{P - \mu}{\sigma}$$

For DIFF mode:
$$\text{Value} = P - \mu$$

For VNORM mode:
$$\text{Value} = \frac{P - \mu}{\sigma} \cdot V$$

For VDIFF mode:
$$\text{Value} = (P - \mu) \cdot V$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aDuration`: Mean duration (seconds)
- `aStdDuration`: Std deviation duration (seconds)
- `aInterval`: Sampling interval (milliseconds)
- `aPriceType`: Price type
- `aMode`: NORM, DIFF, VNORM, VDIFF, VNORM2, VDIFF2
- `aStdDevCap`: Apply minimum std dev cap

**Output:**
- Normalized price deviation or volume-weighted deviation

**Interpretation:**
Measures how many standard deviations price is from its moving average. Values > 2 or < -2 indicate extreme movements.

---

### 30. BollingerNew
**File:** `BollingerNew.cpp`

**Formula:**
Improved variance calculation:
$$\text{Var} = (1-\alpha_{\sigma}) \cdot \text{Var}_{\text{prev}} + \alpha_{\sigma} \cdot (P - \mu)^2$$
$$\sigma = \sqrt{\text{Var}}$$
$$\mu = (1-\alpha) \cdot \mu_{\text{prev}} + \alpha \cdot P$$

**Inputs:**
- Same as Bollinger

**Output:**
- Improved Bollinger bands calculation

**Interpretation:**
More numerically stable version of Bollinger using iterative variance calculation.

---

### 31. StdevPrice
**File:** `StdevPrice.cpp`

**Formula:**
$$\sigma = \sqrt{\text{AvgSqPx} - \text{AvgPx}^2}$$

**Inputs:**
- `aInstrument`: Instrument
- `aDuration`: Duration for averaging (seconds)
- `aStdDuration`: Std deviation duration (seconds)
- `aInterval`: Sampling interval (milliseconds)
- `aPriceType`: Price type

**Output:**
- Price standard deviation

**Interpretation:**
Measures price volatility over specified time window.

---

### 32. SimpleTrend
**File:** `SimpleTrend.cpp`

**Formula:**
$$\text{EWMA} = \text{EWMA}(P)$$
$$\text{Value} = P - \text{EWMA}$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aHalfLife`: EWMA half-life
- `aPriceType`: Price type

**Output:**
- De-trended price

**Interpretation:**
Removes smooth trend from price, highlighting short-term deviations.

---

### 33. PxMomentum
**File:** `PxMomentum.cpp`

**Formula:**
$$\text{ShortMA} = \text{EWMA}_{\text{short}}(P)$$
$$\text{LongMA} = \text{EWMA}_{\text{long}}(P)$$
$$\text{Value} = \text{ShortMA} - \text{LongMA}$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aShortHalfLife`: Short-term half-life
- `aLongHalfLife`: Long-term half-life
- `aPriceType`: Price type

**Output:**
- Momentum indicator (MACD-like)

**Interpretation:**
Positive values indicate upward momentum, negative indicates downward momentum.

---

### 34. Returns
**File:** `Returns.cpp`

**Formula:**
$$\text{Return} = P_t - P_{t-\text{Duration}}$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `Duration`: Lookback duration (seconds)
- `aPriceType`: Price type

**Output:**
- Simple returns over duration

**Interpretation:**
Raw price change over specified period.

---

### 35. Volume
**File:** `Volume.cpp`

**Formula:**
$$\text{Volume} = \sum_{t \in [\text{now}-\text{Duration}, \text{now}]} Q_t$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `Duration`: Time window (seconds)

**Output:**
- Cumulative volume over window

**Interpretation:**
Total traded quantity in recent time window.

---

### 36. OnlineStdev
**File:** `OnlineStdev.cpp`

**Formula:**
Online variance calculation with exponential decay:
$$\text{Sum} = \lambda \cdot \text{Sum}_{\text{prev}} + (1-\lambda) \cdot X$$
$$\text{SumSq} = \lambda \cdot \text{SumSq}_{\text{prev}} + (1-\lambda) \cdot X^2$$
$$\sigma = \sqrt{\text{SumSq} - \text{Sum}^2}$$

**Inputs:**
- `aHalfLife`: Half-life (milliseconds)
- `aHistoricalVal`: Initial standard deviation estimate

**Output:**
- Running standard deviation

**Interpretation:**
Efficiently computes standard deviation in streaming fashion without storing history.

---

## Beta/Correlation Indicators

### 37. DynamicBeta
**File:** `DynamicBeta.cpp`

**Formula:**
$$\beta = \text{EWMA}\left(\frac{\sum P_{\text{lag}}}{\sum P_{\text{lead}}}\right)$$
$$\text{Value} = \beta_{\text{ext}} \cdot \beta \cdot P_{\text{lead}} + (1-\beta_{\text{ext}}) \cdot \mu_{\text{lag}} - P_{\text{lag}}$$

With optional lead filter:
$$\text{If } \left|\frac{P_{\text{lag}} - \mu_{\text{lag}}}{\mu_{\text{lag}}}\right| > \beta_{\text{ext}} \cdot \left|\frac{P_{\text{lead}} - \mu_{\text{lead}}}{\mu_{\text{lead}}}\right|, \text{Value} = 0$$

**Inputs:**
- `aInst1`: Lagging instrument
- `aCoeff`: Coefficient
- `aLeader`: Leading instrument
- `aDuration`: Duration for averaging (seconds)
- `aInterval`: Sampling interval (milliseconds)
- `aHalfLife`: EWMA half-life
- `aExtBeta`: External beta multiplier
- `aUseLead`: Enable lead filter
- `aPriceType`: Price type

**Output:**
- Beta-adjusted spread value

**Interpretation:**
Measures mispri cing between two instruments based on rolling beta relationship. Used for pairs trading.

---

### 38. DynamicBetaBasket
**File:** `DynamicBetaBasket.cpp`

**Formula:**
Same as DynamicBeta but leader is a basket of instruments:
$$\beta = \text{EWMA}\left(\frac{\sum P_{\text{lag}}}{\sum P_{\text{basket}}}\right)$$

**Inputs:**
- `instrument`: Single instrument
- `basket`: Basket of instruments
- `coeff`: Coefficient
- `ext_beta`: External beta
- `use_lead`: Lead filter flag
- `duration`: Duration (seconds)
- `interval`: Interval (milliseconds)
- `halflife`: EWMA half-life
- `price_type`: Price type

**Output:**
- Beta-adjusted spread vs basket

**Interpretation:**
Extends DynamicBeta to measure one instrument against a basket (e.g., stock vs index).

---

### 39. EWBeta
**File:** `EWBeta.cpp`

**Formula:**
$$\alpha = \frac{2}{2.8854 \cdot \text{HalfLife} + 1}$$

For PRICE mode:
$$\Delta_{\text{lag}} = P_{\text{lag}} - P_{\text{lag,prev}}$$
$$\Delta_{\text{lead}} = P_{\text{lead}} - P_{\text{lead,prev}}$$

For RETURN mode:
$$\Delta_{\text{lag}} = \ln(P_{\text{lag}} / P_{\text{lag,prev}})$$
$$\Delta_{\text{lead}} = \ln(P_{\text{lead}} / P_{\text{lead,prev}})$$

$$\text{Mean} = \alpha \cdot \Delta_{\text{lead}}^2 + (1-\alpha) \cdot \text{Mean}_{\text{prev}}$$
$$\text{Var} = \alpha \cdot \Delta_{\text{lag}} \cdot \Delta_{\text{lead}} + (1-\alpha) \cdot \text{Var}_{\text{prev}}$$
$$\beta = \frac{\text{Var}}{\text{Mean}}$$

**Inputs:**
- `aLagger`: Lagging instrument
- `aCoeff`: Coefficient
- `aLeader`: Leading instrument
- `aInterval`: Sampling interval (milliseconds)
- `aHalfLife`: Half-life
- `aPriceType`: Price type
- `aMode`: PRICE or RETURN
- `aRelation`: Relationship type

**Output:**
- Exponentially weighted beta estimate

**Interpretation:**
Computes rolling beta using exponential weighting. More responsive to recent changes than DynamicBeta.

---

### 40. RelBollinger
**File:** `RelBollinger.cpp`

**Formula:**
Computes Bollinger bands for both instruments:
$$\text{Boll}_{\text{lag}} = \frac{P_{\text{lag}} - \mu_{\text{lag}}}{\sigma_{\text{lag}}}$$
$$\text{Boll}_{\text{lead}} = \frac{P_{\text{lead}} - \mu_{\text{lead}}}{\sigma_{\text{lead}}}$$
$$\text{Value} = \text{Boll}_{\text{lag}} - \text{Boll}_{\text{lead}}$$

**Inputs:**
- `aLagger`: Lagging instrument
- `aCoeff`: Coefficient
- `aLeader`: Leading instrument
- `aDuration`: Mean duration (seconds)
- `aStdDevDuration`: Std dev duration (seconds)
- `aInterval`: Sampling interval (milliseconds)
- `aPriceType`: Price type
- `aRelation`: Relationship type
- `aStdDevCap`: Apply std dev cap

**Output:**
- Relative Bollinger position

**Interpretation:**
Measures whether one instrument is overextended relative to another in normalized terms.

---

## Utility Indicators

### 41. ADL (Accumulation/Distribution Line)
**File:** `ADL.cpp`

**Formula:**
$$\text{MFM} = \frac{(P_{\text{close}} - P_{\text{low}}) - (P_{\text{high}} - P_{\text{close}})}{P_{\text{high}} - P_{\text{low}}}$$
$$\text{MFV} = \text{MFM}$$
$$\text{ADL} = \text{EWMA}(\text{MFV})$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `aInterval`: Sampling interval (milliseconds)
- `aHalfLife`: EWMA half-life
- `aPriceType`: Price type

**Output:**
- Smoothed money flow multiplier

**Interpretation:**
Measures buying/selling pressure based on where price closes within its range. Positive indicates accumulation.

---

### 42. BestBid, BestAsk
**Files:** `BestBid.cpp`, `BestAsk.cpp`

**Formula:**
$$\text{Value} = P_{\text{bid/ask},0}$$

Sampled at regular intervals.

**Inputs:**
- `aInstrument`: Instrument
- `aInterval`: Sampling interval (milliseconds)

**Output:**
- Best bid or ask price

**Interpretation:**
Tracks top of book prices at specified frequency.

---

### 43. OBPredictor
**File:** `OBPredictor.cpp`

**Formula:**
$$w_i = 0.5^{\frac{|P_i - P_0|/P_{\text{mid}}}{\text{HalfLife}}}$$
$$\text{WeightedBid} = \frac{\sum w_i \cdot P_{i,\text{bid}} \cdot Q_{i,\text{bid}}}{\sum w_i \cdot Q_{i,\text{bid}}}$$
$$\text{WeightedAsk} = \frac{\sum w_i \cdot P_{i,\text{ask}} \cdot Q_{i,\text{ask}}}{\sum w_i \cdot Q_{i,\text{ask}}}$$
$$\text{CalcMid} = \frac{Q_{\text{bid}} \cdot \text{WeightedAsk} + Q_{\text{ask}} \cdot \text{WeightedBid}}{Q_{\text{bid}} + Q_{\text{ask}}}$$
$$\text{Return} = \frac{\text{CalcMid} - P_{\text{mid}}}{P_{\text{mid}}}$$

**Inputs:**
- `aInstrument`: Instrument
- `aHalfLife`: Distance decay half-life (bps)
- `smoothedHalfLife`: Time smoothing half-life
- `sd_half_life`: Std dev half-life
- `aPriceType`: Price type

**Output:**
- Predicted return based on order book shape
- Metadata: returns, standard deviations, sizes

**Interpretation:**
Uses weighted order book to predict short-term price movement. Positive return suggests upward pressure.

---

### 44. L1Events
**File:** `L1Events.cpp`

**Formula:**
$$\text{Value} = \text{Count of L1 updates in last Duration}$$

**Inputs:**
- `aInstrument`: Instrument
- `aCoeff`: Coefficient
- `duration`: Time window (seconds)
- `level`: Maximum level to track

**Output:**
- Count of level 1 order book events

**Interpretation:**
Measures order book activity level. High counts indicate increased market microstructure activity.

---

### 45. BigMove
**File:** `BigMove.cpp`

**Formula:**
Detects when:
$$|P_t - P_{t-\text{Duration}}| > \text{Percentage} \cdot P_t$$

**Inputs:**
- `aInstrument`: Instrument
- `duration`: Monitoring window (minutes)
- `percentage`: Movement threshold (%)

**Output:**
- Binary indicator (logs when big move detected)

**Interpretation:**
Alert system for large price movements exceeding threshold.

---

### 46. ORB (Opening Range Breakout)
**File:** `ORB.cpp`

**Formula:**
Tracks opening range over first `duration` minutes, then detects breakouts:
- High breakout: $P_{\text{close}} > P_{\text{high,opening}}$
- Low breakdown: $P_{\text{close}} < P_{\text{low,opening}}$

**Inputs:**
- `aInstrument`: Instrument
- `duration`: Opening range duration (minutes)

**Output:**
- Breakout signals (logged)

**Interpretation:**
Classic day-trading pattern. Breakouts from opening range often indicate trend continuation.

---

### 47. Sweep
**File:** `Sweep.cpp`

**Formula:**
Compares current vs previous order book to detect sweeps through multiple levels.

**Output:**
- Sweep bid/ask values

**Interpretation:**
Detects when aggressive orders consume multiple levels, indicating strong directional pressure.

---

### 48. Sampler
**File:** `Sampler.cpp`

**Formula:**
Adaptive sampling based on price change or event count with exponential decay.

**Inputs:**
- `aInstrument`: Instrument
- `aHalfLife`: Decay half-life
- `aCutOff`: Threshold for sampling
- `aDependantPriceType`: Price type to monitor
- `aSampleMode`: Sampling mode

**Output:**
- Decay factor for adaptive sampling

**Interpretation:**
Provides time-adaptive weights for other indicators based on market activity.

---

### 49. ZeroIndicator
**File:** `ZeroIndicator.cpp`

**Formula:**
$$\text{Value} = 0$$

**Output:**
- Always returns 0

**Interpretation:**
Placeholder indicator for testing or as baseline.

---

## Notes

### Common Parameters

- **HalfLife**: Time for exponential weight to decay to 50%. Shorter = more responsive.
- **Interval**: Sampling frequency in milliseconds.
- **Duration**: Lookback window in seconds.
- **Level**: Number of order book levels to include.
- **Multiplier**: Geometric decay factor for weighting levels (typically < 1).
- **Mode**:
  - NORM: Normal calculation
  - LOG: Logarithmic transformation
  - VNORM/VDIFF: Volume-weighted variants
- **PriceType**: MID_PX, BID_PX, ASK_PX, LAST_PX, etc.

### EWMA Calculation

Most indicators use Exponentially Weighted Moving Average (EWMA):
$$\text{EWMA}_t = \alpha \cdot X_t + (1-\alpha) \cdot \text{EWMA}_{t-1}$$

where:
$$\alpha = \frac{2}{2.8854 \cdot \text{HalfLife} + 1}$$

### Sign Conventions

- **Positive values** typically indicate:
  - Buying pressure
  - Upward momentum
  - Bid-side strength

- **Negative values** typically indicate:
  - Selling pressure
  - Downward momentum
  - Ask-side strength

---

## Usage Guidelines

1. **Trend Following**: Use SimpleTrend, PxMomentum, TradeTrend
2. **Mean Reversion**: Use Bollinger, OrderBookSkew, RelBollinger
3. **Liquidity Analysis**: Use OrderBookImbalance, BookSize, OrderBookSlope
4. **Order Flow**: Use TradeImbalance, BookDeltaTop, CTrade
5. **Pairs Trading**: Use DynamicBeta, EWBeta, PriceDiff
6. **Volatility**: Use StdevPrice, Bollinger bands
7. **Market Microstructure**: Use L1Events, Sweep, OBPredictor

---

**Last Updated:** 2025-10-13
