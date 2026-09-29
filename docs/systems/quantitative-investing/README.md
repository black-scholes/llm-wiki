# Quantitative investing

A quantitative investment process turns imperfect forecasts of returns, risk
and trading costs into positions, then uses realised performance to improve
those forecasts. These notes explain that process using Paleologo's
*The Elements of Quantitative Investing*, with complementary theory and
implementation guidance.

The [source record](../../papers/elements-of-quantitative-investing.md) maps all
chapters of the saved 2024-07-10 draft and distinguishes it from the 2025 book.
The [complete draft PDF](../../../reference/books/elements-of-quantitative-investing/eqi.pdf)
is retained in the reference directory. All unqualified chapter citations below
refer to that draft.

## Priority reading

1. [Factor models](factor-models.md): the common language of exposures, risk and alpha.
2. [Statistical factor models](statistical-factor-models.md): PCA, weighted estimation, residual risk and changing universes.
3. [Shrinkage](shrinkage.md): covariance, eigenvalue, specific-risk and hedge estimation.
4. [Multiple-market models](multi-market-models.md): global versus regional models, linkage and asynchronous trading.
5. [Currency rebasing](currency-rebasing.md): local, investor-currency and funding-adjusted returns.
6. [Risk-model validation](risk-model-validation.md): whether the resulting model improves actual decisions.

## The rest of the process

| Stage | Pages |
| --- | --- |
| Understand the opportunity | [Investment process](investment-process.md); [Returns](returns.md); [Performance and capacity](performance-and-capacity.md) |
| Estimate and evaluate | [Volatility](volatility-estimation.md); [Fundamental factor models](fundamental-factor-models.md); [Many-characteristic models](many-predictor-factor-models.md); [Backtesting and multiple testing](backtesting.md) |
| Construct and trade | [Portfolio optimisation](portfolio-optimisation.md); [Constraints and estimation error](constraints-and-estimation-error.md); [Signal aggregation](signal-aggregation.md); [Market impact](market-impact.md); [Hedging](hedging.md) |
| Allocate and learn | [Dynamic risk allocation](dynamic-risk-allocation.md); [Performance attribution](performance-attribution.md) |
| Supporting mathematics | [Linear algebra](linear-algebra.md); [Optimisation mechanics](optimisation-mechanics.md) |

Complementary guidance is labelled on each page. Examples illustrate mechanics;
they do not report backtest performance. Related research is available in the
[Gappy paper collection](../../papers/gappy/README.md), especially its
[return properties](../../papers/gappy/returnproperties/) and
[portfolio construction](../../papers/gappy/portfolioconstruction/) groups.
