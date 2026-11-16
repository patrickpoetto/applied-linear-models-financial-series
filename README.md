# Applied Linear Models - Financial Time Series

> **University coursework project.** This repository contains the final
> project for a university "Applied Linear Models" course, originally
> written and knitted as an R Markdown / PDF report dated **2023-07-10**.
> The source report lists a single credited author, **Patrick Poetto**
> (this repository's owner). No other co-author names appear anywhere in
> the original document text, so none are listed here — this README
> deliberately avoids asserting group authorship that isn't evidenced in
> the source material. The exact university and course code are not
> stated in the extracted text, so they are not invented here; the report
> itself is simply titled "Applied Linear Models - Final Project."

## What this is

The project builds and diagnoses a multiple linear regression model that
tries to explain the behaviour of the Italian banking sector stock index
using a handful of macro-financial indicators. It walks through the full
applied-statistics workflow taught in the course: exploratory data
analysis, model fitting, subset selection, multicollinearity checks, OLS
diagnostics, model refinement, inference, goodness-of-fit, prediction and a
small simulation exercise.

## Dataset

The ORIGINAL project used six daily financial time series, described as
individually collected from [Investing.com](https://www.investing.com) and
merged into a single data frame, covering **01/01/2019 to 31/12/2022**
(1,019 observations per variable):

| Variable | Description |
|---|---|
| `Banks` (response) | FTSE Italia All-Share Banks index - the stock index of all listed banks on the Italian market |
| `BTP` | Price of the 10Y Italian government bond (BTP), in EUR |
| `VIX` | VSTOXX - the European stock market volatility futures index |
| `Spread` | 10Y BTP-Bund spread, in basis points |
| `EUR` | EUR/USD exchange rate |
| `CDS` | 10Y Italian sovereign credit-default-swap spread, in basis points |

**The raw CSV files behind the original `data_fin1` were never part of the
source material for this migration** - only the knitted PDF report (text +
code + printed output) was available, and that report begins right after
`data_fin1` is already loaded, so the import step itself was never visible.

**This repository now ships a synthetic substitute dataset instead of a
placeholder**, generated directly inside `analysis.Rmd`'s `load-data`
chunk, so the document actually runs end-to-end rather than stopping at
the first chunk. Why synthetic rather than a real substitute: a
like-for-like free, no-API-key daily source covering all six original
series (FTSE Italia Banks index, BTP price, VSTOXX, BTP-Bund spread,
EUR/USD, Italian 10Y sovereign CDS) for 2019-2022 was not reliably
available - CDS and BTP-Bund spread data in particular are normally behind
paid terminals (Bloomberg/Refinitiv) or Investing.com's own (non-free) API,
and mixing a couple of real series (e.g. `^VIX`/`EURUSD=X` via Yahoo
Finance) with synthetic ones for the rest would have made the
collinearity/leverage/outlier analysis internally inconsistent. The
synthetic generator instead:

- builds each of the 5 predictors as an AR(1)-smoothed series (so they are
  persistent/trending like real daily financial data, not i.i.d. noise);
- correlates the 5 predictors' innovations via a Cholesky factor of a
  target correlation matrix chosen to reproduce the same *qualitative*
  collinearity pattern the report describes (BTP strongly correlated with
  Spread and EUR; Spread strongly correlated with CDS; VIX comparatively
  uncorrelated with the rest);
- builds the response (`Banks`) from `log(VIX)` (not `VIX`) plus
  autocorrelated noise whose variance increases with VIX, so that the
  baseline model `Banks ~ .` (linear in VIX) genuinely shows the
  non-linearity / non-constant-variance-in-VIX pattern the report diagnoses
  in Section 7, and so that log-transforming VIX and removing high-leverage
  points/outliers in Section 8 genuinely improves the fit on this data too
  - not only in the preserved historical text.

With `set.seed(2024)`, this lands the live, freshly-computed baseline
adjusted R^2 around **0.70-0.71** and the refined model around **0.71-0.72**
- close to, though not identical to, the original's reported 0.6656 / 0.7233,
with the same qualitative behaviour (significant predictors, VIF < 10,
non-normal residuals, improvement after cleanup + log transform).

**None of the live numbers produced by actually running `analysis.Rmd`
should be read as a restatement of the original project's real-world
findings** - they describe properties of synthetic data designed to behave
like the original, not the original data itself. For that reason, the
historical, exact figures printed in the original PDF (e.g. "Adjusted
R-squared: 0.6656", "89 high-leverage points", "16 outliers") are kept
throughout `analysis.Rmd` as static, non-executed text blocks immediately
after the relevant live code chunk, for historical fidelity. When you
render the document, you will see two sets of numbers for the same
quantity: the live one (computed, varies with the synthetic data and
`set.seed`) and the static one (the original report's printed value, fixed
text). This is intentional, not a rendering error.

## Methodology

1. **Exploratory analysis** - summary statistics, per-variable histograms,
   and a scatterplot matrix to inspect pairwise relationships.
2. **Baseline model** - `lm(Banks ~ .)` on all five predictors (no
   categorical variables, no interactions).
3. **Subset selection** - best-subset selection (`leaps::regsubsets`)
   evaluated via:
   - Leave-one-out cross-validation (LOOCV) of prediction MSE.
   - BIC, Mallow's Cp, adjusted R2, and AIC criteria.
   Both approaches agree that the full 5-predictor model is preferred.
4. **Multicollinearity check** - correlation matrix and variance inflation
   factors (`car::vif`); no VIF exceeds 10, so multicollinearity is ruled
   out despite two correlations above 80% (BTP-Spread, BTP-EUR).
5. **OLS diagnostics** on the baseline model:
   - Residuals vs. fitted / vs. covariates (`car::residualPlots`) show
     non-linearity and non-constant variance, concentrated in VIX.
   - An F-test for variance homogeneity (`var.test`, split at VIX = 50)
     confirms non-constant variance.
   - Q-Q plot and Shapiro-Wilk test show the residuals are not normal
     (fat tails / skewness).
   - Half-normal plot of leverages identifies 89 high-leverage points;
     studentized residuals identify 16 outliers, clustered around
     February 2022 and March 2020 - i.e. the start of the Russia/Ukraine
     war and the COVID-19 crash.
   - Cook's distance is well below 1 for every observation, so there are
     no actually influential points despite the leverage/outlier counts.
6. **Model improvement** - the 89 high-leverage points and 16 outliers are
   removed, and VIX is log-transformed (its distribution and Q-Q plot
   improve markedly under `log()`). The refined model,
   `lm(Banks ~ log(VIX) + Spread + CDS + BTP + EUR)`, reduces the residual
   standard error by ~15% (717.3 -> 606.2) and raises adjusted R2 by ~8.6%
   (0.66 -> 0.72). Residual autocorrelation (`acf`) remains unaddressed and
   is flagged as a known limitation, as is the still-non-normal error
   distribution.
7. **Inference** - per-coefficient t-tests, and a nested-model F-test
   comparing the full refined model against one dropping `Spread` (the
   predictor with the highest VIF), confirming it contributes significant
   explanatory power.
8. **Goodness of fit** - sigma, coefficient covariance matrix, standard
   errors, R2 (0.72) and adjusted R2 (0.72) are computed from first
   principles via `summary(mod2)`.
9. **Prediction** - an out-of-range prediction exercise using the
   macro conditions of Italy's 2011 sovereign-debt crisis, illustrating the
   danger of extrapolating a linear model far outside the observed data.
10. **Simulation** - predictors are resampled from `rnorm()` using their
    observed mean/sd, and the model's predictions on the simulated data
    are compared against predictions on the real data.

## Key findings

*(as reported in the original project, based on its real data; see
"Dataset" above for why the live code in this repository now runs on a
synthetic substitute that reproduces the same qualitative story rather
than these exact figures)*

- A simple linear model with five macro-financial predictors explains
  roughly **67-72%** of the variance in the Italian bank stock index,
  depending on whether the raw or the cleaned/transformed data is used.
- **CDS spread and VIX (log-transformed) are the strongest negative
  drivers**; BTP and Spread are positive drivers; EUR/USD has a sizeable
  negative coefficient.
- No meaningful multicollinearity among predictors (all VIFs < 10).
- The main unresolved statistical issues are **non-normal, fat-tailed
  residuals** and **strong autocorrelation of the errors over time** -
  both are typical, expected complications of financial time-series data
  and are called out explicitly as limitations rather than "fixed" by
  the model.
- The 16 identified outliers line up with two real historical shocks
  (COVID-19 crash, start of the war in Ukraine), which is a reassuring
  sanity check that the statistical outlier detection is finding genuine
  market stress events rather than noise.

## Repository contents

- `analysis.Rmd` - the reconstructed R Markdown source (narrative +
  runnable `{r}` code chunks), reproducing the knitted report, now with a
  working synthetic data loader so the whole document knits end-to-end. A
  few code chunks whose source was not echoed in the original PDF (only
  their printed output was visible) are clearly marked inline as
  *"partially reconstructed from PDF export, original .Rmd source was not
  available"* rather than silently guessed; all of them were verified
  during this review to behave correctly against the rest of the document
  (e.g. `vif(mod1)`, `shapiro.test()`, the `HL`/`OL` leverage/outlier
  objects, `rstudent()`, `cooks.distance()`). OCR-style artifacts in the
  extracted text (e.g. `ˆ` instead of `^`, curly quotes, escaped
  underscores/brackets) have been corrected.
- `LICENSE` - MIT license.
- `.gitignore` - standard R project ignores.

### Bugs fixed during the quality-review pass

1. **`RSS0`/`RSS1` computed as `sum(residuals) ^ 2`** instead of
   `sum(residuals ^ 2)` (Section 11, nested F-test) - a genuine bug in the
   original coursework code (squaring the sum instead of summing the
   squares), preserved faithfully with a comment in the first migration
   pass and now corrected, with a comment explaining why.
2. **`halfnorm(hat, 4, labs = data_fin1$Data, ...)`** referenced a `Data`
   column that `data_fin1` never has (per its own documented 6-numeric-
   column structure) - this would have evaluated to `NULL` and broken the
   leverage plot's point labelling. Fixed to use a standalone `Date` vector
   created alongside `data_fin1`.
3. **The simulation chunk hardcoded `rnorm(921, ...)`** - `921` being the
   exact row count of the *original* project's cleaned dataset. With any
   other dataset this silently desyncs the simulated predictors' length
   from `nrow(data_fin2)`, breaking the final comparison plot. Replaced
   with `nrow(data_fin2)`.

## How to run

Requires R with the following packages (verified against R 4.5.2, `car`
3.1-5, `leaps` 3.2, `faraway` 1.0-9, `effects` 4.2-5, `rmarkdown` 2.32 -
`car`/`faraway`/`effects` pull in `lme4`, which needs `nloptr`, which needs
CMake to compile from source on a fresh machine):

```r
install.packages(c("knitr", "leaps", "car", "faraway", "effects", "rmarkdown"))
```

Then simply:

```r
rmarkdown::render("analysis.Rmd")  # or output_format = "html_document" if no LaTeX/pandoc PDF toolchain is installed
```

**This was actually verified during this review**: `analysis.Rmd` knits
end-to-end to HTML with no errors on a clean R 4.5.2 install (R 4.5.2,
pandoc 3.14), producing the synthetic `data_fin1`/`data_fin2`, fitting
`mod1`/`mod2`/`nestedmod`, running the LOOCV and criterion-based subset
selection, every diagnostic (VIF, Shapiro-Wilk, half-normal leverage plot,
studentized residuals, Cook's distance, `var.test`, `acf`), the nested
F-test, the prediction, and the final simulation - no placeholder step
remains.

## Role / attribution

This repository is published by Patrick Poetto as the credited author of
the source report for this university course deliverable. If this was in
fact completed with unlisted collaborators, please treat this README as
incomplete on that point - it reflects only what the original document
text states.
