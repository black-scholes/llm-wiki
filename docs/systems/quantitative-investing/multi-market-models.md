# Risk models across multiple markets

## Intuition and source boundary

Two regional books can each have an accurate local risk model while their
combined risk remains badly estimated. Cross-market covariances determine how
much they diversify. A single date label also hides different closing times,
holidays and available information.

The saved draft names this problem in §6.7, p. 161, but leaves model linkage
unfinished. This page supplements it with the inspected linkage discussion in
published §6.6.1 and standard covariance algebra. The equations below are
independent derivations; they are not attributed to the unfinished draft.

## Three model designs

| Design | Useful when | Main limitation |
| --- | --- | --- |
| Joint global model | Cross-market trading needs common factors and one shared asset universe | Data alignment and missing observations can dominate estimation |
| Linked local models | Regional models should retain their local risk estimates | Cross-market covariance and block consistency need separate estimation |
| Global factors plus local residual factors | Global exposures should be interpretable while regional structure remains flexible | Independence of local residual blocks must be tested |

The published linkage discussion describes these three approaches. In a
global/local model, regress local factors on global factors and examine the
remaining cross-market residual dependence. Orthogonality to global factors
alone does not imply that two local residual blocks are uncorrelated.

## Complementary guidance: preserve local blocks

For market $a$, let $r_a=B_af_a+\epsilon_a$ and
$\Sigma_a=B_a\Phi_aB_a^\top+\Psi_a$. Stack the loadings block-diagonally.
If specific risk is assumed block-diagonal, cross-market covariance comes from
the off-diagonal blocks of the factor covariance $\Phi$:

$$
\Sigma_{ab}=B_a\Phi_{ab}B_b^\top.
$$

This makes an important assumption explicit: direct residual dependence across
markets is omitted. Dual listings, ADRs and related instruments may need
separate treatment.

Suppose a joint estimate $V$ has diagonal blocks $V_{aa}$ that differ from
trusted local factor covariances $\Phi_a$. When these blocks are positive
definite, define

$$
L_a=\Phi_a^{1/2}V_{aa}^{-1/2},\qquad
L=\operatorname{blockdiag}(L_a),\qquad \widetilde\Phi=LVL^\top.
$$

Using symmetric matrix roots gives
$\widetilde\Phi_{aa}=\Phi_a$. Congruence preserves positive semidefiniteness
when $V$ is positive semidefinite. This is a column-vector convention for
block covariance alignment; it is not an ordinary factor rotation because
the asset loadings are left fixed and the risk model changes.

Let $\Phi_0=\operatorname{blockdiag}(\Phi_a)$. A common retention coefficient
$\gamma\in[0,1]$ gives

$$
\Phi_\gamma=(1-\gamma)\Phi_0+\gamma\widetilde\Phi.
$$

The diagonal blocks remain local and the cross blocks are multiplied by
$\gamma$. Both endpoints must be valid covariance matrices. Arbitrary,
independently chosen multipliers for each pair of markets do not have the same
positive-semidefinite guarantee. A singular block needs regularisation or a
carefully specified reduced factor space before inverse roots are used.

## Asynchronous markets and horizons

The published discussion considers multiday aggregation, covariance corrections
and state-space models. Complementary implementation guidance is to specify
the observation interval before estimating any cross-market relationship:

- Use a common decision-time snapshot where reliable executable prices exist.
- Otherwise model lead–lag relationships or use longer aligned return intervals.
- Preserve exchange calendars, timestamps, price freshness and missing-data flags.
- Treat a closed market as missing or stale, rather than as an observed zero.

For stationary vector returns, $\Gamma_k=\operatorname{Cov}(r_t,r_{t-k})$ gives
the exact covariance of an $H$-period sum:

$$
\Sigma_H=H\Gamma_0+
\sum_{k=1}^{H-1}(H-k)(\Gamma_k+\Gamma_k^\top).
$$

Thus $H\Sigma_1$ is appropriate only when the relevant serial and cross-lag
covariances vanish. A long-run HAC covariance is another object; it is not
automatically the covariance for a particular finite holding horizon.
For compounded simple returns this sum formula is a small-return approximation;
for log-return sums it applies directly.

Pairwise estimation on different holiday samples can yield an invalid matrix.
A common complete panel gives a positive-semidefinite sample covariance, but
not necessarily a positive-definite one. Missing-data models, shrinkage and
constrained estimation each introduce assumptions that need validation.

## Practical checks and limits

Verify that portfolios contained in one market retain their local risk; test
mixed-market portfolios separately. Use the same horizon, price convention,
currency and dated universe in all blocks. Evaluate residual lead–lag dependence
and both ordinary and stressed regional correlations.

Combine local returns and [currency factors](currency-rebasing.md) before
reporting investor-currency risk. Country exposure, listing venue and currency
exposure describe different things; one is not a substitute for the others.

## Sources

- Paleologo, [saved draft and evidence boundaries](../../papers/elements-of-quantitative-investing.md), §6.7.
- Paleologo, published §6.6.1, inspected in the [online text mirror](https://dokumen.pub/elements-of-quantitative-investing.html).

Related: [fundamental models](fundamental-factor-models.md),
[statistical models](statistical-factor-models.md), [risk validation](risk-model-validation.md).
