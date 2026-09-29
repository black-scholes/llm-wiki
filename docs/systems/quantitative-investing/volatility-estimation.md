# Estimating time-varying volatility

## Intuition

Risk changes faster than a long-window sample variance can follow, while a
very short window reacts strongly to noise. The draft compares conditional
variance models, realised measures and latent-state filtering to manage this
trade-off (chapter 2).

## Mechanics from the draft

With mean-adjusted innovation $\epsilon_t$, a GARCH(1,1) model is

$$
h_t=\omega+\alpha\epsilon_{t-1}^2+\beta h_{t-1}.
$$

For the standard covariance-stationary construction, $\omega>0$,
$\alpha,\beta\ge0$ and $\alpha+\beta<1$ give unconditional variance
$\omega/(1-\alpha-\beta)$. Stronger moment assumptions require stronger
conditions; existence of a variance does not imply existence of a fourth moment.

EWMA uses $h_t=\lambda h_{t-1}+(1-\lambda)\epsilon_{t-1}^2$.
It gives geometrically decaying weights and can be viewed as the zero-intercept,
unit-persistence boundary of GARCH. That boundary has no finite stationary
long-run variance determined by an intercept. The draft also derives EWMA
through a latent-state estimation argument (§2.3.1).

Realised variance sums squared intraday returns. Finer sampling creates more
observations, but bid–ask effects and observation noise eventually damage the
estimate (§2.2.4). Combining realised measures with a conditional model can
use intraday information without assuming the measure is noise-free.
The Kalman-filter appendix (§2.4) supplies prediction and updating machinery
for linear latent-state models; stochastic-volatility applications may need
transformations or approximations.

## Complementary guidance

Fit a forecast at the information time preceding the return it predicts.
Separate intraday and overnight intervals, account for market closures, and
state whether variance is per day or for a multi-day holding period.
An EWMA weight half-life is $\log(1/2)/\log\lambda$; an observation's amplitude
weight and its covariance weight are not always the same parameterisation.

Compare forecasting losses and portfolio outcomes, rather than selecting the
smoothest chart. For factor models, it can be useful to adapt marginal
volatility faster than the dependence structure, but that is an empirical
design choice rather than a universal timescale separation.

## Limits and sources

Normal innovations are often an inadequate tail model. Forecast variance is
not a bound on tomorrow's loss. A state estimate depends on its observation
noise and process dynamics; smoothing with future observations cannot be
used as an ex-ante trading forecast.

- Paleologo, [saved draft](../../papers/elements-of-quantitative-investing.md), §§2.2–2.4, pp. 50–70.

Related: [risk-model validation](risk-model-validation.md),
[statistical models](statistical-factor-models.md), [dynamic allocation](dynamic-risk-allocation.md).
