# Time-to-Event Analyses

Per-analysis-type operating rules for `/analyze-stats`, loaded on demand once Phase 2 has
fixed the analysis type (see the routing table in `SKILL.md` § Analysis-Specific Guidelines).
Paths below are relative to the `analyze-stats` skill directory.

### Survival Analysis

- **Methodology guide**: `references/analysis_guides/survival.md` (**load before generating code** — competing risks first [naive 1−KM overestimates → produce the Aalen–Johansen/Fine–Gray CIF; cause-specific vs subdistribution for which question, produce-side of probe S3]; PH check → RMST when violated; reverse-KM follow-up + C-index variant [S6]; estimand provenance [S8])
- Table type guide: `references/table-standards/table-types/survival_results.md` (Cox results table: events/person-time, reverse-KM median follow-up, univariable + adjusted HR with CI, PH-assumption footnote, EPV/sparse-stratum and RMST-when-PH-violated rules)
- Kaplan-Meier curves with number-at-risk table
- Log-rank test for group comparison
- Cox proportional hazards: report HR (95% CI)
- **Events-per-variable (EPV) gate**: check `events / n_covariates >= 10` before fitting Cox (mirror of the logistic EPV rule). Warn if violated and fall back to a Firth/penalized Cox or profile-likelihood CIs; do not report Wald CIs from a sparse-event model as if stable
- **Nested observation units (cluster-robust CI)**: when a subject contributes more than one analysed unit (multiple lesions, both eyes, repeated episodes), pass a subject id so the HR CIs use a robust cluster-sandwich variance (`coxph(..., cluster = id)` / `robust = TRUE` in R, `cluster_col=` in lifelines, e.g. `survival_analysis.py --cluster <id>`). Treating correlated rows as independent understates the standard errors and narrows the CI artificially
- Check proportional hazards assumption (Schoenfeld residuals)
- **PH violation → do not report a single time-averaged HR.** If the Schoenfeld global test is significant (or a covariate's residual trends with time), a single Cox HR averages a changing effect and is misleading. Report a piecewise / time-stratified HR (split follow-up at a clinically sensible cut, or `tt()` time-transform), or switch to RMST difference at a fixed horizon, and state the violation explicitly
- **Horizon vs follow-up.** Do not read a KM/CIF estimate at a horizon beyond the data: if a reported time point (e.g., a 15-year cumulative incidence) exceeds the reverse-KM median follow-up, either restrict the horizon to where the risk set is non-trivial or report the number-at-risk at that horizon so the reader can judge the extrapolation
- Report median survival with 95% CI
- **Warranty period / quantile estimands (T25 etc.)**: Time to a fixed cumulative incidence. Use `quantile()` from the KM/`survfit` object and **always emit the 95% CI** (the lower/upper from `quantile(km, conf.int=TRUE)`, or a log-transformed / bootstrap CI) alongside the events/n that define it. A quantile point estimate reported without its CI is incomplete. If the event rate is below the target quantile, report "not reached" and consider Weibull parametric extrapolation (also with an interval)

### Interval-Censored Survival

When exact event times are unknown (e.g., health screening cohorts where status changes are detected at periodic visits), standard KM underestimates time-to-event. Use interval-censored methods:

- **R packages**: `icenReg` (parametric/semi-parametric IC regression), `interval` (NPMLE/Turnbull), `survival` (Surv type "interval2")
- **Turnbull estimator**: Non-parametric MLE for interval-censored data — analogous to KM but accounts for the interval between last negative and first positive observation
- **Parametric IC models**: Weibull or log-logistic via `icenReg::ic_par()`. Report shape/scale parameters and compare AIC across distributions
- **Mid-point imputation**: Simple approximation — event time = midpoint of (last negative, first positive). Acceptable as sensitivity analysis but NOT as primary method
- **When to use**: Serial measurement cohorts (e.g., health screening databases), cancer screening intervals, repeated biomarker assessments
- **Auto-trigger**: if the event date is defined by a periodic visit / scheduled re-examination (the event is detected *at* a visit, not observed exactly), interval-censoring is not optional — make an IC model the **primary** analysis, or at minimum a mandatory pre-specified sensitivity analysis, and do not present a right-censored Cox `coxph()` on visit-dated events as if the times were exact
- **Multistate / transition models**: for repeated transitions (e.g., `msm`), account for subject-level clustering with a subject random effect or a sandwich (robust) variance, and check the time-homogeneity assumption (constant transition intensities) before trusting a single rate
- **Reporting**: State the interval-censored nature of the data explicitly in Methods. Report both standard KM (for comparability with prior literature) and IC estimates (as primary or sensitivity)

### Competing Risks

When death or other events preclude the outcome of interest, standard KM overestimates cumulative incidence (treats competing events as censored). Use competing risk methods:

- **R packages**: `cmprsk` (Fine-Gray), `tidycmprsk` (tidy interface), `survival` (cause-specific Cox)
- **Cumulative incidence function (CIF)**: `cmprsk::cuminc()` — replaces 1-KM for each event type. Gray's test for group comparison
- **Fine-Gray subdistribution hazard**: `cmprsk::crr()` or `tidycmprsk::crr()` — reports subdistribution HR (sHR) with 95% CI. Interpretable as effect on CIF directly. **Check the subdistribution-PH assumption** the same way you check it for Cox (a time-interaction term on the subdistribution scale, or inspection of scaled-residual analogues); a constant sHR is an assumption, not a given. Report the cause-specific HR alongside it so the etiologic and prognostic readings are both visible
- **Cause-specific Cox**: Standard Cox censoring competing events — reports cause-specific HR. Better for etiology; Fine-Gray better for prognosis/prediction
- **When to use**: Mortality studies with multiple causes of death, cardiovascular events when non-CV death is frequent, any outcome where competing events are common (>5% of total events)
- **Reporting**: Present CIF plots (NOT 1-KM) when competing risks exist. Report both cause-specific HR and subdistribution HR when the research question is etiologic. State which competing events were defined. When a CIF is quoted at a horizon beyond the median follow-up, report the number-at-risk at that horizon (or restrict the horizon) — a CIF extrapolated past the data is not a stable estimate
