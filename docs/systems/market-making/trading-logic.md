# Trading Logic

> Source: migrated from `reference/archive/reference-local/docs/references/market-making/`
> on 2026-07-14. Treat implementation details as historical unless verified
> against the owning code.

This note explains the trading logic in the C++ market-making reference code: how valuation feeds execution, and how quoting and aggressive hedging are driven by adjusted theo.

## End-To-End Strategy

The whole strategy is:

```text
market data + reference data + previous state
   -> fit a live smile
   -> compute raw theo and greeks
   -> penalise theo for inventory / risk
   -> decide passive quotes and aggressive hedge trades
   -> send orders
   -> fills update positions
   -> positions feed back into the next adjusted theo
```

### High-level intuition

| Layer | What it is trying to do |
|---|---|
| valuation | decide what each option is worth right now |
| APS adjustment | make that fair value inventory-aware |
| quoter | earn spread by resting around adjusted fair value |
| hitter / swing | take liquidity when market is clearly through adjusted fair value |
| feedback loop | after fills, make the next fair value more conservative in the crowded direction |

So the strategy is not:

```text
fit smile -> quote raw theo
```

It is:

```text
fit smile -> raw theo -> inventory-aware theo -> trade only with enough edge
```

### Does the whole chain run on every tick

No.

The code uses separate cadences:

```text
full valuation cadence
   -> rebuild smile, greeks, APS, credits

fast base-price cadence
   -> reuse last smile / greeks
   -> refresh fast theo and adjTheo

quote / fill cadence
   -> execution reacts using latest stored valuation state
```

| Event type | What runs |
|---|---|
| full valuation trigger | full `runAlgo()` pipeline |
| base-price update that fails the full-run check | fast theo + adjusted-theo refresh only |
| quote tick | execution only, using latest stored theo / adjTheo |
| fill / order response | incremental position and adjusted-theo update |

So the long path:

```text
forward -> IV -> smile -> theo -> greeks -> APS -> credits
```

is not recomputed on every tick.

### How refit is triggered

The refit trigger lives in `RunAlgoCtrl`.

A full `runAlgo()` happens when a base-price event satisfies at least one of:

| Trigger | Condition |
|---|---|
| small move + minimum wait | `|basePx - lastRunBasePx| > bpxChgThresMin` and `timeGap > intervalMin` |
| large move + shorter wait | `|basePx - lastRunBasePx| > bpxChgThresMax` and `timeGap > intervalMax` |
| timer refresh | periodic refresh timeout path |

So:

- quote ticks do **not** directly trigger smile refit
- base-price updates are what trigger the refit check
- if the refit check fails, the code only recomputes fast theo / adjTheo from
  the last stored valuation state

## System Shape

| Layer | Main role | Key files |
|---|---|---|
| Valuation | Build fair values, greeks, credits, and adjusted theo | `Valuator.cpp`, `theoProv.cpp` |
| Strategy | Receive valuation and market events, maintain local risk, route execution | `Strategy.cpp` |
| Quoting | Resting maker quotes | `QuoteStrat.cpp`, `QuoteStratImpl.cpp` |
| Hitting / hedge | Aggressive IOC / STP / GTC hedge trades | `SwingStrat.cpp`, `SwingStratImpl.cpp` |
| Trigger strategy | Lighter stop-style trigger logic | `BlitzStrat.cpp` |
| Order state machines | Own the OMS lifecycle and working-order state | `OrderExecutionGtc.cpp`, `OrderExecutionStp.cpp` |

```text
market data + base price
        |
        v
    Valuator
        |
        v
theo + greeks + credits + self-adjustment
        |
        v
    Strategy
   /   |    \
  /    |     \
quote swing  blitz
  |      |      |
  v      v      v
 GTC   IOC/STP  STP
        |
        v
 fills -> positions -> risk -> adjusted theo update
```

## Valuation Output

The execution layer does not trade raw model prices. It trades adjusted theo.

| Quantity | Meaning | Source |
|---|---|---|
| `mTheo_` | raw option theo | `ValDataPayload` |
| `mDelta_`, `mGamma_`, `mVega_` | per-instrument greeks | same payload |
| `mFixedCredit_` | minimum fixed edge required to trade | same payload |
| `mPerSzCredit_` | size-dependent extra edge | same payload |
| `mValErrCredit_` | valuation uncertainty edge | same payload |
| `mPerSzSelfAdj_` | diagonal self-adjustment term | same payload |

The main valuation loop is `Valuator::runAlgo()`, which builds:

1. fitted vol smile
2. theo and greeks
3. credits
4. self-adjustment / adjusted theo
5. `ValDataPayload`

