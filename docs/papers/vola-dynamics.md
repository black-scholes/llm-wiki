# Vola Dynamics source notes

These notes catalogue the public material used for the volatility-system pages.
They are a provenance record, not an endorsement of every product claim.

## Primary sources

| Source | What it supports |
| --- | --- |
| [Global Derivatives Chicago 2017](https://voladynamics.com/pdf/Klassen_GD_Chicago_2017.pdf) | Normalised strike, parametric curves, fitting, error bars, wings, and arbitrage modes |
| [State of the Smile, QuantMinds 2023](https://voladynamics.com/pdf/Klassen_StateOfTheSmile_QMI_20231115_final.pdf) | Curve families, fit metrics, term structure, SSR, and event/smile observations |
| [Real-Time Implied Volatility Surfaces, CBOE 2025](https://voladynamics.com/pdf/Klassen_IVS-BlackArts_CBOE_20251002_distrib.pdf) | Forward, discount, event, dividend, and risk-management context |
| [Secrets of the Implied Volatility Surface, Bloomberg 2025](https://voladynamics.com/pdf/Klassen_SecretsVolSurface_BBQ_20250224.pdf) | Forward, borrow, dividend, and risk-neutral-density workflow |
| [Spot-vol Dynamics and Deltas for SPX Options](https://voladynamics.com/pdf/Vola_SpotVolDynamicsDeltasSPX_LI_20200107.pdf) | Spot-vol dynamics and adjusted risk sensitivities |
| [Vola Dynamics media index](https://voladynamics.com/media) | Public talks, videos, and supporting material |

## Evidence boundaries

- The public material specifies S3/SSVI more fully than proprietary flexible
  C-family equations.
- Exact production definitions for some metrics, including `aK` and `aT`, are
  not fully published. Local implementations must label proxies as proxies.
- YouTube pages were usable for titles and author descriptions in the source
  review, but not as independently verified spoken-word transcripts.
- Published methodology claims are hypotheses or starting points for empirical
  work, not universal constants across assets and horizons.

## Related notes

- [Implied-volatility surfaces](../systems/volatility/implied-volatility-surfaces.md)
- [Spot-vol dynamics](../systems/volatility/spot-vol-dynamics.md)
- [No-arbitrage and fit quality](../systems/volatility/no-arbitrage-and-fit-quality.md)
