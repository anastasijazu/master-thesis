# Full experiment summary

## 1. Study overview

This study develops a real-data analytical pipeline for **cooling-energy prediction, benchmark-based operational assessment, and intelligent control simulation** in a telecom central office environment.

The final selected site is:

- **790ZHB**

The study starts from raw operational telemetry and proceeds through:

1. site and signal qualification  
2. target engineering  
3. dataset construction  
4. exploratory analysis  
5. forecasting-model comparison  
6. rolling-origin validation  
7. feature interpretation and ablation  
8. forecast-based benchmarking  
9. intelligent control strategy design and simulation  
10. controller validation and sensitivity analysis  

The contribution is not a new forecasting algorithm, but an integrated framework that connects:

- **target engineering from fragmented cumulative subsystem signals**
- **comparative forecasting under realistic validation**
- **expected-vs-actual cooling benchmarking**
- **intelligent control simulation under explicit operational constraints**

---

## 2. Site and data source logic

The study did not start from a pre-defined clean dataset. It first evaluated site suitability and signal availability.

The broader DB-related source structures considered included:

- `enmos_datapoint`
- `datapoint`
- `measurement`
- `measurement_all`

The final analytical branch was built only after checking:
- whether signals actually existed,
- whether they were populated,
- whether they had temporal continuity,
- and whether their semantics were clear enough for forecasting and operational interpretation.

---

## 3. Target engineering

A key methodological issue was that hourly cooling demand was **not directly available** as one ready-made variable.  
It had to be reconstructed from cumulative subsystem telemetry.

The final cooling target was engineered from the following component IDs:

- `120`
- `230`
- `46`
- `49`
- `78`

### Target-construction logic
The target was built by:

1. selecting cooling-related cumulative subsystem signals
2. aligning them to an hourly timestamp
3. differencing cumulative values into hourly increments
4. clipping negative or physically implausible steps
5. summing valid hourly component contributions
6. producing the final hourly cooling-demand target `y`

This means the final target is:

> **an engineered hourly cooling-demand series reconstructed from fragmented cumulative subsystem measurements**

This is one of the main methodological contributions of the study.

---

## 4. Final datasets

Two main dataset concepts were used:

- **main dataset**
- **compare dataset**

### Main dataset
The main dataset contains:
- engineered cooling target `y`
- selected internal operational features
- selected environmental/weather features
- time features

This is the preferred final analytical dataset because it gives the best balance between:
- predictive usefulness
- interpretability
- lower circularity risk

### Compare dataset
The compare dataset extends the main dataset with sibling cooling-related proxy variables.  
Its purpose was to test whether target-adjacent signals materially improve prediction.

### Compare-dataset conclusion
The compare dataset gave only marginal extra gain and was therefore not adopted as the main analytical basis.

---

## 5. Weather and contextual variables

External weather was retained because internal meteo-like signals were incomplete or not robust enough for direct use as primary weather inputs.

The main external variables included:
- `wx_temp_2m`
- `wx_dewpoint_2m`
- `wx_shortwave_rad`
- `wx_relhum_2m`
- `wx_cloudcover`

Time/context variables included:
- `hour`
- `dayofweek`
- `month`
- `is_weekend`

---

## 6. Internal temperature proxy

A sparse internal temperature signal was initially excluded due to missingness, but later re-evaluated.

It turned out to be:
- not a once-daily reading,
- but a **sparse low-frequency internal thermal-state signal**.

It was therefore transformed into:

- `internal_temp_proxy`

### Construction logic
The raw sparse signal was:
1. extracted from aligned telemetry,
2. aligned to hourly timestamps,
3. interpolated cautiously,
4. forward/back filled,
5. interpreted as a **slow internal thermal-state proxy**

It was **not** treated as a normal hourly ambient-weather signal.

This variable was then included in the final analytical pipeline as a contextual feature.

---

## 7. Final main dataset dimensions

The final 2025 main dataset contains:

- **8603 rows**
- **46 columns**
- time range: **2025-01-01 01:00:00** to **2025-12-31 23:00:00**

After lag construction, the final lagged forecasting/control table contains:

