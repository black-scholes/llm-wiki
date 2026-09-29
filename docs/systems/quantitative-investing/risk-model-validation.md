# Validating a risk model

## Intuition

A risk model is useful when it improves the decisions made with it. Good
asset-volatility forecasts, stable factor portfolios and accurate optimisation
can favour different models. Draft chapter 5 groups evaluation by covariance,
precision matrix and secondary uses.

## Mechanics from the draft

Given a portfolio chosen at $t-1$, compare realised squared PnL with forecast
variance $v_t=w_{t-1}^\top\widehat\Sigma_{t-1}w_{t-1}$. The draft uses MSE
and QLIKE losses and discusses the difficulty of evaluating multivariate
forecasts through arbitrary portfolio distributions (§5.1).

Minimum-variance and alpha-driven portfolio tests examine the precision matrix
$\widehat\Sigma^{-1}$ (§5.2). A matrix can appear acceptable in ordinary
directions while producing badly estimated portfolios after inversion.
Mahalanobis diagnostics use centred returns:

$$
d_t^2=r_t^\top\widehat\Sigma_{t-1}^{-1}r_t.
$$

With the true nonsingular covariance and a zero-mean Gaussian $N$-asset
distribution, $d^2$ has a $\chi_N^2$ distribution. Estimated matrices,
heavy tails and changing universes invalidate a literal use of that reference
distribution without further calibration.

Ancillary tests inspect model turnover, hedge betas and explanatory fit.
High in-sample $R^2$ alone does not establish useful risk forecasts (§5.3).
The draft explicitly recommends regenerating portfolios with each candidate
model, rather than letting one model choose portfolios and only asking the
other to score them (§5.1.2).

## Complementary guidance

For non-negative variance proxy $y_t$ and positive forecast $v_t$, a convenient
QLIKE comparison is

$$
L(y_t,v_t)=\log v_t+\frac{y_t}{v_t}.
$$

It differs from the non-negative ratio form by terms independent of the
forecast and remains usable when $y_t=0$. Robust ranking arguments need a
conditionally unbiased proxy and the relevant regularity assumptions; they
do not justify an arbitrary noisy measurement.

Use chronological comparisons with identical data availability, capital,
constraints and execution assumptions. Report calibration and net portfolio
results together. Examine stressed periods, held-out assets, related-share
residuals, FX exposures, cross-region portfolios and short histories.

Check symmetry, positive semidefiniteness and conditioning before optimisation.
Regularisation should be visible in the model configuration. Any diagnostic
that uses subsequent returns to construct a test direction must be labelled
as an ex-post diagnostic, not as a deployable alpha backtest.

## Limits and sources

No single score ranks all risk models for all purposes. Nonstationarity limits
the significance of historical averages. Comparison is strongest when a model
is evaluated through the real decision process it will support.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 5, pp. 127–138; §14.1.

Related: [shrinkage](shrinkage.md), [backtesting](backtesting.md),
[portfolio optimisation](portfolio-optimisation.md).
