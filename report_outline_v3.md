# Report Outline: Forecasting Monthly Vehicle Traffic on the Øresund Bridge

**Course:** Predictive Analytics — CBS F26
**Data:** Monthly car crossings, Øresund Bridge, July 2000 – March 2026 (T = 309, no gaps)
**Methods:** ARIMA(2,1,2)(0,1,1)[12] with COVID dummies | ETS(A,Ad,A)
**Forecast horizon:** April 2026 – March 2028 (24 months)
**Target length:** ~15 pages (hard cap — professor will not read beyond 15)

---

## 1. Introduction (~0.5 page)

**What:** Monthly vehicle crossings on the Øresund Bridge — the only fixed link between Sweden and Denmark — from July 2000 through March 2026 (309 consecutive monthly observations, no gaps). Measured in absolute vehicle counts.

**Why:** Accurate medium-term traffic forecasts are directly relevant to the bridge operator (revenue planning, capacity management), transport policy makers, and regional economic analysis. The series is non-trivial: it contains a long-run trend, strong and stable seasonality, and a severe COVID-19 disruption — making it a challenging but tractable forecasting problem.

**How:** We follow the full course forecasting routine — Define → Identify → Estimate → Diagnostic checks → Forecast → Evaluate — and apply two complementary model classes: ARIMA (extended with COVID impulse dummies) and ETS (exponential smoothing with damped trend). Both point forecasts and prediction intervals are reported for the 24-month horizon April 2026 – March 2028.

**Scope sentence:** Section 2 describes the data; Section 3 presents the methodology; Section 4 presents and discusses results; Section 5 concludes.

---

## 2. Data (~2 pages)

### 2.1 Source and Description

- Source: Øresund Bridge operator's publicly available traffic statistics (`Trafficstat(EN).csv`).
- Variable: monthly car crossings, July 2000 – March 2026, T = 309 observations, no missing values.
- Unit: absolute vehicle count per month.
- **Figure to include:** Time-series plot of raw Cars series (autoplot). Describe the three dominant visual features immediately (trend, seasonality, COVID drop).

### 2.2 Main Characteristics

Three structural features drive every modelling decision:

1. **Long upward trend:** Traffic grew steadily for two decades. The subseries plot shows growth visibly slowing post-2018 — summer peaks plateau while winter months continue rising modestly. This motivates the damped trend specification in ETS.
2. **Strong, stable annual seasonal cycle:** An inverted-U shape peaking in July–August each year, consistent across all 25 years. The seasonal subseries plot confirms near-identical amplitudes year to year. The seasonal pattern is treated as deterministic and stable.
3. **COVID-19 disruption (March 2020 – July 2021):** Traffic collapsed by up to 70% during border closures. April 2020 (144,759 cars) is the single lowest monthly count — a ~70% drop from seasonal expectation. The series recovered fully by mid-2022.

- **Figures to include:** seasonal plot (gg_season), subseries plot (gg_subseries). Reference both explicitly when describing the three features.

### 2.3 Variance Stabilisation and Transformation

- The seasonal amplitude grows with the level — higher summer peaks as traffic grows — which motivates a functional transformation before modelling.
- Guerrero's method selects **λ = 0.7899**. This is close to but meaningfully different from 1 (no transformation) and from 0 (log). The goal is residuals with stable variance and a distribution closer to normal — required for valid prediction intervals.
- We apply Box-Cox(λ = 0.79) throughout all subsequent analysis.
- **Figure to include:** Multi-panel differencing plot showing: Cars → Box-Cox Cars → Annual difference → Doubly-differenced. This simultaneously motivates d=1, D=1.

### 2.4 Stationarity

**Tests on the raw Box-Cox transformed series:**

| Test | Statistic | p-value | Conclusion |
|------|-----------|---------|------------|
| KPSS (H₀: stationary) | 1.6096 | 0.01 | Reject → non-stationary |
| ADF (H₀: unit root) | −2.9251 | 0.186 | Fail to reject → non-stationary |

