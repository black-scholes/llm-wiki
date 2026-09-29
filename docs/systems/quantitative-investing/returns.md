# Measuring investment returns

## Intuition

A return is a change in economic value over a specified interval. Its
definition depends on distributions, financing, currency and the capital
against which PnL is measured. Small inconsistencies here can invalidate an
otherwise careful model. Draft §2.1 develops these distinctions.

## Mechanics from the draft

For an asset bought at $P_{t-1}$ and valued at $P_t$, with a cash distribution
$D_t$ during the interval, its simple total return is

$$
r_t=\frac{P_t+D_t-P_{t-1}}{P_{t-1}}.
$$

An adjusted total-return series incorporates distributions and corporate
actions according to its provider's convention. Avoid adding a dividend again
to a series that already includes it. Simple excess return subtracts a
period-matched cash return: $r_t^e=r_t-r_{f,t}$.

For positive gross returns, $\ell_t=\log(1+r_t)$ compounds additively through
time. Simple returns compound as $\prod_t(1+r_t)-1$. Portfolio returns are
linear combinations of asset simple returns when weights represent beginning
capital fractions and intervening cash flows are handled consistently. Log
returns do not have that portfolio linearity.

The draft also examines midquotes, transaction prices, noisy observations and
stale prices (§2.1.4). Its stylised facts (§2.1.5) include heavy tails,
volatility clustering and leverage effects. These are empirical patterns, not
identities that every asset and sampling interval must exhibit.

## Complementary guidance

State the denominator for a long–short portfolio: NAV, gross exposure, net
exposure and allocated capital answer different questions. Dividing by a
near-zero net exposure produces unstable ratios even when dollar PnL is
well-defined. Use dated signed notionals for PnL and an explicit capital series
for performance returns.

Price corrections, splits, dividends and delisting proceeds need economic
checks. A real large loss should not disappear as a data-cleaning outlier.
Record missing prices and market closures rather than inserting zero returns.
Use executable prices for trading simulations and an independently specified
valuation convention for risk and accounting.

## Limits and sources

The approximation $\log(1+r)\simeq r$ loses compounding effects during large
moves. Log returns require positive gross wealth or prices and are unsuitable
for some instrument price series. Currency conversion belongs in the return
definition, before a global model is fitted.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §2.1, pp. 42–49.

Related: [currency rebasing](currency-rebasing.md),
[volatility estimation](volatility-estimation.md), [performance metrics](performance-and-capacity.md).
