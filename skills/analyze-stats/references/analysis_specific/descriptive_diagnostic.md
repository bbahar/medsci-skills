# Descriptive, Diagnostic-Accuracy & Agreement Analyses

Per-analysis-type operating rules for `/analyze-stats`, loaded on demand once Phase 2 has
fixed the analysis type (see the routing table in `SKILL.md` § Analysis-Specific Guidelines).
Paths below are relative to the `analyze-stats` skill directory.

### Table 1 (Demographics)

- Template: `references/templates/table1_demographics.py`
- Table type guide: `references/table-standards/table-types/table1_demographics.md`
- Continuous variables: mean +/- SD if normal, median (IQR) if skewed
- Categorical variables: n (%)
- Binary variables: show only one level (e.g., Male n (%), not both Male and Female)
- Compare groups: t-test/Mann-Whitney for continuous, chi-square/Fisher for categorical
- Report standardized mean differences (SMD) if requested (preferred over P for PS-matched studies)
- RCTs: P values in Table 1 are usually unnecessary per CONSORT
- gtsummary `tbl_summary()` with journal theme for R pipeline

### Diagnostic Accuracy

- **Methodology guide**: `references/analysis_guides/diagnostic_accuracy.md` (**load before generating code** — every metric with a CI on a stated analysis unit; the confidence-weighted trap [unweighted-baseline AUC + monotonic-encoding check, produce-side of probe D9]; paired DeLong vs MRMC for reader-generalising claims; per-stratum admissibility [D10]; one-scale-per-comparison [D11])
- Template: `references/templates/diagnostic_accuracy.py`
- Always report: sensitivity, specificity, PPV, NPV, accuracy, AUC
- CIs: Wilson score for proportions, DeLong for AUC
- ROC curve: include diagonal reference line, AUC in legend
- If comparing models: DeLong test for AUC comparison
- Youden's index for optimal threshold when applicable
- Include calibration assessment (Brier score, calibration plot) for prediction models
- **NRI/IDI**: When comparing two models (e.g., base model vs model + AI score), report:
  - Category-based NRI (with clinically defined risk categories)
  - Continuous NRI (note: tends to be inflated — report alongside category-based)
  - IDI (Integrated Discrimination Improvement)
  - Bootstrap 95% CIs (1000+ iterations)
  - These supplement, not replace, DeLong AUC comparison
- Table type guide (added value beyond a baseline): `references/table-standards/table-types/incremental_value.md` (paired ΔAUC + DeLong CI, continuous NRI with event/non-event split, IDI, net benefit at a prespecified threshold, same-patient/calibrated-first discipline). Pairs the decision-curve exemplar `make-figures` `references/exemplar_plots/decision_curve.md`.
- Reader study (MRMC): `references/table-standards/table-types/reader_study.md` (per-reader + reader-averaged AUC with an Obuchowski–Rockette/DBM reader+case CI, per-patient vs per-lesion unit, superiority vs non-inferiority margin). Use an MRMC method (not a fixed-reader DeLong CI) for a claim that generalises to readers. Pairs `make-figures` `references/exemplar_plots/mrmc_roc.md`.

### Inter-rater Agreement

- **Methodology guide**: `references/analysis_guides/agreement_reliability.md` (**load before generating code** — the pseudoreplication trap for clustered/repeated measurements + the pseudoreplication-safe per-subject / mixed-effects code, ICC model/type selection, agreement-vs-reliability distinction; pairs with self-review probe O18)
- Table type guide: `references/table-standards/table-types/agreement.md` (ICC with model/type + CI, weighted κ for ordinal, Bland–Altman bias + LoA, reliability-vs-agreement distinction, common errors)
- Template: `references/templates/agreement_analysis.py`
- 2 raters + categorical: Cohen's kappa
- 2+ raters + categorical: Fleiss' kappa (or Krippendorff's alpha)
- Continuous: ICC (specify model: one-way, two-way random/mixed; type: single/average)
- Always report interpretation labels (Landis & Koch or Cicchetti)
- Bland-Altman plot for continuous paired measurements
- Bootstrap CIs (1000 iterations, seed=42)

### Survey/Likert

- Descriptive: median, IQR, frequency distribution per item
- Internal consistency: Cronbach's alpha with item-total correlations
- **Reverse-coding guard (run before reliability)**: a negatively-worded scale item must be recoded `(min+max) - x` before computing the scale total or Cronbach's alpha. An un-recoded reverse item produces a *negative* item-rest correlation and a negative alpha — which is a coding bug, **not** evidence of a multidimensional construct (do not defend it as such; you lose a review round). `likert_summary.py` prints the per-item item-rest correlations, flags negative ones as reverse-code suspects, warns loudly on a negative alpha, and accepts `--reverse-items E3 ...` to apply the recode before scoring. To screen at cleaning time, run `/clean-data` `scripts/check_reverse_coding.py`. See the global rule `survey-scale-reliability.md`.
- If comparing groups: Mann-Whitney or Kruskal-Wallis (ordinal data)
- Visualization: diverging stacked bar chart

### Group Comparison

- 2 independent groups: t-test or Mann-Whitney U
- 2 paired groups: paired t-test or Wilcoxon signed-rank
- 3+ independent groups: ANOVA or Kruskal-Wallis, with post-hoc
- 3+ paired groups: repeated measures ANOVA or Friedman, with post-hoc
- Always report: test statistic, degrees of freedom, p-value, effect size

### Correlation

- Pearson r (if bivariate normal) or Spearman rho (if not)
- Report: coefficient, 95% CI, p-value
- Scatter plot with regression line and CI band
- For multiple variables: correlation matrix heatmap