The two tests now agree on the raw series: the series is non-stationary. KPSS rejects stationarity due to the strong upward trend; ADF fails to reject the unit root. `unitroot_ndiffs = 1` and `unitroot_nsdiffs = 1` confirm d = 1, D = 1.

**After applying d = 1, D = 1:**

| Test | Statistic | p-value | Conclusion |
|------|-----------|---------|------------|
| KPSS | 0.02957 | 0.10 | Fail to reject → stationary |
| ADF | −6.9127 | 0.01 | Reject unit root → stationary |

Both tests now agree: d = 1, D = 1 achieves stationarity. The trend is treated as **stochastic** (unit root), consistent with standard practice for economic time series of this length. A stochastic trend allows the level to drift, which better captures long-range forecast uncertainty than a fixed deterministic trend would.

### 2.5 Seasonality

- The seasonal plot and subseries plot confirm a consistent, strong 12-month cycle across all 25 years. No evidence of changing seasonal shape.
- `unitroot_nsdiffs = 1` confirms one seasonal difference is needed, consistent with the dominant spike at lag 12 in the ACF of the transformed series.
- The seasonal pattern is treated as **deterministic and stable** — the near-zero seasonal smoothing parameter (γ ≈ 0.0001) estimated in ETS(A,Ad,A) later confirms that seasonal indices do not update from year to year. This is consistent evidence from two independent methods.

### 2.6 COVID-19: Structural Break Analysis and Dummy Design

**Identifying the outlier period:**
- A robust STL decomposition (`robust = TRUE`, `trend(window = 13)`) was applied to the raw Cars series. Robust STL downweights extreme observations during trend/seasonal estimation, pushing the COVID signal cleanly into the remainder. Z-scores on the remainder flagged months outside ±2σ.
- The most severely affected months (most negative z-scores): April 2020 (z = −8.07), May 2020 (z = −6.14), June 2020 (z = −5.27), March 2020 (z = −5.47). The contamination window runs March 2020 – July 2021 (17 months).
- Note: several positive outliers also appear (e.g., July 2023 z = +3.05, July 2025 z = +2.21), reflecting genuine post-COVID recovery demand — not controlled for.
- **Figure to include:** Z-score plot of STL remainder with ±2σ dashed lines. This motivates both the window choice and the impulse dummy selection visually.

**Is the COVID disruption a permanent structural break?**
- A QLR test (Andrews 1993 supremum-F) on the doubly-differenced Box-Cox series scans all candidate breakpoints within the trimmed window (15% from each end). The F-statistic peaks in the COVID period — the intuitively correct location — but reaches only **sup.F = 2.369, p = 0.697**, far below the 5% critical value of ~8.68. Structural stability is strongly confirmed at every candidate date.
- As a supplementary cross-check, a single-point Chow test at August 2021 (the post-COVID recovery point) returns **F = 0.059, p = 0.808**, consistent with the QLR result.
- COVID was a temporary, externally-driven shock that did not alter the underlying DGP. The full 25-year history is used for estimation.

**Dummy variable design for ARIMA:**
- `covid`: period dummy = 1 for March 2020 – July 2021, 0 otherwise (17 months).
- `imp_2020_apr`, `imp_2020_may`, `imp_2020_mar`: impulse dummies for the three most extreme individual months identified by the z-score analysis.
- **Logic:** The period dummy absorbs the average disruption across all COVID months. The three impulse dummies capture the additional severity of the hardest lockdown months beyond the average. Months in the period but outside the impulse list receive only β_covid; March, April, and May 2020 receive β_covid + β_month.
- For the out-of-sample forecast, all dummy values are set to zero.
- ETS cannot accommodate external regressors — this is one reason ARIMA is the primary model.

---

## 3. Methodology (~2.5 pages)

*Note: Do not explain what an ARIMA or ETS model is in general. Explain WHY each specification is the right one for this series.*

