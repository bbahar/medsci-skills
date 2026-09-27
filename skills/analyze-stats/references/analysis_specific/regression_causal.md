# Regression, Causal-Adjustment & Population Analyses

Per-analysis-type operating rules for `/analyze-stats`, loaded on demand once Phase 2 has
fixed the analysis type (see the routing table in `SKILL.md` § Analysis-Specific Guidelines).
Paths below are relative to the `analyze-stats` skill directory.

### Logistic Regression

- **Guide**: Load `analysis_guides/regression.md` before generating code
- **Template**: `references/templates/regression.py` (set `regression_type = "logistic"`)
- Run univariable analysis first, then multivariable with clinically selected variables
- Required outputs: OR table (univariable + multivariable), C-statistic (95% CI), and **calibration** (intercept + slope + flexible plot — **not** Hosmer–Lemeshow, which is deprecated; see the calibration guide)
- **Prediction-model calibration guide**: `references/analysis_guides/calibration.md` (**load before generating code** for any model that outputs a risk used for a decision — the apparent slope of exactly 1.00 is the in-sample tell, so produce the **bootstrap optimism-corrected** slope/intercept; Van Calster's calibration levels; scaled Brier; why Hosmer–Lemeshow is dropped; produce-side of probe S7)
- Check VIF < 5, EPV >= 10 (warn if violated)
- **Nested observation units**: when rows are clustered within subjects (multiple lesions/visits per patient), use cluster-robust standard errors (`cov_type="cluster"`, `cov_kwds={"groups": id}` in statsmodels) or a mixed-effects logistic model — a naive logit CI assumes independent rows and is too narrow
- Box-Tidwell test for continuous predictor linearity
- Forest plot of adjusted ORs
- NRI/IDI if comparing models (incremental value assessment)

### Linear Regression

- **Guide**: Load `analysis_guides/regression.md` before generating code
- **Template**: `references/templates/regression.py` (set `regression_type = "linear"`)
- Required outputs: coefficient table (β, 95% CI, P), R²/adjusted R², VIF
- Always generate 4-panel diagnostic plot (residuals vs fitted, Q-Q, scale-location, leverage)
- Check assumptions: normality of residuals, homoscedasticity, multicollinearity
- Report both unstandardized β (primary) and standardized β (for effect size comparison)

### Propensity Score

- **Guide**: Load `analysis_guides/propensity_score.md` before generating code
- **Template**: `references/templates/propensity_score.py`
- Step 1: PS estimation (logistic regression)
- Step 2: Apply method (matching with caliper = 0.2 × SD logit PS, IPTW/SIPTW with stabilized weights, or overlap weighting)
- Step 3: Balance assessment — SMD < 0.10 for all covariates, Love plot mandatory
- Step 4: Weighted/matched outcome analysis with robust SE
- Step 5: Sensitivity analysis (E-value for unmeasured confounding)
- Always state the estimand (ATE/ATT/ATO) explicitly
- Recommend overlap weighting as default (no extreme weight issues)
- **SIPTW**: Stabilized IPTW variant used in emulated target trial frameworks; report effective sample size

### Survey-Weighted Analysis

- **Guide**: Load `analysis_guides/survey_weighted.md` before generating code
- **Template**: `references/templates/survey_weighted_analysis.py`
- For KNHANES/NHANES/KCHS and similar complex survey designs
- Always declare survey design (strata, cluster/PSU, weight) before analysis
- Use correct weight variable (interview vs exam vs nutrition)
- R `survey` package strongly recommended over Python for publication
- Sequential model building: Model 1 (age+sex) → Model 2 (full adjustment)
- Report weighted odds ratios (wOR) with 95% CI
- Cross-national: analyze each country separately, never pool
- Subgroup analysis: exclude the stratification variable from covariates

### Mediation Analysis

- **Guide**: Load `analysis_guides/mediation.md` before generating code
- Bootstrapped product-of-coefficients (a×b) indirect effect (R `mediation` / `CMAverse` / PROCESS); ≥2000 resamples, bias-corrected percentile CI — not the Sobel test
- **Binary outcome**: counterfactual / natural-effects decomposition (`CMAverse`, `regmedint`), not the naive OR product
- Report total, direct, indirect effects each with a bootstrap CI; **proportion mediated only with uncertainty and only when the total effect is well-estimated** (unstable / can exceed 100% when total is near-null)
- **Identification, not the bootstrap, is the issue**: mediation needs no unmeasured mediator–outcome confounding (sequential ignorability) → report an **E-value for the indirect effect** (or ρ-based sensitivity). A cross-sectional design cannot order X→M→Y — frame as association-level (review probe O13)
- Report against **AGReMA**

### Interaction & Effect Modification

- **Choose and state the scale.** A public-health / biological **synergy** claim is an **additive**-scale statement → report **RERI**, **AP** (attributable proportion), or **S** (synergy index), each **with a CI** — not only a multiplicative OR/HR product term. A non-significant multiplicative interaction is compatible with a large additive one (and vice versa)
- **"Joint association" via a combined multi-level exposure** (high/high vs low/low) shows joint *categories*, not interaction — add the product term (multiplicative) and/or RERI (additive) to claim interaction
- **Stratified-only** "stronger in A than B" is the difference-in-significance fallacy — report the formal interaction term, not two separate stratum estimates
- R `interactionR` / `epiR` for RERI/AP/S with CIs; follow Knol & VanderWeele interaction-reporting recommendations. Review-side probe: O14 in `observational_confounding.md`

### Multiple Testing & High-Dimensional Screening

- **Guide**: Load `analysis_guides/multiplicity.md` before generating code
- For agnostic many-exposure scans (ExWAS / EWAS / MWAS / proteome-/nutrient-wide) and any "screen N predictors, report the significant ones" pass
- **Match the correction to the claim**: FWER (Bonferroni / Holm / permutation-based study-wide threshold) for a confirmatory single hit; FDR (Benjamini–Hochberg q-value) for discovery — then frame as hypothesis-generating
- Report the correction method **and the number of tests `m`** (the denominator), applied to the whole tested set — never shrink `m` to the winners
- **Replication is the real safeguard**: split-half / second cohort / cross-cycle with directional concordance and a reported replication rate; a single-cohort FDR-significant scan is exploratory
- Correlated exposures → raw Bonferroni is over-conservative (permutation or effective-number-of-tests via `poolr::meff()`); a univariate hit may be a marker for a correlated cause (consider WQS / quantile g-computation / BKMR before causal reading)
- Report full results (all effect sizes + p/q), not only winners; complex surveys combine design-based SEs (`survey_weighted.md`) WITH the correction. Review-side probe: O17 in `observational_confounding.md`

### Mendelian Randomization

- **Guide**: Load `analysis_guides/mendelian_randomization.md` before generating code
- For genetic-instrument causal inference: two-sample summary-data MR, one-sample MR, MVMR, drug-target / cis-MR, non-linear MR
- **State and evidence the 3 IV assumptions**: relevance (F-statistic / R²), independence (ancestry + confounder scan), exclusion restriction (no horizontal pleiotropy — the untestable one)
- **Pre-specify the full sensitivity suite, not IVW alone**: IVW + MR-Egger (intercept = directional pleiotropy) + weighted median + weighted mode + MR-PRESSO; Cochran's Q + leave-one-out; concordance across methods is the robustness claim (R `TwoSampleMR` / `MendelianRandomization`)
- Address **reverse causation** (Steiger / bidirectional), **sample overlap** (report fraction; overlap + weak instruments inflate type-1 error), **winner's curse** (select instruments in an independent GWAS), and ancestry matching
- **Drug-target / cis-MR**: GLS-IVW for correlated cis variants + colocalization (`coloc`) + positive control + adverse-effect phenome scan. **Non-linear MR**: residual/doubly-ranked shapes can be artefactual → require negative/positive controls + extreme-stratum sensitivity
- Interpret as a **lifelong genetic-proxy effect direction**, not a clinical-intervention magnitude; report against **STROBE-MR**. Review-side probes: MR1–MR8 in `mendelian_randomization.md`

### Polygenic Risk Score (PRS / PGS)

- **Guide**: Load `analysis_guides/polygenic_risk_score.md` before generating code
- For developing/validating/applying a genome-wide polygenic score as a predictor or risk-stratifier (distinct from MR: PRS is prediction, MR is causal inference)
- **Base (discovery GWAS) and target/validation samples must be independent**; tune (P+T / LDpred2 / PRS-CS shrinkage / quantile cut) on a separate tuning set and evaluate out-of-sample (avoid overfitting / winner's curse). Tools: PRSice-2, LDpred2 (`bigsnpr`), PRS-CS / PRS-CSx, BridgePRS
- **Ancestry portability is the central issue**: report performance **separately per target ancestry** (within-group differences can rival between-group); prefer ancestry-matched / multi-ancestry discovery + PCs; do not extend a European-derived score to other ancestries without per-ancestry validation
- Report **OR/HR per SD** (CI) + quantile **absolute risk**; discrimination (C/AUC) **and** calibration (plot + slope/intercept) in the target population — discrimination ≠ calibration
- **Incremental value is the clinical crux**: report PRS **on top of** the guideline clinical model (SCORE2/QRISK3/PCE/Tyrer-Cuzick) — ΔC-statistic (CI), NRI/IDI, net benefit — not PRS-alone AUC. A screening claim needs detection-rate-at-fixed-FPR / likelihood ratio, not AUC
- Prefer prospective/incident validation (prevalent case–control overstates utility); report against **PGS-RS** / TRIPOD+AI. Review-side probes: PG1–PG8 in `polygenic_risk_score.md`

### NHIS Claims-Based Studies

- **Guide**: Load `analysis_guides/nhis_icd10_mapping.md` for disease definition patterns
- Claims-based algorithms: N-claim rule, claim+medication, look-back period
- Always specify ICD-10 code ranges, claim count requirement, and time windows
- Charlson comorbidity index: cite Quan 2005 adaptation
- Anchor covariates to most recent data prior to index date
- Sensitivity analysis: test stricter/looser disease definitions

### Burden of Disease, Decomposition & Forecasting

- **Guide**: Load `analysis_guides/burden_decomposition_forecasting.md` before generating code
- For a burden-of-disease estimate, attributable-risk (PAF / comparative risk assessment), temporal-trend (joinpoint / AAPC), decomposition (Das Gupta; Arriaga life-expectancy), or forecast (BAPC / age-period-cohort)
- **The value-add-layer playbook** — a descriptive rate is rarely publishable alone; bolt on ONE layer: decomposition (*why* the rate changed — aging vs population growth vs epidemiological change), PAF (*how much* is modifiable), joinpoint/AAPC pre-vs-post (did a datable policy/shock bend the trend), forecast (*where* it is going), Arriaga (which ages/causes drove ΔLE)
- **Uncertainty intervals, not CIs**: report a draw-based 95% UI (2.5th–97.5th percentile of 250–500 draws propagated end-to-end); a UI crossing the null means insufficient evidence for direction, not a non-significant test
- **Single-center cohort adaptation**: three layers port onto existing follow-up without new data — trend-break (reslice around a datable guideline/scanner-era change), forecast (project a serial imaging trajectory), and global framing (place the individual-level effect next to the published GBD burden from GHDx — contextualization, not re-estimation, so no GATHER trigger). The ecological "UI-replaces-confounding-control" shortcut does **not** port: individual-level data still needs the DAG / E-value / negative-control toolkit
- Report the estimate against **GATHER** (`/check-reporting`); keep burden/attribution/decomposition/forecast descriptive or associational unless a causal design (natural experiment, MR — `analysis_guides/mendelian_randomization.md`) is in place

### Repeated Measures

- **Guide**: Load `analysis_guides/repeated_measures.md` before generating code
- **Template**: `references/templates/repeated_measures.py`
- Default method: **LMM** (handles missing data, no sphericity assumption)
- RM ANOVA only if: no missing data AND few time points AND sphericity met
- GEE for: population-averaged effects or non-normal outcomes
- Always convert wide → long format first
- **Time × Group interaction is the key result** — always report and interpret
- Generate spaghetti plot (individual trajectories) + group mean trajectory plot
- For LMM: report random effects structure, covariance structure (CS/AR1/UN), AIC/BIC
- For RM ANOVA: report Mauchly's test, epsilon, correction method (Greenhouse-Geisser)
- If missing > 5%: load `analysis_guides/missing_data.md` and apply MICE before analysis

### Covariate Pitfalls: Structural Zeros & Dose/Duration Variables

Applies to any multivariable adjustment (logistic / linear / Cox / propensity-score / survey-weighted). Two coupled failure modes around a **dose/duration variable anchored to a categorical exposure** (pack-years under smoking status, grams/week under alcohol use, cessation-duration under former-smoker):

- **Structural-zero guard (do not impute):** a never-smoker's `pack_years` is a *structural zero*, not missing-at-random — the value is known to be 0 by definition of the category. Feeding it to MICE/MNAR imputation as if it were missing fabricates a non-zero dose for unexposed subjects and corrupts the exposure contrast. Before imputing any dose/duration column, set the implied zero explicitly (`IF status == 'never' THEN dose = 0`) and impute only the genuinely-missing residual among the exposed. `/clean-data` flags categorical-implied-zero contradictions (a `never` row with a NULL dose) and ships `scripts/check_structural_zero.py`.
- **Complete-case collapse warning (use status, not dose, for adjustment):** when a dose/duration variable enters a *complete-case* multivariable model, the unexposed stratum — which carries structural zeros often stored as NULL — is dropped wholesale, collapsing n (commonly 40–60%) and distorting subgroup estimates (a small stratum can shrink to a handful of subjects). For confounder adjustment use the **categorical status** variable (never/former/current); reserve the continuous **dose** for an exposed-only (e.g., ever-smoker-restricted) *secondary* analysis. Always report n before and after model fitting and confirm the denominator did not silently collapse.

### Covariate Selection: Over-adjustment in a Cross-Sectional Outcome Model

Applies to any cross-sectional / single-visit outcome regression (the exposure and outcome are measured at one time point, so temporal order is not observed). The selection rule is **causal, not statistical**:

- **Do not adjust for a consequence or mediator of the outcome.** A covariate that the outcome physiologically *drives* sits on or after the causal path; adjusting for it is over-adjustment / collider bias and removes part of the effect under study. The signature case is a renal-function outcome: with **eGFR** as the outcome, **serum uric acid** is renally excreted (a lower eGFR mechanically raises urate), so uric acid is an outcome-consequence, not a confounder; blood pressure and HbA1c are often similarly downstream. Classify each candidate covariate against a DAG as confounder / mediator / outcome-consequence / collider, and keep only confounders in the primary model.
- **"It differs in Table 1" is not a confounder-selection criterion.** Baseline imbalance by exposure justifies *considering* a variable, but a mediator or outcome-consequence stays out regardless of how imbalanced it is. A kitchen-sink "adjust for everything that differs" model is over-adjusted by construction.
- **Report the suspect-covariate sensitivity + VIF.** Make a parsimonious, history-/design-based model the primary one; report the fuller model as a sensitivity analysis that **drops** the suspect covariate (or adds it, if you start parsimonious), and show whether the headline estimate moves. Always print VIF (collinearity between an outcome-consequence and the outcome's other correlates is common) and the n actually fitted. If dropping the covariate materially changes the estimate, propagate to the abstract and conclusion.
- **Compare adjusted-vs-unadjusted on the SAME frame (extended-adjustment missingness trap).** When an extended-adjustment model adds covariates that carry missingness, the analytic n shrinks (e.g. 84 → 49 events). Comparing that adjusted estimate to the **full-frame** unadjusted/base estimate confounds *adjustment* with *case-concentrated missingness* — it can look as if "adjustment inflated the estimate" when the drift is who-was-dropped. The fair anchor is the **unadjusted estimate refit on the reduced complete-case frame** (the same rows the adjusted model used): report unadjusted-and-adjusted **on the reduced frame** alongside the full-frame estimate, and never describe "adjustment changed the estimate" from a comparison across different frames. (Equivalently, use multiple imputation so all models share one frame.)
