# Experiment summary

## 1. Thesis topic and positioning

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

This positioning is consistent with both the final analytical workflow and the abstract, which states that rule-based operational strategies are evaluated using historical datasets and model predictions to analyse potential improvements in cooling energy efficiency. :contentReference[oaicite:1]{index=1}

---

## 2. Site, source data, and signal screening

The final selected site is:

- **790ZHB**

The broader DB-related source structures considered during discovery included:

- `enmos_datapoint`
- `datapoint`
- `measurement`
- `measurement_all`

The site-selection workflow used lightweight metadata and support information before heavy extraction. The final recommended site was **790ZHB**, because it had the strongest cooling- and energy-related signal environment and the best combined metadata/support score. In the notebook, 790ZHB had **567 ENMOS signals**, including **515 cooling-related** and **548 energy-related** signals, compared with much lower richness for the next candidate 790LIM. :contentReference[oaicite:2]{index=2}

Signal mapping then showed:
- `datapoint_total = 223`
- `enmos_total = 567`
- `matched_signals_inner = 223`
- `datapoint_without_match = 0`
- `enmos_without_match = 344`

This justified anchoring the experiment on the datapoint-mapped signal set.

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
6. apply light quantile clipping,
7. produce final hourly cooling-demand target `y`.

The final target was clipped at the **1st and 99th percentiles** after differencing and aggregation.

This leads to the final interpretation that the target is:

> **an engineered hourly cooling-demand variable reconstructed from fragmented cumulative subsystem telemetry**

and this is one of the core methodological contributions of the thesis.

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

### Compare dataset
The compare dataset extends the main dataset with sibling cooling-related proxy signals to test whether target-adjacent variables materially improve forecasting.

### Main conclusion from this comparison
The sibling-expanded compare dataset improved RMSE only **marginally** relative to the main dataset. In the Random Forest lagged comparison:

- `compare` RMSE = **261.214454**
- `main` RMSE = **261.884303**

This improvement was too small to justify the reduced interpretability and higher target-proximity risk. Therefore, the **main dataset** remained the preferred analytical basis. :contentReference[oaicite:3]{index=3}

This also became an important argument that the selected 5-signal target is **sufficiently representative** and not severely too narrow.

---

## 5. Weather and contextual variables

External weather was retained because the internal meteo candidates were too sparse for direct use as reliable final predictors.

The retained external weather variables in the final main dataset were:

- `wx_temp_2m`
- `wx_dewpoint_2m`
- `wx_shortwave_rad`
- `wx_rh_2m`
- `wx_cloud_cover`

Time/context variables included:
- `hour`
- `dayofweek`
- `month`
- `is_weekend`

The final weather correlations with the target in `dataset_main` were:

- `wx_temp_2m`: **0.850695**
- `wx_dewpoint_2m`: **0.813200**
- `wx_shortwave_rad`: **0.409636**
- `wx_cloud_cover`: **-0.259317**
- `wx_rh_2m`: **-0.412941**

These results support the interpretation that cooling demand depends on both environmental conditions and internal operational behavior. :contentReference[oaicite:4]{index=4}

---

## 6. Internal temperature branch: final decision

A sparse internal temperature signal was reconsidered during the analysis and compared against external weather. The comparison later showed a strong relationship with external temperature:

- `corr(internal_temp, external_temp) = 0.958923`
- `mae = 1.483250`

This supported the conclusion that the internal signal tracked the external temperature context too closely and did **not** provide sufficiently distinct internal-state information for the final main forecasting specification. In the notebook text, it was explicitly concluded that it was **not retained in the final main forecasting specification** and treated, at most, as a sensitivity variable rather than a core predictor. :contentReference[oaicite:5]{index=5}

Therefore, the final thesis story should **remove the internal-temperature branch from the main analytical pipeline**.

---

## 7. Final 2025 dataset size

After final cleaning:

- `dataset_main`: **8603 rows × 46 columns**
- `dataset_compare`: **8603 rows × 69 columns**

The final main dataset spans:

- **2025-01-01 01:00:00**
- to **2025-12-31 23:00:00**

After lag construction, the final lagged forecasting/control table is:

- `lagged_df`: **8435 rows × 54 columns**
- date range: **2025-01-08 01:00:00** to **2025-12-31 23:00:00**