### 3.1 Modelling Strategy

We apply two complementary model classes following the course forecasting routine. ARIMA explicitly models the autocorrelation structure of the differenced series and accommodates external COVID regressors. ETS handles trend and seasonality through adaptive smoothing without requiring pre-differencing or dummy variables. Together they provide a full picture: ARIMA is the recommended primary model; ETS is the comparison model.

Model selection within each class uses two criteria: (1) AIC for within-class comparison during estimation, (2) expanding-window cross-validated RMSE as the primary selection criterion. AIC cannot compare ARIMA against ETS — the two classes use different likelihood normalizations.

### 3.2 Model 1: SARIMA with COVID Regressors

**What:** ARIMA(p,1,q)(0,1,1)[12] on the Box-Cox transformed series, with COVID dummies via `xreg`. Three candidate specifications estimated:
- ARIMA(0,1,0)(0,1,1)[12] — baseline implied directly by the seasonal ACF/PACF
- ARIMA(2,1,2)(0,1,1)[12] — manually identified from the non-seasonal correlogram
- ARIMA(2,1,3)(0,1,1)[12] — extended MA order candidate
- auto.arima (stepwise=FALSE, approximation=FALSE) — as an independent data-driven check

**Why d=1, D=1:** Stationarity tests confirm this (Section 2.4). After differencing, the ACF and PACF of the doubly-differenced series are inspected.

**Why PDQ = (0,1,1) — the seasonal part:**
- A clear negative spike at lag 12 in the ACF of the doubly-differenced series, with decay at lags 24, 36 — the textbook signature of a seasonal MA(1) process.
- D = 1 was already required for stationarity.

**Why pdq = (2,1,2) — the non-seasonal part:**
- After controlling for the seasonal structure, the non-seasonal ACF/PACF shows an oscillating, decaying pattern across multiple lags — the signature of a mixed ARMA process, not a clean AR or MA cutoff.
- AR(2) and MA(2) terms are the parsimonious description of this oscillation.
- `auto.arima` independently selects the same ARIMA(2,1,2)(0,1,1)[12] specification — the ACF/PACF identification and the information-criterion search converge by different routes.
- ARIMA(2,1,3) was also tested as a more flexible alternative but is rejected by CV (see Section 4.3).
- **Figure to include:** ACF and PACF plots of doubly-differenced Box-Cox series (lag_max = 48). Annotate the seasonal spike at lag 12 and the oscillating non-seasonal pattern.

**Why COVID dummies are necessary:**
- Without explicit COVID control, the 17 months of severe disruption would distort the estimated autocorrelation structure.

**Assumptions:**
(a) Error dynamics are independent of the COVID dummy regressors (exogeneity — plausible since COVID was an exogenous public health event).
(b) The DGP is unchanged post-COVID (QLR test, sup.F = 2.369, p = 0.697).
(c) The Box-Cox transformed, differenced series is approximately normally distributed — required for valid prediction intervals.

### 3.3 Model 2: ETS(A,Ad,A)

**What:** ETS(A,Ad,A) — additive errors, additive damped trend, additive seasonality — estimated directly on the Box-Cox transformed series. Five ETS variants were evaluated; ETS(A,Ad,A) is selected (see Section 4.4 for comparison).

**Why additive errors (A):** Box-Cox transformation has stabilised variance, so additive errors are appropriate.

**Why additive damped trend (Ad):**
- The subseries plot shows visibly slowing growth — summer peaks have plateaued. This motivates *damped* rather than linear trend.
- `ets_auto` selects ETS(A,**N**,A): no trend. This is rejected: the series has a clear upward trajectory. A no-trend specification would immediately produce flat forecasts inconsistent with the historical pattern.
- ETS(A,Ad,A) captures slowing growth via the damping parameter φ = 0.98.

**Why additive seasonality (A):** After Box-Cox transformation, seasonal fluctuations are approximately constant in magnitude.

