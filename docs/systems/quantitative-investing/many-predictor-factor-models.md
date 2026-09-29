# Factor models with many characteristics

## Intuition and evidence boundary

When an asset cross-section has more candidate characteristics than assets,
a regression using every characteristic cannot uniquely identify all factor
returns. Compressing the characteristics into a few time-varying loadings is
one possible response.

Draft §6.6, pp. 160–161, sketches such an approach and explicitly marks it as
too speculative or advanced for inclusion in the book. It is preserved here
as an exploratory idea, not an established production recommendation.

## Mechanics from the draft

Let $A_t\in\mathbb R^{N\times P}$ contain dated, standardised characteristics.
Seek $K$ loadings $B_t=A_tC$, with a common coefficient matrix
$C\in\mathbb R^{P\times K}$, by minimising

$$
\sum_t\|r_t-A_tCf_t\|_2^2.
$$

Unlike PCA on the return matrix, this restricts loadings to combinations of
observed characteristics. Unlike an ordinary fundamental model, the
combinations themselves are estimated from returns.

For one factor, hold $c$ fixed to estimate

$$
f_t=\frac{c^\top A_t^\top r_t}{c^\top A_t^\top A_tc},
$$

then hold those factor returns fixed to solve

$$
c=\left(\sum_tf_t^2A_t^\top A_t\right)^+
\left(\sum_tf_tA_t^\top r_t\right).
$$

The superscript $+$ denotes a pseudoinverse. These alternating least-squares
updates clarify the draft's sketch. Scale must be identified: rescaling $c$
requires the inverse rescaling of $f_t$ to preserve fitted returns. Zero
denominators and rank deficiency require explicit treatment.

## Complementary guidance

This is a bilinear, non-convex estimation problem. Exact alternating updates
can reduce the training objective but do not guarantee a unique or globally
optimal answer. Try multiple initialisations and evaluate only subsequent
returns. Normalising characteristics with future data invalidates that test.

The rank of a sum of $T$ matrices $A_t^\top A_t$ is at most
$\min(P,NT)$; the draft's smaller rank bound should not be used as a general
identity. Even if the stacked data identify the coefficients, poor conditioning
and many attempts can make the fitted combinations unreliable.

Ridge penalties or other declared restrictions can regularise $C$, but their
selection is part of the research search. Evaluate exposures, net portfolio
performance and turnover against simpler fundamental or statistical models.
Adding economically meaningful characteristics is useful only if it improves
the out-of-sample decision process.

## Sources

- Paleologo, [saved draft and its evidence boundaries](../../papers/elements-of-quantitative-investing.md), §6.6, pp. 160–161.

Related: [fundamental models](fundamental-factor-models.md),
[statistical models](statistical-factor-models.md), [backtesting](backtesting.md).
