# Transfusion Medicine & Hemovigilance Guide

Transfusion data look like ordinary clinical data and are not. Three structural facts drive
most of the ways these analyses fail review: (1) the **denominator is ambiguous** — units
issued, units transfused, transfusion episodes, and patients transfused are four different
rates; (2) **exposure accrues after baseline and is caused by the outcome's own risk
factors** — sicker, bleeding, longer-surviving patients receive more units; and (3) **donors
and recipients are cross-classified**, not nested — one recipient receives units from many
donors and one donor's components reach many recipients. This guide produces an analysis that
survives those three facts. The general survival, propensity-score, and missing-data rules
still apply; load those guides alongside this one.

---

## When to Use

- **Hemovigilance / adverse-reaction surveillance** — reaction rates by component type,
  reaction category (FNHTR, allergic, TACO, TRALI, TAD, hemolytic, TTI), severity, imputability.
- **Recipient-outcome studies of a transfusion exposure** — transfused vs not, dose (units),
  strategy (restrictive vs liberal Hb threshold), component ratio (plasma:RBC:platelet in
  massive transfusion), product attributes (storage duration, donor sex/parity/age, pathogen
  reduction, irradiation, washing).
- **Donor-side studies** — donor iron/ferritin, deferral, adverse donation events, repeat-donor
  health outcomes.
- **Alloimmunization and refractoriness** — RBC alloantibody incidence, HLA-mediated platelet
  refractoriness, corrected count increment (CCI).
- NOT for: diagnostic accuracy of a coagulation / viscoelastic assay →
  `analysis_guides/diagnostic_accuracy.md`; hemoglobin trajectories alone →
  `analysis_guides/repeated_measures.md`.

---

## 1. Name the denominator before computing any rate

A hemovigilance rate is uninterpretable until its denominator is stated. State in Methods which
of these is used, and do not mix them across components or years:

| Denominator | Answers | Typical unit |
|---|---|---|
| Components **issued** / distributed | Supplier-level safety of the product | per 100,000 components |
| Components **transfused** | Recipient risk per exposure | per 100,000 components |
| Transfusion **episodes** | Risk per clinical administration event (a multi-unit episode is one) | per 1,000 episodes |
| Patients **transfused** | Person-level risk over the study window | per 1,000 patients |

- Rare events need **exact (Poisson / Garwood) CIs**, not Wald. With **zero events**, report the
  one-sided 95% upper bound (≈ 3/n, the rule of three), never "0% (0–0)".
- Compare rates across component types or periods with a **Poisson / negative-binomial model with
  a log(denominator) offset**, checking overdispersion across sites or years.
- Passive surveillance under-reports. Report the rate for **all reported reactions** and for
  **imputability probable/definite (or "likely/certain")**, and state which surveillance
  definition was applied (for example the ISBT / IHN definitions, the 2018-revised TACO criteria,
  the 2019 TRALI redefinition). A rate shift that coincides with a definition change, a switch
  from passive to active surveillance, or a new reporting system is an artefact until shown
  otherwise; report it as a segmented / interrupted time series with the change point, not as a
  before–after contrast.

```python
# Exact Poisson rate per 100,000 components, with a zero-event upper bound
from scipy.stats import chi2

def poisson_rate(events: int, denom: int, per: int = 100_000, alpha: float = 0.05):
    if events == 0:                                   # one-sided upper bound (~3/n at alpha=0.05)
        return 0.0, 0.0, chi2.ppf(1 - alpha, 2) / 2 / denom * per
    lo = chi2.ppf(alpha / 2, 2 * events) / 2          # two-sided Garwood interval
    hi = chi2.ppf(1 - alpha / 2, 2 * (events + 1)) / 2
    return events / denom * per, lo / denom * per, hi / denom * per

print(poisson_rate(7, 412_350))   # rate, ci_lower, ci_upper per 100,000 (synthetic numbers)
print(poisson_rate(0, 58_900))    # zero events: report the upper bound only
```

---

## 2. Recipient outcomes: exposure is time-varying and confounded by indication

**Immortal-time bias.** Classifying patients as "ever transfused" at admission or surgery credits
the transfused group with the time they had to survive in order to be transfused. It biases
toward either direction depending on when death occurs relative to transfusion. Use one of:

- a Cox model with transfusion as a **time-varying covariate** (`survival::tmerge` in R, or a
  counting-process long format in `lifelines.CoxTimeVaryingFitter`);
- a **landmark** analysis (exposure fixed at a stated landmark; exclude those with the event before it);
- a **target trial emulation** with clone–censor–weight when the question is a strategy
  ("transfuse at Hb < 7 vs < 9 g/dL"). Report against **TARGET** (`/check-reporting`).

**Confounding by indication and reverse causation.** Bleeding, anemia severity, and illness
severity cause both transfusion and death. "More units → higher mortality" is expected under the
null and is **not** a dose–response. Hemoglobin is a time-varying confounder that is itself
affected by prior transfusion, so conditioning on it in a standard regression is wrong; a strategy
comparison needs g-methods (IPW marginal structural model, parametric g-formula) or trial data.

