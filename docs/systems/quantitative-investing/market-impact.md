# Market impact and multiperiod optimisation

## Intuition

The frictionless target portfolio is not always worth reaching immediately.
Trading changes prices, and alpha can decay before an order finishes. The
investor must choose a path of positions, not just a final position.
Draft chapter 10 develops this interaction.

## Mechanics from the draft

The draft distinguishes transaction costs and market impact, discusses
temporary impact and participation, and formulates multiperiod trading.
Its linear–quadratic treatment connects current positions, risk aversion,
impact and the persistence of the expected-return signal (§§10.1–10.3).
The no-impact limit recovers faster adjustment towards the frictionless target;
optimal liquidation is a related special case.

The AR(1) example uses a centred signal with persistence $\rho$. A current signal
forecast projects as $\mathbb E_t[s_{t+k}]=\rho^ks_t$ under the model. A slow
signal justifies a different adjustment path from a fast-decaying signal with
the same immediate expected return. Mixing signals needs their persistence
and covariance as well as their current amplitude (§10.3.6).

## Complementary guidance

A schematic finite-horizon formulation is

$$
\max_{w_1,\ldots,w_H}\mathbb E_t
\sum_{k=1}^{H}\beta^{k-1}
\left[
\mu_k^\top w_k-\frac\lambda2w_k^\top\Sigma_kw_k
-c_k(w_k-w_{k-1})-h_k(w_k)
\right],
$$

where $w_0$ is current holdings, $c_k$ trading cost and $h_k$ holding or
financing cost. This is an illustrative convex-control formulation with
consistent fixed units; a full self-financing wealth model also needs cash
and position evolution.

Linear spread cost, quadratic impact or other calibrated convex costs change
the policy in different ways. Include short borrow, fees and funding where
they apply. A terminal position condition or terminal value prevents artificial
end-of-horizon behaviour. In receding-horizon use, solve a planned path,
execute its first trade and re-estimate at the next decision time.

Estimate costs in the same dollar/share/weight convention as trades. A
dimensionless participation rate alone does not define dollar impact. Validate
fill timing, volume, alpha decay and adverse selection on executable data.

## Limits and sources

Quadratic impact and AR(1) dynamics are tractable approximations, not universal
laws. Market impact can change with liquidity and stress. Missing execution
costs can turn a seemingly optimal path into a loss-making strategy.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 10, pp. 261–278; §§14.3–14.6.
- Boyd et al. (2017), [Multi-Period Trading via Convex Optimization](https://arxiv.org/abs/1705.00109).

Related: [capacity](performance-and-capacity.md), [signal aggregation](signal-aggregation.md),
[optimisation mechanics](optimisation-mechanics.md).
