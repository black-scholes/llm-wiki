# Spot-vol dynamics

## Intuition

When the underlying moves, the implied-volatility smile moves too. The common
shortcuts “sticky strike” and “sticky delta” are useful labels, but neither is a
universal law for equity options. The movement of the ATM level and the movement
of the smile slope need to be modelled together.

## SSR

Vola Dynamics describes a dimensionless skew-sensitivity ratio, commonly written
as

```text
SSR = (d sigma0 / d F) / (d sigma / d K)
```

where `sigma0` is ATM volatility and the denominator is the local smile slope.
Under this convention, sticky-delta corresponds approximately to `SSR = 0` and
sticky-strike to `SSR = 1`. Published material reports equity behaviour often
between those extremes, with the level depending on the market and horizon.

The ratio is a diagnostic, not a constant of nature. Intraday and close-to-close
measurements can differ, and the relationship may change in stressed markets or
the wings.

## Smile movement

The useful modelling question is not “which stickiness rule is correct?” but:

1. how the ATM level responds to the forward;
2. how the normalised shape responds to that move; and
3. how both responses affect the risk sensitivities used for PnL.

The Vola material describes equity shapes as more stable in normalised-strike
space than in raw strike or delta space. This is why spot-vol dynamics affects
adjusted delta and gamma: a conventional Greek that freezes the smile can omit a
material part of the mark-to-market move.

## Events

An earnings or macro event can create a jump component, a temporary variance
spike, negative central curvature, or a W-shaped smile. Treating that shape as a
normal smooth smile can misstate both the surface and the calendar term
structure.

Two distinct ideas should not be conflated:

- a jump model decomposes a “dirty” event surface into background volatility and
  jump scenarios;
- an event-variance model changes how variance accrues through event time while
  preserving the appropriate total variance.

Both require explicit event timing and should be evaluated separately from
ordinary diffusion-driven smile movement.

## Practical implications

- Record the horizon and forward convention when estimating smile dynamics.
- Do not use sticky-strike or sticky-delta as an unqualified production truth.
- Recompute risk sensitivities consistently with the chosen surface dynamics.
- Keep event variance and jump effects explicit rather than hiding them in a
  generic curvature parameter.

## Limits and evidence

SSR is useful for communicating and measuring spot-vol behaviour, but published
examples do not establish one universal value across assets, horizons, or
regimes. The event descriptions are methodology notes; a trading system still
needs its own data, calibration, and out-of-sample validation.

## Sources

- [Klassen, State of the Smile](https://voladynamics.com/pdf/Klassen_StateOfTheSmile_QMI_20231115_final.pdf)
- [Klassen, Spot-vol Dynamics and Deltas for SPX Options](https://voladynamics.com/pdf/Vola_SpotVolDynamicsDeltasSPX_LI_20200107.pdf)
- [Vola Dynamics media index](https://voladynamics.com/media)
