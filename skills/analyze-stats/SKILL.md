---
name: analyze-stats
description: Statistical analysis for medical research papers. Generates reproducible Python/R code with publication-ready tables and figures. Supports diagnostic accuracy, inter-rater agreement, meta-analysis, survival analysis, survey data, group comparisons, regression, propensity score, and repeated measures.
triggers: statistics, statistical analysis, analyze data, run stats, table 1, demographics table, ROC curve, agreement analysis, ICC, kappa, survival analysis, Kaplan-Meier, group comparison, logistic regression, linear regression, regression, propensity score, PSM, IPTW, SIPTW, overlap weighting, repeated measures, mixed model, GEE, longitudinal, survey weighted, KNHANES, NHANES, NHIS cohort, complex survey, wOR, weighted odds ratio, claims-based, ICD-10, transfusion, hemovigilance, haemovigilance, transfusion reaction, blood product, massive transfusion, red cell storage, donor-recipient, alloimmunization, platelet refractoriness
tools: Read, Write, Edit, Bash, Grep, Glob
model: inherit
---

# Statistical Analysis Skill

You are assisting a medical researcher with statistical analyses for medical research papers.
Generate reproducible code (Python preferred, R when necessary) that produces publication-ready
tables and figures following journal standards for medical imaging research.

## Data Privacy Check

Before reading any data file, check whether it might contain Protected Health Information (PHI):

