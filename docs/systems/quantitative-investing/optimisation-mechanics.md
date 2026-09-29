# Optimisation mechanics for portfolios

## Intuition

The value of a portfolio solution lies partly in understanding why it has
those positions. Convexity, first-order conditions and dual variables help
explain the solution and identify which limits are costing the most.
The draft introduces these tools in §8.7 and returns to linear–quadratic
control in appendix chapter 14.

## Mechanics from the draft

A concave quadratic objective with convex constraints admits a global optimum
when it is feasible and the optimum is attained. For positive-definite
quadratic risk, the unconstrained first-order equation is
$\lambda\Sigma w=\mu$. Equality constraints introduce Lagrange multipliers;
inequality constraints add feasibility and complementary-slackness conditions.

The Karush–Kuhn–Tucker conditions can certify optimality under the appropriate
convexity and constraint qualifications. Dual variables express local changes
in optimal objective as constraint bounds change. The draft uses local analysis
to connect optimisation choices with their effect on the portfolio.

Multiperiod problems introduce a state, such as current holdings, and a
control, such as the next trade. Quadratic costs and linear state evolution
lead to linear–quadratic regulator methods. Discounting and terminal conditions
affect the optimal feedback policy (§§14.3–14.4).

## Complementary guidance

Define the variable convention, cash balance, objective terms and feasible set
before choosing a solver. Check residual constraint violations and objective
components after solving; a success status alone does not verify that the
economic problem was specified correctly.

Report an infeasible risk budget as infeasible rather than silently relaxing
it. Position and turnover scales can differ by orders of magnitude, so use
reasonable numerical scaling. A shadow price is only local: changing a bound
far enough can change the active constraints and invalidate extrapolation.

For an impact-aware policy, verify the no-cost limit, a zero-alpha liquidation
case and behaviour under binding execution constraints. Receding-horizon
planning executes the first control and updates the rest as information changes.

## Limits and sources

Convexity is a property of the specified model. It does not make the model's
return, risk or execution assumptions correct. Fixed costs, discrete lot
sizes and some other trading rules create different optimisation problems.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §8.7, pp. 229–238; §§14.3–14.7, pp. 338–344.
- Boyd et al., [Multi-Period Trading via Convex Optimization](https://arxiv.org/abs/1705.00109).

Related: [constraints](constraints-and-estimation-error.md),
[market impact](market-impact.md), [linear algebra](linear-algebra.md).