## Self Adjustment And Risk Adjustment

These adjustments start after the fitted smile has already produced raw theo and
greeks.

```text
fitted smile
   |
   v
raw theo + greeks
   |
   v
AdjProv builds APS matrix A
   |
   v
position vector p
   |
   v
adj_i = - sum_j A_ij p_j
   |
   v
clip by adj limit
   |
   v
adjTheo_i = theo_i + clipped(adj_i)
```

### 1. Risk dev per instrument

`AdjProv::onUpdate(...)`
first converts each configured risk into a per-instrument `theoRiskDev`.

| Quantity | Formula |
|---|---|
| adjustment scale | `adjFactor = sqrt(2 x overall_multiplier x adjustment_factor)` |
| per-risk dev | `theoRiskDev_i = adjFactor x dev_i^exp x riskTheo_i` |

Notes:

- `riskTheo_i` is the selected greek-like risk sensitivity from `RiskProv`.
- `dev_i` can be a scalar or a per-instrument dev vector from `DevProv`.
- `exp` is the configured `dev_exponent`.

### 2. APS matrix

`AdjProv` then accumulates an APS matrix `A`.

| Corr type | Matrix contribution |
|---|---|
| `cauchy_corr` | `A_ij += theoRiskDev_i x theoRiskDev_j x corr_ij` |
| `iden_corr` | `A_ii += theoRiskDev_i^2` |
| `unit_corr` | `A_ij += theoRiskDev_i x theoRiskDev_j` |

So:

```text
A = sum of all configured corr / iden / unit adjustment blocks
```

Intuition:

```text
row i, col j of A
   = if I already hold 1 extra lot of j,
     how much should instrument i's fair value deteriorate
```

So the matrix is a table of pairwise inventory pain.

Important notation:

| Quantity | Meaning |
|---|---|
| position `p_j` / `pos_j` | signed contract or lot count in instrument `j` |
| not vega | the vega scaling is already embedded inside `A_ij` through `riskTheo` and `theoRiskDev` |

| Matrix slice | Meaning |
|---|---|
| diagonal `A_ii` | own-name pain: how much instrument `i` hurts itself |
| off-diagonal `A_ij` | cross-name pain: how much position `j` should move fair value of `i` |

The code builds each entry from three ingredients:

```text
risk size in i
   x
risk size in j
   x
how connected i and j are
```

More explicitly:

| Ingredient | Trading meaning |
|---|---|
| `riskTheo_i` | how sensitive option `i` is to the chosen risk |
| `dev_i` | how stressed / important that risk is right now |
| `corr_ij` | whether options `i` and `j` should share inventory pain |

So the matrix gets large when:

- both instruments are very sensitive to the same risk
- that risk is currently stressed
- the two instruments are strongly linked by the chosen correlation model

### How instrument correlation is estimated

For the correlated adjustment blocks, the code does not estimate correlation
from time series. It uses a deterministic kernel on option-chain distance.

```text
pick an x-coordinate for each strike
   -> measure distance |x_i - x_j|
   -> turn that distance into a correlation-like weight
```

The active correlated kernel is:

| Quantity | Formula |
|---|---|
| Cauchy correlation | `corr_ij = (1 + shiftDown) / (1 + (|x_i - x_j| / spreadFactor)^2)^power - shiftDown` |

In the common fast-path case `power = 1`, `shiftDown = 0`:

```text
corr_ij = 1 / (1 + (dx / spreadFactor)^2)
```

Intuition:

| Distance in smile space | Correlation weight |
|---|---|
| same / nearby strikes | high |
| far-apart strikes | low |

So this is really a **smile-distance coupling rule**, not a historical return
correlation estimate.

In the shipped config:

| Config | Value |
|---|---|
| kernel type | Cauchy |
| `spread_factor` | `0.25` |
| `shift_down_factor` | `0.4` |
| `power` | `1.0` |
| x-axis | `xProv2 = explogmoneyness` |

So two strikes are considered strongly linked if they are close in
`explogmoneyness`, and weakly linked if they are far apart on that smile axis.

#### What the Cauchy kernel means

The kernel is a smooth distance-decay rule.

```text
same smile neighbourhood
   -> strong coupling
far smile neighbourhood
   -> weak coupling
very far neighbourhood
   -> can even become mildly negative if shiftDown > 0
```

Intuition:

| Distance `|x_i - x_j|` | Effect |
|---|---|
| `0` | `corr_ij = 1`, so the two strikes are treated as fully linked |
| small | still strongly linked |
| moderate | influence decays smoothly |
| very large | with positive `shiftDown`, the tail can go below `0` |