- **8435 rows**
- lagged setup starting at **2025-01-08 01:00:00**
- ending at **2025-12-31 23:00:00**

---

## 8. Exploratory data analysis

EDA showed that the cooling target has:

- strong seasonal structure
- strong short-term persistence
- non-Gaussian distribution
- larger variability in the warm season
- meaningful sensitivity to external conditions and internal operational context

The target is clearly not a simple memoryless response to weather.  
It behaves as a persistence-driven operational process with both internal and external influences.

---

## 9. Forecasting pipeline

The forecasting pipeline was developed in multiple stages.

### Initial model families
The first benchmark included:
- naive persistence
- seasonal naive
- ARIMA
- SARIMA
- linear regression
- Random Forest
- LSTM

A strong initial result was that naive persistence was already very competitive, indicating that the problem is strongly persistence-dominated at the 1-hour horizon.

### Final lagged setup
A richer supervised forecasting setup was then built using explicit lagged target features.

Final lag list:
- `1`
- `2`
- `3`
- `6`
- `12`
- `24`
- `48`
- `168`

The final lagged model comparison included:
- Linear Regression + lags
- Random Forest + lags
- Gradient Boosting + lags
- LSTM
- ARIMAX / SARIMAX with exogenous inputs

### Main validation stage
The main model-selection stage was **rolling-origin backtesting**.

This is the strongest model-selection stage in the thesis because it is temporally realistic and more robust than a single chronological holdout.

---

## 10. Final forecasting model selection

The final result of the rolling-origin validation was:

- **Gradient Boosting + lags** = final selected forecasting model
- **Random Forest + lags** = strongest secondary benchmark
- naive = strong baseline
- LSTM = improved after preprocessing/scaling but weaker than best tree-based models
- SARIMAX / ARIMAX = weaker and less stable

### Why Gradient Boosting remained final
The final model choice is based on the strongest validation stage, namely rolling-origin backtesting, not on one later interpretive plot alone.

Therefore:
- **Gradient Boosting** remains the final selected forecasting model
- **Random Forest** is retained as a highly competitive secondary benchmark and as the most interpretable model for feature-group analysis

---

## 11. Feature interpretation and ablation

Feature interpretation used:
- tree-based importance
- permutation importance
- grouped feature importance
- feature-group ablation

### Main result
The most important predictor was:

- `y_lag_1`

This confirms that short-term cooling forecasting is heavily persistence-driven.

### After excluding `y_lag_1`
The most important contextual predictors included:
- `90`
- `100`
- `14`
- `72`
- `internal_temp_proxy`
- longer lags
- selected operational variables

After adding `internal_temp_proxy`, it became one of the strongest non-lag contextual variables.

### Feature-group ablation
Feature groups compared included:
- `lags_only`
- `lags_plus_weather`
- `lags_plus_operational`
- `lags_weather_operational`
- `lags_weather_operational_siblings`

#### Random Forest ablation
Random Forest suggested that:
- operational and weather context improve over lags alone
- siblings add only marginal extra benefit
- operational context is particularly important after lagged history

#### Gradient Boosting ablation
Gradient Boosting suggested a stronger dependence on lag structure and less benefit from wider feature additions.

### Final interpretation
This difference does **not** invalidate the final model choice.  
Instead it shows that:
- Gradient Boosting is more compact and lag-dominant
- Random Forest is more useful for interpreting feature-group contributions

---

## 12. Forecast-based benchmarking

The final selected Gradient Boosting model was used to generate **expected cooling demand**.  
This benchmark was bias-corrected before operational interpretation.

### Full-year result
After bias correction:
- total actual cooling and expected cooling were effectively equal
- annual net difference was approximately zero

### Interpretation
There is **no evidence of large-scale systematic annual overcooling**.

### Warm-season result
The warm-season analysis showed:
- raw warm-season excess above expected cooling ≈ **1.34%**
- reduced-cooling share ≈ **1.36%**
- net warm-season balance ≈ **-0.02%**

### Interpretation
Even in the warm season, the site remains broadly balanced around expected cooling demand.

---

## 13. Conservative reduction-potential analysis

The study did not treat every positive residual as directly reducible waste.