:contentReference[oaicite:6]{index=6}

---

## 8. Exploratory data analysis

EDA showed that the cooling target is:

- strongly seasonal,
- strongly persistence-driven,
- non-Gaussian,
- and more variable during the warm season.

The monthly target distribution confirmed:
- lower winter demand,
- rising spring demand,
- elevated summer levels,
- and a decrease toward late autumn and winter.

The target also showed strong nonlinear relationships with outside temperature and dew point, supporting the need for both lagged and contextual predictors. :contentReference[oaicite:7]{index=7}

---

## 9. Forecasting pipeline

The forecasting pipeline was built in several stages.

### Initial benchmark layer
The first benchmark on a single chronological split included:
- naive persistence
- seasonal naive
- ARIMA
- SARIMA
- linear regression
- Random Forest
- LSTM

Results:

- `Naive`: MAE **180.213829**, RMSE **310.964207**
- `Random Forest`: MAE **414.638606**, RMSE **513.386080**
- `Linear Regression`: MAE **499.839085**, RMSE **641.596075**
- `Seasonal Naive`: MAE **401.951191**, RMSE **767.137918**
- `SARIMA`: MAE **796.387167**, RMSE **884.804906**
- `ARIMA`: MAE **1413.108501**, RMSE **1463.849270**
- `LSTM`: MAE **2247.237316**, RMSE **2338.217299**

This showed that naive persistence was already a very strong reference, and that the problem is strongly persistence-driven. :contentReference[oaicite:8]{index=8}

### Final lagged forecasting setup
A richer supervised setup was then built using lagged target features:

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
- LSTM multivariate
- SARIMAX + exogenous weather

On the single chronological lagged benchmark:

- `Random Forest + lags`: MAE **186.268427**, RMSE **261.884303**
- `Naive`: MAE **178.606995**, RMSE **310.448222**
- `Gradient Boosting + lags`: MAE **258.247380**, RMSE **323.813578**
- `Linear Regression + lags`: MAE **288.075342**, RMSE **355.100944**
- `LSTM multivariate (scaled)`: MAE **969.879350**, RMSE **1026.151097**

This showed that adding explicit lag structure substantially improved the supervised models. :contentReference[oaicite:9]{index=9}

---

## 10. Final forecasting model selection

The final model choice was based on **rolling-origin backtesting**, which became the strongest and most defensible validation stage.

### Rolling-origin summary
Average fold performance:

- `Gradient Boosting + lags`
  - `MAE_mean = 228.051749`
  - `RMSE_mean = 306.739060`

- `Random Forest + lags`
  - `MAE_mean = 222.487411`
  - `RMSE_mean = 308.307356`

- `Naive`
  - `MAE_mean = 254.359444`
  - `RMSE_mean = 369.302055`

- `Linear Regression + lags`
  - `MAE_mean = 287.317139`
  - `RMSE_mean = 374.264193`

- `SARIMAX + exog`
  - `MAE_mean = 604.428176`
  - `RMSE_mean = 741.967120`

Pooled metrics confirmed the same ranking:

- `Gradient Boosting + lags`: `RMSE_pooled = 316.644365`
- `Random Forest + lags`: `RMSE_pooled = 319.491879`
- `Naive`: `RMSE_pooled = 382.450216`

Therefore, the final selected forecasting model is:

- **Gradient Boosting with lagged target features**

The strongest secondary benchmark is:

- **Random Forest with lagged target features**

Random Forest remained highly competitive and more interpretable, but Gradient Boosting performed best overall under the main rolling-origin selection criterion. :contentReference[oaicite:10]{index=10}

---

## 11. Forecasting quality

The final forecast-quality summary for the selected benchmark pipeline was:

- **MAE = 115.062724**
- **RMSE = 161.786664**
- **Mean actual = 3377.843983**
- **CVRMSE = 4.789643%**
- **NMAE = 3.406395%**
- **R² = 0.985501**

### Interpretation
These values mean that:
- forecast error is small relative to the scale of the target,
- RMSE is about **4.8% of mean hourly cooling demand**,
- average absolute error is about **3.4% of mean hourly demand**,
- and the benchmark explains about **98.6% of target variance**.

