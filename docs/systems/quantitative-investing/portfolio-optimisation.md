# Mean–variance portfolio optimisation

## Intuition

An optimiser converts expected returns and risk into positions. It may short
an asset with a positive expected return because that asset is a useful hedge
for a more attractive investment. The result depends on the quality of the
inputs, not just on solving the mathematical problem accurately.

## Mechanics from the draft

For excess-return forecast $\mu$, positive-definite covariance $\Sigma$ and
risk-aversion coefficient $\lambda>0$, the unconstrained problem is

$$
\max_w\;\mu^\top w-\frac\lambda2w^\top\Sigma w,
\qquad w^*=\lambda^{-1}\Sigma^{-1}\mu.
$$

This is the basic formulation in draft chapter 8. A volatility budget changes
the scale of the same direction when there are no other constraints. Adding
budget, leverage, exposure or trading constraints changes the solution.

In factor coordinates, $\Sigma=B\Phi B^\top+\Psi$. A minimum-specific-risk
portfolio with unit exposure to factor $j$ and zero exposure to the other
factors is

$$
p_j=\Psi^{-1}B(B^\top\Psi^{-1}B)^{-1}e_j,
\qquad B^\top p_j=e_j.
$$

This requires positive-definite $\Psi$ and full-rank $B$. Factor-mimicking
portfolios still have specific risk. Trading a factor return cannot automatically
be treated as trading an instrument with no implementation cost (§8.3).

In a diagonal specific-risk model, an alpha-orthogonal portfolio has direction
$\Psi^{-1}\alpha_\perp$. Risk-weighted orthogonality must be specified;
Euclidean orthogonality alone does not generally guarantee factor neutrality.
The draft relates information ratio to information coefficient and
diversification in §8.5.

## Complementary guidance

Solve $\Sigma x=\mu$ through a stable factorisation rather than constructing
the inverse. Specify whether $w$ contains dollar positions, NAV weights or
volatility allocations, and scale $\lambda$ accordingly. Include an explicit
cash position or financing assumption when required.

A simple identity illustrates how optimiser inputs matter: in an eigenvector
direction, the unconstrained position is proportional to alpha divided by
variance. A falsely tiny variance produces an excessive position. Combine
[shrinkage](shrinkage.md), forecast calibration and realistic constraints.

## Limits and sources

Mean–variance objectives use only two moments and do not control arbitrary
tail or liquidity risks. The familiar square-root breadth interpretation of
information ratio depends on independence and comparable bet quality; the
number of assets is not automatically the number of independent opportunities.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §§8.1–8.5, pp. 211–222.

Related: [constraints](constraints-and-estimation-error.md),
[optimisation mechanics](optimisation-mechanics.md), [signal aggregation](signal-aggregation.md).