**Estimated smoothing parameters:**
- α = 0.9999 (level): the level essentially resets to each new observation. **Note for paper: discuss what α ≈ 1 means in practice — the model assigns near-zero weight to all past observations beyond the most recent one, making the level effectively a random walk. This explains why STL's COVID downweighting adds nothing: by 2026, exponential discounting has already reduced the COVID observations to negligible influence regardless. This parameter result should be explicitly interpreted, not just reported.**
- β = 0.0059 (trend slope): slope updates slowly — consistent with gradual, stable growth.
- γ = 0.0001 (seasonality): seasonal indices are effectively fixed — consistent with the visually stable seasonal pattern.
- φ = 0.98 (damping): slow damping — trend persists but gradually attenuates over the forecast horizon.

**Why ETS is the comparison, not the primary model:**
All five ETS variants fail Ljung-Box. ETS has no mechanism to model the autocorrelation dynamics of the error term — structurally, this is what AR and MA terms do. Residual autocorrelation means prediction intervals are likely too narrow.

### 3.4 Forecast Evaluation Strategy

**Train/test split:** Training: July 2000 – March 2024. Test (held out): April 2024 – March 2026 (24 months).

**Cross-validation:** Expanding-window CV with initial window of 120 months (.init = 120) and step of 12 months (.step = 12). CV RMSE computed at h = 6, 12, and 24 months. CV is the primary model selection criterion.

**Accuracy metrics reported:** RMSE (primary), MAPE (interpretable percentage), MASE (scaled by seasonal naive baseline — MASE < 1 means the model beats naive seasonal).

**Prediction intervals:** 80% intervals are reported throughout, following the convention in Hyndman & Athanasopoulos (2021, *fpp3*) — the course textbook. At a 24-month horizon on this series, 95% intervals span several hundred thousand vehicles and are practically uninformative; 80% intervals remain interpretable while still quantifying forecast uncertainty.

---

## 4. Results and Discussion (~4–5 pages)

### 4.1 Model Estimation Results

**ARIMA(2,1,2)(0,1,1)[12] coefficient table:**

| Coefficient | Estimate | Std. Error |
|---|---|---|
| ar1 | 1.4459 | 0.0592 |
| ar2 | −0.6691 | 0.0549 |
| ma1 | −1.8300 | 0.0332 |
| ma2 | 0.9468 | 0.0299 |
| sma1 | −0.6923 | 0.0520 |
| covid | −12,437.79 | 805.99 |
| imp_2020_apr | −11,833.52 | 1,326.68 |
| imp_2020_may | −4,417.17 | 1,239.82 |
| imp_2020_mar | −3,121.68 | 1,256.70 |

σ² = 2,240,333; log-likelihood = −2583.94; **AIC = 5187.87**

*Interpret the COVID coefficients:* The `covid` coefficient represents the average reduction during the COVID period. The `imp_2020_apr` impulse captures the additional severity of April 2020 on top of the period effect. All COVID coefficients are highly significant.

**ETS(A,Ad,A) parameters:** α = 0.9999, β = 0.0059, γ = 0.0001, φ = 0.9800; σ² = 4,220,883; **AIC = 6504.08** (not comparable to ARIMA AIC — different scales).

**AIC comparison within ARIMA class:**

| Model | AIC | Notes |
|---|---|---|
| ARIMA(2,1,2)(0,1,1)[12] | 5187.87 | **Selected; auto.arima independently confirms** |
| ARIMA(0,1,0)(0,1,1)[12] | 5240.62 | Simpler baseline |

`auto.arima` (stepwise=FALSE, approximation=FALSE) independently selects ARIMA(2,1,2)(0,1,1)[12] — convergence of manual ACF/PACF identification and information-criterion search is strong validation.

### 4.2 Diagnostic Checks

