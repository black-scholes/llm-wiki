# Performance attribution and estimation uncertainty

## Intuition

Attribution explains realised PnL through exposures and residual returns.
It is a model-based explanation, not automatic evidence that a manager has
skill. Draft chapter 13 examines the errors and ambiguities in this explanation.

## Mechanics from the draft

For positions selected before the return interval,

$$
\operatorname{PnL}_t=w_{t-1}^\top r_t
=b_{t-1}^\top f_t+w_{t-1}^\top\epsilon_t,
\qquad b_{t-1}=B_{t-1}^\top w_{t-1}.
$$

An intercept or expected residual component can be retained separately when
the model includes it. Summing over time and groups produces factor and
asset-specific attribution (§13.1).

Estimated factor returns and exposures have errors. Apparent factor timing
or idiosyncratic skill may reflect those errors (§13.2). Individual factor PnL
also changes with factor normalisation or rotation even when fitted returns
and total factor PnL do not (§13.3).

The draft's maximal attribution uses the relationship with correlated factor
returns to explain a factor shock. Exposure to one factor alone is insufficient
because other factors can transmit that shock. The result is conditional on
the covariance and distribution assumptions; it is not a causal attribution.

Selection–sizing attribution (§13.4) distinguishes being on the right side of
residual returns from giving greater size to more successful positions, and
links these components with diversification. Hit rates and ratios of average
wins to losses do not by themselves identify the information ratio.
Section §13.5 also discusses time-series attribution.

## Complementary guidance

Keep a stable reporting convention, dated holdings and estimated-factor
versions. Reconcile model PnL to accounting PnL by identifying currency,
cash, funding, execution costs, corporate actions and unexplained differences.
Compounded investor-currency returns require exact translation; a linear
factor attribution will otherwise leave interaction residuals.

Examine uncertainty over many periods. In an illustrative fixed linear
Gaussian model, conditional means use covariance, for example
$\mathbb E[p\mid f_j]=\operatorname{Cov}(p,f_j)f_j/operatorname{Var}(f_j)$
for centred quantities. The conditional uncertainty is still nonzero.
This mathematical explanation should not be presented as proof that a
particular factor caused an observed loss.

## Limits and sources

A residual inherits model omissions. A rotated factor name can change a
manager's apparent source of performance without changing the underlying
portfolio. Attribution should guide investigation and future decisions while
preserving these uncertainty boundaries.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 13, pp. 311–334.

Related: [factor models](factor-models.md), [currency rebasing](currency-rebasing.md),
[risk-model validation](risk-model-validation.md).
