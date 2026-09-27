# Evidence Synthesis & Health-Economic Analyses

Per-analysis-type operating rules for `/analyze-stats`, loaded on demand once Phase 2 has
fixed the analysis type (see the routing table in `SKILL.md` § Analysis-Specific Guidelines).
Paths below are relative to the `analyze-stats` skill directory.

### Meta-analysis

- Prefer R (meta/metafor packages) for meta-analysis
- **Comparative**: `metabin()` for binary outcomes (OR/RR), `metagen()` for continuous
  - Use `method = "Inverse"`, `method.tau = "DL"`, `method.random.ci = "HK"`
  - Avoid deprecated args: `comb.fixed` → `common`, `hakn` → `method.random.ci`
- **Single-arm pooled proportion**: `metaprop()` with `sm = "PLOGIT"`, `method.ci = "CP"`
  - **Small-study test branch**: do **not** use Egger's regression for a single-arm proportion meta-analysis — funnel-asymmetry tests assume an effect-size-vs-SE relationship that does not hold for raw proportions. If a small-study assessment is needed, use Peters' test or an arcsine-based variant, and only when `k >= 10` (note underpowered otherwise)
  - **Standard output**: report `tau-squared` on the logit scale and a **95% prediction interval** (`metaprop(..., prediction = TRUE)`) in addition to the pooled estimate; the PI conveys where a future study's proportion is expected to fall under the random-effects model
- **Nested observation units**: if the proportion's unit is nested within study (e.g., per-lesion within study, per-image within patient), do **not** report a naive Wilson/binomial CI that ignores clustering — use a cluster-bootstrap or a GLMM with a random intercept per study so the CI reflects the design
- Heterogeneity: I-squared, Q test, tau-squared, and a 95% prediction interval for the random-effects pooled estimate
- Forest plot: individual studies + pooled estimate
- Funnel plot + Egger's test for publication bias (comparative effect sizes only; note: underpowered k<10)
- Sensitivity analysis: leave-one-out (`metainf()`)
- Subgroup: `update(res, subgroup = variable)`

### DTA Meta-Analysis

- Template: `references/templates/dta_meta_analysis.R`
- Prefer R (`mada`, `meta`, `metafor` packages) for DTA meta-analysis
- **Bivariate model** (Reitsma): `mada::reitsma()` — recommended over separate pooling of Se/Sp
  - Accounts for correlation between sensitivity and specificity
  - Produces SROC curve with confidence + prediction regions
- **Key outputs**: Pooled Se/Sp (95% CI), positive/negative LR, DOR, SROC AUC
- **Threshold effect**: Spearman correlation between logit(Se) and logit(FPR)
  - If significant: interpret single pooled Se/Sp with caution, emphasize SROC curve
- **Forest plots**: Paired (sensitivity + specificity side by side)
- **Publication bias**: Deeks' funnel plot asymmetry test (NOT standard funnel plot)
  - Standard funnel plots are inappropriate for DTA studies
  - Note: underpowered for k < 10
- **Dual approach** (comparative + single-arm):
  - Primary: `metabin()` for comparative studies (OR/RR)
  - Secondary: `metaprop()` with `sm = "PLOGIT"` for single-arm pooled proportion
  - Use `method = "Inverse"`, `method.tau = "DL"`, `method.random.ci = "HK"`
- **Small studies (k < 10)**: bivariate model may not converge; consider narrative synthesis
- **Alternative**: If `mada` unavailable, use `metafor::rma.mv()` with bivariate structure

### Network Meta-Analysis

- **Guide**: Load `analysis_guides/network_meta_analysis.md` before generating code
- For ≥3 interventions via combined direct + indirect evidence (incl. component NMA); pairwise machinery (search/screening/random-effects model) via the Meta-analysis section above
- **Assess transitivity before pooling**: compare effect-modifier distributions across comparisons (box plots / table) and/or network meta-regression — it is a clinical judgment, not a test
- **Test consistency** globally (design-by-treatment) AND locally (node-split / back-calculation); a **star network (no closed loops) cannot be checked** — state it; investigate the source of any inconsistency (often one trial)
- R `netmeta` (frequentist: `netsplit`, `decomp.design`, `netheat`, `netrank` P-scores, comparison-adjusted `funnel`) or Bayesian `gemtc` / `multinma` / `BUGSnet` (node-split, SUCRA, DIC)
- Present a **network plot** (node ∝ sample size, edge ∝ #trials); report global **τ²**; **ranking (SUCRA/P-score) is not a superiority test** — report it with the league table, intervals, and certainty
- Certainty **per estimate** via **CINeMA / GRADE-NMA** (downgrade indirect-only); component NMA assumes **additivity** (state/check it). Report against **PRISMA-NMA**; risk of bias via **RoB-NMA**. Review-side probes: NM1–NM8 in `network_meta_analysis.md`

### Health Economic Evaluation

- **Guide**: Load `analysis_guides/health_economic_evaluation.md` before generating code
- For cost-effectiveness (CEA), cost-utility (CUA, QALY), cost-benefit (CBA), cost-minimisation, or budget-impact analyses; trial-based or decision-model-based (decision tree, **Markov/state-transition**, discrete-event simulation)
- Compute **incremental cost ΔC, incremental effect ΔE, and the ICER = ΔC/ΔE**; with ≥3 options remove **dominated / extended-dominated** strategies before sequential ICERs; prefer **net benefit (INMB = λΔE − ΔC)** for regression/probabilistic summaries
- State and justify the **perspective, time horizon (lifetime for chronic disease), discount rate (both costs and outcomes), currency + price year**; QALYs from a named preference-based instrument + value set
- **Uncertainty is the analytic core**: one-way / **tornado** for drivers, **probabilistic sensitivity analysis (PSA)** with justified parameter distributions (beta for probabilities/utilities, gamma/log-normal for costs) → **cost-effectiveness plane + CEAC**; scenario analyses for structural choices
- R `heemod` / `dampack` / `hesim` / `BCEA` (state-transition + PSA + CEAC + EVPI), `flexsurv` for survival extrapolation. Report against **CHEERS 2022**; make the "cost-effective" conclusion conditional on a stated willingness-to-pay threshold. Review-side probes: HE1–HE8 in `health_economic_evaluation.md`