**ARIMA(2,1,2)(0,1,1)[12]:**
- Ljung-Box (lag = 24, dof = 5): **Q = 14.23, p = 0.770** — fail to reject white-noise residuals. The AR(2)/MA(2) terms have successfully absorbed the oscillating non-seasonal autocorrelation.
- **Figures to include:** residual time plot, ACF of residuals (all bars within bounds), histogram (approximately normal).
- The baseline ARIMA(0,1,0)(0,1,1)[12] fails: Q = 38.53, p = 0.022 (dof = 1). The AR(2)/MA(2) terms are necessary and justified.
- The auto model is run with dof = 6 in the code, producing Q = 14.23, p = 0.714. However, the Q-statistic is **bit-for-bit identical** to arima212011 (14.22552), proving the residuals — and therefore the fitted model — are identical. The auto model's `report()` output shows no constant/drift term, confirming it has the same 5 free ARMA parameters (ar1, ar2, ma1, ma2, sma1). dof = 6 was likely set anticipating auto might include a mean term; it did not. The correct dof is 5 for both, giving p = 0.770 for both. **Do not report the auto LB result separately** — note only that auto confirms the same specification, evidenced by the identical Q-statistic.

**ETS(A,Ad,A):**
- Ljung-Box (lag = 24, dof = 4): **Q = 50.89, p = 0.000165** — significant residual autocorrelation. ETS has no mechanism to model error dynamics.
- All five ETS variants fail Ljung-Box:

| Model | Q-stat | p-value |
|---|---|---|
| ets_AAdA | 50.89 | 0.000165 |
| ets_auto | 55.56 | 9.89×10⁻⁵ |
| stl_auto | 68.06 | 2.41×10⁻⁶ |
| stl_AAN | 67.99 | 1.36×10⁻⁶ |
| stl_AAdN | 68.04 | 7.21×10⁻⁷ |

- **Figures to include:** ETS residual plot and ACF showing the autocorrelation structure ETS leaves unmodelled.

**Implication for prediction intervals:** ARIMA's prediction intervals rest on the white-noise residual assumption — met (p = 0.770). ETS's prediction intervals assume uncorrelated errors — violated (p = 0.000165). ETS prediction intervals are unreliable and likely too narrow.

### 4.3 Forecast Accuracy — Cross-Validation and Test Window

**Expanding-window CV RMSE (h = 6, 12, 24) — ARIMA models:**

| Model | h = 6 | h = 12 | h = 24 |
|---|---|---|---|
| ARIMA(2,1,2)(0,1,1)[12] | **34,859** | **36,759** | **69,618** |
| ARIMA(2,1,3)(0,1,1)[12] | 45,024 | 40,116 | 72,417 |
| ARIMA(0,1,0)(0,1,1)[12] | 96,836 | 90,712 | 106,580 |

ARIMA(2,1,2)(0,1,1)[12] is best at every horizon. ARIMA(2,1,3) is worse than (2,1,2) at h=6 and h=24, confirming the additional MA term adds no forecast value. ARIMA(0,1,0) is decisively rejected.

**Expanding-window CV RMSE (h = 6, 12, 24) — ETS models:**

| Model | h = 6 | h = 12 | h = 24 |
|---|---|---|---|
| ets_AAdA | **36,873** | **109,407** | **162,367** |
| ets_auto | 37,445 | 111,796 | 163,212 |
| stl_AAdN | 38,467 | 109,607 | 162,436 |
| stl_auto | 41,172 | 110,978 | 163,926 |
| stl_AAN | 40,715 | 113,563 | 173,912 |

ETS(A,Ad,A) is best or tied at every horizon across all five ETS variants and is selected as the ETS representative.

**ARIMA vs ETS (best of each class):**

| Model | h = 6 | h = 12 | h = 24 |
|---|---|---|---|
| ARIMA(2,1,2)(0,1,1)[12] | **34,859** | **36,759** | **69,618** |
| ETS(A,Ad,A) | 36,873 | 109,407 | 162,367 |

