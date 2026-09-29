# Backtesting and multiple testing

## Intuition

A strong historical result can arise from searching many weak ideas. Testing
the winning variant as though it were the only idea tried confuses selection
with evidence. Draft chapter 4 addresses research process and evaluation;
appendix §14.8 compares related testing approaches.

## Mechanics from the draft

Backtests need dated data, historically available characteristics, reliable
corporate actions and realistic costs. Walk-forward evaluation trains using
past information and tests subsequently. The protocol matters as much as a
particular statistical score (§§4.1–4.2).

The Rademacher Anti-Serum (§4.3) organises candidate strategy outcomes into a
matrix $X\in\mathbb R^{T\times M}$: observations are rows and candidates are
columns. Random signs are shared across each row, preserving the relationships
between candidates. A representative empirical complexity term is

$$
\widehat{\mathcal R}(X)
=\mathbb E_\xi\left[
\max_{m\le M}\frac1T\sum_t\xi_tX_{tm}
\right],\qquad \xi_t\in\{-1,+1\}.
$$

The draft uses this quantity in bounds on expected performance, together with
sampling-error terms under specified distribution assumptions. Dependence
between candidates matters: duplicating one strategy is different from adding
many independent strategies. The term is not a generic numerical haircut to
apply to any reported Sharpe ratio.

Appendix §14.8 discusses White's reality check, Romano–Wolf familywise error
control and the Hansen–Lunde–Nason model confidence set. These answer different
questions about beating a benchmark, selecting non-null candidates or retaining
a set of competitive models.

## Complementary guidance

Retain the full attempted candidate family and research history. A matrix
containing only surviving strategies understates the search that produced
them. Use independent chronological holdouts for consequential choices,
including factor count, shrinkage, constraints and cost parameters.

Random folds can leak information when labels share future return intervals.
Remove overlapping training labels at a test boundary and allow a gap when
information availability or serial dependence warrants it. The required gap
follows from the strategy's timestamps and horizon, not a universal constant.

Serial dependence can require block resampling or aggregation. Row-wise sign
randomisation alone does not solve every dependence problem. Corrected
significance cannot repair survivorship bias, impossible fills or a missing
borrow fee.

## Limits and sources

The draft's finite-sample results depend on stated boundedness or tail and
sampling assumptions. Heavy tails and regime changes need additional care.
Statistical evidence and deployable capacity remain separate questions.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 4, pp. 105–126; §14.8, pp. 345–348.

Related: [performance and capacity](performance-and-capacity.md),
[risk validation](risk-model-validation.md), [market impact](market-impact.md).
