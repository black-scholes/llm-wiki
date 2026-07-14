# Implied Vol and Forward Construction

> Source: migrated from `reference/archive/reference-local/docs/references/market-making/`
> on 2026-07-14. Treat implementation details as historical unless verified
> against the owning code.

This note explains how the reference market-making code converts option quotes into implied vol, how it estimates the forward used in that inversion, and how those pointwise observations become a fitted smile.

## Shapes

For one expiry:

| Symbol | Meaning |
|---|---|
| `n_inst` | number of option instruments in the expiry chain |
| `n_strike` | `n_inst / 2` |

Instrument order is:

```text
[call(K0), put(K0), call(K1), put(K1), ...]
```

| Vector | Shape |
|---|---|
| option prices, sizes, deltas, vegas | `n_inst` |
| implied-vol smile, strike, x-coordinate | `n_strike` |

## Current-Cycle Data Flow

Inputs for one valuation cycle are only:

| Input class | Examples |
|---|---|
| reference data | `refPx`, strike vector, expiry, `sqrtTau`, `df` |
| market data | option bid/ask prices and sizes |
| previous state | previous smile, previous offset memory, solver warm-starts |

The current-cycle data flow is:

```text
reference data
+ market data
+ previous state
   |
   v
bootstrap ATM sigma estimate
   |
   v
raw forward quotes per instrument
   |
   v
raw instrument forward f_i
   |
   v
expiry forward F_exp
   |
   v
raw instrument offset instBo_i
   |
   v
tgtBo_i
   |
   v
memBo_i
   |
   v
fitBo_i
   |
   v
boRes_i
   |
   v
final instrument forward F_i
   |
   v
bid IV / ask IV per option
   |
   v
observed IV + error + weight
   |
   v
per-strike target IV
   |
   v
vol memory
   |
   v
vol smile spline fit
```

## Order Around Bid/Ask IV

The correct ordering is:

```text
quote
   -> raw forward / offset pipeline
   -> final F_i
   -> bid IV / ask IV
   -> later vol-smile fitting
```

So:

| Ordering statement | Correct? |
|---|---|
| `quote -> F_i -> bid/ask IV -> later vol-smile spline fitting` | yes |
| `quote -> F_i -> current-cycle vol-smile spline fitting -> bid/ask IV` | no |

Why this looked contradictory:

| Spline stage | Happens when | Purpose |
|---|---|---|
| base-offset spline | before bid/ask IV | creates `fitBo_i`, then `boRes_i`, then `F_i` |
| vol-smile spline | after bid/ask IV | fits the current-cycle IV smile |

## After Bid/Ask IV

Once bid IV and ask IV exist for each option, the next stages are:

```text
bid IV / ask IV
   -> status mark / fill missing IV quotes
   -> observed IV + error + weight per option
   -> per-strike target IV
   -> vol memory
   -> fitted smile
```

| Stage | Output |
|---|---|
| `VFVolaPolator` | status-marked IV quotes |
| `VolaSrcProv` | one observed IV, error, weight per option |
| `VolaTgtProv` | one target IV, error, weight per strike |
| `VolaMemProv` | remembered / smoothed IV per strike |
| `VolaFitter` / `VolaFitProv` | fitted smile per strike |

## Stages Up To Per-Strike Target IV

This is the ordered flow from final forward `F_i` to per-strike target IV.

```text
final F_i
   -> bid IV / ask IV per option
   -> status-mark / fill missing IV quotes
   -> observed IV + error + weight per option
   -> per-strike target IV
```

### 1. Final forward `F_i`

| Input | Meaning |
|---|---|
| `refPx` | current reference price |
| `boRes_i` | final residual base offset |

In the shipped config:

```text
F_i = refPx + boRes_i
```

### 2. Quote -> bid IV / ask IV

For each option instrument, bid and ask are inverted separately.

| Input | Used by |
|---|---|
| option bid / ask quote | current market data |
| `F_i` | final instrument forward |
| `K` | strike |
| `sqrtTau`, `df` | expiry / discount inputs |

| Output | Shape |
|---|---|
| bid IV | `n_inst` |
| ask IV | `n_inst` |

`df` is needed here because the inversion is Black-76:

```text
Call = df x (F x N(d1) - K x N(d2))
Put  = df x (K x N(-d2) - F x N(-d1))
```

So quote -> bid/ask IV uses:

- quote
- `F_i`
- `K`
- `sqrtTau`
- `df`

In the shipped config, `df` comes from the `DFProv` interest rate:

```text
df = exp(-0.08 x calT2E)
```

because:

```text
interest_rate = 0.08
stt_rate = 0
```

### 3. Status mark / fill missing IV quotes

`VFVolaPolator` marks each IV quote:

| Status | Meaning |
|---|---|
| `0` | direct market IV |
| `1` | interpolated |
| `2` | extrapolated |
| `<0` | unusable |

This still remains per option side.

Only the configured OTM side is filled. The code runs calls and puts
separately.

#### Interpolated IV

If a missing IV lies between two valid IV points `(x0, y0)` and `(x1, y1)`:

```text
slope = interpSlopeMult x (y1 - y0) / (x1 - x0)
iv(x) = max( y0 + (x - x0) x slope, minVola )
status = 1
```

#### Extrapolated IV

If a missing IV lies outside the observed range:

```text
slope = clamp(
  extrapSlopeMult x (y1 - y0) / (x1 - x0),
  minSlope,
  maxSlope
)

iv(x) = max( y0 + (x - x0) x slope, minVola )
status = 2
```

There are separate slope clamps for:

- lower-strike extrapolation
- higher-strike extrapolation

#### Unstable / unusable IV

At this stage, unstable means the IV quote is not trusted as a direct market
observation.

| Case | What happens |
|---|---|
| direct market IV exists | keep value, `status = 0` |
| missing but fillable on OTM side | interpolate / extrapolate, `status = 1 or 2` |
| missing and not fillable | leave unusable, status stays negative |

Then in the next stage (`VolaSrcProv`), unstable quotes are penalised through
larger error floors / multipliers:

| Status pattern | Error treatment |
|---|---|
| both direct | `minErr` |
| one side filled | `minLerpErr` |
| both sides filled | `minNoMktErr` |
| crossed filled market | extra `minCrossErr` and multiplier |
| negative status | effectively infinite error / zero weight |

### 4. Bid/ask IV -> observed IV + error + weight

`VolaSrcProv` collapses bid IV and ask IV into one observed IV per option.

With the shipped `mid_px` quote pricer:

```text
srcIV_i  = (bidIV_i + askIV_i) / 2
rawErr_i = (askIV_i - bidIV_i) / 2
```

Then error is inflated by quote quality:

```text
srcErr_i = max(minErrFloor_i, statusMultiplier_i x rawErr_i) + max(0, tickSzFactor / vega_i)
srcWt_i  = 1 / srcErr_i^2
```

So output of this stage is:

| Output | Shape |
|---|---|
| `srcIV_i` | `n_inst` |
| `srcErr_i` | `n_inst` |
| `srcWt_i` | `n_inst` |

### 5. Per-option IV -> per-strike target IV

`VolaTgtProv` merges call-side and put-side information into one strike-level
target IV.

For each strike:

1. decide which side is OTM and which side is ITM
2. apply strike-relative weights
3. apply an ITM down-weight / adjustment factor
4. optionally blend with memory IV

The target weight is:

```text
tgtWt = itmFactor x itmWt + otmWt + memWt
```

and the target IV is:

```text
tgtIV = (memWt x memIV + itmFactor x itmWt x itmIV + otmWt x otmIV) / tgtWt
```

with:

```text
tgtErr = 1 / sqrt(tgtWt)
```

So this is the first point where the flow becomes strike-level:

| Output | Shape |
|---|---|
| `tgtIV` | `n_strike` |
| `tgtErr` | `n_strike` |
| `tgtWt` | `n_strike` |

### 6. Per-strike target IV -> remembered IV

`VolaMemProv` takes the strike-level target IV and produces the IV vector that
will actually be fit.

In the shipped config:

```text
memIV  = tgtIV
memErr = tgtErr
```

because `VolaMemProv.type = default`.

If memory mode were `kalman`, this stage would smooth target IV over time
before fitting.

### 7. Remembered IV -> fitted smile

`VolaFitter` passes the remembered strike-level IVs into `VolaFitProv`.

Inputs to the fit are:

| Input | Shape | Meaning |
|---|---|---|
| `X` | `n_strike` | fit x-coordinate per strike |
| `memIV` | `n_strike` | y-values to fit |
| `memErr` | `n_strike` | per-strike error |
| `tgtWt` | `n_strike` | per-strike fit weight |
| `vega` | `n_inst` | used for optional fit reweighting |
| `aEstAtmVola` | scalar | ATM-vol reference |

The fit step is:

```text
fitted smile = smoothing spline through (X, memIV, tgtWt)
```

Output is:

| Output | Shape |
|---|---|
| `fitIV` | `n_strike` |
| fitted ATM IV | scalar |
| fit error | `n_strike` diagnostic / error vector |

So after one IV per strike exists, the next use is:

```text
tgtIV
   -> remembered IV
   -> smoothing spline fit
   -> fitted smile
```

### What happens after the fitted smile

After the fitted smile exists, the valuation path continues as:

```text
fitted smile
   -> option theo
   -> greeks
   -> self-adjustment / risk adjustment
   -> spread and credit model
   -> publish valuation payload to execution
```

More explicitly:

| Stage | What it produces |
|---|---|
| `theoProv::onUpdate` | option fair values / theo from `boRes_i` + fitted smile |
| `theoProv::computeGreeks` | delta, gamma, vega and related greeks |
| `AdjProv::onUpdate` | self-adjustment matrix / inventory-risk adjustment inputs |
| `theoProv::onPosition` | position-adjusted theo |
| `SpreadProv::onUpdate` | market spread vs theo inputs |
| `CreditProv::onUpdate` | quoting / hitting credits and commissions |
| valuation payload publish | sends theo, greeks, credits, offsets to strategy/execution |

### Vol-smile spline model

This is a single strike-level spline over the remembered IV points, not the
base-offset spline.

| Input | Shape |
|---|---|
| `X` | `n_strike` |
| `memIV` | `n_strike` |
| `tgtWt` | `n_strike` |

The knots are the valid strike-level `X` points used in the fit:

```text
knots = { X[k] : memIV[k] is finite and fit weight is finite }
```

If fewer than 4 valid points exist, the spline is not built and the values are
passed through.

The model is a **natural cubic smoothing spline**.

On interval `[x_j, x_{j+1}]`:

```text
f_j(x) = d0_j + (x - x_j) x d1_j
       + 0.5 x (x - x_j)^2 x d2_j
       + (1/6) x (x - x_j)^3 x d3_j
```

Natural boundary condition:

```text
f''(left end)  = 0
f''(right end) = 0
```

Smoothing strength:

```text
lambda = smoothing_factor x smoothingFactorMult
```

In `VolaFitProv`, the fit weight is:

```text
fitWt_k = tgtWt_k
```

optionally multiplied by average strike-pair vega if `bVegaExponent != 0`.

#### How many parameters it has

It is not a fixed low-parameter model.

If there are `N` valid strike points in the fit:

| Quantity | Count |
|---|---|
| knots | `N` |
| spline segments | `N - 1` |
| raw cubic coefficients per segment | `4` |
| raw segment coefficients total | `4 x (N - 1)` |

But those raw coefficients are linked by continuity and natural-boundary
constraints, so they are not all independent.

The practical answer is:

| View | Parameter count |
|---|---|
| raw piecewise-cubic representation | `4 x (N - 1)` coefficients |
| independent spline degrees of freedom | roughly `N`, with smoothness controlled by `lambda` |

So unlike a low-dimensional parametric vol-dynamics model, the spline is a
flexible curve whose complexity grows with the number of valid strike points.

It is not forced to pass exactly through every point, because this is a
**smoothing** spline, not an exact interpolating spline.

| Statement | True? |
|---|---|
| roughly one degree of freedom per valid strike point | yes |
| smooth curve over strike/x points | yes |
| must pass exactly through all points | no |

### Cubic spline math

Suppose the valid fit knots are:

```text
x_0 < x_1 < ... < x_N
```

On each interval `[x_j, x_{j+1}]`, the spline is one cubic polynomial:

```text
f_j(x) = a_j + b_j (x - x_j) + c_j (x - x_j)^2 + d_j (x - x_j)^3
```

In this code, the same segment is written as:

```text
f_j(x) = d0_j
       + (x - x_j) d1_j
       + 0.5 (x - x_j)^2 d2_j
       + (1/6) (x - x_j)^3 d3_j
```

where:

```text
a_j = d0_j
b_j = d1_j
c_j = d2_j / 2
d_j = d3_j / 6
```

The spline is not just "many unrelated cubics". Adjacent segments are tied
together by smoothness constraints at every interior knot `x_j`:

```text
f_{j-1}(x_j)   = f_j(x_j)
f'_{j-1}(x_j)  = f'_j(x_j)
f''_{j-1}(x_j) = f''_j(x_j)
```

So:

| Constraint | Meaning |
|---|---|
| value continuity | no jumps |
| slope continuity | no kinks |
| curvature continuity | smooth bending |

Natural boundary conditions are:

```text
f''(x_0) = 0
f''(x_N) = 0
```

