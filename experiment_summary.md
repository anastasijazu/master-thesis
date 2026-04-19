## 1. Thesis topic and overall positioning

The thesis studies **data-driven prediction and analysis of cooling energy use in telecom central office environments**, with a focus on:

- reconstructing a usable hourly cooling-demand target from fragmented telemetry,
- forecasting short-term cooling demand,
- comparing forecasting model families under realistic temporal validation,
- benchmarking actual versus expected cooling demand,
- and designing **forecast-informed intelligent control simulation** strategies.

The final thesis positioning is:

- **not** full MPC deployment,
- **not** reinforcement learning control,
- **not** actuator-level industrial control validation,

but:

- **a methodologically integrated applied study** combining target engineering, forecasting, benchmark-based operational assessment, and intelligent control simulation under real telemetry constraints.

This positioning is consistent with the literature review and with the available telemetry.

---

## 2. Site, source data, and signal screening

The final selected site is:

- **790ZHB**

The broader DB-related structures and metadata layers considered during exploration included:

- `enmos_datapoint`
- `datapoint`
- `measurement`
- `measurement_all`

The study did not assume all nominal signals were usable. Instead, it first checked:
- availability,
- temporal continuity,
- meaning,
- completeness,
- and whether signals were suitable for target construction, forecasting, or control context.

A separate raw signal listing later confirmed that there are many available signals, but most are still power-like telemetry and do not provide a rich actuator/state layer. That supported the decision to keep the thesis at the supervisory-control level.

---

## 3. Target engineering

A major contribution of the thesis is that the cooling target was **not directly available** as a ready hourly variable. It had to be reconstructed.

The final engineered cooling target was built from cumulative subsystem signals with IDs:

- `120`
- `230`
- `46`
- `49`
- `78`

### Target construction logic
The target pipeline was:

1. select cooling-related cumulative subsystem signals,
2. align them to hourly timestamps,
3. difference cumulative values into interval consumption,
4. clip negative / implausible steps,
5. aggregate retained hourly increments,
6. produce final hourly cooling-demand target `y`.

This led to the key interpretation that the target is:

> **an engineered hourly cooling-demand variable reconstructed from fragmented cumulative subsystem telemetry**

and this is a central methodological contribution of the thesis, not a hidden preprocessing detail.

---

## 4. Final dataset construction

Two main dataset concepts were developed:

- **main dataset**
- **compare dataset**

### Main dataset
The main dataset contains:
- engineered target `y`
- selected internal operational features
- selected external weather variables
- time/context variables

This is the preferred final dataset because it gives the best balance of:
- predictive usefulness,
- interpretability,
- and reduced circularity risk.

### Compare dataset
The compare dataset extends the main dataset with sibling cooling-related proxies to test whether target-adjacent variables change the forecasting and benchmarking results substantially.

### Main conclusion from this comparison
The sibling-expanded compare dataset changed results only **marginally**, which became an important argument that the selected 5-signal target is **sufficiently representative** and not severely too narrow.

This became one of the strongest arguments against the concern that small estimated overcooling might be only an artifact of an overly narrow target.

---

## 5. Weather and contextual variables

External weather was retained because internal meteo-like signals were incomplete or unsuitable as direct replacements.

The external variables used included:
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

A sparse internal temperature signal was first excluded because of missingness, but later re-examined.

### What we found
It was not just a once-daily reading.  
It had roughly **2000 non-null values** in 2025 and appeared in sparse intra-day blocks.

This led to a reinterpretation:
- it is **not** a normal hourly weather variable,
- but it can be treated as a **slow internal thermal-state signal**.

### How it was handled
It was converted into:
- `internal_temp_proxy`

The pipeline was:
1. extract the sparse raw signal,
2. align it to hourly timestamps,
3. fix timezone mismatch,
4. interpolate cautiously,
5. forward/back fill,
6. retain it as a slow internal thermal-state proxy.

This proxy was then added into the main feature set.