1. If `*_deidentified.*` files exist in the working directory, use those preferentially.
2. If only raw CSV/Excel files exist (no `*_deidentified.*` counterpart), warn the user (ask in the user's preferred language):
   > "Does this data contain patient identifiers (names, national ID / RRN, contact details, etc.)?
   > If so, please de-identify it first with the `/deidentify` skill."
3. If the user confirms the data is already de-identified or contains no PHI, proceed.
4. **NEVER** display raw PHI values (names, phone numbers, RRN) in your output. If you
   encounter them while reading data, warn the user and suggest running `/deidentify`.

## Reference Files

- **Templates**: `${CLAUDE_SKILL_DIR}/references/templates/` -- reusable analysis scripts
- **Analysis guides**: `${CLAUDE_SKILL_DIR}/references/analysis_guides/` -- on-demand methodology references
- **Per-type operating rules**: `${CLAUDE_SKILL_DIR}/references/analysis_specific/` -- read after Phase 2 fixes the analysis type (routing table in § Analysis-Specific Guidelines)
- **Table standards**: `${CLAUDE_SKILL_DIR}/references/table-standards/` -- journal-specific table formatting
  - `table-standards.md` -- universal rules, AMA rules, footnote system, mistakes checklist
  - `journal-profiles/` -- YAML profiles per journal (radiology, jama, nejm, lancet, eur_rad, ajr)
  - `table-types/` -- templates per table type (Table 1, diagnostic accuracy, regression, survival/Cox, agreement/reliability, meta-analysis, model comparison, incremental value, reader study (MRMC))
  - `tool-comparison.md` -- R/Python tool comparison and recommended pipelines
- **Figure style**: `${CLAUDE_SKILL_DIR}/references/style/figure_style.mplstyle`
- **Project data**: See CLAUDE.md for data locations under `2_Data/`

Read relevant templates before generating analysis code. For complex analysis types
(regression, propensity score, repeated measures), also load the corresponding guide
from `analysis_guides/` to ensure correct methodology and reporting.

## Workflow

### Phase 1: Data Assessment

1. **Read the data file** (CSV, Excel, TSV, or other tabular format).
2. **Report to the user**:
   - Shape (rows x columns)
   - Column names and inferred types (continuous, categorical, ordinal, binary, datetime)
   - Missing values per column (count and percentage)
   - First 5 rows preview
   - Unique value counts for categorical columns
3. **Identify the analysis unit**: patient, exam, lesion, image, rater, study, etc.

### Phase 2: Analysis Plan

**Precondition (observational studies).** Before proposing an analysis plan for an observational design (cohort, case-control, cross-sectional, registry, or survey), confirm that a literature-grounded variable operationalization exists — a `variable_operationalization.md` from `/define-variables`, or an equivalent codebook-backed definition table. If none exists, **warn** the user and recommend running `/define-variables` first, so exposure / outcome / covariate definitions and cutoffs are citation-backed rather than invented ad hoc from the data dictionary (ad-hoc phenotype/cutoff definitions are a common reviewer-rejection trigger for observational work — see the dictionary-first discipline). This is a WARN, not a hard block: proceed on explicit user confirmation, recording that the operationalization artifact was not available. For stricter projects, treat the missing artifact as a hard stop until `/define-variables` has run. (This mirrors the same precondition already enforced in `/write-protocol` before drafting Methods.)

Based on the data structure and research question, propose an analysis plan:

1. **Auto-detect analysis type** from the table below, or accept user specification.
2. **List specific tests** to be performed.
3. **Identify primary and secondary endpoints**.
4. **State assumptions** that will be checked (normality, homogeneity, independence).
5. **Note any data cleaning** needed (recoding, outlier handling, missing data strategy).
6. **Anchor the estimand to the research question.** If interaction/synergy/effect-modification is the question, the primary estimand is the **interaction parameter itself** (a likelihood-ratio test of the interaction term, or the interaction OR/HR on a single consistent scale) — not a main-effect OR whose CI is then read as "no synergy." If the claim is equivalence or non-inferiority, declare the margin up front (a TOST procedure, or the CI compared against a pre-stated MCID); a non-significant difference is not equivalence without a margin.

7. **Screen every categorical/binary predictor for separation — before fitting anything.**
   A predictor that perfectly predicts the outcome breaks maximum likelihood: no finite MLE
   exists. The failure is silent — `glm` does not error, it returns an odds ratio near 0 (or
   enormous), *p* ≈ 0.99, and an AUC that then gets written into a table. This is routine in
   diagnostic imaging, because the good signs are the pathognomonic ones (T2-FLAIR mismatch,
   the string sign, a halo sign): 100% specificity means an empty cell by construction.

   ```bash
   python3 "${CLAUDE_SKILL_DIR}/scripts/check_separation.py" \
     --data cohort.csv --outcome idh_mutant --auto --strict
   ```

   `COMPLETE_SEPARATION` (an empty cell) and `QUASI_SEPARATION` (a cell below the sparsity
   floor) both halt the plan. The remedy is a **design** decision, not a numerical one:
   Firth's penalised likelihood keeps one model, while a **two-stage rule** — classify the
   sign-positive cases directly, model only the sign-negative remainder — is usually the
   clinically meaningful choice for a pathognomonic sign, because a sign-positive patient is
   already diagnosed and the real question is what to do with everyone else. Decide this in
   the plan; do not discover it in the output.

Present the plan and **wait for user approval** before executing.

| Type | When to use | Python packages | R packages | Primary output |
|------|-------------|-----------------|------------|----------------|
| Table 1 (Demographics) | Baseline characteristics | pandas, scipy | tableone | Demographics table |
| Diagnostic Accuracy | Sensitivity/specificity/AUC | sklearn, scipy | pROC | ROC curve, performance table |
| Inter-rater Agreement | Multiple raters rating same items | krippendorff, pingouin | irr, psych | ICC/Kappa table |
| Meta-analysis | Pooling effect sizes across studies | -- | meta, metafor | Forest + funnel plots |
| DTA Meta-analysis | Pooling diagnostic accuracy across studies | -- | meta, metafor, mada | SROC + paired forest plots |
| Survey/Likert | Ordinal rating scales | pingouin, scipy | psych | Descriptive + reliability |
| Survival | Time-to-event outcomes | lifelines | survival | KM curves, Cox table |
| Group Comparison | Comparing 2+ groups | scipy, pingouin | -- | Test results + effect sizes |
| Correlation | Association between variables | scipy, pingouin | -- | Scatter + correlation matrix |
| Logistic Regression | Binary outcome + predictors | statsmodels, sklearn | -- | OR table, C-statistic, forest plot |
| Linear Regression | Continuous outcome + predictors | statsmodels | -- | Coefficient table, R², diagnostic plots |
| Propensity Score | Observational treatment comparison | sklearn, statsmodels | MatchIt, WeightIt, cobalt | Balance table, Love plot, weighted analysis |
| Survey-Weighted | Complex survey data (KNHANES, NHANES, KCHS) | statsmodels | survey, tableone, gWQS | Weighted Table 1, wOR table, subgroup results |
| Repeated Measures | Longitudinal / multi-timepoint data | pingouin, statsmodels | lme4, nlme, geepack | Spaghetti plot, LMM/GEE/RM ANOVA results |

For **Logistic Regression**, **Linear Regression**, **Propensity Score**, **Survey-Weighted**, and **Repeated Measures**:
load the corresponding guide from `${CLAUDE_SKILL_DIR}/references/analysis_guides/` before generating code.
For **Survey-Weighted** analysis, also load `survey_weighted.md`. For NHIS claims-based studies, load `nhis_icd10_mapping.md`.
For **transfusion-medicine / hemovigilance** data (reaction rates, transfusion as an exposure, component ratios, unit or donor attributes, donor cohorts, alloimmunization), load `transfusion_hemovigilance.md` — the denominator, immortal-time, and donor–recipient cross-classification rules there apply before any model choice.
For test selection guidance, load `${CLAUDE_SKILL_DIR}/references/analysis_guides/test_selection.md`.

### Phase 3: Execute

Generate and run a Python (preferred) or R script following these rules:

#### Script Structure

Every script MUST start with a reproducibility header:

```python
"""
Analysis: {description}
Date: {YYYY-MM-DD}
Random seed: 42
Python: {version}
Key packages: {package==version, ...}
"""
import numpy as np
import pandas as pd
np.random.seed(42)
```

#### Execution Rules

1. **Random seed**: Always `np.random.seed(42)` or `set.seed(42)`.
2. **Figure style**: Always load the matplotlib style file:
   ```python
   import matplotlib.pyplot as plt
   style_path = os.path.join(os.environ.get('CLAUDE_SKILL_DIR', '.'), 'references/style/figure_style.mplstyle')
   if os.path.exists(style_path):
       plt.style.use(style_path)
   ```
3. **Output files**: Save all outputs to the same directory as the input data, or to a
   user-specified output directory.
4. **Tables**: Save as CSV (for downstream use) AND print a formatted markdown/console version.
5. **Figures**: Save as both PDF (vector) and PNG (300 DPI).
6. **Console output**: Print a summary formatted for direct copy-paste into a Results section.

#### Assumption Checking

Before running parametric tests, always check and report:

- **Normality**: Shapiro-Wilk test (n < 50) or Kolmogorov-Smirnov (n >= 50), plus visual QQ plot
- **Homogeneity of variance**: Levene's test
- **If assumptions violated**: Use non-parametric alternatives and report why

#### Multiple Comparisons

- If running 3+ tests on the same dataset, apply Bonferroni or Benjamini-Hochberg correction.
- Always report both uncorrected and corrected p-values.
- State the correction method used.

#### Stratified & Ordinal-Trend Reporting

- **Strata disjointness gate (before any ordinal trend test).** Before running a Cochran-Armitage trend test (or any analysis that treats tiers as an ordered partition), assert the strata are mutually exclusive and exhaustive: `sum(n per stratum) == unique N` and `sum(events per stratum) == total events`. A trend test on overlapping or non-exhaustive strata is invalid. Emit the per-stratum N/event table and the reconciliation in the output (this is the analysis-side mirror of `/self-review` `check_cohort_arithmetic.py` `PARTITION_OVERLAP`).
- **Secondary stratum-HR validation checklist.** Every secondary stratum hazard/odds ratio must be reported with (a) its **reference contrast** (which category is the referent), (b) the **event count** in each stratum, and (c) a **sparse-stratum caveat** when any stratum has a low event count (a rule of thumb: < 10 events makes the estimate unstable). A bare "HR 1.55 in lean participants" without the referent and the events is uninterpretable.
- **Proportion CI lower-bound clamp.** Clamp every proportion confidence-interval lower bound to `max(0, lower)`; a zero-event Wilson/score interval can emit a negative or absurd tiny-exponent lower bound (e.g., `3.47e-16`) that is a display artifact, not a real bound. Report `0` (or `0.0%`) instead, and prefer an exact (Clopper-Pearson) interval for zero/near-zero cells.

#### Output Manifest

After all analyses complete, save `_analysis_outputs.md` in the output directory.
Use the [output format and bound binary workflow](references/analysis_run_workflow.md)
when producing the analysis outputs.

This manifest enables downstream skills (`/make-figures`, `/write-paper`) to auto-discover analysis outputs without user intervention.

For **prespecified binary predictions on independent units**, use the bundled
`scripts/run_analysis.py run` workflow described in
[`references/analysis_run_workflow.md`](references/analysis_run_workflow.md).
It executes the existing diagnostic template and embeds data/configuration/code/
output hashes, exact counts, metric-specific denominators and the reproduction
command in this same manifest. `audit` checks recorded versions without rewriting
them; `compare` separates declared context and recorded numeric equality from byte
drift. It does not select thresholds or establish study validity, privacy clearance
or reuse rights. The original synthetic example runs with
`python3 ${CLAUDE_SKILL_DIR}/scripts/demo_analysis_run.py --out demo-project`.

### Phase 3.5: Generated-Code Quality Gate

Before reporting any script as final, lint every emitted `.py`/`.R` file for the
reproducibility-hygiene "slop" that AI-generated analysis code recurrently carries:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/check_generated_code.py {script.py} --strict
# or scan a whole output directory:
python3 ${CLAUDE_SKILL_DIR}/scripts/check_generated_code.py --code-dir {analysis_dir} --strict
```

**Major findings (fix before reporting the script):**
- `MISSING_SEED` — randomness used (sampling, bootstrap, train/test split, rng) with no
  `np.random.seed` / `set.seed` / `random_state=` / `default_rng`. Non-reproducible.
- `HARDCODED_DATA_LITERAL` — a hand-typed, table-shaped numeric literal instead of
  `read_csv()`/`read.csv()` + subset. This is the data-integrity rule "never hand-type CSV
  data into scripts."
- `HARDCODED_ABS_PATH` — an absolute path literal (`/Users/`, `/home/`, `C:\`, `~/Documents`).
  Non-portable and a PII risk.
- `INPLACE_SOURCE_OVERWRITE` — writing to the same path read as input; this overwrites raw
  data. Write derived outputs to a new path ("never modify raw data").

**Flags (fix when tidying):** `DEBUG_LEFTOVER` (a `breakpoint()` / `browser()` / debug print
/ TODO marker left in) and `UNUSED_IMPORT` (a dead Python dependency).

The gate is conservative on the Major checks — it fires `HARDCODED_DATA_LITERAL` only on
genuinely table-shaped literals and `MISSING_SEED` only on a real randomness call — so it
stays quiet on legitimate analysis code. It is the analysis-side mirror of the
data-integrity and reproducibility checks `/self-review` is built to catch downstream.

### Phase 4: Report

After execution, generate manuscript-ready text:

1. **Results paragraph**: 3-8 sentences with specific numbers, formatted as:
   - Continuous: "mean +/- SD" or "median (IQR)"
   - Proportions: "n/N (XX.X%)"
   - Test results: "statistic = X.XX, p = 0.XXX"
   - Effect sizes: "Cohen's d = X.XX (95% CI: X.XX-X.XX)"
   - AUC: "AUC = 0.XXX (95% CI: 0.XXX-0.XXX)"
2. **Table/figure captions**: Draft captions referencing table/figure numbers.
3. **Methods snippet**: 2-3 sentences describing the statistical methods used, suitable for
   the Methods section.

## Statistical Reporting Rules (Always Enforced)

These rules apply to ALL analyses without exception:

1. **Exact p-values**: Report exact values (e.g., p = 0.034), not inequalities.
   Exception: report as p < 0.001 when the value is below 0.001.
2. **Confidence intervals**: Always report 95% CIs for primary endpoints.
3. **Effect sizes**: Report alongside every p-value (Cohen's d, eta-squared, odds ratio,
   risk ratio, etc., as appropriate).
4. **Parametric vs non-parametric**: Choose based on assumption checks, not convenience.
   Report the assumption test results.
5. **Multiple comparisons**: Apply and explicitly report the correction method when
   performing 3+ comparisons.
6. **Sample size reporting**: Always state n for each group/analysis.
7. **Missing data**: Report how many cases were excluded and why.
8. **Decimal places**: p-values to 3 decimals, proportions to 1 decimal, means/SDs to
   appropriate precision for the measurement.
9. **Design/power statistics are code outputs, never hand-computed.** Any minimum detectable
   effect (MDE), a-priori or post-hoc power, or required sample size that will appear in the
   manuscript MUST be emitted by this committed script — printed with its method and inputs
   (n per arm, alpha, power, allocation ratio, one/two-sided) — not computed in a side tool
   (G*Power, an online calculator) and pasted in. Use one method family consistently
   (e.g. the exact noncentral-t via `statsmodels` `TTestIndPower` or `scipy`'s `nct`); do not
   mix a normal approximation for some values with exact-t for others. A value that exists only
   in the manuscript with no script that reproduces it is the failure mode `/self-review`
   Phase 2.5a-2 is built to catch.
10. **Estimand & CI output contract.** Every primary point estimate — including quantile
    estimands (T25, median time-to-event), pooled proportions, and subdistribution HRs, not
    just ORs/HRs/AUCs — MUST be emitted together with its 95% CI. In the output CSV, carry the
    interval as explicit columns (`estimate, ci_lower, ci_upper`) or as a single text column in
    `est (lo–hi)` form; never emit a point estimate with no interval in an adjacent column.
    Round ORs/HRs/sHRs to 2 decimals and AUC/C-statistic to 3. This is the output side of the
    `/self-review` §C assertion that "all primary metrics have 95% CIs."

### Effect-Size Real-World Translation

Whenever a primary result is a correlation, a standardized coefficient, a regression slope, an
OR/HR/RR, or a Cohen's d, also report it as a **plain-language unit shift** a non-statistician can
act on. The coefficient answers "is there an association"; the translation answers "how much, in
units I use". This complements rule 3 above (report effect sizes) — it does not replace it.

**When to apply**
- Any continuous-exposure to continuous-outcome association reported as Spearman's rho, Pearson's r,
  or a standardized slope.
- Any OR/HR/RR where the audience needs an absolute-risk feel.
- Reader / expert-elicitation studies, clinical-utility framing, abstracts, and figure captions.

**Procedure**
1. **Pick an anchored contrast on the exposure**, not a 1-unit step. Default: 25th to 75th percentile
   (IQR). State both endpoints in native units.
2. **Translate to the outcome scale.**
   - For a rank/standardized association (Spearman's rho or a per-SD slope) under an approximately
     monotonic-linear assumption:
     `delta_outcome ~= ((x_p75 - x_p25) / SD_x) * |rho| * SD_outcome`.
     Report as: "going from {x_p25} to {x_p75} {units} is associated with about {delta_outcome}
     {outcome units} on average."
   - For a regression slope b: `delta_outcome = b * (x_p75 - x_p25)` (cleaner; no monotonicity caveat).
   - State the assumption explicitly; the IQR translation is a more defensible verbal guide than an
     SD-scaled one.
3. **For OR/HR/RR**, accompany the relative measure with an absolute one at a stated baseline risk:
   the absolute risk difference, and NNT = 1 / ARR (or NNH = 1 / ARI). Always state the baseline risk used.
4. **Bound the claim**: report the contrast, the assumption, and a CI on the coefficient; do not imply
   causation from a crude or unadjusted estimate.

**Worked example (synthetic)**
rho = 0.39 between a fasting marker (IQR 0.6 to 3.5 units, SD 3.05) and an index (SD 2.13):
`((3.5 - 0.6) / 3.05) * 0.39 * 2.13 ~= 0.8` -> "Going from the 25th to the 75th percentile of the
marker is associated with about 0.8 index units higher on average (monotonic-linear approximation;
crude, unadjusted)."

**Output contract (clinical-utility is a default, not an optional add-on).** Report every
primary effect in units a clinician acts on, by default — do not leave these as prose to be
added later:
- **OR/HR/RR primary outcomes** → report the relative measure **and** the absolute risk at
  a stated baseline + absolute risk difference + **NNT** (or NNH = 1/ARI), baseline risk
  explicit. A relative-only headline is incomplete.
- **Continuous outcomes** → add the IQR/clinically-anchored "Real-world translation" line
  beneath the effect size.
- **Prediction / classification (incl. medical-AI) models** → a **decision-curve /
  net-benefit** pass at the relevant threshold is standard output, not just AUC +
  calibration. An incremental claim reports added **net benefit / NRI / IDI over the
  established clinical model**, not the new model's AUC alone. See
  `references/table-standards/table-types/incremental_value.md` and the `make-figures`
  `decision_curve` exemplar (and `render_core_figures.py` for the rendered curve).

## Error Handling

- If a script fails to execute, report the error in one line, diagnose the likely cause
  (missing package, data format mismatch, wrong column name), and present a fix.
- Do NOT retry the same script more than once without modifying it or asking the user.
- If an R package is unavailable, suggest `install.packages()` and wait for user confirmation.
- For prediction models: always include calibration assessment (Brier score, calibration plot,
  or calibration slope/intercept) alongside discrimination metrics. AUC alone is insufficient.

## Output Conventions

### Tables

**Before generating any publication table**, load the journal profile and table type template:
1. Load `${CLAUDE_SKILL_DIR}/references/table-standards/journal-profiles/{journal}.yaml` if a target journal is known
2. Load `${CLAUDE_SKILL_DIR}/references/table-standards/table-types/{type}.md` for the relevant table type
3. If no journal specified, default to AMA style (Radiology profile)

**Output formats** (always generate all three):
- CSV file (for downstream use and archival)
- Console markdown rendering (for user review)
- R gtsummary code (for publication-quality Word/LaTeX export)

**Universal rules** (enforced regardless of journal):
- No vertical lines — horizontal rules only (top, below header, bottom)
- Binary variables: show only one level (e.g., Male only, not Male + Female)
- Units in column headers, not repeated in cells
- Consistent decimal places within each column
- All abbreviations defined in footnotes, self-contained per table
- Exact P values always (never "NS" or "significant")
- Name the statistical test in footnote or general note
- Variability measure always stated: mean (SD) or median (IQR)

**Journal-specific parameters** (from loaded YAML profile):
- Footnote markers: letters (AMA) vs symbols (NEJM/Lancet)
- P value format: case, leading zero, italic
- CI separator: comma (Radiology) vs "to" (JAMA/NEJM/Lancet)
- Title format: period (AMA) vs colon (Lancet)
- Abbreviation order: appearance (Radiology) vs alphabetical (JAMA)

**Footnote placement order** (universal):
1. General note (no marker) — e.g., "Data are mean (SD) unless noted"
2. Abbreviations — in order per journal convention
3. Specific notes (superscript markers) — per-cell explanations
4. Probability notes — significance thresholds (if applicable)

**gtsummary pipeline** (recommended for R table generation):
```r
theme_gtsummary_journal("{journal}")  # "jama", "lancet", "nejm"
theme_gtsummary_compact()
# ... build table ...
tbl %>% as_flex_table() %>% flextable::save_as_docx(path = "table.docx")
```

**Validation checklist** (run before finalizing any table):
- [ ] Binary variables show only one level
- [ ] Units in headers, not cells
- [ ] Consistent decimal places per column
- [ ] Statistical test named (footnote or general note)
- [ ] Effect sizes per clinically meaningful unit (per 10 years, not per 1 year)
- [ ] Reference category stated for categorical predictors
- [ ] No "NS" — exact P values only
- [ ] Abbreviations defined in footnotes

### Figures

- Format: PDF (vector, for journal) + PNG (300 DPI, for review)
- Style: Use `figure_style.mplstyle` for consistent appearance
- Font: Arial, 8-10pt
- Colors: Colorblind-safe palette
- Size: 3.5 inches (single column) or 7.0 inches (double column) width
- Always include axis labels with units

### Console Output

- Formatted for direct copy-paste into the Results section of a manuscript
- Include all numbers that would appear in the text
- Use the reporting format conventions above

## Analysis-Specific Guidelines

The per-type operating rules (templates, required outputs, pitfalls) live in
`references/analysis_specific/` and are read **after** Phase 2 has fixed the analysis type —
read only the file(s) the approved plan needs, together with the methodology guide the Phase 2
table points to.

| Analysis type(s) in the approved plan | Read | Cost if read blindly |
|---|---|---|
| Table 1, diagnostic accuracy (incl. NRI/IDI), inter-rater agreement, survey/Likert (reverse-coding guard), group comparison, correlation | `references/analysis_specific/descriptive_diagnostic.md` | ~1,400 tokens |
| Meta-analysis, DTA meta-analysis, network meta-analysis, health economic evaluation | `references/analysis_specific/evidence_synthesis.md` | ~1,500 tokens |
| Survival, interval-censored survival, competing risks | `references/analysis_specific/time_to_event.md` | ~1,700 tokens |
| Logistic / linear regression, propensity score, survey-weighted, mediation, interaction, multiple testing, Mendelian randomization, PRS, NHIS claims, burden of disease, repeated measures | `references/analysis_specific/regression_causal.md` | ~4,300 tokens |
| **Any multivariable adjustment** (covariate pitfalls: structural zeros and dose/duration variables; over-adjustment in cross-sectional outcome models) | `references/analysis_specific/regression_causal.md` § Covariate Pitfalls / § Covariate Selection | (same file) |
| Transfusion medicine / hemovigilance (reaction rates, transfusion exposure, component ratios, storage age / donor attributes, donor studies, alloimmunization) | `references/analysis_guides/transfusion_hemovigilance.md` plus the file above for the model family used | ~3,000 tokens |

**Load-on-demand**: read `${CLAUDE_SKILL_DIR}/references/analysis_specific/<file>.md` once the
Phase 2 plan is approved and before generating code for that analysis type. A plan that mixes
types (e.g. Table 1 + Cox + propensity score) reads each matching file.

## Language

- Code and output: English
- Communication with user: Match user's preferred language
- Medical terms: English only

## What This Skill Does NOT Do

- Does not fabricate or simulate data to fill gaps
- Does not choose analysis endpoints -- the user decides the research question
- Does not interpret clinical significance -- only statistical results
- Does not replace biostatistician review for complex designs (e.g., adaptive trials)

## Anti-Hallucination

- **Never fabricate variable names, dataset column names, or variable codings.** If a variable mapping is uncertain, output `[VERIFY: variable_name]` and ask the user to confirm against the data dictionary.
- **Never fabricate statistical results** — no invented p-values, effect sizes, confidence intervals, or sample sizes. All numbers must come from executed code output.
- **Never generate references from memory.** Use `/search-lit` for all citations.
- If a function, package, or API does not exist or you are unsure, say so explicitly rather than guessing.