At h=24, ARIMA CV RMSE is 69,618 vs ETS's 162,367 — less than half.

**Held-out test window accuracy (April 2024 – March 2026):**

| Model | RMSE | MAE | MAPE | MASE |
|---|---|---|---|---|
| ARIMA(2,1,2)(0,1,1)[12] | **22,988** | **16,869** | **3.03%** | **0.360** |
| ETS(A,Ad,A) | 37,775 | 28,825 | 4.74% | 0.616 |

ARIMA is decisively better — 39% lower RMSE. MASE = 0.360 means ARIMA beats the seasonal naive baseline by 64%. Both the expanding CV and the held-out test window consistently favour ARIMA at all horizons.

**Figure to include:** Plot of 24-month test period forecasts from both models overlaid on actual data (filter from "2022 Jan"), with 80% prediction intervals. This is the key visual.

**Why ARIMA(0,1,0) is rejected:** Both diagnostics (fails Ljung-Box, p = 0.022) and CV accuracy (RMSE 96,836 at h=6) are substantially worse. The AR(2)/MA(2) terms are necessary.

**Why ARIMA(2,1,3) is rejected:** Worse CV RMSE than ARIMA(2,1,2) at h=6 (45,024 vs 34,859) and h=24 (72,417 vs 69,618). No diagnostic or accuracy justification for the additional MA term.

### 4.4 Why STL Variants Add Nothing (and why plain ETS(A,Ad,A) is selected)

Five ETS variants were investigated. The motivation for STL+ETS was compelling: robust STL would downweight COVID months during decomposition, providing implicit outlier protection without explicit dummies.

Empirically, ETS(A,Ad,A) matches or beats all STL variants at every CV horizon (see full table above). The mechanism: with α ≈ 1, ETS's level updates almost entirely from each new observation, so COVID observations are exponentially discounted to near-zero weight by 2026 regardless of STL downweighting. STL's robust treatment and ETS's exponential discounting converge to the same effective outcome.

Additionally, STL variants fail Ljung-Box more severely (p ≈ 10⁻⁶ to 10⁻⁷) than plain ETS(A,Ad,A) (p = 0.000165). ETS(A,Ad,A) is selected on parsimony, better diagnostics within the ETS class, and equal or better accuracy.

### 4.5 Final Forecasts: April 2026 – March 2028

Both models are refitted on the full dataset (July 2000 – March 2026) before producing the final 24-month forecast. Forecast plots are produced at 6-month, 12-month, and 24-month horizons with 80% prediction intervals.

**Full 24-month point forecast table:**

| Month | ARIMA(2,1,2)(0,1,1)[12] | ETS(A,Ad,A) |
|---|---|---|
| Apr 2026 | 573,684 | 545,345 |
| May 2026 | 623,386 | 599,234 |
| Jun 2026 | 666,824 | 640,040 |
| Jul 2026 | 829,232 | 765,212 |
| Aug 2026 | 736,961 | 702,728 |
| Sep 2026 | 587,886 | 596,958 |
| Oct 2026 | 586,772 | 601,249 |
| Nov 2026 | 499,228 | 536,118 |
| Dec 2026 | 500,190 | 519,277 |
| Jan 2027 | 464,198 | 491,608 |
| Feb 2027 | 466,910 | 494,372 |
| Mar 2027 | 524,842 | 530,642 |
| Apr 2027 | 572,944 | 551,960 |
| May 2027 | 624,814 | 605,730 |
| Jun 2027 | 672,601 | 646,439 |
| Jul 2027 | 840,235 | 771,456 |
| Aug 2027 | 751,842 | 708,958 |
| Sep 2027 | 604,712 | 603,246 |
| Oct 2027 | 604,667 | 607,481 |
| Nov 2027 | 516,445 | 542,404 |
| Dec 2027 | 516,504 | 525,549 |
| Jan 2028 | 479,103 | 497,897 |
| Feb 2028 | 480,710 | 500,615 |
| Mar 2028 | 538,088 | 536,766 |