**Survivor bias in ratio analyses.** In massive transfusion, a patient who dies early has had no
time to receive thawed plasma or platelets, so a high plasma:RBC ratio is partly a marker of
having survived long enough. Compute the ratio as a **time-varying** exposure over fixed windows
(for example cumulative ratio at 3, 6, 24 hours) and exclude deaths before the window, or use the
randomized evidence; never analyze the 24-hour ratio against 24-hour mortality as a fixed covariate.

```r
# Time-varying transfusion exposure with survival::tmerge (synthetic column names)
library(survival)
base <- tmerge(pts, pts, id = id, death = event(fu_days, died))           # one row per patient
tv   <- tmerge(base, tx, id = id, transfused = tdc(tx_day))               # switches 0 -> 1 at first unit
tv   <- tmerge(tv,  tx, id = id, units = cumtdc(tx_day, n_units))         # cumulative units, if needed
fit  <- coxph(Surv(tstart, tstop, death) ~ transfused + age + sofa, data = tv, cluster = id)
```

---

## 3. Product-attribute studies: donors and recipients are cross-classified

Studies of storage duration, donor sex / parity / age, or donor–recipient sex mismatch assign an
exposure that lives on the **unit**, to an outcome that lives on the **recipient**.

- **Pre-specify the recipient-level exposure summary** (any exposure, proportion of exposed units,
  maximum / mean storage age, first unit only) and the exposure window. Picking the summary after
  seeing results is a multiplicity problem; report the others as sensitivity analyses.
- **Adjust for the number of units.** A recipient of 20 units has more opportunities to receive
  an "exposed" unit than a recipient of 1 unit; the number of units is a strong confounder of any
  "any exposed unit" comparison. Adjust for it flexibly (splines or categories) or restrict to
  recipients with the same number of units.
- **Blood group and inventory drive unit attributes.** ABO/RhD group, hospital, and calendar time
  determine which units are available, so they confound storage-age and donor-attribute contrasts.
  Adjust for or stratify on them.
- **Correlation structure.** A donor contributing to many recipients correlates their outcomes.
  Use a cross-classified random-effects model (`lme4::glmer(... + (1|recipient) + (1|donor))`) or
  cluster-robust SEs on the dominant cluster, and state which. Ignoring the donor dimension
  overstates precision when prolific donors are common.

---

## 4. Donor studies

- **Healthy donor effect.** Donors are selected for health and repeat donors are further
  selected by continued eligibility. Donor-vs-general-population comparisons are biased toward
  better outcomes; compare within donors (by donation intensity) and handle deferral and
  lapse as informative drop-out rather than ignorable censoring.
- **Repeated measures.** Ferritin and hemoglobin across donations are repeated within donor; use
  mixed models or GEE (`analysis_guides/repeated_measures.md`), and model inter-donation interval
  explicitly, since it is chosen partly because of prior values.

---

## 5. Alloimmunization and platelet refractoriness

- **Detection depends on the screening schedule.** Antibodies can be evanescent; an antibody
  that develops and falls below detection between screens is missed. Incidence is
  interval-censored at the screen dates and should be reported per patient with the screening
  schedule stated, with death as a **competing risk** (`analysis_guides/survival.md`).
- **Denominator**: patients exposed (with a stated number of units and follow-up), not units.
  Report cumulative incidence by cumulative number of units received, not a crude proportion.
- **Refractoriness**: define with CCI (or percent platelet recovery) and the post-transfusion
  sampling time (for example 10–60 min and 18–24 h), requiring at least two consecutive
  ABO-compatible transfusions; state how non-immune causes (fever, sepsis, splenomegaly, DIC,
  drugs) were handled.

---

## Reporting

- Registry / hemovigilance database / EHR → **STROBE + RECORD**; strategy emulation → **TARGET**;
  randomized threshold or product trials → **CONSORT**. Run `/check-reporting`.
- Every rate states its numerator definition (reaction category, severity, imputability), its
  denominator (section 1 table), and the surveillance mode (passive / active).
- Every recipient-outcome estimate states how exposure time was handled (time-varying,
  landmark, emulation) and how the number of units entered the model.

## Review checklist (produce-side)

- [ ] Denominator named and consistent across groups and years
- [ ] Exact Poisson CIs; zero-event rates reported as an upper bound
- [ ] Surveillance definition and imputability threshold stated; definition changes modeled as change points
- [ ] No "ever transfused" exposure fixed at baseline (immortal time)
- [ ] Units-transfused not presented as a dose–response without addressing confounding by indication
- [ ] Ratio exposures computed time-varying, with early deaths handled
- [ ] Unit-attribute exposure summary pre-specified; number of units, ABO group, site, and calendar time adjusted
- [ ] Donor clustering handled or its omission justified
