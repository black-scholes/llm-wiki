# Dynamic risk allocation and fractional Kelly

## Intuition

Repeatedly investing a fraction of wealth is different from holding a fixed
dollar allocation. Compounding makes excessive leverage costly: a strategy
with positive arithmetic expected return can have poor long-run growth.
Draft chapter 12 uses the Kelly criterion to connect growth and risk allocation.

## Mechanics from the draft

For excess risky return $r$ and fraction $f$ in the risky investment, maximise
expected log gross excess wealth in the simple approximation

$$
\max_f\mathbb E[\log(1+fr)].
$$

The feasible set must keep gross wealth positive. A second-order expansion
gives

$$
\mathbb E[\log(1+fr)]
\simeq f\mu-\frac{f^2}{2}(\sigma^2+\mu^2).
$$

If $\mu^2$ is negligible, $f^*\simeq\mu/\sigma^2$. In the standard continuous
diffusion model the corresponding log-growth rate is
$g(f)=f\mu-f^2\sigma^2/2$, with optimum $\mu/\sigma^2$.
These are different model statements; neither is an exact formula for every
discrete return distribution.

Fractional Kelly chooses $f=c f^*$ with $0<c<1$. The draft discusses uncertain
parameters, finite horizons, costs, drawdown control and practical heuristics
(§§12.2–12.4). Lower exposure sacrifices modelled growth to reduce volatility
and sensitivity to estimation error.

## Complementary guidance

In the diffusion example with known constant parameters,
$g(cf^*)=(2c-c^2)g(f^*)$ and volatility is $c$ times full-Kelly volatility.
Thus half Kelly retains 75% of modelled optimal growth with half its
volatility. This is a mathematical illustration, not a performance promise
for a real strategy.

For several investments the local growth approximation leads to direction
$\Sigma^{-1}\mu$. Estimate uncertainty in both inputs, incorporate capacity,
funding and drawdown tolerances, and use the same period units for Sharpe
and volatility. Capital fractions, leverage and volatility budgets are
different allocation variables.

## Limits and sources

Kelly optimises a particular long-run objective under a model. It does not
maximise every investor's utility or guarantee a tolerable drawdown. Fat tails,
nonstationarity, borrowing limits and return-dependent costs can make an
estimated full-Kelly allocation unusable. Fractional sizing reduces exposure;
it does not remove those modelling problems.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 12, pp. 293–310.

Related: [performance metrics](performance-and-capacity.md),
[constraints](constraints-and-estimation-error.md), [volatility estimation](volatility-estimation.md).