So the curve is free to bend in the middle, but the end curvature is pinned to
zero.

### What the smoothing does

An interpolating spline would try to hit all points exactly.

This code uses a **smoothing** spline, which instead balances:

```text
fit to data
vs
curve roughness
```

Conceptually, it solves:

```text
minimise
  sum_k w_k (y_k - f(x_k))^2
  + lambda x roughness(f)
```

where:

| Term | Meaning |
|---|---|
| `y_k` | strike-level IV input |
| `w_k` | fit weight |
| `lambda` | smoothing strength |
| `roughness(f)` | penalty on excessive curvature |

Interpretation:

| If `lambda` is small | spline follows the points closely |
|---|---|
| If `lambda` is large | spline becomes smoother, less point-sensitive |

### Vol-dynamics style model vs this spline approach

#### Low-parameter vol-dynamics / parametric smile

A parametric smile model usually says:

```text
iv(x) = g(x; theta)
```

where `theta` is a small vector such as:

```text
theta = [atm, skew, curvature, ...]
```

Examples of the idea:

| Model style | Typical parameters |
|---|---|
| quadratic smile | ATM level, slope, curvature |
| SABR-like | alpha, beta, rho, nu |
| SVI-like | level, skew, wing, shift, curvature |

These models have:

| Property | Parametric model |
|---|---|
| number of parameters | small, fixed |
| shape flexibility | limited by formula |
| interpretation | often economically cleaner |
| local fit quality | can be worse if market shape is irregular |

#### This code’s spline smile

This code effectively says:

```text
iv(x) = smooth piecewise cubic curve through strike-level IV observations
```

with complexity that grows with the number of valid strike points.

So:

| Property | Spline approach here |
|---|---|
| number of parameters | grows with data points |
| shape flexibility | high |
| interpretation | less structural, more geometric |
| local fit quality | usually better for irregular live smiles |

### Practical difference

Parametric vol-dynamics thinking:

```text
find a small set of smile parameters
-> generate the whole curve
```

This code’s spline thinking:

```text
observe many strike IV points
-> smooth them into one curve
```

So the main difference is:

| Approach | Core object |
|---|---|
| vol-dynamics / parametric | a few structural parameters |
| this code | a smooth nonparametric curve over strike/x |

### What `itmWt`, `otmWt`, and `itmFactor` mean

For each strike, the code first decides which side is OTM and which is ITM.

| Quantity | Meaning |
|---|---|
| `otmWt` | source IV weight of the OTM side after strike-relative weighting |
| `itmWt` | source IV weight of the ITM side after strike-relative weighting |
| `itmFactor` | extra multiplier applied to the ITM side before it enters the strike target |

Formally:

```text
otmWt = srcWt_otm x relWt_otm
itmWt = srcWt_itm x relWt_itm
```

Then:

```text
tgtWt = itmFactor x itmWt + otmWt + memWt
```

So `itmFactor` controls how much the ITM side contributes relative to the OTM
side.

In the shipped config:

| Config | Value |
|---|---|
| `itm_factor.type` | `one` |
| `use_delta_ratio_mult` | `false` |
| `itm_factor_mult` | `1.0` |

So here:

```text
itmFactor = 1
```

and therefore:

```text
tgtWt = itmWt + otmWt + memWt
```

Meaning: in the shipped config, ITM and OTM weights are treated symmetrically
at this step, apart from whatever difference already exists in `srcWt` and
`relWt`.

### Weight map

The weights are built in layers:

| Name | Formula |
|---|---|
| source weight | `srcWt_i = 1 / srcErr_i^2` |
| OTM weight at strike | `otmWt = srcWt_otm x relWt_otm` |
| ITM weight at strike | `itmWt = srcWt_itm x relWt_itm` |
| memory weight | `memWt = 1 / memErr^2` if memory is used, else `0` |
| target strike weight | `tgtWt = itmFactor x itmWt + otmWt + memWt` |

### Why this feels contradictory

There are two different spline fits in the valuation loop:

| Spline stage | Happens when | Purpose |
|---|---|---|
| base-offset spline | before bid/ask IV | turns memory base offsets into `fitBo`, then `boRes_i`, then `F_i` |
| vol-smile spline | after bid/ask IV | fits the current-cycle implied-vol smile |

So the fully correct ordering is:

```text
quote
   -> raw forward estimate f_i
   -> base-offset spline / boRes_i
   -> final F_i
   -> bid IV / ask IV
   -> vol-smile spline
```

## Timing: Why This Is Not A Same-Step Circular Solve

The code does not solve `F` and `sigma` simultaneously from the same quote in
one equation.

It uses a staged update:

```text
previous smile sigma
   |
   v
quote -> forward estimate
   |
   v
final forward F_i
   |
   v
quote -> current IV
   |
   v
current smile fit
   |
   v
used on the next update
```

| Stage | Sigma source |
|---|---|
| `reverse_b76` forward estimation | existing smile state from `mVolaFitter_` |
| final quote-to-IV inversion | final `F_i` from `VFFutProv` |
| next cycle forward estimation | updated smile from the previous cycle |

So the circularity is broken by time and sequencing.

## Cold Start: First Sigma From Reference Data And Market Quotes

At cold start, there is no valid fitted smile yet:

| State at start | Value |
|---|---|
| `mIsVolaValid_` | `false` |
| previous fitted smile | not yet trusted |
| previous ATM vol | `NaN` on the first cycle |

So the first cycle uses a bootstrap path.

```text
reference data + bid/ask
   |
   v
ATM vol estimate from near-ATM straddle quotes
   |
   v
synthetic forward fallback
   |
   v
quote -> first instrument IV
   |
   v
first fitted smile
```

### Step 1. Bootstrap ATM vol

The code looks at the two strikes around `refPx`:

- nearest strike at or below `refPx`
- nearest strike at or above `refPx`

For each of those two strikes, it forms one call price and one put price using
the configured quote pricer, then computes:

```text
sigma_atm(strike) = (call_px + put_px) / (2 * sqrt(1 / (2pi)) * refPx * sqrtTau)
```

Then it averages the lower and upper estimates:

```text
first_atm_sigma = 0.5 * (sigma_atm_lower + sigma_atm_upper)
```

If this is not available:

| Fallback order | Value |
|---|---|
| 1 | previous ATM vol |
| 2 | config fallback `fallback_atm_vola = 0.15` |

### Step 2. Bootstrap forward

Because `mIsVolaValid_` starts as `false`, `BFPerInstrFutVecProv` does not use
`reverse_b76` on the first pass.

It switches to the configured fallback:

| Config field | Value |
|---|---|
| `impl_fut_quote_prov_type` | `reverse_b76` |
| `fallback_impl_fut_quote_prov_type` | `synthetic_min_sz` |

