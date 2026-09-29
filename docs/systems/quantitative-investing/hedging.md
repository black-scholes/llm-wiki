# Hedging with factors and tradeable instruments

## Intuition

A hedge adds a position that offsets an unwanted source of risk. It can also
add specific risk, remove profitable exposure and incur costs. The correct
hedge therefore depends on the investor's objective and estimation uncertainty.
Draft chapter 11 starts with a simple example and relaxes its assumptions.

## Mechanics from the draft

Let a core unit return be $r_c$, a hedge unit return $r_h$ and their signed
notionals $x_c,x_h$. The minimum-variance hedge is

$$
x_h^*=-x_c\frac{\operatorname{Cov}(r_c,r_h)}{\operatorname{Var}(r_h)}
=-x_c\beta_{c|h}.
$$

With zero expected hedge excess return and no trading costs, this also
preserves the core expected PnL while reducing risk. Nonzero hedge expected
return changes the return–risk trade-off (§11.1).

Factor-mimicking portfolios isolate chosen model exposures better than a
benchmark that carries several style or sector bets, but still have specific
risk. The draft formulates factor hedging with alpha, factor and residual
variance, and execution costs in the same objective (§11.2).

Time-series hedges use estimated betas to traded instruments or baskets.
Beta estimation error motivates a smaller hedge. Section §11.4 considers
constructing portfolios that track a nontradeable factor time series.

## Complementary guidance

For several hedges with covariance $\Sigma_{hh}$ and covariance vector
$c=\operatorname{Cov}(r_h,r_c)$, the unconstrained minimum-variance solution
is $x_h^*=-\Sigma_{hh}^{-1}c\,x_c$. Use a stable linear solve and regularise
near-redundant hedge instruments. Include contract multipliers and currency
conversion before translating these exposures into orders.

Check the regression direction: the relevant coefficient predicts the core
return from hedge return, not the inverse regression. Verify the actual beta
to a factor; zero characteristic exposure may leave beta through correlated
factors. Stress beta and correlation estimates as well as marginal volatility.

Separate currency-translation hedging from equity-market hedging. A foreign
cash balance, an overseas equity position and a margined derivative need
different cash-flow treatment.

## Limits and sources

Historical covariance can break when it is most needed. Futures rolls,
forward carry, borrow and market impact change hedge economics. A smaller
estimated variance is not sufficient evidence that a hedge improves net
performance.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 11, pp. 279–290.

Related: [shrinkage](shrinkage.md), [currency rebasing](currency-rebasing.md),
[factor models](factor-models.md), [risk-model validation](risk-model-validation.md).
