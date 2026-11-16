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

Six daily financial time series, individually collected from
[Investing.com](https://www.investing.com) and merged into a single data
frame, covering **01/01/2019 to 31/12/2022** (1,019 observations per
variable):

| Variable | Description |
|---|---|
| `Banks` (response) | FTSE Italia All-Share Banks index - the stock index of all listed banks on the Italian market |
| `BTP` | Price of the 10Y Italian government bond (BTP), in EUR |
| `VIX` | VSTOXX - the European stock market volatility futures index |
| `Spread` | 10Y BTP-Bund spread, in basis points |
| `EUR` | EUR/USD exchange rate |
| `CDS` | 10Y Italian sovereign credit-default-swap spread, in basis points |

**Note:** the raw CSV/data files behind `data_fin1` are not included in
this repository - only the knitted report (text + code + output) was
available when this project was migrated, and the report begins after the
data was already loaded. `analysis.Rmd` documents the expected shape of
`data_fin1` in a placeholder chunk so the rest of the analysis can be
reproduced once the same six series are collected and merged the same way.

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
  runnable `{r}` code chunks), reproducing the knitted report. A few code
  chunks whose source was not echoed in the original PDF (only their
  printed output was visible) are clearly marked inline as
  *"partially reconstructed from PDF export, original .Rmd source was not
  available"* rather than silently guessed. OCR-style artifacts in the
  extracted text (e.g. `ˆ` instead of `^`, curly quotes, escaped
  underscores/brackets) have been corrected.
- `LICENSE` - MIT license.
- `.gitignore` - standard R project ignores.

## How to run

Requires R (tested conceptually against the packages below; versions are
not pinned since the original `.Rmd`/session info was not part of the
extracted source):

```r
install.packages(c("knitr", "leaps", "car", "faraway", "effects", "rmarkdown"))
```

Then, once the six source series are collected/merged into a `data_fin1`
data frame matching the structure documented in `analysis.Rmd` (see the
`load-data` chunk):

```r
rmarkdown::render("analysis.Rmd")
```

## Role / attribution

This repository is published by Patrick Poetto as the credited author of
the source report for this university course deliverable. If this was in
fact completed with unlisted collaborators, please treat this README as
incomplete on that point - it reflects only what the original document
text states.