Why that shape makes sense:

- nearby strikes usually share the same local smile / vega risk
- far-apart strikes are less interchangeable
- opposite wings can sometimes act like partial hedges, so a small negative tail
  is plausible

So this kernel is not saying:

```text
historical realised correlation
```

It is saying:

```text
inventory in nearby smile regions should propagate strongly
inventory in distant smile regions should propagate weakly
```

### What `riskTheo` is

`riskTheo_i` is the option-level cash risk exposure used inside the APS build,
before the extra dev scaling is applied.

The code constructs it from a risk factor level and a risk sensitivity:

| Risk type | Formula | Intuition |
|---|---|---|
| first-order risk | `riskTheo_i = riskFactor x sensitivity_i` | delta-like or vega-like cash risk |
| second-order risk | `riskTheo_i = 0.5 x riskFactor1 x riskFactor2 x sensitivity_i` | gamma / vomma / vanna style quadratic exposure |

Examples from the shipped config:

| Risk block | `riskTheo_i` intuition |
|---|---|
| `black_delta` | underlying level x delta |
| `black_vega` | vol level x vega |
| `shadow_gamma_sqd` | roughly `0.5 x S x S x gamma` |
| `shadow_vomma_sqd` | roughly `0.5 x vol x vol x vomma` |

Then `theoRiskDev` is just the stressed version of that cash risk:

```text
theoRiskDev_i = stress_scale x riskTheo_i
stress_scale  = sqrt(2 x overall_multiplier x adjustment_factor) x dev_i^exp
```

So:

| Quantity | Meaning |
|---|---|
| `riskTheo_i` | raw cash exposure to one risk |
| `theoRiskDev_i` | stressed / weighted cash exposure used in APS |

### How different Greeks become one matrix

The code does not first collapse all greeks into one scalar per option.
Instead, it builds one adjustment block per chosen risk, then sums the blocks.

```text
delta block
+ vega block
+ gamma block
+ vomma block
...
= final APS matrix
```

For one risk block `r`:

| Quantity | Formula |
|---|---|
| stressed risk size | `g_r(i) = sqrt(2 x overall_multiplier x adjustment_factor_r) x dev_r(i)^exp_r x riskTheo_r(i)` |
| block contribution | `A_r(i,j) = g_r(i) x g_r(j) x corr_r(i,j)` |

Then:

```text
A(i,j) = sum over risk blocks r of A_r(i,j)
```

So all the different greeks are integrated by **adding their quadratic penalty
matrices**.

In the shipped config, the active blocks are:

| Risk block | Corr type | Intuition |
|---|---|---|
| black delta | `unit_corr` | portfolio-level directional crowding |
| black vega | `cauchy_corr` | local smile vega crowding |
| black vega | `unit_corr` | market-wide vega crowding |
| shadow gamma squared | `unit_corr` | convexity crowding |
| shadow vomma squared | `unit_corr` | vol-convexity crowding |

Examples of `riskTheo`:

| Risk | `riskTheo_i` |
|---|---|
| delta | `S x delta_i` |
| vega | `sigma x vega_i` |
| gamma-like | `0.5 x S^2 x gamma_i` |
| vomma-like | `0.5 x sigma^2 x vomma_i` |

So one option can be large in several blocks at once:

- high delta
- high vega
- high gamma

and each of those contributes separately to inventory pain.

### Why this method makes sense

This is a quadratic inventory penalty model.

```text
position vector p
   -> penalty ~ p' A p
   -> marginal penalty = A p
```

That is a natural fit for options because:

| Reason | Intuition |
|---|---|
| options carry several risks at once | one scalar inventory limit is too crude |
| nearby smile points are substitutes | local coupling should be stronger than distant coupling |
| risk should grow faster than linearly | quadratic penalties make crowding more expensive as size grows |
| different risk types matter simultaneously | summing risk blocks lets delta, vega, gamma, vomma all contribute |

So the method says:

```text
inventory pain should depend on
1. how much cash risk each option carries
2. which smile region that risk lives in
3. how much of that risk the book already owns
```

### What happens after the matrix adjustment

The APS matrix is the last **inventory / portfolio-risk** adjustment to theo,
but it is not the last execution adjustment overall.

```text
raw theo
   -> APS / position adjustment
   -> adjTheo
   -> execution credits and commissions
   -> final quote / hit threshold
```

After `adjTheo`, the execution layer still applies:

| Adjustment after APS | Purpose |
|---|---|
| fixed credit | minimum edge to justify trading |
| per-size credit | extra edge for larger size |
| valuation-error credit | extra edge for model uncertainty |
| bid / ask commission | transaction-cost allowance |
| base-price-error credit | extra edge for underlying uncertainty |

So:

- **no further inventory matrix adjustment** after APS
- **yes, further execution-price adjustments** before a quote or aggressive order
  is actually sent

## From Theo To Quoter And Hitter

The execution path starts from raw theo, but strategies trade on `adjTheo`.

```text
fitted smile
   -> raw theo
   -> APS adjustment
   -> adjTheo
   -> credits / commissions
   -> passive quote price or aggressive hit threshold
```

### 1. Theo -> adjusted theo

| Quantity | Formula |
|---|---|
| raw theo | `theo_i` |
| raw inventory adjustment | `adj_i = -sum_j A_ij p_j` |
| adjusted theo | `adjTheo_i = theo_i + clip(adj_i, -L_i, +L_i)` |

This is the fair value seen by execution.

### 2. Adjusted theo -> quoter

Passive quoting happens in `QuoteStratImpl`.

```text
adjTheo
   -> add required edge
   -> add commission / uncertainty buffers
   -> round to tick
   -> clamp to market / bands
   -> build quote ladder
```

Top-level quote formula:

| Side | Formula |
|---|---|
| bid | `quoteBid ~= adjTheo - requiredEdge - bidComm` |
| ask | `quoteAsk ~= adjTheo + requiredEdge + askComm` |

where `requiredEdge` includes:

| Edge term | Meaning |
|---|---|
| fixed credit | minimum edge to make any quote worthwhile |
| per-size credit | extra edge as size increases |
| valuation-error credit | model uncertainty buffer |
| base-price-error credit | underlying uncertainty buffer |
| tolerance / offset logic | anti-churn / quote-stability control |

Then the quoter:

- rounds to tick
- optionally avoids crossing the live market
- creates deeper levels by stepping away in ticks
- caps size using freeze qty, risk limits, and `perSzSelfAdj`

Intuition:

```text
quoter = "show a price only if I still have enough edge after inventory,
uncertainty, and costs"
```

### 3. Adjusted theo -> hitter

Aggressive execution happens in `SwingStratImpl`.

```text
adjTheo
   -> compare with executable market price or trade print
   -> check if available edge exceeds threshold
   -> size by portfolio / local limits
   -> send IOC / GTC / STP hedge order
```

Decision rule:

| Case | Trigger intuition |
|---|---|
| buy aggressively | market ask is cheap enough versus `adjTheo` |
| sell aggressively | market bid is rich enough versus `adjTheo` |

Equivalent inequality:

| Side | Condition |
|---|---|
| aggressive buy | `adjTheo - marketAsk > requiredEdge` |
| aggressive sell | `marketBid - adjTheo > requiredEdge` |

The same credit family is used here too:

- fixed credit
- per-size credit
- valuation-error credit
- base-price-error credit
- commissions

Then size is clipped by:

- freeze quantity
- local position room
- portfolio delta room
- portfolio vega room
- option-notional room
- displayed book size
- max order amount

Intuition:

```text
hitter = "take liquidity only if the market is far enough through my
inventory-adjusted fair value"
```

### 4. Main difference

| Strategy | Uses `adjTheo` how? |
|---|---|
| quoter | posts around `adjTheo`, but stays back by the required edge |
| hitter | waits for the live market to cross through `adjTheo` by more than the required edge |

### 5. Dedicated hedger

Yes. The main unwanted-risk hedger in this snapshot is `SwingStrat`.

```text
portfolio risk update or market opportunity
   -> SwingStrat
   -> compare executable price to adjTheo
   -> if edge is good enough, send hedge order
```

It is a **price-sensitive hedger**, not a blind neutraliser.

| Risk / limit it watches | Use |
|---|---|
| portfolio delta | trigger and size cap |
| portfolio vega | trigger and size cap |
| option notional | size cap |
| local instrument / strike position | size cap |

So the hedge rule is:

```text
reduce unwanted risk
only when the market gives an acceptable price
```

`BlitzStrat` can act as a lighter trigger-based hedge helper, but `SwingStrat`
is the main hedging engine.

### 5. Passive quote size and number of levels

The passive ladder starts from a configured template, then live risk filters cut
it down.

```text
configured ladder template
   -> top / bottom size caps
   -> skip-level logic
   -> price-band checks
   -> cumulative risk cap
   -> final active levels
```

Configured per-level size template:

| Level `k` | Formula |
|---|---|
| template size | `templateSz_k = clip(topSize + stepSize x k, 0, maxSize)` |