So the first forward quotes come from put-call parity, not from a supplied
smile sigma.

For each strike:

```text
F_bid = (C_bid - P_ask) / df + K
F_ask = (C_ask - P_bid) / df + K
```

Those bid/ask forward quotes are then reduced to one per-instrument estimate by
the quote pricer.

#### How synthetic `F_bid` and `F_ask` are used

For synthetic forward generation, the code writes the same forward book into
both option slots at the strike:

| Slot at strike `K` | Stored bid | Stored ask |
|---|---|---|
| call slot | `F_bid` | `F_ask` |
| put slot | `F_bid` | `F_ask` |

Then the normal quote-pricer path runs on each slot:

| Quote pricer | Per-slot forward estimate | Per-slot forward error |
|---|---|---|
| `mid_px` | `f_i = (F_bid + F_ask) / 2` | `err_i = (F_ask - F_bid) / 2` |
| `wgt_px` | size-weighted average of `F_bid` and `F_ask` | `(F_ask - F_bid) / 2` |

So the synthetic bid/ask pair is not used directly in IV inversion. It is first
collapsed into one forward estimate `f_i` and one error `err_i`, then those are
used in the expiry-level forward aggregation.

### Step 3. First instrument IV

Once the synthetic-forward path has produced the final `F_i`, the code runs the
instrument IV inversion:

```text
call quote + F_call_i -> call IV
put quote  + F_put_i  -> put IV
```

This is the first point where instrument IV is computed.

### Step 4. First fitted smile

The per-instrument IVs are then:

- status-stamped
- converted into per-option IV/error/weight
- merged into per-strike targets
- optionally memory-smoothed
- spline-fit into the first smile

After that first smile exists and `mIsVolaValid_` becomes true, later cycles
can use `reverse_b76` for forward estimation.

## If A Previous Forward Existed

There is no direct previous-`F` state that is carried forward as the new
forward input.

| Reused on later cycles | Used directly? |
|---|---|
| previous fitted smile sigma | yes |
| previous solved `x = log(F/K)` | yes, as warm-start cache |
| previous final `F_i` | no |
| previous weighted expiry forward `F_exp` | not as the main solver input |

So once the model is warm:

```text
current quote + previous smile sigma + previous x warm-start
   |
   v
reverse_b76
   |
   v
new forward quotes
```

The warm-start cache only helps the numerical solver converge faster. It does
not force the new forward to stay near the old one if the current quotes move.

## When Synthetic Is Used Versus Reverse B76

| Case | Forward method used |
|---|---|
| no valid vol state yet | synthetic fallback |
| valid vol state available | `reverse_b76` |
| later cycle but vol state becomes invalid again | synthetic fallback again |

So it is not "only the initial fit uses synthetic". The switch depends on
whether the volatility state is currently valid.

## What Reverse B76 Calculates

`reverse_b76` does not output IV. It outputs a forward quote.

For one option quote, it solves:

```text
x = log(F / K)
```

using:

- option quote `px`
- strike `K`
- supplied sigma
- `sqrtTau`
- discount factor `df`

Then it returns:

```text
F = K * exp(x)
```

### Call path

Inputs:

```text
V = sigma * sqrtTau
NORMPX = call_px / (K * df)
initial x = previous warm-start x, else log(refPx / K)
```

The solver iterates on `x` until the Black-76 call price implied by `x` and
`V` matches the observed normalised call price.

### Put path

Inputs:

```text
V = - sigma * sqrtTau
NORMPX = - put_px / (K * df)
initial x = previous warm-start x, else log(refPx / K)
```

It again solves for `x = log(F / K)` and returns:

```text
F = K * exp(x)
```

### Output shape

For each strike pair:

| Output | Meaning |
|---|---|
| call bid forward quote | from call bid |
| call ask forward quote | from call ask |
| put bid-side forward quote | from put ask |
| put ask-side forward quote | from put bid |

So the output of `reverse_b76` is a forward bid/ask book, not an implied-vol
book.

## Are Call And Put Using Different `F`?

Yes, at the option-level forward-estimation stage they can be different.

For one strike:

| Quantity | Source |
|---|---|
| call-side forward quote | from call quote |
| put-side forward quote | from put quote |

After quote-pricer collapse, the code has:

| Quantity | Meaning |
|---|---|
| `f_call(K)` | forward estimate from the call slot |
| `f_put(K)` | forward estimate from the put slot |

These are then included separately in the expiry-level weighted average:

```text
F_exp = sum_i(w_i * f_i) / sum_i(w_i)
```

So the averaging happens across all option-level forward estimates in the
expiry, including both call-derived and put-derived forwards.

## Weight Used In The Expiry Forward

For option-level forward estimate `f_i`, the weight is:

```text
w_i = relWt_i x deltaWt_i / err_i^2
```

where:

| Term | Meaning |
|---|---|
| `err_i` | forward quote error after quote-pricer collapse |
| `deltaWt_i` | `1` if greek weighting disabled, else `max(abs(delta_i), 1)` |
| `relWt_i` | configurable strike/x weight |

In the shipped config:

| Config | Effect |
|---|---|
| `weight_gen.type = err` | `relWt_i = 1` |
| `greek_delta = ""` | `deltaWt_i = 1` |

So here:

```text
w_i = 1 / err_i^2
```

## From Instrument Offset To Final Instrument Forward

### Raw instrument offset

The first offset is:

```text
instBo_i = f_i - anchor
```

with:

| Anchor choice | Formula |
|---|---|
| if `calc_wrt_opt_exp_fwd = true` | `anchor = F_exp` |
| if `calc_wrt_opt_exp_fwd = false` | `anchor = refPx` |

In the shipped config:

```text
instBo_i = f_i - refPx
```

### Target / memory merge

The base-offset target step merges the raw offset and memory by inverse
variance:

```text
instWt_i = 1 / instErr_i^2
memWt_i  = useMem / memErr_i^2
tgtBo_i  = (instWt_i x instBo_i + memWt_i x memBo_i) / (instWt_i + memWt_i)
```

### Final residual offset

That target offset is then fit / bounded by the base-offset pipeline, producing
the final residual offset:

```text
boRes_i
```

### Final instrument forward

In `VFFutProv`:

| Mode | Final forward |
|---|---|
| `base` | `F_i = refPx + boRes_i` |
| `smooth` | `F_i = boRes_i + F_exp - optExpBo` |

In the shipped config:

```text
F_i = refPx + boRes_i
```

## `refPx`, Expiry Forward Use, And Call/Put Final Forward

### What is `refPx`?

`refPx` is the current base / reference price passed through the valuation
pipeline. It is the anchor price used by several stages:

- ATM-vol bootstrap
- raw instrument offset when `calc_wrt_opt_exp_fwd = false`
- final instrument forward in `VFFutProv.type = base`