### Important interpretation
It was **not** treated as a normal ambient temperature measure.  
It was treated as a contextual internal-state variable.

Later feature-importance analysis showed that `internal_temp_proxy` became one of the strongest non-lag contextual features, which supported its inclusion.

---

## 7. Final 2025 main dataset size

The final main dataset dimensions were:

- **8603 rows**
- **46 columns**
- time range: **2025-01-01 01:00:00** to **2025-12-31 23:00:00**

After lag construction, the final lagged table used for forecasting and control became:

- **8435 rows**
- start: **2025-01-08 01:00:00**
- end: **2025-12-31 23:00:00**

---

## 8. Exploratory data analysis

EDA established that the cooling target is:

- strongly seasonal,
- strongly persistence-driven,
- non-Gaussian,
- and more variable in the warm season.

The target behaves as a persistence-dominated operational process rather than as a simple direct weather response.

External weather and internal operational variables both contribute explanatory value, but lagged target history remains dominant.

---

## 9. Forecasting pipeline

The forecasting pipeline was built in multiple stages.

### Initial family benchmark
Initial models included:
- naive persistence
- seasonal naive
- ARIMA
- SARIMA
- linear regression
- Random Forest
- LSTM

A major finding was that naive persistence is already very competitive at the 1-hour horizon, confirming that the problem is strongly persistence-driven.

### Final lagged setup
A richer supervised setup was then built with explicit lagged target features:

Final lag list:
- `1`
- `2`
- `3`
- `6`
- `12`
- `24`
- `48`
- `168`

Models compared in the lagged setup included:
- Linear Regression + lags
- Random Forest + lags
- Gradient Boosting + lags
- LSTM
- ARIMAX / SARIMAX with exogenous inputs

### Main validation stage
The final model choice was based on **rolling-origin backtesting**, which became the strongest and most defensible validation stage.

---

## 10. Final forecasting model choice

The final selected model is:

- **Gradient Boosting with lagged target features**

The strongest secondary benchmark is:

- **Random Forest with lagged target features**

### Why GB remained final
Final model selection was based on the main rolling-origin validation stage, not only on later interpretive diagnostics.

### Why RF was retained
Random Forest remained:
- highly competitive,
- and more interpretable in feature-group ablation.

So the final logic became:
- **GB** = final benchmark model
- **RF** = interpretive and robustness benchmark

---

## 11. Forecasting metrics and interpretation

Final forecast metrics shown later for the chosen model were:

- **MAE = 117.413399**
- **RMSE = 166.253246**
- **Mean actual = 3377.843983**
- **CVRMSE = 4.921875%**
- **NMAE = 3.475986%**
- **R² = 0.984690**

### Interpretation
These numbers mean:
- forecast error is small relative to the target scale,
- RMSE is about **4.9% of mean hourly cooling demand**,
- average absolute error is about **3.5% of mean hourly demand**,
- and the model explains about **98.5% of target variance**.

This became the best way to express forecasting quality.

We explicitly decided **not** to say “95% accurate,” because that would be less academically precise than CVRMSE, NMAE, and R².

---

## 12. Feature interpretation and ablation

Feature interpretation used:
- Random Forest feature importance
- Gradient Boosting permutation importance
- grouped importance
- feature-group ablation

### Main findings
The dominant feature was:
- `y_lag_1`

This confirmed that the short-term cooling-demand problem is heavily persistence-driven.

### After excluding `y_lag_1`
Strong contextual variables included:
- `90`
- `100`
- `14`
- `72`
- `internal_temp_proxy`
- longer lags
- selected internal operational features

The addition of `internal_temp_proxy` changed the contextual ranking substantially and showed that it carries meaningful thermal context.

### Feature-group ablation
Ablation groups included:
- `lags_only`
- `lags_plus_weather`
- `lags_plus_operational`
- `lags_weather_operational`
- `lags_weather_operational_siblings`