Instead, it introduced conservative filters:
- meaningful excess threshold
- persistence requirement
- operating-context restrictions
- bounded action caps

At one stage of the warm-season funnel:
- total warm-season hours = **3537**
- positive residual hours = **1832**
- meaningful excess hours = **458**
- eligible persistent hours = **38**

### Bounded scenario estimates
- raw warm-season excess = **1.34%**
- persistent 5% cap = **0.046%**
- persistent 10% cap = **0.057%**

### Interpretation
Once supervisory realism is imposed, plausible intervention potential becomes very small.

---

## 14. Intelligent control framing

The control part of the study is formulated as:

> **forecast-informed intelligent control strategy simulation**

It is **not**:
- full model predictive control deployment
- reinforcement learning control
- actuator-level industrial control

It is:
- forecast-informed
- context-aware
- bounded
- supervisory
- historically simulated
- comparatively evaluated

This is the strongest control level defensibly supported by the available telemetry.

---

## 15. Intelligent control architecture

The intelligent control section was structured into four layers:

1. **forecasting layer**  
   estimates expected cooling demand

2. **state/context layer**  
   defines residual state, load context, thermal context, time context, and operating mode

3. **supervisory decision layer**  
   assigns control states

4. **action simulation layer**  
   applies bounded hypothetical reductions to historical cooling demand

The manipulated variable is represented abstractly as:

- **cooling-reduction intensity**

rather than a direct actuator command.

---

## 16. Operational proxy selection for intelligent control

Several candidate operational proxies were tested for the control layer:
- `179`
- `188`
- `80`
- `90`

The final proxy was **not** selected by raw target correlation alone.

### Final proxy sensitivity results
For the revised control-validation pipeline:

- `179`:  
  - `S2_active_hours = 30`  
  - `S2_reduction_pct = 0.043542`  
  - `S3_active_hours = 30`  
  - `S3_reduction_pct = 0.041931`

- `188`:  
  - `S2_active_hours = 9`  
  - `S2_reduction_pct = 0.010786`  
  - `S3_active_hours = 5`  
  - `S3_reduction_pct = 0.005317`

- `80`:  
  - `S2_active_hours = 9`  
  - `S2_reduction_pct = 0.011077`  
  - `S3_active_hours = 5`  
  - `S3_reduction_pct = 0.005759`

- `90`:  
  - `S2_active_hours = 23`  
  - `S2_reduction_pct = 0.035380`  
  - `S3_active_hours = 15`  
  - `S3_reduction_pct = 0.024061`

### Final proxy choice
The final load proxy retained for intelligent control was:

- **`179`**

because it gave the strongest balance of:
- supervisory usefulness
- stable S2/S3 behavior
- meaningful operating-mode discrimination

Thus, proxy selection was based on **control-context usefulness**, not only on predictive similarity.

---

## 17. Thermal and contextual state design

The intelligent control state layer includes:
- `residual_bc`
- `load_proxy`
- `internal_temp_proxy`
- warm-season filter
- allowed-hour filter
- persistence logic
- operating modes

### Warm-season / allowed-hour context
- warm-season hours = **3537**
- allowed-hour hours = **3159**

### Residual structure
- positive residual hours = **4130**
- meaningful excess hours = **1033**
- persistent excess hours = **213**

### State distributions
- load state counts:
  - `LOW = 4816`
  - `MEDIUM = 755`
  - `HIGH = 2864`

- thermal state counts:
  - `LOW = 2784`
  - `MEDIUM = 2785`
  - `HIGH = 2866`

- excess state counts:
  - `NONE = 4305`
  - `MILD = 3097`
  - `STRONG = 1033`

---

## 18. Operating-mode discovery

K-means clustering was used to discover operating modes.  
Several values of `K` were tested, and the final choice remained:

- **K = 3**

because it gave the best balance of:
- separation
- cluster quality
- supervisory interpretability

The final operating modes were interpreted as:
- `LOW_RISK`
- `MEDIUM_RISK`
- `HIGH_RISK`

The cluster design was validated rather than treated as purely hardcoded.

---

## 19. Intelligent control strategies