This became the preferred way to express forecasting quality, instead of using a vague “95% accurate” phrasing. :contentReference[oaicite:11]{index=11}

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

This confirmed that the problem is highly persistence-driven.

### Secondary predictors
After excluding `y_lag_1`, the strongest secondary variables included:

For Random Forest:
- `wx_temp_2m`
- `90`
- `100`
- `14`
- `72`
- longer lags
- several internal operational signals

For Gradient Boosting permutation importance:
- `90`
- `wx_temp_2m`
- `72`
- `100`
- `35`
- `y_lag_12`
- `14`
- `187`
- `171`
- `108`

### Grouped importance
For Gradient Boosting:
- lagged target: **387.803936**
- internal operational: **84.769746**
- weather: **19.863015**
- time: **0.634776**

This confirmed that:
- lagged target history dominates,
- internal operational features form the strongest secondary group,
- weather contributes useful but smaller additional value,
- and time features remain weak. :contentReference[oaicite:12]{index=12}

### Feature-group ablation
Random Forest ablation results:

- `lags_weather_operational_siblings`: RMSE **261.193115**
- `lags_weather_operational`: RMSE **261.874440**
- `lags_plus_operational`: RMSE **273.307318**
- `lags_only`: RMSE **296.699644**
- `lags_plus_weather`: RMSE **305.706284**

This showed:
- lagged features alone are not sufficient for best performance,
- internal operational signals are very important,
- weather adds further refinement,
- sibling proxies add only a marginal extra gain.

Gradient Boosting ablation under a fixed split showed that `lags_only` performed best there, but because this was less consistent with the broader interpretation and the main rolling-origin results, the thesis retained:
- **RF ablation** as the primary grouped-comparison tool,
- and **GB permutation importance** as the main interpretation tool for the final benchmark model. :contentReference[oaicite:13]{index=13}

---

## 13. Benchmarking actual vs expected cooling

The final selected Gradient Boosting model was used to generate expected cooling demand, then bias-corrected for residual benchmarking.

### Full-year summary
- `total_actual = 2.849211e+07`
- `total_predicted_bias_corrected = 2.849211e+07`
- `net_difference_bias_corrected ≈ 0`

### Interpretation
There is **no evidence of large-scale systematic annual overcooling**.

### Warm-season summary
- `warm_excess_pct = 1.340705%`
- `warm_reduced_pct = 1.360980%`
- `warm_net_pct = -0.020275%`

This means the warm season is also broadly balanced around expected cooling demand. :contentReference[oaicite:14]{index=14}

A consistency check using Random Forest gave very similar warm-season results:
- `warm_excess_pct = 1.511112%`
- `warm_reduced_pct = 1.472109%`
- `warm_net_pct = 0.039003%`

So the operational benchmarking conclusion was stable across the two near-tied tree-based models. :contentReference[oaicite:15]{index=15}

---

## 14. Conservative reduction-potential analysis

The study did not treat every positive residual as reducible energy. It introduced conservative filters:
- meaningful excess threshold,
- non-critical load filter,
- persistence,
- bounded action caps.

### Warm-season filtering funnel
- `warm_hours_total = 3537`
- `positive_residual_hours = 1832`
- `meaningful_excess_hours = 458`
- `control_eligible_hours = 38`

### Reduction scenarios
- `raw_warm_excess = 1.340705%`
- `persistent_5pct_cap = 0.046356%`
- `persistent_10pct_cap = 0.057228%`

### Sensitivity analysis
Across different thresholds and persistence rules, the estimated reduction potential stayed small. For example:

- at `0.75 residual quantile`, `0.8 load quantile`, `2h persistence`, `5% cap`:
  - **38 eligible hours**
  - **0.04635%** of warm-season actual

- at stricter settings such as `0.90`, `3h`, `5% cap`:
  - **0 hours**
  - **0.00000%**

This showed that once supervisory realism is imposed, plausible intervention potential becomes very small. :contentReference[oaicite:16]{index=16}

---

## 15. Intelligent control framing

The control section is explicitly framed as:

> **forecast-informed intelligent control simulation**

It is:
- forecasting-model output translated into supervisory decision logic,
- then simulated on historical data,
- with explicit context and risk filtering.

It is **not**:
- full MPC,
- reinforcement learning,
- or actuator-level validated control.