#### RF ablation
RF suggested that:
- operational and weather context improve prediction over lags alone
- siblings add only marginal value

#### GB ablation
GB suggested a stronger reliance on lag structure and less benefit from wider contextual features.

### Interpretation
This did **not** invalidate GB as final model.  
It simply meant:
- GB is more compact and lag-driven,
- RF is more useful for understanding grouped feature contributions.

---

## 13. Benchmarking actual vs expected cooling

The final selected GB model was used to generate:
- expected cooling,
- bias-corrected predictions,
- and benchmark residuals.

### Full-year benchmarking
After bias correction:
- total actual cooling and total expected cooling were effectively equal,
- annual net difference was approximately zero.

### Interpretation
There is **no evidence of large-scale systematic annual overcooling**.

### Warm-season benchmarking
For the warm season:
- raw excess above expected cooling ≈ **1.34%**
- reduced-cooling share ≈ **1.36%**
- net warm-season balance ≈ **-0.02%**

### Interpretation
Even in the warm season, the site remains broadly balanced around expected demand.

---

## 14. Conservative reduction-potential logic

The study did not treat every positive residual as directly reducible waste.  
It introduced conservative filters:
- meaningful excess threshold,
- persistence,
- contextual screening,
- bounded action caps.

At one stage of the reduction funnel:
- warm-season hours = **3537**
- positive residual hours = **1832**
- meaningful excess hours = **458**
- eligible persistent hours = **38**

This led to bounded scenario estimates:
- raw warm-season excess = **1.34%**
- persistent 5% cap = **0.046%**
- persistent 10% cap = **0.057%**

### Interpretation
Once supervisory realism is introduced, the plausible intervention margin becomes very small.

---

## 15. Intelligent control framing

The control section was explicitly framed as:

> **forecast-informed intelligent control simulation**

This was an important wording shift.

It is not:
- full MPC,
- reinforcement learning,
- or actuator-level validated control.

It is:
- forecasting-model output translated into supervisory decision logic,
- then simulated on historical data,
- with explicit context and risk filtering.

We discussed multiple times that this is the strongest defensible control level supported by the available telemetry.

---

## 16. Why not full MPC or RL

We clarified that full MPC would require:
- validated dynamic process model,
- explicit manipulated variables,
- richer internal thermal-state feedback,
- physical constraints,
- disturbance forecasts in control-ready form,
- and a safe deployment or digital-twin environment.

We also clarified why reinforcement learning was not used:
- no validated closed-loop environment,
- no safe exploration space,
- no actuator-level response model,
- no action-outcome label structure.

This became part of the planned discussion section.

---

## 17. Why there is no ground-truth control accuracy

We clarified multiple times that forecasting and control are validated differently.

### Forecasting
Has a true observed target:
- actual cooling demand

So forecasting accuracy is directly measurable.

### Intelligent control
Does **not** have:
- a true optimal action at each hour,
- a deployed controller trying alternatives,
- a validated digital twin,
- or labeled action-outcome data.

Therefore, control is not evaluated by “accuracy,” but by:
- effectiveness,
- selectivity,
- risk-awareness,
- and robustness.

This became an important defense point.

---

## 18. Bidirectional benchmark interpretation

We discussed whether the framework should identify only reduction opportunities or also possible insufficient-cooling periods.

The final conclusion was:
- the proposal/abstract do **not** force reduction-only logic,
- the benchmark framework can be interpreted bidirectionally,
- but the under-expected side should be treated more cautiously.

So the final interpretation became:

- **positive residuals** → candidate excess cooling / reduction side
- **negative residuals** → candidate insufficient-cooling diagnostics

We implemented this as a diagnostic extension rather than as a fully validated increase controller.

### Under-expected diagnostic result
The pipeline found:
- `negative_residual_hours = 4327`
- `meaningful_under_hours = 2164`
- `persistent_under_hours = 854`
- but `candidate_insufficient_cooling_hours = 0`

