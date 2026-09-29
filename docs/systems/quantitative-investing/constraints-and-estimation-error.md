# Constraints and estimation error

## Intuition

An optimiser can turn a small forecasting error into a large position. A
constraint can prevent that error from dominating the book, but can also
remove a real opportunity. Draft chapter 9 examines this trade-off.

## Mechanics from the draft

The chapter starts with a two-asset example showing how a high correlation
creates large hedged positions. It distinguishes uncertainty in alpha from
uncertainty in risk, and examines how both affect Sharpe efficiency (§§9.1, 9.3).

Practical formulations use limits on gross exposure, net exposure, individual
positions, factor exposure, turnover and liquidity. They can also use penalties
for holding or trading positions (§9.2). A binding constraint has a shadow
price: a local measure of the objective improvement from relaxing its limit.
It is a diagnostic for that problem and those inputs, not a stable economic
constant.

## Complementary guidance

An $L_2$ holding penalty provides a transparent link to regularisation:

$$
\max_w\;\mu^\top w-
\frac\lambda2w^\top\Sigma w-\frac\eta2\|w\|_2^2
\quad\Longleftrightarrow\quad
\Sigma_{\mathrm{effective}}=\Sigma+\frac\eta\lambda I.
$$

This identity describes the optimiser's penalty; it does not by itself produce
a calibrated covariance forecast. An $L_1$ turnover penalty instead depends
on the distance from current holdings and can create no-trade regions.

For illustrative ellipsoidal alpha uncertainty $\mu=\widehat\mu+u$ with
positive-definite uncertainty matrix $C$ and
$\|C^{-1/2}u\|_2\le\rho$, worst-case expected return is

$$
\min_\mu\mu^\top w
=\widehat\mu^\top w-\rho\sqrt{w^\top Cw}.
$$

This standard robust-optimisation derivation complements the draft; the
uncertainty set needs independent calibration. It makes exposure to estimation
uncertainty costly rather than pretending forecasts are known.

Choose constraints from actual trading limits and forecast uncertainty.
Compare constrained and unconstrained outcomes under the same chronological
protocol. Searching many limit combinations is itself a source of overfitting.

## Limits and sources

Constraints may improve realised performance under noisy estimates; they
cannot improve the oracle maximum of the same objective by shrinking its
feasible set. Apparent improvements need to distinguish input error, costs
and a change in the economic objective.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 9, pp. 239–260.

Related: [shrinkage](shrinkage.md), [optimisation mechanics](optimisation-mechanics.md),
[risk-model validation](risk-model-validation.md).
