# Factor models: exposures, risk and alpha

## Intuition

Assets often move together. A factor model explains their common movements
through a small set of exposures and leaves a residual for asset-specific
behaviour. The draft uses the same structure for risk forecasting, alpha
research, portfolio construction and attribution (chapter 3).

## Mechanics from the draft

Write excess returns as

$$
r=\alpha+Bf+\epsilon,
\qquad \operatorname{Cov}(f,\epsilon)=0.
$$

$B$ contains asset loadings, $f$ factor returns and $\epsilon$ residual returns.
With factor covariance $\Phi$ and residual covariance $\Psi$,

$$
\Sigma=B\Phi B^\top+\Psi.
$$

For portfolio $w$, exposure is $b=B^\top w$ and variance is
$b^\top\Phi b+w^\top\Psi w$. A strict factor model assumes diagonal
$\Psi$; an approximate model allows some remaining cross-asset dependence.

| Model | Inputs treated as known | Estimated quantities |
| --- | --- | --- |
| Fundamental/characteristic | Returns and asset characteristics | Factor and residual returns |
| Macroeconomic/time-series | Returns and observed factor series | Asset loadings and residuals |
| Statistical | Returns | Loadings, factors and residuals |

This classification is about identification, not about whether a factor earns
a premium. The expected-return vector can have a component spanned by $B$
and an orthogonal component; the meaning of orthogonality depends on the
chosen inner product (§3.3).

For invertible $Q$, replacing $B$ with $BQ$ and $f$ with $Q^{-1}f$ preserves
fitted returns. Transforming $\Phi$ consistently preserves covariance.
Factor labels and individual attribution can change under this rotation while
aggregate risk remains unchanged (§3.4).

## Complementary guidance

Zero loading on factor $j$ does not necessarily imply zero beta to its return.
For centred returns,

$$
\beta_{w,j}=\frac{b^\top\Phi e_j}{\Phi_{jj}}
$$

under the residual-orthogonality assumptions. Correlated factors transmit
risk through the other exposures. Likewise, individual contributions to
variance need cross terms; adding only $b_j^2\Phi_{jj}$ generally misses
covariance. A standard Euler allocation is $b_j(\Phi b)_j$, whose sum equals
factor variance and whose terms can be negative.

Keep forecast alpha separate from realised unexplained returns. A profitable
residual can be estimation error, a missing factor, or skill. Test these
interpretations before labelling it alpha.

## Limits and sources

Linearity and residual orthogonality are model assumptions. Exposure
definitions, normalisations and dated universe membership need to remain
consistent across forecasting, optimisation and reporting.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), chapter 3, pp. 73–104.

Related: [fundamental models](fundamental-factor-models.md),
[statistical models](statistical-factor-models.md), [attribution](performance-attribution.md).
