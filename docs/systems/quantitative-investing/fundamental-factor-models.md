# Fundamental factor models

## Intuition

A fundamental model chooses interpretable asset characteristics first, then
asks what returns those characteristics explain. Industry, country, size,
momentum and valuation loadings help turn the risk model into a language
shared by researchers and portfolio managers. Draft chapter 6 develops the
estimation process and its pitfalls.

## Mechanics from the draft

With characteristics $B_{t-1}$ known before period $t$, estimate

$$
\widehat f_t=\arg\min_f
(r_t-B_{t-1}f)^\top W_t(r_t-B_{t-1}f).
$$

If $W_t=\Psi_t^{-1}$ and the loadings have full column rank, the solution is
$(B^\top WB)^{-1}B^\top Wr_t$. The weighting reduces the influence of noisy
specific returns. Estimate factor covariance from $\widehat f_t$ and specific
risk from $\widehat\epsilon_t=r_t-B_{t-1}\widehat f_t$.

An intercept and a complete set of industry or country dummies create rank
deficiency. Impose an identification convention, such as a weighted sum of
industry returns equal to zero, and transform covariance consistently
(§§6.2.1–6.2.2). The convention can change individual factor returns without
changing fitted asset returns.

The draft discusses separate factor-volatility and correlation estimation,
short-term updates, serial dependence and specific-risk estimation
(§§6.3–6.4). It also identifies clusters of correlated residuals, including
related securities, which a diagonal residual matrix misses. Returns should
be inspected for data errors, but genuine jumps should not disappear through
automatic winsorisation (§6.5).

## Complementary guidance

Maintain separate estimation and investment universes. A clean, liquid
estimation universe may improve factor estimates; omitted investment assets
still require loadings and specific-risk forecasts. Standardise characteristics
using the information and cross-section available at each historical date.

Use QR or SVD-based weighted least squares rather than explicitly inverting
an unstable normal-equations matrix. Record constraints and weights so
factor-mimicking portfolios, covariance and attribution use compatible
definitions. A tiny specific-variance estimate should not give an asset
unlimited influence.

Country, industry and FX factors need separate identification. A local-market
return model can be expanded into investor-currency space, but adding FX
without its covariance with local factors is incomplete.

## Limits and sources

Interpretability is not evidence of superior forecasting. A large menu of
characteristics can create redundancy and selection error. Section §6.6 on
[many predictors](many-predictor-factor-models.md) is explicitly labelled
speculative in the draft. Sections §6.4.5 on variance shrinkage and §6.7.1 on
linkage are unfinished; complementary methods are identified in the related
pages rather than presented as completed draft results.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 6, pp. 139–166.

Related: [shrinkage](shrinkage.md), [multiple markets](multi-market-models.md),
[currency rebasing](currency-rebasing.md), [statistical models](statistical-factor-models.md).