### Interpretation
Below-expected cooling does occur often, but **not** under the simultaneous high-risk conditions required by the strict diagnostic flag. So the data do **not** support a strong claim of systematic insufficient cooling.

This became a strong finding because it made the benchmark-control framework balanced rather than one-sided.

---

## 19. Operational proxy selection for intelligent control

Several proxies were tested:
- `179`
- `188`
- `80`
- `90`

Initially, `80` and `188` had the strongest raw correlation with actual cooling (~0.85), but they proved less useful as supervisory context variables because they behaved too much like direct cooling-state indicators and often collapsed S2/S3 into near-zero or trivial results.

The final proxy retained for intelligent control became:

- **`179`**

because it gave the strongest balance of:
- contextual usefulness,
- stable S2/S3 behavior,
- and interpretable operating-mode separation.

This was an important conceptual shift:
proxy choice was based on **supervisory usefulness**, not only on raw predictive correlation.

---

## 20. Intelligent control state design

The intelligent control state layer uses:
- `residual_bc`
- load proxy
- `internal_temp_proxy`
- warm-season filter
- allowed-hour filter
- persistence logic
- operating modes

We later revised the thermal/risk design further in response to feature-importance and clustering results.

The candidate operational load proxies used in the intelligent-control section were not drawn arbitrarily from the final compact forecasting dataset. They were manually shortlisted from the broader hourly telemetry exploration after the main forecasting dataset had already been fixed. This shortlist was intended specifically for the control-design stage, where one additional operational-context variable was needed. The tested candidates were selected because they had plausible operational meaning, sufficient availability, and potential value as contextual indicators of site burden. The final proxy was then chosen based on supervisory usefulness, including operating-mode separation and control behavior, rather than on raw predictive correlation alone.

### State summary from one major control version
At one point:
- warm-season hours = **3537**
- allowed-hour hours = **3159**
- positive residual hours = **4130**
- meaningful excess hours = **1033**
- persistent excess hours = **213**

State counts included:
- load state: `LOW = 4816`, `MEDIUM = 755`, `HIGH = 2864`
- excess state: `NONE = 4305`, `MILD = 3097`, `STRONG = 1033`

---

## 21. Operating modes and high-risk definition

We spent a lot of time clarifying what `HIGH_RISK` means and how to justify it.

### Important clarification
`HIGH_RISK` does **not** mean directly observed thermal failure.  
It means:

> a data-driven supervisory operating context with comparatively higher load and/or thermal burden, in which cooling reduction is less defensible.

### How high-risk was built
Originally, risk assignment used a cluster-level ranking based on:
- `load_proxy_mean`
- `thermal_context_mean`

We noticed a problem where `MEDIUM_RISK` and `HIGH_RISK` could tie under a simple average-rank approach.

So the recommendation became:
- rank clusters primarily by **load burden**
- use thermal context as secondary refinement
- use clustering for empirical structure, but risk labels for conservative filtering

We also created visualizations to show:
- how clusters are formed,
- how cluster-level risk is ranked,
- and how LOW / MEDIUM / HIGH risk differ in load proxy and thermal context.

---

## 22. K-means cluster validation

We tested:
- `K = 2`
- `K = 3`
- `K = 4`
- `K = 5`

Diagnostics used:
- silhouette score
- Calinski–Harabasz score
- Davies–Bouldin score

At one stage, `K = 3` gave the best overall clustering quality and remained the preferred choice.

This allowed us to defend that the number of clusters was **not purely hardcoded**, but checked against diagnostics.

---

## 23. Final intelligent control strategy family

The final strategy family was:

### `B0_simple_threshold`
A simple threshold-based baseline:
- warm season
- residual threshold
- bounded action
- no persistence
- no context awareness

### `S1_forecast_only`
Uses:
- warm season
- residual benchmark
- persistence

### `S2_load_aware`
Adds:
- load proxy
- allowed-hour filtering

### `S3_load_mode_aware`
Adds:
- mode awareness
- more refined action logic