**Discussion of divergence:**
- At h=6 (Apr–Sep 2026), forecasts are broadly comparable. Both models agree on the seasonal shape.
- At h=24, summer peaks diverge substantially: ARIMA projects ~829,000–840,000 for July peaks; ETS projects ~765,000–771,000. The gap is ~64,000–69,000 vehicles.
- The divergence reflects different views on trend persistence. ARIMA's AR(2)/MA(2) structure extrapolates current momentum; ETS's damped trend (φ = 0.98) gradually attenuates growth, producing more conservative long-horizon projections.
- Given ARIMA's superior CV and test-window accuracy, ARIMA point forecasts are the primary recommendation. ETS provides a useful conservative bound.

**Forecast combination:** Combining ARIMA and ETS forecasts (e.g., simple average) was not pursued — ARIMA outperforms ETS at every horizon in both CV and the held-out test, so averaging would only dilute accuracy. Combination is most valuable when models have complementary errors; here, ARIMA dominates on all metrics.

**Figures to include:**
- 6-month forecast plot (Apr–Sep 2026) with 80% PI for both models.
- 12-month forecast plot (Apr 2026–Mar 2027) with 80% PI for both models.
- 24-month forecast plot (Apr 2026–Mar 2028) with 80% PI for both models, overlaid on historical data from Jan 2023.

---

## 5. Conclusions (~1 page)

**Plain-language summary (for a non-technical reader / CEO):** Monthly car traffic on the Øresund Bridge is forecast to continue growing moderately over the next two years. Based on our primary model, the bridge can expect roughly 570,000–830,000 vehicle crossings per month across 2026–2027, with the summer peak (July) reaching approximately 830,000–840,000 — slightly above current record levels. The forecasts carry meaningful uncertainty: at the two-year horizon, the 80% prediction interval spans roughly ±150,000 vehicles around the central estimate. Both models agree on the seasonal pattern and the direction of growth; they differ mainly on the pace, with ARIMA projecting a somewhat stronger trend than ETS.

**Summary of findings:** We forecasted monthly vehicle crossings on the Øresund Bridge using 309 monthly observations (July 2000 – March 2026). After Box-Cox transformation (λ = 0.79) and d = 1, D = 1 differencing to achieve stationarity, we estimated an ARIMA(2,1,2)(0,1,1)[12] model with explicit COVID dummies and an ETS(A,Ad,A) model with damped trend.

**Primary model: ARIMA(2,1,2)(0,1,1)[12].** Clear recommendation on every criterion:
- Passes Ljung-Box (Q = 14.23, p = 0.770): residuals are white noise; prediction intervals are statistically valid.
- Best CV RMSE at all horizons (h=6: 34,859; h=12: 36,759; h=24: 69,618).
- Best test-window accuracy (RMSE = 22,988, MAPE = 3.03%, MASE = 0.360).
- `auto.arima` independently confirms the (2,1,2)(0,1,1)[12] specification.
- The explicit COVID dummy structure isolates the outlier period cleanly.

**ETS(A,Ad,A) as comparison:** ETS provides a useful methodological contrast and a more conservative long-horizon projection (damped trend, φ = 0.98). However, all five ETS variants fail Ljung-Box, making prediction intervals unreliable. On accuracy, ETS is consistently inferior (test RMSE = 37,775 vs 22,988; CV RMSE at h=24: 162,367 vs 69,618). Retained as the required second method.

**Limitations:**
(a) 17 COVID-affected observations necessarily influence estimated coefficients, though the QLR test (sup.F = 2.369, p = 0.697) supports structural stability across the full sample.
(b) Neither model can forecast unforeseen macroeconomic or policy shocks.
(c) At h=24, even ARIMA's prediction intervals span hundreds of thousands of vehicles — point forecasts are best understood as central scenarios.

---

## Appendix (Optional — does not count toward 15 pages)

