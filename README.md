# Motor Pricing GLM

A claim frequency and severity pricing model for French motor third-party liability insurance, built with generalised linear models in Python (pandas, statsmodels, matplotlib).

**Status:** steps 1–3 of 9 are complete (data, EDA, frequency model). The severity model is next.

## Data

[freMTPL2](https://www.openml.org/d/41214) is a public French motor portfolio of about 678k policies. It has two tables:

- `freMTPL2freq`: one row per policy, with exposure, claim count and rating factors (driver age, vehicle age, vehicle power, brand, fuel, BonusMalus, area, density, region)
- `freMTPL2sev`: one row per claim, with the claim amount

The notebook downloads both tables from OpenML. If the download fails, it reads `freMTPL2freq.csv` and `freMTPL2sev.csv` from the repo root instead.

## How to run

```bash
pip install -r requirements.txt
jupyter notebook pricing_glm.ipynb
```

The notebook takes about 3 minutes to run from start to finish. The train/test split uses a fixed seed (42), so results can be reproduced.

## Progress

| # | Step | Status |
|---|------|--------|
| 1 | Data | Done |
| 2 | Cleaning and EDA | Done |
| 3 | Frequency model | Done |
| 4 | Severity model | Next |
| 5 | Model selection | Partly done: train/test split, AIC, test-set Poisson deviance |
| 6 | Pure premium | To do |
| 7 | Rate relativities | Partly done: exponentiated frequency coefficients |
| 8 | Validation | To do |
| 9 | Writeup | In progress (this README) |

### Done so far

**1. Data**
- Loaded both tables from OpenML.
- Fixed OpenML quirks: `IDpol` loads as a float in the freq table, and `VehGas` values come with literal quotes.
- Summed claim amounts to one row per policy and left-joined them onto the policy table.
- Capped `ClaimNb` at 4. This affects 16 policies with 5–16 claims, which are likely data errors.
- 9,116 policies report claims but have no amounts in `freMTPL2sev`. This is a known issue with the dataset. It does not affect the frequency model, but the severity model will need to handle it.

**2. Cleaning and EDA**
- Capped `Exposure` at 1 year (1,224 policies were above it) and `BonusMalus` at 150.
- Grouped driver age, vehicle age and BonusMalus into bands.
- Made an 80/20 train/test split before EDA, so the test set is never used for modelling choices.
- Claim counts: 95% of policies have no claims.
- Severity: the median claim is about EUR 1,170. The largest 1% of claims make up about 39% of total claim cost, and the largest single claim is about EUR 4.1M.
- One-way claim frequency (claims per exposure year) is highest for drivers aged 18–20, for vehicles aged 0, and for high BonusMalus levels.

**3. Frequency model**
- Fitted a Poisson GLM with a log link and `log(Exposure)` as offset. The predictors are the three banded factors plus Area, VehGas, VehBrand, Region, VehPower (categorical) and log(Density).
- The base level of each banded factor is its highest-exposure band.
- Overdispersion: Pearson χ²/df = 2.24, so the Poisson standard errors are too small.
- Also fitted a negative binomial (NB2) model:

| | Poisson | Negative binomial |
|---|---|---|
| AIC | 224,791 | 224,331 |
| Test mean Poisson deviance | 0.3148 | 0.3149 |
| Test predicted / actual claims | 7,170 / 7,239 | 7,262 / 7,239 |

- The negative binomial fits better on AIC (alpha ≈ 0.8). Relativities and test-set accuracy are almost the same for both models, so overdispersion mainly affects standard errors and confidence intervals.
- BonusMalus is the strongest rating factor: the 126–150 band has about 6.9 times the frequency of BonusMalus 50.
- Drivers aged 18–20 have about 2.4 times the base frequency one-way, but a relativity of only about 1.05 in the GLM. New drivers start on a high BonusMalus, so the BonusMalus bands already capture most of the young-driver effect.

## Next steps

4. **Severity model.** Fit a Gamma GLM (log link) with average claim amount as the response, weighted by claim count, and compare it with a lognormal model. Also decide how to handle large claims (for example, capping them and spreading the excess loading across all policies) and the policies that have claims but no amounts.
5. **Model selection.** Compare candidate models on AIC, deviance and test-set Poisson deviance. Drop weak variables (Area and parts of VehBrand look weak so far) and merge adjacent bands whose relativities are similar. Use the negative binomial standard errors for significance.
6. **Pure premium.** Calculate expected frequency × expected severity for each policy, and compare total predicted losses with actual losses on the test set.
7. **Rate relativities.** Build a relativities table against a base class, and plot relativities by age band and vehicle power.
8. **Validation.** Produce a lift chart, a Gini coefficient, and actual vs predicted by decile.
9. **Writeup.** Complete this README with assumptions, limitations and key findings, and add a one-page summary.

## Repository contents

- `pricing_glm.ipynb`: the analysis notebook, saved with outputs
- `requirements.txt`: Python dependencies
