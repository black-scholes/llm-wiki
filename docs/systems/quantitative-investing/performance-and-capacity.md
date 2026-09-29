# Performance, Sharpe ratio and capacity

## Intuition

A strategy with an attractive return per unit of risk may produce little
total profit. Scaling it can increase trading costs, change the portfolio and
reduce that ratio. Performance and capacity therefore need to be evaluated
together. The draft covers these metrics in §8.6 and capacity trade-offs in §9.4.

## Mechanics from the draft

Expected excess PnL $m$ and volatility $s$ define a period Sharpe ratio
$\mathrm{SR}=m/s$. The units and sampling interval must match. Leverage scales
both quantities equally only when returns, funding and costs remain linear.
Information ratio instead compares active or residual return with the relevant
tracking or specific volatility; its denominator must be stated.

Capacity concerns how performance changes as more capital is deployed. Market
impact, available volume and constraints affect the feasible strategy. A small
high-Sharpe portfolio and a larger lower-Sharpe portfolio can have different
economic value. The draft's analysis links this trade-off to objective and
constraint choices rather than treating maximum Sharpe as the only goal.

## Complementary guidance

The common annualisation $\mathrm{SR}_{\mathrm{annual}}\simeq\sqrt A\,
\mathrm{SR}_{\mathrm{period}}$ assumes comparable periods and sufficiently
uncorrelated returns. For stationary scalar excess returns, the variance of
an $A$-period sum is

$$
\operatorname{Var}\left(\sum_{t=1}^{A}r_t\right)
=A\gamma_0+2\sum_{k=1}^{A-1}(A-k)\gamma_k.
$$

Serial covariance changes the scaling. Sharpe ratios estimated from short
or selected histories have material uncertainty even when annualisation is
performed correctly.

Run the entire construction and execution process at several capital levels.
Report net PnL, volatility, turnover, participation, financing and borrow costs,
drawdown and concentration. Define capacity operationally: for example, the
largest capital satisfying specified net-performance and execution constraints.
There is no single capital figure independent of these requirements.

## Limits and sources

Sharpe does not describe all tail behaviour, and a high ratio may hide stale
valuation or a favourable funding convention. Estimated future capacity is
a conditional forecast, not a fixed asset of the strategy.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §8.6, pp. 223–228; §9.4, p. 256.

Related: [backtesting](backtesting.md), [market impact](market-impact.md),
[dynamic allocation](dynamic-risk-allocation.md).