- Full R code for data loading, transformation, EDA, model estimation, diagnostics, CV, and forecasting
- Full residual ACF/PACF panels for both models
- Full 24-month forecast tables (duplicated from Section 4.5 for convenience)
- Full CV RMSE tables for all 5 ETS variants and 3 ARIMA candidates
- Multi-panel differencing plot (raw → Box-Cox → annual diff → doubly-differenced)
- STL z-score remainder plot

---

## Numbers Reference Sheet (verified against r3_clean.pdf)

| Item | Value |
|---|---|
| Observations | 309 (Jul 2000 – Mar 2026) |
| Guerrero λ | 0.7899 |
| KPSS stat (raw) | 1.6096, p = 0.01 |
| **ADF** stat (raw) | −2.9251, p = 0.186 (fail to reject → non-stationary) |
| ndiffs | 1 |
| nsdiffs | 1 |
| KPSS stat (d=1,D=1) | 0.02957, p = 0.10 |
| **ADF** stat (d=1,D=1) | −6.9127, p = 0.01 |
| QLR test (sup.F) | sup.F = 2.369, p = 0.697 |
| Chow test (supplementary) | F = 0.059, p = 0.808 |
| COVID window | Mar 2020 – Jul 2021 (17 months) |
| April 2020 actual cars | 144,759 |
| April 2020 STL z-score | −8.07 |
| March 2020 STL z-score | −5.47 |
| ARIMA(2,1,2) AIC | 5187.87 |
| ARIMA(0,1,0) AIC | 5240.62 |
| ARIMA(2,1,2) LB | Q = 14.23, p = 0.770 (lag=24, dof=5) |
| ARIMA(0,1,0) LB | Q = 38.53, p = 0.022 (lag=24, dof=1) |
| auto.arima LB | Q = 14.23, p = 0.714 (lag=24, dof=6) |
| ETS(A,Ad,A) AIC | 6504.08 |
| ETS(A,Ad,A) α | 0.9999 |
| ETS(A,Ad,A) β | 0.0059 |
| ETS(A,Ad,A) γ | 0.0001 |
| ETS(A,Ad,A) φ | 0.9800 |
| ETS(A,Ad,A) LB | Q = 50.89, p = 0.000165 (lag=24, dof=4) |
| ets_auto LB | Q = 55.56, p = 9.89×10⁻⁵ |
| stl_auto LB | Q = 68.06, p = 2.41×10⁻⁶ |
| stl_AAN LB | Q = 67.99, p = 1.36×10⁻⁶ |
| stl_AAdN LB | Q = 68.04, p = 7.21×10⁻⁷ |
| ARIMA CV RMSE h=6 | 34,859 |
| ARIMA CV RMSE h=12 | 36,759 |
| ARIMA CV RMSE h=24 | 69,618 |
| ARIMA(2,1,3) CV RMSE h=6 | 45,024 |
| ARIMA(2,1,3) CV RMSE h=12 | 40,116 |
| ARIMA(2,1,3) CV RMSE h=24 | 72,417 |
| ETS CV RMSE h=6 | 36,873 |
| ETS CV RMSE h=12 | 109,407 |
| ETS CV RMSE h=24 | 162,367 |
| ARIMA test RMSE | 22,988 |
| ARIMA test MAPE | 3.03% |
| ARIMA test MASE | 0.360 |
| ETS test RMSE | 37,775 |
| ETS test MAPE | 4.74% |
| ETS test MASE | 0.616 |
| ARIMA Apr 2026 forecast | 573,684 |
| ETS Apr 2026 forecast | 545,345 |
| ARIMA Jul 2026 forecast | 829,232 |
| ETS Jul 2026 forecast | 765,212 |
| ARIMA Jul 2027 forecast | 840,235 |
| ETS Jul 2027 forecast | 771,456 |
| ARIMA Mar 2028 forecast | 538,088 |
| ETS Mar 2028 forecast | 536,766 |
