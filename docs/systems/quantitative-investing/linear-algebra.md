# Linear algebra for factor models

## Intuition

Regression, PCA and risk decomposition are ways to project information into
smaller spaces. A projection can simplify a model without creating new
information; an unstable coordinate system can make the same underlying
structure appear to change.

## Mechanics from the draft

For full-column-rank $B$, ordinary least squares projects $r$ into its column
space with $P=B(B^\top B)^{-1}B^\top$. The residual is $(I-P)r$ and is
Euclidean-orthogonal to the columns of $B$. Weighted least squares uses
$P_W=B(B^\top WB)^{-1}B^\top W$; orthogonality is then in the $W$ inner
product (§§3.7.1–3.7.2).

The Frisch–Waugh–Lovell result obtains a variable group's coefficient by
residualising both the target and that group against the controls, then
regressing the residuals. It helps separate a candidate signal from exposures
already represented in a model (§3.7.3).

SVD represents a matrix as $USV^\top$. Singular values reveal rank and poorly
identified directions. Truncated SVD gives the best low-rank approximation
under the Frobenius norm, and connects cross-sectional regression to
time-series loadings in statistical factor models (§3.7.4; §7.1).

## Complementary guidance

For a linear random-vector transformation $y=Ar$, covariance transforms as
$\Sigma_y=A\Sigma_rA^\top$. A true change of factor coordinates must transform
loadings and factor covariance together. Changing covariance alone changes
the model, as in block alignment of regional models.

Condition number measures sensitivity to input errors. Forming normal
equations squares the design matrix's spectral condition number. Prefer
QR/SVD solves; use Cholesky for positive-definite covariance systems and a
declared regularisation or rank treatment when that assumption fails.

With positive-definite $\Psi$ and $\Phi$, the factor-model inverse can be
applied through the Woodbury identity:

$$
(\Psi+B\Phi B^\top)^{-1}
=\Psi^{-1}-\Psi^{-1}B
(\Phi^{-1}+B^\top\Psi^{-1}B)^{-1}B^\top\Psi^{-1}.
$$

This can replace an asset-sized solve with smaller factor operations. Singular
factor covariance needs a suitable alternative factorisation; the displayed
inverse formula cannot simply be applied unchanged.

The draft's Gaussian-conditioning appendix (§14.7) supports attribution and
linear risk scenarios. For $r\sim N(\mu,\Sigma)$ and a constraint $A^\top r=b$,
the correctly dimensioned conditional moments, when $A^\top\Sigma A$ is
invertible, are

$$
\mu_c=\mu+\Sigma A(A^\top\Sigma A)^{-1}(b-A^\top\mu),
\qquad
\Sigma_c=\Sigma-\Sigma A(A^\top\Sigma A)^{-1}A^\top\Sigma.
$$

This restates the result with typographical inconsistencies removed. Gaussianity
is essential for interpreting the full conditional distribution this way;
covariance alone does not determine a general conditional distribution.

## Limits and sources

Numerical rank depends on scale and tolerance. Rotations in a nearly repeated
eigenspace can be harmless for risk but troublesome for factor-by-factor
reporting. Robust numerical algebra cannot repair incorrectly dated data.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §3.7, chapter 7 exercises; §§14.2, 14.5, 14.7.

Related: [statistical models](statistical-factor-models.md),
[multi-market models](multi-market-models.md), [portfolio optimisation](portfolio-optimisation.md).