### Is expiry forward `F_exp` used in final instrument `F_i`?

| Config | Is `F_exp` used directly in final `F_i`? |
|---|---|
| `VFFutProv.type = base` | no |
| `VFFutProv.type = smooth` | yes |

In the shipped config:

```text
F_i = refPx + boRes_i
```

so `F_exp` is not used directly in the final instrument forward.

### Do call and put have the same final instrument `F`?

Not necessarily.

| Stage | Call/put relationship |
|---|---|
| synthetic forward quote stage | same forward bid/ask copied into both slots |
| expiry forward averaging | both contribute separately |
| final instrument forward | can differ because `boRes_i` is per instrument |

So in the shipped config:

```text
F_call_i = refPx + boRes_call_i
F_put_i  = refPx + boRes_put_i
```

Call and put at the same strike only match if their final residual offsets
happen to be the same.

## Where `F_exp` Is Used

`F_exp` is used as an expiry-level anchor in several places.

| Component | Use of `F_exp` | Active in shipped config? |
|---|---|---|
| `BFInstBoProv` | can define raw instrument offset relative to `F_exp` | no |
| `VFFutProv` `smooth` mode | can enter final instrument forward directly | no |
| `XProv` | can be the reference for moneyness-style x-coordinates | no |
| `BoResProv` | can be used as the bound anchor instead of `refPx` | no |
| logging / diagnostics | recorded as expiry forward summary | yes |

Inactive here means the shipped config sets:

```text
calc_wrt_opt_exp_fwd = false
VFFutProv.type = base
use_opt_exp_fwd = false
cap_wrt_opt_exp_fwd = false
```

So in the shipped config, `F_exp` is mostly an internal summary / diagnostic
anchor, not a direct input to final instrument `F_i`.

## How `boRes_i` Is Calculated

`boRes_i` is the final residual base offset used to build:

```text
F_i = refPx + boRes_i
```

It is not just the raw difference `F - refPx`.

### Step 1. Raw instrument base offset

In the shipped config:

```text
instBo_i = f_i - refPx
```

where `f_i` is the option-level forward estimate before expiry aggregation.

### Step 2. Merge with memory

The raw offset is merged with memory by inverse variance:

```text
instWt_i = 1 / instErr_i^2
memWt_i  = useMem / memErr_i^2
tgtBo_i  = (instWt_i x instBo_i + memWt_i x memBo_i) / (instWt_i + memWt_i)
```

This is Kalman-like.

| Statement | True? |
|---|---|
| `tgtBo_i` is a precision-weighted blend of current offset and prior offset | yes |
| that algebra matches a Kalman-style measurement update | yes, in that narrow sense |
| the full `boRes_i` pipeline is just a Kalman filter | no |

### Step 3. Memory update

The memory layer then stores the target offset.

| Memory mode | Behaviour |
|---|---|
| `default` | pass-through |
| `kalman` | Kalman smoothing |

In the shipped config:

```text
BoMemProv.type = default
```

so this stage is not acting as a Kalman filter here.

### Step 4. Fit offset curves

The memory offsets are spline-fit separately for calls and puts:

```text
fitBo_i = spline_fit(memBo)
```

### Step 5. Bound the fitted offset

`BoResProv` then clips the fitted offset into option-theoretic bounds derived
from:

- strike
- fitted vol
- base price
- discount factor
- time to expiry
- deep OTM probability
- configured long/short bounds

Conceptually:

```text
boRes_i = clip(fitBo_i, lowerBound_i, upperBound_i)
```

## Is Base Offset Essentially A Kalman Filter On `F - underlying`?

No.

| Statement | Correct? |
|---|---|
| raw starting point is roughly `F - refPx` | yes |
| final `boRes_i` is just that raw difference | no |
| shipped config uses Kalman smoothing for base offset | no |

The better description is:

```text
raw offset
   -> target merge with memory
   -> memory store
   -> call/put spline fit
   -> bounded residual offset boRes_i
```

## `tgtBo` Versus `boRes`

| Quantity | What it is |
|---|---|
| `tgtBo_i` | local target offset after blending raw instrument offset with memory |
| `boRes_i` | final residual offset after memory, spline fit, and bounds |

More explicitly:

```text
instBo_i
   + memBo_i
   -> tgtBo_i
   -> memBo_i(next)
   -> fitBo_i
   -> boRes_i
```

So:

| Comparison | Result |
|---|---|
| `tgtBo_i` vs `boRes_i` | not the same object |
| can `boRes_i` equal `tgtBo_i` sometimes? | yes, if fit/bounds do little |
| is `boRes_i` generally smoother / constrained? | yes |

Bid/ask implied vol uses the final instrument forward built from `boRes_i`, not
the intermediate `tgtBo_i`.

```text
tgtBo_i
   -> memory / fit / bounds
   -> boRes_i
   -> F_i
   -> bid/ask IV
```

### Exact `tgtBo_i -> boRes_i` path

In the shipped config, the conversion is:

```text
tgtBo_i
   -> memBo_i
   -> fitBo_i
   -> boRes_i
```

| Step | Formula / action |
|---|---|
| memory | `memBo_i = tgtBo_i` because `BoMemProv.type = default` |
| fit | spline-fit calls and puts separately: `fitBo_i = spline(memBo)` |
| bound | `boRes_i = min(max(fitBo_i, lowerBound_i), upperBound_i)` |

The spline fit uses:

- `X =` configured strike/x coordinate
- `Y = memBo`
- weights from `BoTgtProv`

### Base-offset spline input and output

This spline stage is `BFFitBoProv`.

| Input | Shape | Meaning |
|---|---|---|
| `XVector` | `n_strike` | x-coordinate per strike |
| `YVector` | `n_inst` | memory offset `memBo_i` |
| `aErrVector` | `n_inst` | error of `memBo_i` |
| `aWgtVector` | `n_inst` | target weights from `BoTgtProv` |

In the shipped call:

```text
XVector   = strike-x provider output
YVector   = mBoMemProv_->getBoVector()
aErrVector = mBoMemProv_->getBoErrVector()
aWgtVector = mBoTgtProv_->mOutputWeightVector_
```

| Output | Shape | Meaning |
|---|---|---|
| `mOutputVector_` | `n_inst` | fitted offset `fitBo_i` |
| `mOutputErrorVector_` | `n_inst` | fit error per instrument |
| `mRecordCall_` | spline record | call-side spline coefficients |
| `mRecordPut_` | spline record | put-side spline coefficients |

The bounds are generated in `BoResProv` from:

- strike
- fitted vol
- base price
- discount factor
- time to expiry
- deep OTM probability
- configured long/short bound parameters

So `tgtBo_i` is not transformed into `boRes_i` by one formula. It is:

```text
copy / smooth
   -> cross-strike fit
   -> per-instrument clipping
```

### How `memBo` becomes `fitBo`

`BFFitBoProv` takes the memory offsets as the `Y` values and runs two spline
fits:

| Fit | Indices used |
|---|---|
| call spline | `0, 2, 4, ...` |
| put spline | `1, 3, 5, ...` |

The mechanics are:

| Step | Action |
|---|---|
| 1 | normalise `X` by `norm_factor_x` |
| 2 | normalise `Y = memBo` by `norm_factor_y` |
| 3 | run smoothing spline separately for call and put slices |
| 4 | de-normalise the fitted output |
| 5 | compute fit error versus input `memBo` |

So conceptually:

```text
fitBo_call = spline_call(X, memBo_call, weights)
fitBo_put  = spline_put(X, memBo_put,  weights)
```

In the shipped config, `smoothing_weight_trivial = true`, so the spline fit
uses uniform weights for this stage.

### Spline model

The model is a **natural cubic smoothing spline**, fit separately for calls and
puts.

For each segment starting at knot `x0`, the fitted curve is:

```text
f(x) = d0 + dx x ( d1 + dx x (0.5 x d2 + dx x d3 / 6) )
dx   = x - x0
```

So each segment is a cubic polynomial.

Natural boundary condition:

```text
f''(left end)  = 0
f''(right end) = 0
```

The smoothing strength is controlled by:

```text
lambda = smoothing_factor
```

### How the knots are defined

The knots are not fixed externally. They are the `X` values of the valid input
points for that side.

For the call spline:

```text
knots = { X[k] : memBo_call(k) is finite and weight is finite }
```

For the put spline:

```text
knots = { X[k] : memBo_put(k) is finite and weight is finite }
```

So:

| Spline | Knot set |
|---|---|
| call spline | valid call-side strike/x points |
| put spline | valid put-side strike/x points |

If fewer than 4 valid points exist, the code does not build the spline and just
passes the input values through.

### Detailed fitted form

Let the knots be:

```text
x_0 < x_1 < ... < x_n
```

On interval `[x_j, x_{j+1}]`, the spline is:

```text
f_j(x) = d0_j + (x - x_j) x d1_j
       + 0.5 x (x - x_j)^2 x d2_j
       + (1/6) x (x - x_j)^3 x d3_j
```

with:

| Coefficient | Meaning in code |
|---|---|
| `d0_j` | fitted value at knot `x_j` |
| `d1_j` | first derivative at knot `x_j` |
| `d2_j` | second derivative at knot `x_j` |
| `d3_j` | third-derivative segment coefficient |

The code sets:

```text
d2_0 = 0
d2_n = 0
```

which is the natural-spline boundary condition.

### What the bound is

The bound is a permitted range for the final residual offset `boRes_i`.

It is built from a reduced option premium:

```text
theoNew = max(
  minDiscOptPrem,
  theo - theo x optPremDiscRate - expMarginFactor x refPx x T2E
)
```

Then the code asks:

```text
What forward shift would make Black-76 price equal to theoNew?
```

For calls:

```text
mFutPxNew = forward implied by call price theoNew
BOBound   = mFutPxNew - refPx
```

For puts:

```text
mFutPxNew = forward implied by put price theoNew
BOBound   = mFutPxNew - refPx
```

That gives an intermediate symmetric-style offset range around `refPx`, which
is then clipped again by configured long/short hard caps.

So the final bound is:

```text
lowerBound_i <= boRes_i <= upperBound_i
```

where the interval comes from:

- Black-76 option value at fitted vol
- premium discount / margin haircut
- inversion back to forward
- hard long/short offset caps

### Why it is not a Kalman filter in the shipped config

Because the code takes the `default` branch, not the `kalman` branch.

| Config | Value |
|---|---|
| `BoMemProv.type` | `default` |

In `BoMemProv::update(...)`:

| If type | What happens |
|---|---|
| `default` | `mResBoVector_ = aBoVector` and `mResBoErrVector_ = aErrVector` |
| `kalman` | `mKalmanFilter_->update(...)` is called |

So in this snapshot, the stored base offset is passed through from the target
offset and then spline-fit and bounded. The Kalman code exists, but it is not
the active path.

## 1. Per-Instrument Forward Used Before IV Inversion

### Raw forward quote per option

Source:

- `ImplFutQuoteProv.cpp`
- `BFPerInstrFutVecProv.cpp`

Two modes exist.

### `reverse_b76`

For a call or put quote, the code solves for:

```text
x = log(F / K)
```

then sets:

```text
F = K * exp(x)
```

This uses the current smile vol as input to the inverse map.

#### What it does for one strike pair

For strike `K_m`, the code starts with the current strike-level smile vol
`sigma_m = aInputVolaVector[m]` and reference forward anchor `refPx`.

It then creates four raw forward-side quotes:

| Option quote | Solver call | Raw forward output |
|---|---|---|
| call bid | `_SOR_IM_CALL(refPx, K, sigma_m, C_bid, sqrtTau, df)` | `F_bid_call = K * exp(x_call_bid)` |
| call ask | `_SOR_IM_CALL(refPx, K, sigma_m, C_ask, sqrtTau, df)` | `F_ask_call = K * exp(x_call_ask)` |
| put bid-side forward | `_SOR_IM_PUT(refPx, K, sigma_m, P_ask, sqrtTau, df)` | `F_bid_put = K * exp(x_put_from_ask)` |
| put ask-side forward | `_SOR_IM_PUT(refPx, K, sigma_m, P_bid, sqrtTau, df)` | `F_ask_put = K * exp(x_put_from_bid)` |

The put inputs are intentionally flipped on bid and ask.

| Put-side flip | Reason |
|---|---|
| bid forward uses put ask | higher put price implies lower forward |
| ask forward uses put bid | keeps the derived forward bid/ask orientation consistent |

So `reverse_b76` does not combine call and put into one forward immediately.
It first creates one forward quote stream from the call and another from the
put.

#### Exact quantity being solved

`_SOR_IM_CALL` and `_SOR_IM_PUT` do not solve for IV. They solve for:

```text
x = log(F / K)
```

with `sigma` supplied as an input.

| Solver | Normalised price used by the solver |
|---|---|
| `_SOR_IM_CALL` | `NORMPX = px / (K * df)` |
| `_SOR_IM_PUT` | `NORMPX = - px / (K * df)` |

Both use:

```text
V = sigma * sqrtTau
initial guess x_0 = log(refPx / K)   if no warm start exists
```

and iterate on `x` until the Black-76 normalised price matches the observed
option quote.

#### Where that sigma comes from

The forward-estimation step is called before the current-cycle IV inversion.
It takes sigma from the fitter state already held by the `Valuator`:

| Config flag | Sigma vector used by `BFPerInstrFutVecProv` |
|---|---|
| `use_fit_prov_for_input = true` | `mVolaFitter_->mFitVolaVector_` |
| `use_fit_prov_for_input = false` | `mVolaFitter_->mOutputVolaVector_` |

In the shipped valuation config, `use_fit_prov_for_input` is `true`.

### `synthetic`

Using put-call parity:

| Quantity | Formula |
|---|---|
| synthetic bid forward | `F_bid = (C_bid - P_ask) / df + K` |
| synthetic ask forward | `F_ask = (C_ask - P_bid) / df + K` |

## 2. Bid/Ask Forward Quotes -> One Forward Estimate Per Instrument

Source: `QuotePricer.cpp`

| Quote pricer | Forward estimate | Raw error |
|---|---|---|
| `mid_px` | `(bid + ask) / 2` | `(ask - bid) / 2` |
| `wgt_px` | `(bid * askSz + ask * bidSz) / (bidSz + askSz)` | `(ask - bid) / 2` |
| strict variants | same formula | return `NaN, Inf` if any side or size is invalid |

The effective error is floored:

```text
err_eff = max(raw_err, minBookError)
```

For one strike pair, that gives:

| Instrument slot | Forward estimate after quote pricer |
|---|---|
| call instrument at `K` | `f_call(K)` |
| put instrument at `K` | `f_put(K)` |

These are still separate. The code does not collapse them into one
strike-level forward at this stage.

So `f_i` is instrument-specific.

| Quantity | Meaning |
|---|---|
| `f_i` | one forward estimate for option instrument `i` |
| strike-specific only? | no |
| instrument-specific? | yes |

It is computed by first building a forward bid/ask quote for that instrument,
then collapsing that bid/ask pair with the configured quote pricer.

With `mid_px`:

```text
f_i   = (F_bid_i + F_ask_i) / 2
err_i = (F_ask_i - F_bid_i) / 2
```

For `reverse_b76`:

```text
F_bid_i = K * exp(x_bid_i)
F_ask_i = K * exp(x_ask_i)
```

For synthetic fallback:

```text
F_bid = (C_bid - P_ask) / df + K
F_ask = (C_ask - P_bid) / df + K
```

and that same synthetic forward bid/ask is copied into both call and put slots
at the strike before quote-pricer collapse.

## Is The Offset Pipeline Involved In Quote -> IV?

Yes.

```text
quote
   -> raw forward estimate f_i
   -> offset pipeline
   -> final F_i
   -> bid/ask IV
```

The distinction is:

| Quantity | Role |
|---|---|
| `f_i` | raw forward estimate from quote-derived forward bid/ask |
| `boRes_i` | final residual offset after the offset pipeline |
| `F_i` | final forward used in IV inversion |

In the shipped config:

```text
F_i = refPx + boRes_i
```

So the offset pipeline is between raw forward estimation and final IV
inversion.

## 3. Forward Weighting Across Strikes

Source:

- `BFPerInstrFutVecProv.cpp`
- `BFWtProv.cpp`

### Base weight

For instrument `i`:

```text
baseWt_i = deltaWt_i / err_eff_i^2
```

where:

```text
deltaWt_i = 1                    if greek weighting disabled
deltaWt_i = max(abs(delta_i), 1) otherwise
```

### Relative strike / x weight

The final weight multiplies the inverse-error weight by a configurable relative weight:

```text
w_i = baseWt_i * relWt_i
```

#### Strike-selection multiplier

Every weighting scheme includes:

```text
selectMultiplier =
  selectStrikeWeightMult   if strike % selectStrikeGap == 0
  1                        otherwise
```

#### Supported relative weights

Let `phi(z)` be the normal PDF.

| Type | Call relative weight | Put relative weight |
|---|---|---|
| `err` | `selectMultiplier` | `selectMultiplier` |
| `strike_norm` | `phi((x - refPx) / std_call) * selectMultiplier` | `phi((x - refPx) / std_put) * selectMultiplier` |
| `opt_norm` | `phi((x - mu_call) / std_call) * selectMultiplier` | `phi((x - mu_put) / std_put) * selectMultiplier` |
| `opt_norm_max` | `max(phi_call, phi_put) * selectMultiplier` | same as call |
| `opt_lognorm` | `phi((log(x) - mu_call) / std_call) * selectMultiplier` | `phi((log(x) - mu_put) / std_put) * selectMultiplier` |
| `opt_lognorm_max` | `max(phi_call, phi_put) * selectMultiplier` | same as call |

### Expiry-level forward

The expiry forward is the weighted average:

```text
F_exp = sum_i(w_i * f_i) / sum_i(w_i)
```

This average runs over all option instruments in the expiry:

```text
[call(K0), put(K0), call(K1), put(K1), ...]
```

So the weight is attached to each option-level forward estimate, not to one
pre-merged strike-level forward.

### How the weighted expiry forward is used

The weighted expiry forward is not passed directly into the IV solver as the
final answer. It is an intermediate anchor.

```text
option-level forward estimates f_i
   |
   v
weighted average
   |
   v
F_exp
   |
   v
combine with base-offset residual
   |
   v
final instrument forward F_i
   |
   v
quote + F_i -> instrument IV
```

| Stage | Use of `F_exp` |
|---|---|
| `BFOptExpFutProv` | holds the expiry-level forward summary |
| `BFInstBoProv` | compares per-instrument forwards against the expiry forward or `refPx` to form base offsets |
| `VFFutProv` in `smooth` mode | builds final per-instrument forward as `F_i = residual_i + F_exp - optExpBo` |
| `XProv` | can use expiry forward as the reference for moneyness-style x-coordinates when configured |

In the shipped valuation config:

| Config | Effect |
|---|---|
| `VFFutProv.type = base` | final `F_i` does not use `F_exp` directly |
| `BFInstBoProv.calc_wrt_opt_exp_fwd = false` | base offsets are measured against `refPx`, not `F_exp` |

So in this config, the weighted expiry forward mainly acts as an internal
diagnostic and anchor for the broader base-offset pipeline, while the final IV
inversion uses:

```text
F_i = refPx + boRes_i
```

## 4. Final Forward Passed Into IV Solver

Source: `VFFutProv.cpp`

The final forward used to invert each option quote is not just the expiry average.

| `VFFutProv` mode | Final forward |
|---|---|
| `base` | `F_i = refPx + boRes_i` |
| `smooth` | `F_i = F_exp + (boRes_i - optExpBo)` |

So the final `F_i` is:

```text
expiry forward level + instrument-specific residual correction
```

## 4A. Is `F` Strike-Specific?

Not exactly.

