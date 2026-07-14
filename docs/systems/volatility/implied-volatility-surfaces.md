# Implied-volatility surfaces

## Intuition

An implied-volatility surface is a compact description of option prices across
strike and expiry. The useful object is not a single volatility number but a
stable shape that can be evaluated, differentiated, and compared across time.

## Mechanics

For an expiry with forward `F`, ATM volatility `sigma0`, and volatility time
`T`, a common dimensionless coordinate is

```text
z = log(K / F) / (sigma0 * sqrt(T))
```

The coordinate removes much of the scale variation caused by the forward, time
to expiry, and ATM level. A shape model can then be written as

```text
sigma(z)^2 = sigma0^2 f(z)
f(z) = 1 + s2 z + 1/2 c2 z^2 + ...
```

Here `s2` describes skew and `c2` describes central curvature. Negative central
curvature can be important for event-driven or multi-modal smiles; a simple
single-bowl model cannot represent every observed surface.

Common model families form a complexity ladder:

- S3/SSVI provides a compact, publicly specified three-parameter shape with
  useful analytical arbitrage conditions.
- S5/SVI adds independent wing flexibility but remains limited for some
  event-driven W-shapes.
- More flexible C-family models add asymmetric put and call wing parameters.
  Public sources describe their parameter structure and behaviour, but not all
  proprietary closed-form equations.

## Term structure and stability

Fitting each expiry independently can produce unstable parameters. Practical
surface construction therefore transfers information across expiries, often by
using error-bar or observation-count weights and a smoothness prior. ATM total
variance should be monitored across expiry; smoothing volatility levels directly
can impose the wrong economic constraint.

Error bars matter twice: they determine how much each quote should influence the
fit, and they determine how much a sparse expiry should borrow from neighbouring
terms. A visually attractive fit with collapsed or untrusted error bars is not
evidence of a good surface.

## Practical implications

- Use forwards and volatility time consistently; mixing spot, forward, or
  calendar-time conventions changes the geometry.
- Treat the curve family as a model-selection decision, not a universal winner.
- Evaluate short-dated, sparse, one-sided, event, and wing regimes separately.
- Keep fit quality, arbitrage diagnostics, and temporal stability as distinct
  dimensions.

## Limits and evidence

The normalised-strike and S3/SSVI ideas are public and source-backed. The exact
equations of proprietary flexible C-curves and the exact production definitions
of some Vola metrics are not public. Claims about a local implementation must
be checked against that implementation rather than presented as properties of
the general method.

## Sources

- [Klassen, Arbitrage-Free Parametric Volatility Surfaces and Real-Time Fitting](https://voladynamics.com/pdf/Klassen_GD_Chicago_2017.pdf)
- [Klassen, State of the Smile](https://voladynamics.com/pdf/Klassen_StateOfTheSmile_QMI_20231115_final.pdf)
- [Gatheral and Jacquier, Arbitrage-free SVI volatility surfaces](https://ssrn.com/abstract=2033323)
- [Hendriks and Martini, Extended SSVI volatility surface](https://doi.org/10.1080/14697688.2019.1571681)