Three intelligent control strategies were defined.

### `S1_forecast_only`
Uses:
- warm-season filter
- residual / excess logic
- persistence

It is the simplest forecast-informed strategy.

### `S2_load_aware`
Adds:
- selected operational load proxy
- allowed operating hours

It is more selective than S1.

### `S3_load_mode_aware`
Adds:
- operating-mode awareness
- more refined action-state logic

It is the most context-aware strategy.

### Control states
The final supervisory states were:
- `PROTECT`
- `NORMAL`
- `RELAXED_1`
- `RELAXED_2`

Mapped to actions:
- `PROTECT` = `0%`
- `NORMAL` = `0%`
- `RELAXED_1` = `5%`
- `RELAXED_2` = `10%`

---

## 20. Baseline comparison

A simple low-complexity benchmark controller was added:

- `B0_simple_threshold`

It uses:
- warm season
- residual threshold
- bounded action
- no persistence
- no load context
- no mode awareness

This makes the control study more systematic by comparing:
- baseline
- S1
- S2
- S3

under the same conditions.

---

## 21. Final intelligent control validation results

The final shared-condition comparison gave:

### `B0_simple_threshold`
- active hours = **496**
- reduction vs historical = **0.641%**
- `RELAXED_1` hours = **496**
- `RELAXED_2` hours = **0**
- mean action intensity = **5.000000%**
- low-risk action fraction = **0.088710**
- medium-risk action fraction = **0.852823**
- high-risk action fraction = **0.058468**
- avoided high-risk hours = **0**

### `S1_forecast_only`
- active hours = **115**
- reduction vs historical = **0.178%**
- `RELAXED_1` hours = **0**
- `RELAXED_2` hours = **115**
- mean action intensity = **10.000000%**
- low-risk action fraction = **0.113043**
- medium-risk action fraction = **0.852174**
- high-risk action fraction = **0.034783**
- avoided high-risk hours = **0**

### `S2_load_aware`
- active hours = **47**
- reduction vs historical = **0.073%**
- `RELAXED_1` hours = **0**
- `RELAXED_2` hours = **47**
- mean action intensity = **10.000000%**
- low-risk action fraction = **0.127660**
- medium-risk action fraction = **0.872340**
- high-risk action fraction = **0.000000**
- avoided high-risk hours = **200**

### `S3_load_mode_aware`
- active hours = **47**
- reduction vs historical = **0.066%**
- `RELAXED_1` hours = **41**
- `RELAXED_2` hours = **6**
- mean action intensity = **5.638298%**
- low-risk action fraction = **0.127660**
- medium-risk action fraction = **0.872340**
- high-risk action fraction = **0.000000**
- avoided high-risk hours = **200**

### Interpretation
The final hierarchy is clear and defensible:

- the simple baseline is the most aggressive
- S1 is much more selective
- S2 is stricter still
- S3 is the most nuanced and context-aware

Importantly, after the integration of `internal_temp_proxy`, S3 is no longer unrealistically restrictive.  
It still acts selectively, but now uses a **mixed intensity pattern**:
- mostly `RELAXED_1`
- only a few `RELAXED_2`

So S3 is now better interpreted as:
- **the most refined intelligent controller**
- not simply the most restrictive one

---

## 22. Monthly intelligent control pattern

Monthly reduction plots show that:
- the baseline acts broadly across months
- S1 still acts in several months
- S2 and S3 act in a much narrower subset

In the final configuration:
- the largest reductions are concentrated in months **5**, **7**, and **8**
- month **9** becomes nearly or fully inactive for S2 and S3

This supports the conclusion that intervention opportunities are:
- seasonal,
- limited,
- and concentrated in specific operational contexts.

---

## 23. Example event windows

Representative example plots were created for:
- S1 around **2025-05-07 08:00:00**
- S1 around **2025-05-07 23:00:00**
- S3 around **2025-08-04 09:00:00**

These plots show:
- historical actual cooling
- expected cooling
- simulated controlled cooling
- control-active interval

They illustrate that the intelligent control logic does not act continuously, but only during bounded intervals where the combined residual and contextual criteria are satisfied.

---