Control states remained:
- `PROTECT`
- `NORMAL`
- `RELAXED_1`
- `RELAXED_2`

Action mapping:
- `PROTECT` = 0%
- `NORMAL` = 0%
- `RELAXED_1` = 5%
- `RELAXED_2` = 10%

---

## 24. Main intelligent control results (before internal temperature revision)

An earlier major control version gave:

### `B0_simple_threshold`
- active hours = **496**
- reduction = **0.641%**

### `S1_forecast_only`
- active hours = **115**
- reduction = **0.178%**

### `S2_load_aware`
- active hours = **25**
- reduction = **0.0356%**

### `S3_load_mode_aware`
- active hours = **5**
- reduction = **0.0090%**

This was already a very good demonstration of the basic hierarchy:
- simple logic → more aggressive
- richer logic → more selective and safer

---

## 25. Main intelligent control results after adding internal temperature proxy

After adding `internal_temp_proxy` into the contextual design, the final validation results shifted.

The final validated strategy comparison became:

### `B0_simple_threshold`
- active hours = **496**
- reduction = **0.641%**
- mean action intensity = **5.0%**
- low-risk action fraction = **0.088710**
- medium-risk action fraction = **0.852823**
- high-risk action fraction = **0.058468**
- avoided high-risk hours = **0**

### `S1_forecast_only`
- active hours = **115**
- reduction = **0.178%**
- mean action intensity = **10.0%**
- low-risk action fraction = **0.113043**
- medium-risk action fraction = **0.852174**
- high-risk action fraction = **0.034783**
- avoided high-risk hours = **0**

### `S2_load_aware`
- active hours = **47**
- reduction = **0.073%**
- mean action intensity = **10.0%**
- low-risk action fraction = **0.127660**
- medium-risk action fraction = **0.872340**
- high-risk action fraction = **0.000000**
- avoided high-risk hours = **200**

### `S3_load_mode_aware`
- active hours = **47**
- reduction = **0.066%**
- `RELAXED_1 = 41`
- `RELAXED_2 = 6`
- mean action intensity = **5.638298%**
- low-risk action fraction = **0.127660**
- medium-risk action fraction = **0.872340**
- high-risk action fraction = **0.000000**
- avoided high-risk hours = **200**

### Interpretation
This became one of the strongest final results.

The hierarchy remained:
- baseline most aggressive,
- S1 selective,
- S2 and S3 much safer,

but now S3 was **not simply more restrictive in hours** than S2.  
Instead, S3 was **more nuanced in action intensity**:
- same active hours as S2
- lower average action intensity
- more refined use of `RELAXED_1` and `RELAXED_2`

This made the intelligent control section stronger, because S3 now looked like a **refined supervisory controller**, not only the strictest one.

---

## 26. Monthly control behavior

Monthly reduction plots showed:
- baseline acts broadly across months
- S1 still acts across several months
- S2 and S3 become much narrower

With internal temperature included:
- month 5 remained the strongest month
- months 6–8 retained moderate activity
- month 9 dropped out for S2 and S3

This reinforced that intervention opportunities are:
- seasonal,
- bounded,
- and concentrated in a narrow set of contexts.

---

## 27. Controller-oriented validation

We developed a full controller validation package including:

### Baseline comparison
- baseline vs S1 / S2 / S3

### Controller metrics
- active control hours
- action intensity
- risk-state distribution
- avoided high-risk hours
- selectivity

### Proxy sensitivity
- comparing `179`, `188`, `80`, `90`

### Cluster-number sensitivity
- comparing `K = 2, 3, 4, 5`

### Threshold / persistence / cap sensitivity
For S3, tested:
- residual quantiles `0.50`, `0.75`, `0.90`
- persistence `1`, `2`, `3`
- cap `0.05`, `0.10`

### Main S3 sensitivity results
Examples:

At `0.50`, `1h`, `0.05`:
- **441 hours**
- **0.449991%**