All template sizes are rounded to lot size.

Then the live caps are:

| Quantity | Rough formula |
|---|---|
| top-level cap | `maxTop ~= min(freezeQty, maxOrderAmount / (theo + maxNumLevel x tick))` |
| deeper-level cap | `maxBot ~= min(freezeQty, tick / perSzSelfAdj, maxOrderAmount / (theo + maxNumLevel x tick))` |

So:

| Level | Size before portfolio cap |
|---|---|
| top | `min(templateSz_0, maxTop)` |
| deeper `k >= 1` | `min(templateSz_k, maxBot)` |

Then the whole ladder is clipped by one cumulative size cap:

| Cumulative cap is based on | Meaning |
|---|---|
| local position room | per-instrument inventory limit |
| portfolio delta room | total delta limit |
| portfolio vega room | total vega limit |
| strike position room | combined call/put strike limit |

The code walks from deepest level back toward the top and removes excess size so
that:

```text
sum(all quoted sizes) <= maxCumSize
```

Number of levels:

| Quantity | Meaning |
|---|---|
| `mMaxNumLevel_` | configured maximum levels |
| `mCurrSkipLevel_` | intentional front levels skipped by tolerance / margin logic |
| `mCumLvl` | non-zero levels left after all filters |

So the practical answer is:

- configured levels = `mMaxNumLevel_`
- actually live quoted levels = `mCumLvl`

### 6. Aggressive hit size

The hitter does not build a ladder. It produces one order size.

| Final swing size is the minimum of | Meaning |
|---|---|
| configured max size | local strategy cap |
| freeze quantity | exchange / instrument cap |
| local position room | instrument position limit |
| portfolio delta room | remaining delta capacity |
| portfolio vega room | remaining vega capacity |
| option-notional room | remaining notional capacity |
| max order amount | price-based cap |
| credit-allowed size | edge available at this price |
| displayed market size | available liquidity |
| strike position room | strike-level combined cap |

After that minimum is taken, it is rounded down to lot size.

### 3. Portfolio risk adjustment

`calcAdjTheoOnPositionImpl(...)`
turns APS and positions into an adjustment vector.

| Quantity | Formula |
|---|---|
| raw adjustment | `adj_i = - sum_j A_ij x pos_j` |
| limit | `adjLimit_i = clip(theo_i x adj_limit_factor, adj_limit_min, adj_limit_max)` |
| adjusted theo | `adjTheo_i = theo_i + clip(adj_i, -adjLimit_i, +adjLimit_i)` |

This is the main **risk adjustment**. It is portfolio-aware because instrument
`i` is adjusted by all positions `j`, not just its own.

Intuition:

| If you already hold | Then the model does |
|---|---|
| a long portfolio that loads the same risk as instrument `i` | lowers `adjTheo_i`, so buying more looks worse and selling looks easier |
| a short portfolio that loads the opposite risk | raises `adjTheo_i`, so buying back risk looks more attractive |
| a very large position | stops moving linearly once the adjustment hits the configured cap |

So the adjustment behaves like an internal shadow cost of inventory:

```text
same-direction inventory -> worse price
opposite-direction inventory -> better price
```

### 4. Self adjustment

The code also publishes the diagonal term:

| Quantity | Formula |
|---|---|
| per-size self adjustment | `selfAdj_i = A_ii` |

This is sent as `mPerSzSelfAdj_` in
`ValDataPayload`.

Meaning:

- `A_ii` is the marginal own-position penalty for instrument `i`
- it is not the full portfolio adjustment
- it is the diagonal slice of the same APS matrix used to build `adjTheo`

Intuition:

| Quantity | Trading meaning |
|---|---|
| small `A_ii` | you can add size in that name without much extra self-penalty |
| large `A_ii` | every extra lot in that name should hurt your valuation quickly |

So `A_ii` is a local slope:

```text
one more lot of this instrument
   -> how fast should my own fair value deteriorate
```

So yes:

| Object | Comes from |
|---|---|
| self adjustment | diagonal of the APS matrix |
| risk adjustment | full matrix times position vector |

### 5. Where each one is used

| Quantity | Used for |
|---|---|
| `adjTheo_i` | fair value for quoting, hitting, and hedge decisions |
| `selfAdj_i = A_ii` | quote-size control in `QuoteStratImpl` |

In the quote ladder, `QuoteStratImpl` uses `mTick / mPerSzSelfAdj_` as one of
the size caps for deeper levels. So the diagonal term acts like a per-lot
inventory penalty slope.

Intuition:

| If `selfAdj_i` is | Then quote sizing does |
|---|---|
| large | shrink size faster |
| small | allow more displayed size |

The idea is:

```text
if one extra lot already costs about one tick of inventory penalty,
do not quote many extra lots at the same level
```

### 6. Fill-time update

After a fill, the strategy updates the same adjustment incrementally instead of
rebuilding it from scratch.

| Quantity | Formula |
|---|---|
| execution update | `adj_i <- adj_i - A_row,i x execSz` |
| updated adjusted theo | `adjTheo_i = theo_i + clip(adj_i, -adjLimit_i, +adjLimit_i)` |

So:

- **risk adjustment** = full matrix-times-position adjustment applied to theo
- **self adjustment** = diagonal APS term, published separately and used mainly
  as a local quoting-size penalty

## Strategy Event Flow

### Valuation update

`Strategy::doEvent(const ValDataPayload&)`

| Step | Action |
|---|---|
| 1 | recompute portfolio delta, gamma, vega, option-notional |
| 2 | rebuild fast theo from base price and theo offsets |
| 3 | rebuild adjusted theo using APS and adjustment limits |
| 4 | call execution strategies |

### Base-price update

`Strategy::doEvent(const BpgDataPayload&)`

| Step | Action |
|---|---|
| 1 | update base price and base-price error |
| 2 | rebuild fast theo |
| 3 | rebuild adjusted theo |
| 4 | notify execution strategies if the move is within configured limits |

### Quote update

`Strategy::handleQuote(...)`

| Step | Action |
|---|---|
| 1 | update displayed available sizes |
| 2 | update delayed trade / LTP state |
| 3 | forward the quote event to execution strategies |

### Fill or OMS response

`Strategy::handleOrderResponse(...)`

| Step | Action |
|---|---|
| 1 | update positions and portfolio risk on trade |
| 2 | incrementally update adjusted theo |
| 3 | route the response back to the originating strategy |

## Credits and Edge Model

`CreditProv::onUpdate(...)`

| Output | Meaning | Shape |
|---|---|---|
| fixed credit | minimum fixed premium to justify trading | per instrument |
| per-size credit | extra premium that scales with order size | per instrument |
| valuation-error credit | extra premium for model uncertainty | per instrument |
| bid / ask commission | transaction-cost estimates | per instrument |

These are built from:

- delta and vega exposure
- spread
- base-offset error
- IV error
- RV factors
- transaction costs

## Quoting Logic

### Purpose

`QuoteStrat` is the passive maker. It computes resting bid and ask ladders around adjusted theo and passes them into the GTC executor.

### Quote construction

`QuoteStratImpl::computeQuotesAt(...)`

| Input | Meaning |
|---|---|
| adjusted theo | fair value after inventory/risk adjustment |
| base-price-error credit | extra edge for base-price uncertainty |
| market limit price | clamp against current market / override provider |
| context | delta, lot size, freeze qty, per-size self-adjustment, DPR/LPP bounds, positions |

| Output | Meaning |
|---|---|
| `intend_px[]` | theoretical ladder before market collapse |
| `curr_px[]` | actual ladder sent to OMS |
| `curr_sz[]` | size ladder |
| tick / level count / skipped levels | execution metadata |

The quote formula is:

```text
top_quote ~= adjusted_theo
             +/- required_credit
             +/- commission
             then rounded to tick
```

The ladder then fans out level by level with fixed tick spacing, after which:

- price bands are enforced
- skipped levels are zeroed
- size is clipped by local and portfolio limits

### Quote lifecycle

`QuoteStrat` does not own live orders directly. It sets targets on the GTC executor:

- `OrderExecutionGtc::setTargetAt(...)`
- executor handles place / modify / cancel / refresh / refill

## Aggressive Hedge Logic

### Purpose

`SwingStrat` is the main aggressive hedger. It looks for immediately tradable prices that offer enough edge relative to adjusted theo and enough risk relief to justify taking liquidity.

### Opportunity detection

Two main entry points:

| Path | What it reacts to |
|---|---|
| trade hitter | a trade just printed |
| scan and swing | current executable market price |

Core functions:

- `prepTradeHitterOrderAt(...)`
- `prepOrderAt(...)`

### Order sizing

`prepOrderSzAt(...)`

The final hedge size is the minimum of:

1. configured max swing size
2. freeze quantity
3. local position room
4. portfolio delta room
5. portfolio vega room
6. option-notional room
7. max order amount
8. credit-allowed size
9. displayed market size
10. strike-position room

### Routing

`SwingStrat::placeOrderAt(...)`

