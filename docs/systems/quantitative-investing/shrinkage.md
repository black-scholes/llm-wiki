# Shrinkage in investment models

## Intuition

Shrinkage trades a little bias for less estimation noise. This matters because
a portfolio optimiser can interpret a noisy small eigenvalue as a cheap hedge
and concentrate positions in that direction. The same principle applies to
factor variances, specific risk, forecasts and hedge betas, but the estimators
and targets differ.

The draft treats factor-return estimation noise in §6.3.1, PCA eigenvalue
shrinkage in §§7.1.2 and 7.2.3, and beta uncertainty in §11.3. Its heading
§6.4.5 on specific-variance shrinkage is unfinished.

## Factor covariance and estimation noise

With $r=Bf+\epsilon$, specific covariance $\Psi$ and full-rank $B$, weighted
least squares estimates factor returns as

$$
\widehat f=(B^\top\Psi^{-1}B)^{-1}B^\top\Psi^{-1}r.
$$

If $B$ and $\Psi$ are fixed and factor and residual returns are uncorrelated,

$$
\operatorname{Cov}(\widehat f)
=\Phi+(B^\top\Psi^{-1}B)^{-1}.
$$

The measured factor-return covariance includes the noise of the estimating
portfolio. Subtracting that term estimates latent common-factor covariance;
it does not estimate the full covariance of tradeable factor-mimicking returns.
The draft makes this distinction in §6.3.1.

Complementary guidance: direct subtraction in a finite sample can produce a
negative eigenvalue. Check the result and use a constrained positive-semidefinite
estimator or a justified regularisation step. Shrinkage is not guaranteed to
repair every invalid input.

## Complementary guidance: targets and objectives

A linear estimator combines a sample covariance $S$ with a target $F$:

$$
S_\delta=(1-\delta)S+\delta F,\qquad 0\le\delta\le1.
$$

| Target | What it assumes |
| --- | --- |
| $F=\operatorname{tr}(S)I/N$ | Variances should be closer together; uncertain covariances move towards zero |
| $F=\operatorname{diag}(S)$ | Preserve sample variances and reduce uncertain correlations |
| Structured factor or constant-correlation target | Retain a chosen pattern of dependence |
| Block-diagonal regional target | Preserve local models and reduce uncertain cross-market dependence |

If $S$ is positive semidefinite, $F$ is positive definite and $\delta>0$, the
blend is positive definite. A target is a modelling assumption, not a neutral
choice. The draft illustrates the scaled-identity target; the targets in this
table are broader standard alternatives. The
[Ledoit–Wolf paper](https://ledoit.net/honey.pdf) derives a shrinkage estimator
and tests its use in portfolio construction.

Select intensity for the intended use. A Frobenius-loss optimum need not minimise
out-of-sample portfolio variance or improve net returns. Estimate parameters
inside the training window and validate through the
[portfolio construction process](risk-model-validation.md).

## Spectral and specific-risk shrinkage

For $S=U\operatorname{diag}(\lambda_i)U^\top$, spectral shrinkage keeps sample
directions and changes their variances:

$$
\widehat\Sigma
=U\operatorname{diag}(d_i)U^\top.
$$

PPCA subtracts an estimated isotropic noise floor from the retained common
eigenvalues. The draft also examines corrections under a spiked-covariance
model and contrasts Gaussian with heavy-tailed simulations (§7.2). Those
corrections depend on their asymptotic regime and noise assumptions.
General [nonlinear shrinkage](https://arxiv.org/abs/1207.5322) estimates a
different correction for different eigenvalues; it is not just removing all
components below an arbitrary cutoff.

To complement unfinished §6.4.5, a simple specific-risk estimator is

$$
\widetilde\psi_i=(1-\delta_i)\widehat\psi_i+\delta_i\psi_{0,i},
$$

with a positive pooled target $\psi_{0,i}$. Pool within comparable groups when
equal variance is implausible. Stronger shrinkage for short, noisy histories
can be reasonable, but its strength needs calibration. Preserve real residual
dependencies such as related share classes where they matter to the strategy.

## Hedge and alpha shrinkage

The draft's noisy-beta hedge analysis (§11.3) shrinks the full hedge according
to aggregate beta uncertainty. For estimated portfolio beta
$\widehat b=w^\top\widehat\beta$ and beta-error covariance $\Gamma$, its
practical coefficient is

$$
c=\max\left(0,1-\frac{w^\top\Gamma w}{\widehat b^2}\right),
\qquad x_h=-c\widehat b.
$$

Use $x_h=0$ for zero estimated beta; do not divide by zero. This is the draft's
approximate hedge prescription under its particular error model, not a general
posterior-mean formula. Correlated errors can defeat the apparent benefit of
holding many assets.

Complementary Bayesian illustration: if an alpha estimate $y$ has independent
Gaussian observation noise $N(0,s^2)$ and the true alpha has prior $N(0,\tau^2)$, its posterior mean is
$\tau^2y/(\tau^2+s^2)$. Noisier estimates move further towards zero. The prior
and noise model must be justified; this illustration is not an extra theorem
claimed from the draft.

## Limits and sources

Covariance conditioning, residual calibration and net portfolio outcomes all
matter. Shrinkage cannot correct stale prices, look-ahead, omitted currency
exposure or a structurally wrong universe. Keep the units and prediction horizon
consistent and do not double-count residual noise.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §§6.3.1, 7.1.2, 7.2.3–7.2.4, 11.3.
- Ledoit and Wolf, [linear covariance shrinkage](https://ledoit.net/honey.pdf) and [nonlinear shrinkage](https://arxiv.org/abs/1207.5322).

Related: [statistical models](statistical-factor-models.md),
[hedging](hedging.md), [constraints](constraints-and-estimation-error.md).