| Level | What the code has |
|---|---|
| after `reverse_b76` | option-level forward quotes |
| after `QuotePricer` | option-level forward estimates `f_i` |
| after weighting | one expiry-level forward `F_exp` |
| final IV inversion input | instrument-specific `F_i` |

So the final forward passed into `_SOR_IV_CALL` and `_SOR_IV_PUT` is not a
single shared `F(K)` per strike pair. It is `F_i` per option instrument.

For call and put at the same strike:

| Quantity | Code shape |
|---|---|
| call forward used for IV | `tFc = aInputFutureVector[i]` |
| put forward used for IV | `tFp = aInputFutureVector[j]` |

That means the code allows the call and put at the same strike to use different
final forwards, because `aInputFutureVector` is indexed by instrument, not by
strike.

## 5. Black-76 Inversion: Quote -> Implied Vol

Source:

- `VFImplVolQuoteProv.cpp`
- `formula.hpp`

For a call:

```text
d1 = ln(F / K) / (sigma * sqrtTau) + 0.5 * sigma * sqrtTau
d2 = ln(F / K) / (sigma * sqrtTau) - 0.5 * sigma * sqrtTau

Call = df * (F * N(d1) - K * N(d2))
```

For a put:

```text
Put = df * (K * N(-d2) - F * N(-d1))
```

The code inverts these formulas using a fixed-point SOR-style solver.

### Normalised call inversion

The solver works in:

```text
x = log(F / K)
v = sigma * sqrtTau
```

For `_SOR_IV_CALL`, the update is built around:

```text
d1(x,v) = x / v + v / 2
d2(x,v) = x / v - v / 2
```

and a normalised price.

The returned implied vol is:

```text
iv = v / sqrtTau
```

## 6. Status Stamping: Direct / Interpolated / Extrapolated

Source: `VFVolaPolator.cpp`

Missing IV quotes are filled only on the configured OTM side of the smile.

| Status | Meaning |
|---|---|
| `0` | direct market quote |
| `1` | interpolated |
| `2` | extrapolated |
| `<0` | unusable |

Interpolation:

```text
slope = interpSlopeMult * (y1 - y0) / (x1 - x0)
y(x)  = max(y0 + (x - x0) * slope, minVola)
```

Extrapolation:

```text
slope = clamp(
  extrapSlopeMult * (y1 - y0) / (x1 - x0),
  minSlope,
  maxSlope
)

y(x) = max(y0 + (x - x0) * slope, minVola)
```

## 7. Per-Option IV Observation, Error, and Weight

Source: `VolaSrcProv.cpp`

First, bid/ask implied vols are collapsed to one value and one raw error using the same quote-pricer choices as above.

Then status-based error inflation is applied.

### Status multiplier

```text
mult(0) = 1
mult(1) = interpErrMult
mult(2) = extrapErrMult
```

Combined multiplier:

```text
tMult = mult(bidStatus) * mult(askStatus)
```

### Minimum error floor

```text
tMinErr =
  minErr       if both sides are direct
  minNoMktErr  if both sides are interpolated/extrapolated
  minLerpErr   if exactly one side is interpolated/extrapolated
  Inf          if either side is unusable
```

If a synthetic or interpolated quote crosses:

```text
if bidPx >= askPx:
  tMinErr = max(tMinErr, minCrossErr)
  tMult   = tMult * crossErrMult
```

### Final per-option error and weight

```text
err_i = max(tMinErr, tMult * rawErr_i) + max(0, tickSzFactor / vega_i)
w_i   = 1 / err_i^2
```

## 8. Per-Strike Merge: Call + Put -> One Smile Point

Source: `VolaTgtProv.cpp`

This is the main collapse from per-option data to per-strike smile data.

For strike pair `m`:

1. decide whether the call side or put side is ITM from `atm_x` and `x[m]`
2. label one source point as ITM and the other as OTM
3. apply side-specific relative weights
4. compute an ITM correction factor from `ItmFactorGen`
5. blend ITM, OTM, and memory into one strike-level target

### Raw weighted pieces

```text
itmWt = srcWt_itm * relWt_itm
otmWt = srcWt_otm * relWt_otm
memWt = 1 / memErr^2
```

If memory is disabled or missing:

```text
memWt = 0
memPx = fallbackVol
```

### Strike-level target

Let `itmFactor` come from `ItmFactorGen`.

Then:

```text
tgtWt_m = itmFactor * itmWt + otmWt + memWt
tgtPx_m = (memWt * memPx + itmFactor * itmWt * itmPx + otmWt * otmPx) / tgtWt_m
tgtErr_m = 1 / sqrt(tgtWt_m)
```

If `persist_strike_weight` is enabled, the stored weight becomes:

```text
tgtWt_m = itmWt + otmWt + memWt
```

After the minimum-vol strike is found, a window around that minimum gets extra weight:

```text
tgtWt_m <- tgtWt_m * weightMult
```

for strikes within the configured radius around the minimum target vol.

## 9. X-Axis Used For Spline Fit

Source: `XProv.hpp`

| X type | Formula |
|---|---|
| `strike` | `x = K` |
| `moneyness` | `x = ref / K` |
| `logmoneyness` | `x = log(ref / K)` |
| `expmoneyness` | `x = (ref / K) * powerTau / sqrtTauEff` |
| `explogmoneyness` | `x = log(ref / K) * powerTau / sqrtTauEff` |
| `stdmoneyness` | `x = log(ref / K) * powerTau / (sqrtTauEff * sigma)` |

## 10. Memory Smoothing

Source:

- `VolaMemProv.cpp`
- `KalmanFilter.hpp`

If memory type is `default`:

```text
memPx  = tgtPx
memErr = tgtErr
```

If memory type is `kalman`:

```text
variance = oldErr^2 + dev^2 * processVarMult
gain     = 1 / (1 + measErr^2 / variance)

newPx  = oldPx + gain * (measPx - oldPx)
newErr = sqrt((1 - gain) * variance)
```

## 11. Spline Fit

Source:

- `VolaFitProv.cpp`
- `Spline.cpp`

### Weight normalisation

```text
sumW   = sum_i w_i
wMax   = max_i w_i
wFloor = wMax / (sumW * wtRatioCeil)

w_i <- max(wFloor, w_i / sumW)
```

Optional vega scaling:

```text
avgVega_m = 0.5 * (vega_call_m + vega_put_m)
w_m <- w_m * avgVega_m ^ bVegaExponent
```

### Cubic representation on each interval

If the left knot is `x0` and `dx = x - x0`:

```text
f(x) = d0 + dx * (d1 + dx * (0.5 * d2 + dx * d3 / 6))
```

### Fit error

```text
fitErr = sqrt(sum_i w_i * (y_i - f(x_i))^2)
```

### ATM vol

The fitted ATM vol is simply the spline evaluated at `atm_x`.
