# Statistical factor models

## Intuition

A statistical factor model finds shared movements directly from returns.
Instead of prescribing value, industry or country exposures, it estimates both
the exposures and the factor returns. PCA is useful when characteristics are
unavailable, and as a second view of risks missed by a fundamental model.
Explaining historical variance is a starting point; the goal is usable forecasts
for portfolios. This is the central subject of draft chapter 7, pp. 167–208.

## Mechanics from the draft

Let the demeaned return panel be $X\in\mathbb R^{N\times T}$: assets are rows
and observations are columns. Its SVD is $X=USV^\top$. For $K$ retained factors,

$$
\min_{\operatorname{rank}(Y)\le K}\|X-Y\|_F^2,
\qquad Y=U_KS_KV_K^\top.
$$

One valid normalisation is $B=U_K$, $F=S_KV_K^\top$. With a $1/T$ covariance
convention, the sample eigenvalues are $s_i^2/T$, not the raw squared singular
values. The cross-sectional estimate $F=B^\top X$ recovers the same factors.
An invertible change of factor coordinates changes $B$ and $F$ but preserves
their product (§§7.1.1–7.1.3).

A low-rank fitted panel alone is not a complete risk model: its covariance
is singular. Probabilistic PCA adds isotropic residual noise,

$$
\Sigma=BB^\top+\sigma^2 I,
\qquad
\widehat\sigma^2=\frac{1}{N-K}\sum_{i=K+1}^{N}\lambda_i.
$$

Under its Gaussian, isotropic assumptions, a loading estimate is
$B=U_K\operatorname{diag}(\sqrt{\lambda_i-\widehat\sigma^2})Q$, with orthogonal
$Q$. Factor eigenvalues and total asset eigenvalues therefore have different
meanings. See §7.1.2 and the [PPCA paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/bishop-ppca-jrss.pdf).

Sections 7.2–7.3 discuss finite samples, spectral gaps, factor count and unstable
eigenvectors. A large asset universe does not create a long time history.
Closely spaced eigenvalues can produce unstable individual directions even
when their combined subspace remains fairly stable. Sample eigenvalues can
overstate factor strength; [shrinkage](shrinkage.md) addresses part of this error.

The draft's practical procedure (§7.5.1) separates faster specific-volatility
estimation from slower estimation of common structure:

1. Fit an initial PCA using a recent-weighted return history.
2. Estimate specific-volatility proxies from its residuals.
3. Divide asset returns by those proxies and perform a second PCA with a
   longer effective history.
4. Estimate factor variance and residual noise in the scaled space.
5. Transform loadings and residual variances back into return units.

## Complementary guidance

Here is an explicit normalisation that avoids confusing weights with their
square roots. Let $a_t\ge0$, $\sum_ta_t=1$, $A=\operatorname{diag}(a_t)$ and
$D=\operatorname{diag}(d_i)$ contain positive specific-volatility proxies.
For weighted demeaned returns $X$, decompose

$$
Z=D^{-1}XA^{1/2}=USV^\top.
$$

Then $ZZ^\top$ is the weighted covariance in scaled units; its eigenvalues are
$s_i^2$. A model of the form $\Sigma_z=U_K\Phi U_K^\top+\Psi_z$ becomes

$$
B=DU_K,\qquad \Psi=D\Psi_zD,
\qquad \Sigma=B\Phi B^\top+\Psi.
$$

This is an illustrative restatement, not a literal transcription of Procedure
7.1. In particular, an eigenvalue used for common-factor variance must exclude
the residual-noise contribution; adding a full sample eigenvalue and residual
noise counts that noise twice. A residual-diagonal estimator is another
possible construction, with different assumptions.

For exponential weighting with half-life $h$, use weights proportional to
$2^{-\mathrm{age}/h}$. The approximate effective sample size
$T_{\mathrm{eff}}=1/\sum_ta_t^2$ helps reveal how much information weighting
discards. It is not a licence to replace $T$ with $T_{\mathrm{eff}}$ in every
random-matrix theorem.

Choose $K$, weighting and shrinkage with a chronological
[risk-model evaluation](risk-model-validation.md). Explained-variance targets
and scree plots are diagnostics, not portfolio-quality objectives. Raw-return,
total-volatility and specific-volatility PCA optimise different weighted errors;
the last two are not interchangeable.

## Changing universes and limits

The draft (§7.5.2) discusses sign alignment, changing estimation universes,
regression-based loadings for assets outside the estimation universe, and
characteristic-based imputation for short-history assets. Complementary practice
is to fit with information available by $t-1$ when forecasting period $t$.
Keep imputed loadings identifiable and give them an uncertainty allowance.

Compare consecutive subspaces on common assets. Align signs for isolated
components; use an orthogonal alignment for a near-degenerate group and
transform factor covariance consistently. Basis alignment improves reporting
continuity; it cannot manufacture a stable underlying risk structure.
Missing observations are not zero returns, and close-to-close observations
across time zones may not refer to the same information interval.

Normality, stationarity and pervasive-factor assumptions are approximations.
Statistical factors need not be causal, profitable or cheaply tradeable. Interpret
them through characteristic regressions or clustering (§7.4), and compare them
with [fundamental models](fundamental-factor-models.md).

## Sources

- Paleologo, [saved draft and chapter map](../../papers/elements-of-quantitative-investing.md), chapter 7; appendix §§14.2, 14.5.
- Tipping and Bishop (1999), [Probabilistic Principal Component Analysis](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/bishop-ppca-jrss.pdf).

Related: [factor models](factor-models.md), [shrinkage](shrinkage.md),
[multiple markets](multi-market-models.md).
