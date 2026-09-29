# Combining signals and portfolios

## Intuition

Combining return forecasts and then building one portfolio is different from
adding several independently optimised portfolios. They coincide in a narrow
linear setting; costs, constraints and different risk assumptions often make
them diverge.

This page is complementary guidance. The draft discusses mixing signals in
§10.3.6, p. 276; the published contents places a dedicated aggregation discussion
in §9.6. The following algebra is an independent explanation.

## Mechanics

Suppose every signal uses the same covariance $\Sigma$, risk aversion $\lambda$
and unconstrained mean–variance construction. For
$\mu=\sum_j a_j\mu_j$,

$$
w(\mu)=\lambda^{-1}\Sigma^{-1}\sum_j a_j\mu_j
=\sum_j a_jw(\mu_j).
$$

The constructor is linear in alpha. This equality does not persist in general
with nonlinear costs, signal-specific covariance, different leverage
normalisation, shared position limits or risk budgets.

For example, if two separately managed signals both want a stock position
at the maximum allowable limit, summing their portfolios violates the joint
limit. A central constructor can apply that limit once and allocate capital
among the other opportunities. Conversely, decentralised portfolios can
preserve local modelling expertise that a central model does not capture.

## Practical implications

Calibrate forecasts to common return units and horizons before averaging.
Several highly correlated signals are not several independent confirmations.
Estimate combination weights chronologically and account for correlated
forecast errors.

Fast and slow signals may share a risk model while needing different trading
speeds. Their expected returns at different horizons cannot simply be added
as interchangeable forecasts. A joint multiperiod model can represent both
decay rates and their interaction with current holdings.

Even when teams retain separate portfolio constructors, a shared execution
layer can net opposing orders. Its savings and cost allocation need to be
measured rather than inferred from the gross turnover of individual books.

## Limits and sources

Centralisation depends on available forecasts, information boundaries and
compatible risk estimates. The linear identity is a useful diagnostic, not
a general argument that either organisation always dominates.

- Paleologo, [draft and edition map](../../papers/elements-of-quantitative-investing.md), §10.3.6; published contents §9.6.
- Boyd et al., [Multi-Period Trading via Convex Optimization](https://arxiv.org/abs/1705.00109), for a general framework including forecasts, constraints and costs.

Related: [portfolio optimisation](portfolio-optimisation.md),
[market impact](market-impact.md), [performance attribution](performance-attribution.md).
