# Plan: `{noacs}` — Lab R Package for Trauma Research

## Purpose

Build an R package that standardizes recurring work in the **NOACS (No Accidents)** research lab: cleaning trauma data from multiple registries and trials, running the analyses that recur across student and faculty projects, and producing reproducible study plans and manuscripts.

The package should encode lab conventions so students and collaborators get consistent cleaning rules, clinical scores, descriptive tables, modelling workflows, and reporting defaults—without reinventing them in every repository under [noacs-io](https://github.com/noacs-io).

Repository: [`martingerdin/noacs`](https://github.com/martingerdin/noacs)

---

## Lab context (what the package must serve)

NOACS conducts research to strengthen trauma systems by predicting and preventing mortality and morbidity after trauma, primarily in **Sweden and India** ([lab description](https://ki.se/en/research/research-areas-centres-and-networks/research-groups/health-systems-and-policy-cecilia-stalsby-lundborgs-research-group/no-accidents-improving-trauma-systems)).

### Project themes visible in [noacs-io](https://github.com/noacs-io)

| Theme | Example repos | Recurring methods |
|-------|---------------|-------------------|
| Opportunities for improvement (OFI) | `patients-in-shock-ofi`, `trauma-ofi-icu`, `level-hospital-care-ofi`, `trauma-treatment-delay` | Link registry ↔ quality-review data; logistic regression; OFI categories |
| Audit filters & TQIP | `audit-filters-ofi-performance`, `tqip-cohort-outcomes`, `tqip-effect-on-mortality` | Filter performance (AUC/accuracy); TQIP clinical cohorts; ITS / DiD / GAM |
| Prediction & fairness | `predicting-ofi-in-trauma`, `ml-performance-cohorts` | Discrimination, calibration, subgroup performance |
| Student onboarding | `student-onboarding`, `starter-repo-template`, `student-repo-template` | `main.R` + `manuscript.Rmd` + `functions/`; Vancouver CSL; project-plan templates |

### Source datasets the package must accommodate

The design is **dataset-agnostic**. First-class sources:

| Source | Nature | Typical use in lab work |
|--------|--------|-------------------------|
| **TITCO** | Multicentre Indian trauma cohort (open: [titco-I](https://github.com/titco/titco-I)) | Epidemiology, scores, process delays |
| **TTRIS** | Trauma Triage Study (India; prediction/triage validation) | Model comparison (discrimination, calibration, net benefit) |
| **TAFT** | Trauma Audit Filters Trial (India; controlled ITS) | Segmented GAM / DiD; process and mortality outcomes |
| **SweTrau** | Swedish national trauma registry (Utstein-aligned) | National analyses; Karolinska extracts feed many OFI projects |
| **NTDB** | US National Trauma Data Bank | Cross-system comparisons / external context |

Additional local resources already used in lab packages (especially `{rofi}`): Karolinska trauma registry extracts, trauma care quality / peer-review databases, and KoBo Toolbox collection for prospective studies. `{noacs}` should support these via the same adapter pattern rather than hard-coding one registry’s column names into core verbs.

---

## Relationship to existing lab packages

| Package | Role today | Relationship to `{noacs}` |
|---------|------------|---------------------------|
| [`noacsr`](https://github.com/martingerdin/noacsr) | Project scaffolding (`create()`), `source_all_functions()`, MariaDB import, KoBo helpers, injury-extraction utilities | **Complement / predecessor.** Keep scaffolding & infra helpers in `{noacsr}` (or gradually migrate stable ones). `{noacs}` focuses on analysis-ready cleaning, scores, models, and reporting. |
| [`rofi`](https://github.com/martingerdin/rofi) | Karolinska OFI-specific merge/clean/`create_ofi()` / TQIP cohorts / Table 1 labelling | **Dataset-specific layer.** Generalize reusable pieces (missing-code handling, TQIP cohort logic, Table 1 defaults, inline formatters) into `{noacs}`; leave Karolinska merge keys and Swedish variable defaults in `{rofi}` or thin adapters that call `{noacs}`. |

Design rule: **core verbs never require SweTrau/TITCO/NTDB column names**. Adapters map source columns → a small canonical schema (or accept an explicit `vars` mapping argument).

---

## Design principles

1. **Source-agnostic core + thin adapters.** Analysis functions operate on a documented canonical trauma schema; each registry/trial gets a `as_*()` / `read_*()` / mapping helper.
2. **Convention over configuration.** Lab defaults for reporting (median/IQR; n/%; 5% α; 95% CI) and common outcomes, always overridable.
3. **Tidyverse-first API.** Tibbles in/out; pipe-friendly; clear names.
4. **Composable.** Small functions composing into pipelines; no monolithic “do the paper” function.
5. **Transparent methods.** Document formulas, coefficient sources, and citations for scores and validation metrics.
6. **Manuscript-ready outputs.** Align with `student-onboarding` / starter templates and patterns already used in mature repos (`inline_numbers`, `gtsummary`, Quarto/Rmd).
7. **Reuse external packages.** Wrap `dplyr`, `gtsummary`, `rms`/`mgcv`, `pROC`, `mice`, `broom`, etc.; do not reimplement.
8. **Tested.** Unit tests for scores and recodes; fixtures per source schema (tiny synthetic); optional integration vignettes against public data (e.g. TITCO limited set) or scrambled DB extracts.

---

## Canonical pipeline

```text
source import / adapter → validate → clean/recode → derive scores & times
      → define cohort / outcomes / OFI flags → describe
      → model (assoc. | prediction | ITS) → validate → report
```

---

## Canonical trauma schema (v0 draft)

Core analysis verbs should prefer these **logical fields** (exact R names TBD). Adapters populate what they can; missing fields stay `NULL`/absent with clear messaging.

| Domain | Examples |
|--------|----------|
| Identifiers | `patient_id`, `centre_id`, `encounter_id` |
| Demographics | `age`, `sex` |
| Injury | `mechanism`, `injury_type` (blunt/penetrating), `intention` |
| Physiology (prehospital / ED) | `sbp`, `hr`, `rr`, `spo2`, `gcs_total`, GCS components, `intubated` |
| Severity | `iss`, `niss`, AIS region fields / codes when available |
| Process times | `datetime_injury`, `datetime_arrival`, `datetime_ct`, `datetime_emerg_proc`, intervals derived from these |
| Care process | `transferred`, `tt_activation`, audit-filter flags, emergency procedures |
| Outcomes | `death_inhospital`, `death_24h`, `death_30d`, `gos`, `ofi`, `ofi_category`, ICU/LOS |
| Study design (optional) | `study_arm`, `study_month`, `post_intervention` (for TAFT-like designs) |

Adapters may keep original columns alongside canonical ones.

---

## Package architecture

```text
noacs/
├── R/
│   ├── schema.R             # canonical field helpers, mapping validators
│   ├── adapters-*.R         # titco, ttris, taft, swetrau, ntdb (+ generic map)
│   ├── data-io.R            # read csv/rds; codebook load/validate
│   ├── clean.R              # missing sentinels, factors, physiology fixes
│   ├── times.R              # datetimes, delays, hours-of-arrival
│   ├── scores.R             # SI, RTS, (N)ISS helpers, TRISS, GCS rules
│   ├── cohorts.R            # eligibility, attrition, TQIP-style cohorts
│   ├── ofi.R                # generic OFI construction hooks (source-mapped)
│   ├── describe.R           # Table 1 / stratified summaries
│   ├── model-assoc.R        # logistic/linear (+ RCS); multilevel stubs
│   ├── model-predict.R      # fit/compare prediction models
│   ├── model-its.R          # aggregated monthly series, GAM/ITS, DiD helpers
│   ├── validate.R           # AUC, calibration, net benefit, subgroup performance
│   ├── missing.R            # complete-case / MICE / pool
│   ├── report.R             # inline formatters, exports, acknowledgements
│   └── utils.R
├── inst/
│   ├── mappings/            # YAML/CSV column maps per source
│   ├── templates/           # Quarto/Rmd study plan + manuscript skeletons
│   └── extdata/             # tiny synthetic fixtures per source schema
├── vignettes/
│   ├── getting-started.Rmd
│   ├── adapters-overview.Rmd
│   ├── ofi-cohort-recipe.Rmd
│   ├── prediction-validation.Rmd
│   ├── interrupted-time-series.Rmd
│   └── manuscript-workflow.Rmd
└── tests/testthat/
```

Suggested Imports: `rlang`, `cli`, `dplyr`, `tidyr`, `tibble`, `lubridate`, `stringr`, `purrr`, `gtsummary`, `broom`, `ggplot2`.  
Suggested Suggests: `rms`, `mgcv`, `mice`, `pROC`, `lme4`, `gt`, `quarto`, `knitr`, `readr`, `yaml`, `DiagrammeR` (flowcharts).

---

## Function inventory

Names are proposals (`snake_case`, verb-first). Prefer generic verbs; source-specific entry points live under adapters.

### 1. Schema and adapters

| Function | Role |
|----------|------|
| `trauma_schema()` | Document/list canonical fields and types |
| `validate_mapping()` | Ensure a column map covers required fields for a task |
| `apply_mapping()` | Rename/coerce source data → canonical (+ keep originals option) |
| `read_codebook()` / `validate_against_codebook()` | Generic codebook checks (value sets, types) |
| `adapt_titco()` | Map TITCO codebook names → canonical |
| `adapt_ttris()` | Map TTRIS export → canonical |
| `adapt_taft()` | Map TAFT export → canonical (incl. study design fields) |
| `adapt_swetrau()` | Map SweTrau / Utstein-style extracts → canonical |
| `adapt_ntdb()` | Map common NTDB fields → canonical |
| `adapt_custom()` | User-supplied mapping tibble/YAML |

Heavy Karolinska multi-table merges stay in `{rofi}` (or a future adapter package); `{noacs}` consumes the merged frame via `adapt_swetrau()` / custom map.

### 2. Cleaning and recoding

| Function | Role |
|----------|------|
| `recode_missing_sentinels()` | Configurable 99/999/9999 (and source-specific exceptions, as in `{rofi}` cleaning) |
| `recode_yes_no()` | Logical/factor standardization |
| `clean_gcs()` | Totals/components; intubation / unattainable codes with explicit policy |
| `clean_vitals()` | Unrecordable / gasping / feeble-pulse conventions via policy object |
| `collapse_mechanism()` | Optional collapsing of detailed MOI into analysis categories |
| `standardize_sex()` / `standardize_injury_type()` | Consistent factors and reference levels |
| `flag_missing_patterns()` | Missingness by variable/centre for Methods |

### 3. Times and process measures

| Function | Role |
|----------|------|
| `combine_datetime()` | Date + time → POSIXct with NA policy |
| `interval_hours()` / `interval_minutes()` | Generic delay calculation |
| `add_care_intervals()` | Injury→arrival, arrival→CT, arrival→emerg proc, etc., from canonical datetimes |
| `classify_arrival_hours()` | Office-hours vs after-hours (schedule argument) |

### 4. Clinical scores and physiology

| Function | Role |
|----------|------|
| `shock_index()`, `age_shock_index()` | HR/SBP variants |
| `rts()` | Revised Trauma Score (document coefficients) |
| `iss()`, `niss()` | From AIS inputs when present |
| `triss_ps()` | TRISS Ps with selectable coefficient set |
| `gcs_total()` | From components + intubation rules |
| `add_tqip_cohorts()` | Blunt multisystem ± TBI, isolated severe TBI, severe penetrating (generalized from `{rofi}`) |

Interop with CRAN `{traumar}` where sufficient; implement lab-needed gaps only.

### 5. Cohort definition, OFI, outcomes

| Function | Role |
|----------|------|
| `define_cohort()` | Eligibility rules as composable predicates; auditable attrition |
| `attrition_table()` / `flowchart_data()` | CONSORT-style counts for DiagrammeR/consort |
| `define_outcome()` | Standard outcomes: in-hospital / 24h / 30d death; GOS unfavorable; etc. |
| `define_ofi()` | Generic constructor taking mapped review fields (defaults can mirror `{rofi}` logic) |
| `temporal_split()` | Centre-stratified early/late split for temporal validation |
| `make_monthly_series()` | Aggregate patient-level data for ITS (TAFT-style) |

### 6. Descriptive analysis

| Function | Role |
|----------|------|
| `describe_cohort()` | Table 1 via `gtsummary` with lab defaults |
| `compare_groups()` | Optional bivariate tests with declared policy |
| `format_median_iqr()`, `format_n_pct()`, `format_or_ci()` | Inline Results helpers (as reused across `tqip-cohort-outcomes` etc.) |
| `export_table()` | Word/HTML/LaTeX |

### 7. Modelling

**Association / cohort studies**

| Function | Role |
|----------|------|
| `fit_logit()` / `fit_linear()` | Thin tidy wrappers |
| `fit_logit_rcs()` | Logistic + restricted cubic splines (`rms`) for continuous predictors |
| `fit_multilevel()` | Optional centre random intercept |

**Prediction studies (TTRIS-oriented, also OFI prediction)**

| Function | Role |
|----------|------|
| `evaluate_discrimination()` | AUC + CI |
| `evaluate_calibration()` | Slope/intercept + calibration plot |
| `evaluate_net_benefit()` | Decision-curve helpers |
| `compare_models()` | Side-by-side metric table |
| `subgroup_performance()` | Men/women, TQIP cohorts, shock, etc. (`ml-performance-cohorts` pattern) |

**Interrupted time series / trials (TAFT-oriented)**

| Function | Role |
|----------|------|
| `fit_its_gam()` | Segmented GAM on monthly aggregates (`mgcv`) |
| `fit_did()` | Difference-in-differences helper for binary/continuous outcomes |
| `format_gam_result()` | Manuscript-ready OR/CI/p strings (generalize `ReturnGamResult` pattern) |

### 8. Missing data

| Function | Role |
|----------|------|
| `complete_case()` | Explicit filter + attrition note |
| `impute_mice()` | Opinionated `mice` wrapper (seed, predictor matrix helpers; structural NA policy) |
| `pool_and_tidy()` | Rubin’s-rules pooling to manuscript tables |

### 9. Manuscript and project reporting

| Function / asset | Role |
|------------------|------|
| Quarto/Rmd templates | Study plan + manuscript aligned with `student-onboarding` / starter repos |
| `inline_*` helpers | Consistent Results language |
| `session_footer()` | Dataset version, package versions |
| `acknowledgement_*()` | Optional source-specific acknowledgement snippets (TITCO policy text; SweTrau citation note; etc.) |
| `reporting_checklist()` | STROBE / TRIPOD / CONSORT-ITS reminders with links |

Project scaffolding (`create()`, gitignore, `main.R` stubs) remains primarily `{noacsr}` / starter templates; `{noacs}` supplies the analytical verbs those templates call.

---

## Suggested vignettes / recipes

1. **Getting started** — install; synthetic fixture; Table 1; simple mortality model.  
2. **Adapters overview** — same analysis on two mocked sources via mappings.  
3. **OFI cohort recipe** — outcomes, descriptives, adjusted logit (SweTrau-like).  
4. **Prediction validation** — TTRIS-like split, AUC/calibration/net benefit.  
5. **Interrupted time series** — TAFT-like monthly series + GAM/DiD.  
6. **Manuscript workflow** — from analysis objects to Quarto/Rmd with inline numbers.

---

## Implementation phases

### Phase 0 — Scaffold

- Package skeleton, LICENSE, DESCRIPTION (NOACS title/authors), README, CI (`R CMD check`), `testthat`, pkgdown  
- Document relationship to `{noacsr}` / `{rofi}`  
- Freeze v0 canonical schema + mapping file format  

### Phase 1 — Schema, cleaning, scores

- Mapping helpers + at least two adapters (recommend **SweTrau-like** + **TITCO**, highest reuse)  
- Missing sentinels, GCS/vitals policies, care intervals, SI/RTS/(N)ISS/TRISS basics  
- `define_cohort()` + `attrition_table()`  
- Synthetic fixtures + unit tests  

### Phase 2 — Describe + association models

- `describe_cohort()`, formatters  
- `fit_logit_rcs()`, TQIP cohorts, OFI hooks  
- Vignette: OFI / registry cohort recipe  

### Phase 3 — Prediction + ITS

- Validation metrics, subgroup performance  
- Monthly aggregation, `fit_its_gam()`, `fit_did()`  
- Adapters for **TTRIS**, **TAFT**, **NTDB** (as data access allows)  
- Vignettes: prediction; ITS  

### Phase 4 — Missing data + manuscript tooling

- MICE helpers; templates; acknowledgement helpers  
- Dogfood on one live [noacs-io](https://github.com/noacs-io) student project and one faculty manuscript repo  

### Phase 5 — Harden toward 0.1.0

- API freeze; NEWS; optionally have `{rofi}` depend on `{noacs}` for shared verbs  
- Student onboarding docs updated to recommend `{noacs}` for analysis  

---

## Testing strategy

- Unit tests for score formulas and sentinel recoding policies.  
- Mapping tests: each adapter maps fixture → canonical and validates types.  
- Snapshot tests for attrition and Table 1 structure.  
- Do **not** ship identifiable registry extracts; use synthetic fixtures + public TITCO limited data for examples.  
- Keep vignettes skippable when private data are unavailable.

---

## Documentation and governance

- pkgdown reference organized by: Adapters, Cleaning, Scores, Cohorts/OFI, Describe, Models, Validation, Report.  
- Lifecycle badges (`experimental` → `stable`) on user-facing functions.  
- CONTRIBUTING + tidyverse style.  
- Data-policy notes: centre-specific mortality from shared consortia may need extra permission; SweTrau multi-hospital extracts need steering-group/ethics pathways—document, don’t enforce via code alone.

---

## Out of scope (for v0.1)

- Full free-text → AIS coding pipelines.  
- Replacing `{noacsr}` scaffolding or all of `{rofi}` merges in one step.  
- Shiny GUIs.  
- Automatic journal submission.  
- Re-implementing `{traumar}` / `rms` / `mice` wholesale.

---

## Open decisions (resolve in Phase 0)

1. **Package name/subtitle** — `{noacs}` vs evolving `{noacsr}`; confirm DESCRIPTION title (“NOACS tools for trauma data analysis” or similar).  
2. **Canonical field naming** — British/American spelling, prehospital vs ED prefixes (`sbp_ed` vs `ed_sbp`).  
3. **Default TRISS coefficient set** and office-hours schedule defaults.  
4. **License and authorship** list.  
5. **Migration path** — which `{rofi}` / manuscript helpers move first vs wrap-in-place.  
6. **NTDB access pattern** — vignette against documented public/demo fields only unless lab has a redistributable subset.

---

## Immediate next steps

1. Confirm this multi-source plan and open decisions (especially schema naming and `{noacsr}`/`{rofi}` boundaries).  
2. Scaffold Phase 0.  
3. Implement Phase 1 with SweTrau-like + TITCO adapters, tests, and a cleaning vignette.  
4. Dogfood on one OFI student repo and one prediction or TAFT-style analysis.