## 24. Sensitivity analysis of S3

The intelligent control section was additionally validated with a parameter sensitivity analysis for S3.

Tested values:
- residual quantile = `0.50`, `0.75`, `0.90`
- persistence = `1`, `2`, `3` hours
- cap = `0.05`, `0.10`

### Main results

#### At residual quantile `0.50`
- persistence 1, cap 0.05 → **441 hours**, **0.449991%**
- persistence 1, cap 0.10 → **441 hours**, **0.487835%**
- persistence 2, cap 0.05 → **185 hours**, **0.187373%**
- persistence 2, cap 0.10 → **185 hours**, **0.204505%**
- persistence 3, cap 0.05 → **76 hours**, **0.075382%**
- persistence 3, cap 0.10 → **76 hours**, **0.080893%**

#### At residual quantile `0.75`
- persistence 1, cap 0.05 → **194 hours**, **0.271479%**
- persistence 1, cap 0.10 → **194 hours**, **0.309323%**
- persistence 2, cap 0.05 → **47 hours**, **0.066288%**
- persistence 2, cap 0.10 → **47 hours**, **0.076125%**
- persistence 3, cap 0.05 → **9 hours**, **0.011581%**
- persistence 3, cap 0.10 → **9 hours**, **0.012478%**

#### At residual quantile `0.90`
- persistence 1, cap 0.05 → **75 hours**, **0.127538%**
- persistence 1, cap 0.10 → **75 hours**, **0.164845%**
- persistence 2, cap 0.05 → **8 hours**, **0.016246%**
- persistence 2, cap 0.10 → **8 hours**, **0.020805%**
- persistence 3, cap 0.05 → **0 hours**, **0.000000%**
- persistence 3, cap 0.10 → **0 hours**, **0.000000%**

### Interpretation
This confirms that:
- looser settings produce more action and larger reduction estimates
- stricter settings rapidly reduce the intervention margin
- the final selected middle setting remains conservative but non-zero

This is exactly the desired robustness result.

---

## 25. What was tested but not retained

### 2026 temporal extension
A partial 2026 extension was explored, but not kept as a main thesis result because:
- 2026 currently ends at **2026-04-15 21:00:00**
- warm-season comparison is incomplete
- forecast transfer to early 2026 was weak

### Dwell-time validation
A minimum dwell-time check was also explored, but it inflated active control hours unrealistically and was therefore not used as final evidence.

---

## 26. Final conclusions

The final study supports the following conclusions:

### Forecasting and benchmarking
1. A defensible hourly cooling-demand target can be engineered from fragmented cumulative subsystem telemetry.
2. Short-term cooling forecasting is feasible, and the final selected model is **Gradient Boosting with lagged target features**.
3. Random Forest remains a strong secondary benchmark and the clearest model for feature-group interpretation.
4. The site does not show evidence of large-scale systematic annual overcooling.
5. Warm-season positive residuals exist, but once screened conservatively, they imply only a small plausible intervention margin.

### Intelligent control
6. Intelligent control can be formulated credibly as a **forecast-informed control simulation** based on model outputs and contextual variables.
7. The final controller family is:
   - baseline simple threshold
   - S1 forecast-only
   - S2 load-aware
   - S3 load-and-mode-aware
8. The final contextual design was improved by:
   - adding `internal_temp_proxy`
   - selecting `179` as the final load proxy
   - validating the operating-mode structure
   - validating the supervisory-parameter choices
9. The strongest final intelligent controller, **S3**, is selective, context-aware, and conservative:
   - **47 active hours**
   - **0.066% reduction**
   - mostly `RELAXED_1`
   - no high-risk actions
10. The final result is not that the site contains large hidden savings.  
    The result is that the site contains a **small, bounded, and defensibly identifiable set of forecast-informed control opportunities**.

---

## 27. Final thesis positioning

The study develops a methodologically integrated framework for telecom cooling-energy analysis that bridges real-world target engineering, predictive benchmarking, and intelligent control simulation. It shows that broad reducible cooling demand is not evident at the selected site, but that a small number of bounded and context-aware intervention opportunities can still be identified using forecast-informed intelligent control logic.