If the chosen executor is stop-based, trigger price is chosen by cascade:

```text
configured trigger
  -> fresh LTP
  -> top of book
  -> DPR boundary
```

The order is then fired as IOC, GTC, or STP depending on the strategy instance.

## Blitz Logic

`BlitzStrat` is a lighter trigger strategy. It reacts to quote changes and maintains a small stop-style target rather than a full maker ladder or a broad hedge scanner.

Main path:

- quote update
- compute one trigger/target pair
- set or reset stop order

Source: `BlitzStrat.cpp`

## Order Executors

### GTC executor

`OrderExecutionGtc.cpp`

Owns:

- working ladder state
- open-slot bookkeeping
- refresh / refill
- response handling

### STP executor

`OrderExecutionStp.cpp`

Owns:

- stop-order lifecycle
- trigger-side order bank
- response handling for stop orders

## OMS Executor, Fills, And Orchestration

### What the OMS executor does

The execution strategies do **not** place OMS orders directly. They set targets
on an executor.

```text
execution strategy
   -> target prices / sizes
   -> OMS executor
   -> place / modify / cancel / refresh
   -> OMS responses
```

Main idea:

| Component | Responsibility |
|---|---|
| `QuoteStrat` / `SwingStrat` / `BlitzStrat` | decide what they want |
| `OrderExecutionGtc` / `OrderExecutionStp` | own live order state and talk to OMS |

So the executor is the order-state machine.

| Component | Talks to OMS directly? |
|---|---|
| quoter / hitter strategy | no |
| executor | yes |

### Executor as a state machine

The executor's job is:

```text
desired target state
   ->
live OMS order state
   ->
smallest safe set of actions
   ->
place / modify / cancel / refresh
```

So it continuously answers:

| Question | Meaning |
|---|---|
| what orders should exist now? | current target state from strategy |
| what orders already exist? | current OMS-backed live state |
| what changed? | delta between target and live state |
| what action is allowed now? | pacing, ammo, delay, connection limits |

### Core executor state

Per instrument-side, the executor keeps:

| State | Meaning |
|---|---|
| target price array | what price each order-instance slot should have |
| target size array | what size each slot should have |
| internal / intended price array | pre-collapse or reference ladder price |
| trigger price array | stop trigger level for stop-style executors |
| order-instance bank | handles / metadata for live OMS orders |
| order-instance book bitset | which slots are currently occupied |
| ammo bitset | which slots are free to load / reuse |
| swung bitset | which slots have already been consumed by swing logic |
| delay counters | when each slot is next allowed to refresh |

This is why the executor exists: strategies should not manage all of that
exchange-facing state themselves.

### Main executor operations

| Operation | Meaning |
|---|---|
| `setTargetAt(...)` | update desired order state |
| `fireOrderAt(...)` | send one immediate order |
| `refreshOrdersAt(...)` | reconcile current live orders with target |
| `loadOrdersAt(...)` | create missing live orders for target slots |
| `resetTargetAt(...)` | clear desired state and unwind live orders |
| `handleOrderResponseAt(...)` | absorb OMS acknowledgements, cancels, fills, rejects |

### How it decides what to do

For resting orders, the executor compares:

```text
target slot state
vs
live slot state
```

Then:

| Situation | Typical action |
|---|---|
| target exists, live order missing | place |
| target changed, live order exists | modify |
| target removed, live order exists | cancel |
| target unchanged | do nothing |
| slot blocked by pacing / delay | wait and retry later |

### Why there are different executors

| Executor | Best for |
|---|---|
| `OrderExecutionGtc` | passive resting ladders |
| `OrderExecutionStp` | stop / trigger orders |
| `OrderExecutionIoc` | one-shot immediate orders |

Same pattern, different order semantics.

### GTC executor

`OrderExecutionGtc` stores, per side and instrument:

| State it owns | Meaning |
|---|---|
| target price / size arrays | what the strategy currently wants |
| internal order-instance bank | OMS-local order slots |
| order book bitset | which order-instance slots are occupied |
| ammo / delay counters | pacing / refresh / reload control |
| swung state | how much execution state has already been consumed |

Flow:

```text
setTargetAt
   -> remember desired ladder
   -> refresh / load / cancel as needed
   -> handle OMS responses
```

### STP executor

`OrderExecutionStp` does the same style of state management, but for
trigger-style orders:

| Extra state | Meaning |
|---|---|
| trigger price arrays | stop trigger level |
| target price / size arrays | actual order to place when trigger logic allows |

So:

- GTC executor = resting ladder manager
- STP executor = stop / trigger order manager