This is the highest defensible control level supported by the available telemetry and remains consistent with the abstract, which says that rule-based operational strategies are evaluated using historical datasets and model predictions to analyse potential improvements in cooling energy efficiency. :contentReference[oaicite:17]{index=17}

---

## 16. Why not full MPC or RL

We clarified that full MPC would require:
- a validated dynamic process model,
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

Therefore, the study remains at the forecast-informed supervisory-control level.

---

## 17. Why there is no ground-truth control accuracy

Forecasting and control are validated differently.

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

This became an important defense point and should remain explicit in the discussion section.

---

## 18. Six-hour-ahead sensitivity check

Because the literature suggests that exogenous/contextual structure may become more important at longer forecast horizons, a limited six-hour-ahead sensitivity check was performed.

Under rolling-origin validation, however, the 6-hour machine-learning models did **not** outperform naive:

- `Naive_6h`
  - `MAE_mean = 428.172107`
  - `RMSE_mean = 644.081329`
  - `CVRMSE_mean = 21.193645`
  - `R2_mean = -0.136386`

- `GB_6h`
  - `MAE_mean = 597.511387`
  - `RMSE_mean = 730.251800`
  - `CVRMSE_mean = 26.941288`
  - `R2_mean = -1.795989`

- `RF_6h`
  - `MAE_mean = 668.018085`
  - `RMSE_mean = 771.431043`
  - `CVRMSE_mean = 28.576325`
  - `R2_mean = -2.176720`

So the six-hour-ahead branch was **not retained as a main thesis result**.

However, the 6-hour Random Forest feature-importance pattern suggested stronger contextual dependence, especially for broader contextual variables, which was conceptually consistent with the literature’s horizon-sensitive expectation.

Therefore:
- the final thesis remains centered on the **one-hour-ahead** case,
- while the six-hour result is treated, at most, as a limited sensitivity check.

---

## 19. Bidirectional benchmark interpretation

The benchmark framework was extended conceptually in both directions:

- **positive residuals** → candidate excess cooling / reduction side
- **negative residuals** → candidate insufficient-cooling diagnostics

The under-expected side was treated more cautiously and not interpreted as direct increase commands.

### Under-expected diagnostic result
The final diagnostic found:

- `negative_residual_hours = 4305`
- `meaningful_under_hours = 2153`
- `persistent_under_hours = 848`
- `candidate_insufficient_cooling_hours = 18`

This means that below-expected cooling does occur, but only a very small subset appears under the strict combination of:
- warm season,
- allowed hour,
- persistent under-expected residual,
- high load,
- high thermal state,
- high-risk operating mode.

So the data do **not** support a strong claim of systematic insufficient cooling. :contentReference[oaicite:18]{index=18}

---

## 20. Operational proxy selection for intelligent control

Several candidate operational proxies were tested:
- `179`
- `188`
- `80`
- `90`

These were not part of the original fixed forecasting feature pool. They were manually shortlisted from the broader hourly telemetry exploration as a **control-specific candidate set**.

### Final proxy comparison in the revised validation branch
A later proxy sensitivity check gave:

- `179`
  - `S2_active_hours = 30`
  - `S2_reduction_pct = 0.043542`
  - `S3_active_hours = 30`
  - `S3_reduction_pct = 0.041931`

- `188`
  - `S2_active_hours = 9`
  - `S2_reduction_pct = 0.010786`
  - `S3_active_hours = 5`
  - `S3_reduction_pct = 0.005317`

- `80`
  - `S2_active_hours = 9`
  - `S2_reduction_pct = 0.011077`
  - `S3_active_hours = 5`
  - `S3_reduction_pct = 0.005759`

- `90`
  - `S2_active_hours = 23`
  - `S2_reduction_pct = 0.035380`
  - `S3_active_hours = 15`
  - `S3_reduction_pct = 0.024061`

The final retained load proxy was:

- **`179`**

because it provided the strongest balance of:
- contextual usefulness,
- stable supervisory behavior,
- and interpretable operating-mode separation.

This is why proxy selection should be described as **control-specific candidate screening**, not as part of the original fixed modeling pool.

---

## 21. Intelligent control state design

The final control-state layer uses:
- `residual_bc`
- selected load proxy (`179`)
- warm-season filter
- allowed-hour filter
- persistence of above-expected demand
- thermal-context score from external weather
- operating modes

