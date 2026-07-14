# No-arbitrage and fit quality

## Intuition

“Fits the quotes well” and “is economically usable” are different claims. A
surface can have a small residual while implying negative density, decreasing
total variance across expiry, or implausible wings. Surface review needs both
fit metrics and arbitrage diagnostics.

## Butterfly and calendar checks

For a total-variance curve `w(y)`, the Durrleman density condition can be written
in the form

```text
g(y) = (1 - y w'/(2w))^2
       - (1/4)(1/w + 1/4)(w')^2
       + (1/2)w'' >= 0
```

The exact notation changes with the chosen coordinate and scaling, but the
meaning is stable: the risk-neutral density must not become negative.

Across expiry, calendar sanity is commonly expressed as non-decreasing total
variance at fixed log-moneyness. This is not the same as requiring volatility
itself to increase with expiry.

The Lee wing bound constrains asymptotic total variance growth. It is necessary
for sensible wings but not sufficient to guarantee a fully arbitrage-free
surface.

## How fitting and no-arbitrage interact

The public Vola material supports a layered view:

1. regular fitting uses quote-weighted residuals and soft robustness or
   regularisation penalties;
2. curve families can make some arbitrage violations unlikely or analytically
   tractable, but flexible curves are not automatically perfect;
3. a no-arbitrage mode can impose stronger butterfly and calendar constraints,
   accepting some fit-quality cost.

This is more precise than saying every surface is “arbitrage-free by
construction”. The enforcement mechanism depends on the curve family and the
selected mode.

## Fit metrics

- `chi` or reduced chi-square compares residuals with quote error bars. It is
  only meaningful when the error-bar convention is credible.
- `avE5` is an ATM-focused mean absolute implied-volatility error over five
  strikes, usually reported in basis points. It is not interchangeable with a
  full-smile RMSE.
- `aK` and `aT` are dimensionless butterfly and calendar diagnostics in Vola's
  terminology. Public material describes their purpose, but the exact
  production formulae are not fully public.

When comparing implementations, name the region, weighting, units, and
aggregation explicitly. A single headline score hides sparse data, short-DTE
pathology, wings, and event regimes.

## Practical review checklist

- Check quote and IV error bars before interpreting `chi`.
- Check density and calendar conditions on a sufficiently wide grid.
- Check wings beyond the observed strikes, not only the interpolation region.
- Separate regular and no-arbitrage modes in reports.
- Test representative calm, stressed, short-dated, sparse, and event surfaces.
- Report where a metric is a local proxy rather than a source-equivalent
  implementation.

## Sources

- [Klassen, Arbitrage-Free Parametric Volatility Surfaces and Real-Time Fitting](https://voladynamics.com/pdf/Klassen_GD_Chicago_2017.pdf)
- [Klassen, State of the Smile](https://voladynamics.com/pdf/Klassen_StateOfTheSmile_QMI_20231115_final.pdf)
- [Gatheral and Jacquier, Arbitrage-free SVI volatility surfaces](https://ssrn.com/abstract=2033323)