### What happens on a fill

When `Strategy::handleOrderResponse(...)` receives an OMS trade:

```text
OMS trade
   -> decode local strategy / instrument / side bits
   -> update positions
   -> update portfolio greeks
   -> incrementally update adjusted theo
   -> hand the response to the owning execution strategy
   -> publish execution payload
```

Position and risk update:

| Quantity | Formula |
|---|---|
| instrument position | `pos_i <- pos_i + execSz` |
| portfolio delta | `portDelta <- portDelta + execSz x delta_i` |
| portfolio gamma | `portGamma <- portGamma + execSz x gamma_i` |
| portfolio vega | `portVega <- portVega + execSz x vega_i` |
| option-notional proxy | updated from signed option type x base price x size |

Adjusted-theo update on fill:

| Quantity | Formula |
|---|---|
| incremental adjustment | `adj_k <- adj_k - A_ik x execSz` |
| new adjusted theo | `adjTheo_k = theo_k + clip(adj_k, -L_k, +L_k)` |

Here `i` is the filled instrument and `k` runs over the whole universe.

Intuition:

```text
one fill changes inventory immediately
   -> inventory penalty changes immediately
   -> next quote / hit decision should use the new inventory state
```

### How the orchestrator works

The main orchestrator is `Strategy`.

It owns:

| Responsibility | Meaning |
|---|---|
| latest valuation state | theo, adjTheo, greeks, credits |
| live positions and shadow positions | current book state |
| portfolio risk aggregates | delta, gamma, vega, notional |
| execution-strategy list | quoter, swing, blitz, manual, etc. |
| routing of market / valuation / OMS events | dispatch hub |

Event flow:

```text
Valuation event
   -> Strategy updates stored theo / adjTheo context
   -> notifies execution strategies

Base-price event
   -> Strategy recomputes fast theo / adjTheo
   -> notifies execution strategies

Quote event
   -> Strategy updates avail sizes / delay-trade state
   -> forwards quote to execution strategies

OMS response
   -> Strategy updates positions / risk / adjTheo
   -> forwards response to the originating execution strategy
```

So `Strategy` is the coordinator:

```text
valuation decides fair value
execution strategies decide intent
executors manage live OMS order state
Strategy ties the whole loop together
```

## From Quoter / Hitter To Market

### Passive quoter path

The passive path is:

```text
QuoteStrat
   -> compute ladder arrays
   -> OrderExecutionGtc.setTargetAt(...)
   -> executor stores desired ladder
   -> executor load / refresh loop decides place / modify / cancel
   -> OMS transaction manager
   -> exchange
```

The important boundary is:

| Step | What is passed |
|---|---|
| quoter -> executor | intended prices, current prices, sizes, tick, number of levels, skipped levels |
| executor -> OMS | concrete place / modify / cancel requests |

So the quoter does **not** send an order directly. It updates the executor's
target ladder.

### Aggressive hitter path

The aggressive path is:

```text
SwingStrat
   -> find one target price / size
   -> OrderExecution*.fireOrderAt(...)
   -> executor chooses place / modify path
   -> OMS transaction manager
   -> exchange
```

Unlike the quoter:

| Path | Handoff style |
|---|---|
| quoter | `setTargetAt(...)` on a ladder |
| hitter | `fireOrderAt(...)` for one immediate order |

### How the executor reaches OMS

Before placing, the executor writes strategy-routing bits into the OMS
transaction manager:

```text
instrument id
local strategy id
order-instance id
side
```

Then the executor calls the order object's `place_order(...)` through
`mOmsTranMgr_`.

Conceptually:

```text
executor
   -> fill OmsTranMgr with routing / side / price / size / trigger info
   -> place_order(...)
   -> OMS API call goes out
```

The OMS manager also carries:

- order price
- trigger price for stop orders
- order size
- order term (`GTC`, `IOC`, stop-style)
- connection / block / tap routing metadata

### How market responses come back

After the exchange / OMS replies:

```text
market / OMS response
   -> Strategy::handleOrderResponse(...)
   -> update positions / risk / adjTheo on trades
   -> forward response to owning executor / strategy
```

So the full round-trip is:

```text
quoter / hitter intent
   -> executor target or fire call
   -> OmsTranMgr / place_order
   -> market
   -> OMS response
   -> Strategy
   -> position and risk update
```

## Current Limitation In This Snapshot

The valuation path supports multiple option universes, but execution currently ignores non-zero universe IDs.

Source:

- `Valuator` handles `mOptionUnivId_`
- `Strategy.cpp` returns early unless `option_univ_id == 0`