The final thermal-context score is data-driven and based on:
- `wx_temp_2m`
- `wx_dewpoint_2m`

using PCA.

### Final state summary
- `warm_season_hours = 3537`
- `allowed_hour_hours = 3159`
- `positive_residual_hours = 4130`
- `meaningful_excess_hours = 1033`
- `persistent_excess_hours = 213`
- `thermal_pca_explained_variance = 0.943328`

State counts:
- `load_state_counts = {'HIGH': 2858, 'MEDIUM': 2793, 'LOW': 2784}`
- `thermal_state_counts = {'HIGH': 2868, 'LOW': 2784, 'MEDIUM': 2783}`
- `excess_state_counts = {'NONE': 4305, 'MILD': 3097, 'STRONG': 1033}`

:contentReference[oaicite:19]{index=19}

---

## 22. Operating modes and high-risk definition

Operating modes were discovered with K-means using:

- `load_proxy`
- `thermal_context_score`
- `hour`

### K selection
Candidate K values were tested:

- `K = 2`
- `K = 3`
- `K = 4`
- `K = 5`

Diagnostic results were:

- `K=2`
  - silhouette **0.319056**
  - Calinski–Harabasz **3814.389019**
  - Davies–Bouldin **1.280317**

- `K=3`
  - silhouette **0.366267**
  - Calinski–Harabasz **4683.768571**
  - Davies–Bouldin **1.076463**

- `K=4`
  - silhouette **0.354662**
  - Calinski–Harabasz **5068.663835**
  - Davies–Bouldin **0.909915**

- `K=5`
  - silhouette **0.368762**
  - Calinski–Harabasz **5461.673664**
  - Davies–Bouldin **0.882785**

Although some metrics improved further at larger K, the final rule was designed to avoid over-fragmentation and to preserve interpretability. The final selected value remained:

- **`K = 3`**

because it was within 95% of the best silhouette and supported the clearest interpretable LOW / MEDIUM / HIGH risk structure. :contentReference[oaicite:20]{index=20}

### High-risk definition
`HIGH_RISK` does **not** mean directly observed failure. It means:

> a data-driven supervisory operating context with comparatively higher load and thermal burden, in which cooling reduction is less defensible.

Risk labels were assigned after clustering based on cluster-level burden:
- mean load proxy
- mean thermal-context score

These labels are therefore supervisory categories, not direct physical-failure states.

---

## 23. Final intelligent control strategy family

The final strategy family is:

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
- selected load proxy
- allowed-hour filtering

### `S3_load_mode_aware`
Adds:
- operating-mode awareness
- more refined action logic

Control states:
- `PROTECT`
- `NORMAL`
- `RELAXED_1`
- `RELAXED_2`

Action mapping:
- `PROTECT = 0%`
- `NORMAL = 0%`
- `RELAXED_1 = 5%`
- `RELAXED_2 = 10%`

---

## 24. Main intelligent control results

### Final direct strategy-family results in the notebook
The main strategy-family run in Section F gave:

- `S1_forecast_only`
  - active hours = **106**
  - reduction = **0.158979%**

- `S2_load_aware`
  - active hours = **25**
  - reduction = **0.035591%**

- `S3_load_mode_aware`
  - active hours = **6**
  - reduction = **0.010249%**

This is the cleanest final strategy-family result directly tied to the final F-section pipeline. :contentReference[oaicite:21]{index=21}

### Validation package results
In the validation package, the shared-condition comparison gave:

- `B0_simple_threshold`
  - active hours = **497**
  - reduction = **0.631270%**

- `S1_forecast_only`
  - active hours = **106**
  - reduction = **0.158979%**

- `S2_load_aware`
  - active hours = **25**
  - reduction = **0.035591%**

- `S3_load_mode_aware`
  - active hours = **6**
  - reduction = **0.010249%**

### Interpretation
The hierarchy is clear and defensible:

- baseline is the most aggressive,
- S1 is much more selective,
- S2 is stricter,
- S3 is the most selective and context-aware.

This is exactly the pattern expected from a conservative supervisory controller.

---

## 25. Controller-oriented validation

The intelligent-control validation package included:

### Baseline comparison
- `B0` vs `S1` vs `S2` vs `S3`

