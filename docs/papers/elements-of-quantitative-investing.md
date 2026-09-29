# The Elements of Quantitative Investing: source notes

Giuseppe A. Paleologo connects return modelling, risk estimation, portfolio
construction, trading costs and performance attribution as parts of one
investment process. The wiki pages below extract those connections and add
standard theory and implementation guidance where the draft is brief.

## Source and edition

The primary source is the complete user-provided
[2024-07-10 draft PDF](../../reference/books/elements-of-quantitative-investing/eqi.pdf),
saved unchanged on 2026-09-29. It has 376 PDF pages. Printed page numbers
match one-based PDF page numbers in this copy. Bibliography starts on p. 349
and the index on p. 371. The
[reference directory](../../reference/books/elements-of-quantitative-investing/README.md)
and [manifest](../../reference/books/elements-of-quantitative-investing/manifest.json)
record file identity and checksums.

Wiley's 2025 first edition has print ISBN 9781394265459. Its
[official contents](https://catalogimages.wiley.com/images/db/pdf/9781394265459.toc.pdf)
and [chapter 1 excerpt](https://catalogimages.wiley.com/images/db/pdf/9781394265459.excerpt.pdf)
are available online and saved beside the draft. The
[author's resource index](https://linktr.ee/paleologo) links to the published
book and talks. The author's original public Dropbox draft folder was empty
when checked on 2026-09-29.

## Coverage map

Section and page citations on concept pages refer to the saved **2024-07-10
draft**, unless explicitly labelled otherwise. This table covers its substantive
chapters and technical appendix; it is a navigation map rather than a
transcription of every exercise or proof.

| Draft chapter and pages | Concept pages | Corresponding published chapter |
| --- | --- | --- |
| Preface, pp. 11–19; 1, pp. 23–40 | [Investment process](../systems/quantitative-investing/investment-process.md) | Introduction; 1 |
| 2, pp. 41–72 | [Returns](../systems/quantitative-investing/returns.md); [Volatility estimation](../systems/quantitative-investing/volatility-estimation.md) | 2 |
| 3, pp. 73–104 | [Factor models](../systems/quantitative-investing/factor-models.md); [Linear algebra](../systems/quantitative-investing/linear-algebra.md) | 4 |
| 4, pp. 105–126 | [Backtesting and multiple testing](../systems/quantitative-investing/backtesting.md) | 8 |
| 5, pp. 127–138 | [Risk-model validation](../systems/quantitative-investing/risk-model-validation.md) | 5 |
| 6, pp. 139–166 | [Fundamental models](../systems/quantitative-investing/fundamental-factor-models.md); [Many-characteristic models](../systems/quantitative-investing/many-predictor-factor-models.md); [Shrinkage](../systems/quantitative-investing/shrinkage.md); [Multiple markets](../systems/quantitative-investing/multi-market-models.md); [Currency rebasing](../systems/quantitative-investing/currency-rebasing.md) | 6; speculative §6.6 is not separately listed |
| 7, pp. 167–208 | [Statistical models](../systems/quantitative-investing/statistical-factor-models.md); [Shrinkage](../systems/quantitative-investing/shrinkage.md) | 7 |
| 8, pp. 211–238 | [Portfolio optimisation](../systems/quantitative-investing/portfolio-optimisation.md); [Performance and capacity](../systems/quantitative-investing/performance-and-capacity.md); [Optimisation mechanics](../systems/quantitative-investing/optimisation-mechanics.md) | 9; performance moved to 3 |
| 9, pp. 239–260 | [Constraints and estimation error](../systems/quantitative-investing/constraints-and-estimation-error.md); [Performance and capacity](../systems/quantitative-investing/performance-and-capacity.md) | 10 |
| 10, pp. 261–278 | [Market impact and multiperiod optimisation](../systems/quantitative-investing/market-impact.md) | 11 |
| 11, pp. 279–290 | [Hedging](../systems/quantitative-investing/hedging.md); [Shrinkage](../systems/quantitative-investing/shrinkage.md) | 12 |
| 12, pp. 293–310 | [Dynamic risk allocation](../systems/quantitative-investing/dynamic-risk-allocation.md) | 13 |
| 13, pp. 311–334 | [Performance attribution](../systems/quantitative-investing/performance-attribution.md) | 14 |
| 14, pp. 335–348 | [Linear algebra](../systems/quantitative-investing/linear-algebra.md); [Optimisation mechanics](../systems/quantitative-investing/optimisation-mechanics.md); [Backtesting](../systems/quantitative-investing/backtesting.md); [Statistical models](../systems/quantitative-investing/statistical-factor-models.md) | Material redistributed among appendices |

[Signal and portfolio aggregation](../systems/quantitative-investing/signal-aggregation.md)
is a complementary page: the draft discusses signal mixing in §10.3.6,
while the published contents identifies a separate aggregation section, §9.6.
The published coda, chapter 15, is not present in the supplied draft and is not
claimed as extracted.

## Evidence boundaries and complements

- The draft contains unfinished material, notably §6.7.1 on model linkage,
  §6.4.5 on specific-variance shrinkage, and parts of the appendix. A heading
  in its contents is not evidence of a completed derivation.
- The draft explicitly marks its many-predictor factor sketch (§6.6) as
  speculative. Its exploratory status is preserved in the corresponding page.
- Sections labelled **Complementary guidance** are standard theory, independent
  derivations, or practical recommendations added to explain or extend the
  draft. Worked examples are original illustrations, not empirical results.
- The multiple-market page supplements the draft with the readable published
  §6.6.1 in an [online text mirror](https://dokumen.pub/elements-of-quantitative-investing.html).
  That mirror has missing sections and damaged equation layout. Only the
  inspected linkage discussion is used; it is not a verified complete copy.
- Draft equations and procedures are interpreted with stated assumptions.
  Apparent typographical errors are not carried into implementation advice.
  No half-life, shrinkage intensity, factor count or regional correlation is
  recommended as universally optimal.

## Primary references for complementary material

- Tipping and Bishop (1999),
  [Probabilistic Principal Component Analysis](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/bishop-ppca-jrss.pdf).
- Ledoit and Wolf (2003),
  [Honey, I Shrunk the Sample Covariance Matrix](https://ledoit.net/honey.pdf).
- Ledoit and Wolf (2012),
  [Nonlinear shrinkage estimation of large-dimensional covariance matrices](https://arxiv.org/abs/1207.5322).
- Boyd et al. (2017),
  [Multi-Period Trading via Convex Optimization](https://arxiv.org/abs/1705.00109).

The source PDF's bibliography provides the starting references for the other
concepts. These pages are educational methods, not a trading system's live
contract or a validated investment strategy.
