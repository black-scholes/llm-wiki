# Currency returns and rebasing

## Intuition

An overseas asset produces a local investment return and a currency return.
The investor's funding currency adds another distinction: earning foreign
interest is not the same as earning a free excess return. A global risk model
must keep these components and their covariance together.

Draft §6.7.2, pp. 161–163, develops funding-adjusted currency returns and a
linear transformation for changing the reporting currency. Exact compounding,
joint covariance and portfolio examples below complement that treatment.

## Complementary guidance: exact return identities

Let $S_{c/b,t}$ be units of reporting currency $b$ per unit of local currency
$c$. A EUR/USD quote of 1.10 means USD 1.10 per EUR. Define the spot return
$g_{c/b}=S_{c/b,t}/S_{c/b,t-1}-1$. For a funded asset with local total return
$r_c$, including distributions,

$$
1+r_b=(1+r_c)(1+g_{c/b}),
\qquad r_b=r_c+g_{c/b}+r_cg_{c/b}.
$$

If the asset gains 5% locally and its currency falls 3% against the investor's
currency, its investor-currency return is $1.05\times0.97-1=1.85\%$.
Adding the two returns gives 2%; that is a first-order approximation.
Log returns add exactly, assuming positive gross returns.

Inverting an FX quote also requires compounding:

$$
g_{b/c}=-\frac{g_{c/b}}{1+g_{c/b}}.
$$

Negating a simple FX return is only an approximation. For a third currency $a$,
gross spot returns multiply:
$(1+g_{c/b})=(1+g_{c/a})(1+g_{a/b})$.

## Funding-adjusted returns from the draft

Let $r_{f,c}$ and $r_{f,b}$ be period-matched local and reporting-currency
cash returns. The simple-return excess PnL per unit of reporting-currency
funding is

$$
r_b^e=(1+r_c)(1+g_{c/b})-(1+r_{f,b}).
$$

Define $r_c^e=r_c-r_{f,c}$. At first order,

$$
r_b^e\simeq r_c^e+q_{c/b},
\qquad q_{c/b}=g_{c/b}+r_{f,c}-r_{f,b}.
$$

The $q$ term is the draft's funding-adjusted currency return. Using raw spot
FX instead loses the interest differential. Its triangle identity
$q_{c/b}\simeq q_{c/a}+q_{a/b}$ is a linearised identity, whereas the gross
spot identity above is exact. Keep total returns, simple excess differences,
and ratios to a cash numeraire distinct.

## Complementary guidance: joint risk and rebasing

Let $C_{ic}=1$ when funded asset $i$ has local currency $c$, and zero otherwise.
To first order, stack nonredundant currency factors with local factors:

$$
r_b^e=Bf+Cq_b+\epsilon,
\qquad
\Omega=\operatorname{Cov}\begin{pmatrix}f\\q_b\end{pmatrix}.
$$

If residuals are uncorrelated with the stacked factors,

$$
\Sigma_b=[B\ C]\Omega[B\ C]^\top+\Psi.
$$

This includes local-factor/FX covariance as well as FX/FX covariance.
Assuming those cross terms are zero is an additional modelling choice.
Use one reporting currency and only the independent currency coordinates;
adding every bilateral pair creates exact or approximate linear dependence.

Changing the reporting currency from $b$ to $a$ gives, to first order,
$q_{c/a}\simeq q_{c/b}-q_{a/b}$. If the chosen factor coordinates transform
as $z_a=Jz_b$, transform their covariance as
$\Omega_a=J\Omega_bJ^\top$. A rebased exposure calculation must use the same
coordinate change. Independently rebuilding pairwise currency covariance
without the triangle constraints can introduce spurious risk directions.

## Cash, shorts and hedges

For a local-currency cash account, use its local cash return in the exact
identity. For a fully funded foreign equity position, the currency translation
exposure is approximately its signed value in reporting currency. Opposite
positions in the same foreign currency can offset that translation exposure;
their equity factor risks remain.

The currency of an exchange listing does not describe the issuer's economic
currency sensitivity. FX forwards, futures, swaps and margined derivatives
need explicit contractual cash flows, financing, collateral and hedge notionals;
the funded-equity identity is not a universal derivative-return formula.
Forward hedging replaces an uncertain spot component with contractual terms
and subsequent hedge PnL. Carry, forward points, basis and transaction costs
still matter.

The draft notes that currency exposure depends on institutional funding
arrangements. Build a currency ledger that includes cash, borrowings,
collateral and hedges, not just security positions. Align equity and FX
valuation times and validate realised investor-currency PnL against that ledger.

## Limits and sources

The linear factor representation drops return interaction terms. Use exact
cash-flow translation for performance accounting, especially during large
market or FX moves. Changing currency changes measured risk; it does not
create economic diversification by itself.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §6.7.2.

Related: [returns](returns.md), [multiple-market models](multi-market-models.md),
[hedging](hedging.md), [performance attribution](performance-attribution.md).