At `0.75`, `2h`, `0.05`:
- **47 hours**
- **0.066288%**

At `0.90`, `3h`, `0.05`:
- **0 hours**
- **0.000000%**

### Interpretation
This showed exactly what a trustworthy bounded control estimate should show:
- looser rules give bigger opportunities,
- stricter rules shrink or eliminate them,
- the selected middle setting remains conservative but non-zero.

---

## 28. What we decided not to keep as main evidence

### 2026 extension
A partial 2026 extension was explored, but it was not retained as a main thesis result because:
- 2026 currently ends at **2026-04-15 21:00:00**
- warm-season validation is incomplete
- forecast transfer to early 2026 was weak

### Dwell-time validation
A dwell-time check was implemented, but it inflated activity unrealistically:
- baseline: **497 → 1978**
- S2: **25 → 1308**

So it was not retained as main evidence without further correction.

---

## 29. How we decided to talk about “accuracy”

We clarified that:

### Forecasting
can be evaluated by **accuracy**, because there is a true observed target.

Best metrics:
- MAE
- RMSE
- CVRMSE (%)
- NMAE (%)
- R²

### Intelligent control
should **not** be described with “accuracy,” because there is no ground-truth optimal action sequence.

Instead, control quality should be described through:
- effectiveness
- selectivity
- risk-awareness
- robustness

This became an important defense point.

---

## 30. Final interpretation of trust and quality

The final logic we agreed on was:

- **forecasting quality** is trusted because of strong out-of-sample and normalized error metrics
- **control credibility** is trusted because of layered validation:
  - baseline comparison
  - risk filtering
  - proxy sensitivity
  - clustering sensitivity
  - parameter sensitivity

The key defense sentence became:

> Forecast accuracy is measured directly; control trustworthiness is defended through conservativeness, selectivity, and robustness.

---

## 6-hours ahead sensitivity check since literature describes that 

A limited six-hour-ahead sensitivity check was performed to test whether the literature’s horizon-sensitive expectation was visible in the present dataset. Although the six-hour machine-learning models did not outperform the naive benchmark under rolling-origin validation, the feature-importance pattern suggested that contextual variables, especially the internal temperature proxy, become relatively more important at the longer horizon. For this reason, the final thesis remains centered on the one-hour-ahead case, while the six-hour result is treated as a supporting sensitivity check rather than a core analytical branch.

## 31. Final overall conclusion

The final thesis supports these conclusions:

1. A defensible hourly cooling-demand target can be engineered from fragmented cumulative subsystem telemetry.
2. Short-term cooling demand can be forecast strongly using lagged tree-based models.
3. The final selected forecasting model is **Gradient Boosting**, with **Random Forest** retained as the strongest secondary benchmark and interpretive model.
4. The site does **not** show evidence of large-scale systematic annual overcooling.
5. Raw warm-season positive residuals exist, but the realistic intervention margin becomes very small under conservative supervisory filters.
6. Intelligent control can be formulated credibly as **forecast-informed intelligent control simulation** under real telemetry constraints.
7. The addition of `internal_temp_proxy` improved the contextual design and made the final intelligent control section stronger.
8. The final intelligent control family is:
   - baseline simple threshold
   - forecast-only
   - load-aware
   - load-and-mode-aware
9. The strongest final controller, **S3**, acts in a small and refined set of hours and uses more nuanced action intensity than S2.
10. The final substantive conclusion is not that the site hides large reducible cooling waste, but that it contains a **small, bounded, and defensibly identifiable set of forecast-informed intelligent control opportunities**.

---

## 32. Final thesis positioning

The thesis develops a defensible framework that bridges reconstructed cooling-demand telemetry, forecast-based benchmarking, and intelligent control simulation in a real telecom central office. It shows that broad reducible cooling demand is not evident at the selected site, but that a small number of bounded and context-aware intervention opportunities can still be identified using forecast-informed intelligent control logic.