### Controller metrics
Validation table showed:

- `B0`
  - `active_control_hours = 497`
  - `relaxed_1_hours = 497`
  - `mean_action_intensity_pct = 5.0`
  - `low_risk_action_frac = 0.098592`

- `S1`
  - `active_control_hours = 106`
  - `relaxed_2_hours = 106`
  - `mean_action_intensity_pct = 10.0`
  - `low_risk_action_frac = 0.188679`

- `S2`
  - `active_control_hours = 25`
  - `relaxed_2_hours = 25`
  - `mean_action_intensity_pct = 10.0`
  - `low_risk_action_frac = 0.240000`

- `S3`
  - `active_control_hours = 6`
  - `relaxed_2_hours = 6`
  - `mean_action_intensity_pct = 10.0`
  - `low_risk_action_frac = 1.000000`

This final version shows S3 acting only in `LOW_RISK` contexts, which is a very strong supervisory result. :contentReference[oaicite:22]{index=22}

### S3 sensitivity analysis
Selected final S3 sensitivity results:

- at `0.50`, `1h`, `0.05`:
  - **16 hours**
  - **0.025262%**

- at `0.50`, `2h`, `0.05`:
  - **10 hours**
  - **0.015223%**

- at `0.75`, `2h`, `0.05`:
  - **6 hours**
  - **0.010249%**

- at `0.90`, `2h`, `0.05`:
  - **0 hours**
  - **0.000000%**

This again shows that:
- looser settings increase the opportunity,
- stricter settings shrink or eliminate it,
- and the final selected middle setting remains small but non-zero.

---

## 26. What was not retained as main evidence

### 2026 extension
A partial 2026 extension was explored, but not retained as a main thesis result because:
- 2026 ends at **2026-04-15 21:00:00**
- warm-season comparison is incomplete
- model transfer was weak

### Dwell-time validation
A dwell-time check was implemented, but it inflated active hours unrealistically and was therefore not retained as final evidence.

### Internal-temperature main branch
The internal temperature proxy was explored, but should not remain in the final main story because it was later judged too close to external weather and not sufficiently distinct as a final core predictor.

---

## 27. How accuracy and trust are framed

### Forecasting
Forecasting quality is reported using:
- MAE
- RMSE
- CVRMSE (%)
- NMAE (%)
- R²

### Intelligent control
Intelligent control should **not** be described as having exact accuracy, because there is no ground-truth optimal action sequence.

Instead, control quality is justified through:
- effectiveness,
- selectivity,
- risk-awareness,
- and robustness.

The key defense sentence is:

> Forecast accuracy is measured directly; control trustworthiness is defended through conservativeness, selectivity, and robustness.

---

## 28. Final overall conclusion

The final thesis supports these conclusions:

1. A defensible hourly cooling-demand target can be engineered from fragmented cumulative subsystem telemetry.
2. Short-term cooling demand can be forecast strongly using lagged tree-based models.
3. The final selected forecasting model is **Gradient Boosting**, with **Random Forest** retained as the strongest secondary benchmark and interpretive model.
4. The site does **not** show evidence of large-scale systematic annual overcooling.
5. Raw warm-season positive residuals exist, but the realistic intervention margin becomes very small under conservative supervisory filters.
6. Intelligent control can be formulated credibly as **forecast-informed intelligent control simulation** under real telemetry constraints.
7. The final intelligent-control family is:
   - baseline simple threshold
   - forecast-only
   - load-aware
   - load-and-mode-aware
8. The strongest final controller, **S3**, acts in a very small and carefully filtered set of hours and only in `LOW_RISK` contexts in the final validation view.
9. The bidirectional benchmark extension does not support a strong claim of systematic insufficient cooling.
10. The final substantive conclusion is not that the site hides large reducible cooling waste, but that it contains a **small, bounded, and defensibly identifiable set of forecast-informed intelligent control opportunities**.

---

## 29. Final thesis positioning

The thesis develops a defensible framework that bridges reconstructed cooling-demand telemetry, forecast-based benchmarking, and intelligent control simulation in a real telecom central office. It shows that broad reducible cooling demand is not evident at the selected site, but that a small number of bounded and context-aware intervention opportunities can still be identified using forecast-informed intelligent control logic.