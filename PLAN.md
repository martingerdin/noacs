# Plan: `{noacs}` — Lab R Package for Trauma Research

## Purpose

Build an R package that standardizes the recurring work of the research group: cleaning trauma registry data (especially TITCO-style cohorts), running the analyses that appear across our papers, and producing reproducible study plans and manuscripts.

The package should encode lab conventions so students and collaborators get consistent eligibility rules, score calculations, descriptive tables, model workflows, and reporting defaults—without reinventing them in every project.

Repository: [`martingerdin/noacs`](https://github.com/martingerdin/noacs)  
Reference corpus: [titco.org publications](https://titco.org/publications) and open data ([titco/titco-I](https://github.com/titco/titco-I))

---

## Design principles

1. **Convention over configuration.** Sensible TITCO/lab defaults (variable names, outcome definitions, reporting style), with explicit overrides.
2. **Tidyverse-first API.** Functions take/return tibbles; pipe-friendly; clear column naming.
3. **Composable, not monolithic.** Small functions that combine into analysis pipelines; avoid one giant “do the paper” function.
4. **Transparent methods.** Document formulas, coefficient sources, and citations (ISS, RTS, TRISS, ICISS, shock index, etc.).
5. **Reproducible manuscripts.** Quarto/R Markdown templates and helpers that match the student-kit study-plan style and common journal formats.
6. **Depend on well-tested packages where possible.** Prefer `dplyr`, `tidyr`, `lubridate`, `gtsummary`, `rms`, `pROC`, `mice`, `broom` over reimplementing; wrap them with lab-specific interfaces.
7. **Tested against known values.** Unit tests for scores and cleaning rules; integration examples on the public TITCO limited dataset (or a tiny synthetic fixture derived from the codebook).

---

## Target users and workflows

| User | Typical job |
|------|-------------|
| Student / early analyst | Load data → clean → describe → simple models → draft Methods/Results |
| Experienced analyst | Custom models with lab helpers for scores, splits, missing data, validation |
| PI / coauthor | Review reproducible report; regenerate tables/figures from frozen analysis |

Canonical pipeline the package should support:

```text
import → validate → clean/recode → derive scores & times
      → define cohort/outcomes → describe → model → validate → report
```

---

## Package architecture

```text
noacs/
├── R/
│   ├── data-io.R            # import, codebook checks
│   ├── clean.R              # recodes, missingness, eligibility
│   ├── times.R              # date/time combine, delays, hours
│   ├── scores.R             # GCS, SI, RTS, ISS/NISS, TRISS, ICISS helpers
│   ├── cohort.R             # inclusion, outcomes, temporal splits
│   ├── describe.R           # Table 1 / stratified summaries
│   ├── model.R              # regression + RCS wrappers, multilevel stubs
│   ├── validate.R           # discrimination, calibration
│   ├── missing.R            # complete-case / MICE helpers
│   ├── report.R             # formatting, acknowledgements, exports
│   └── utils.R
├── inst/
│   ├── templates/           # Quarto study plan + manuscript skeletons
│   └── extdata/             # tiny synthetic demo data + codebook snippet
├── vignettes/
│   ├── getting-started.Rmd
│   ├── cleaning-titco.Rmd
│   └── manuscript-workflow.Rmd
└── tests/testthat/
```

Suggested dependencies (initial): `rlang`, `cli`, `dplyr`, `tidyr`, `tibble`, `lubridate`, `stringr`, `gtsummary`, `broom`, `pROC`, `ggplot2`.  
Suggested Suggests: `rms`, `mice`, `lme4`/`mgcv`, `quarto`, `knitr`, `gt`, `readr`.

---

## Function inventory

Names below are proposals; finalize naming during scaffolding (`snake_case`, verb-first).

### 1. Data I/O and validation

| Function | Role |
|----------|------|
| `read_titco()` | Read TITCO CSV (limited/full), normalize types using codebook metadata |
| `read_codebook()` | Load codebook as a tibble (name, label, type, values, notes) |
| `validate_against_codebook()` | Check allowed values, types, unexpected levels; return structured report |
| `label_variables()` | Attach variable labels from codebook for tables |

### 2. Cleaning and recoding

| Function | Role |
|----------|------|
| `recode_yes_no()` | Standardize `"Yes"`/`"No"` (and variants) to logical or factor |
| `recode_vital_zeroes()` | Treat documented special zeroes (unrecordable SBP, gasping RR, etc.) per codebook rules—with explicit options |
| `clean_gcs_components()` | Handle `1c` / `1t` / `1v` (and similar) codes; optionally compute total GCS |
| `collapse_moi()` | Collapse RTI subcategories → `"Road traffic injury"` (as recommended in TITCO notes) |
| `standardize_sex()`, `standardize_ti()` | Factor levels with consistent reference |
| `flag_missing_patterns()` | Summarize missingness by variable/centre for Methods text |

### 3. Times, delays, and temporal context

These map directly to recurring TITCO themes (prehospital time, “third delay”, office-hours vs after-hours).

| Function | Role |
|----------|------|
| `combine_datetime()` | Merge date + time columns → POSIXct with clear NA policy |
| `time_to_*()` helpers | e.g. injury→arrival, arrival→admission, arrival→CT, arrival→OT |
| `classify_arrival_hours()` | Office-hours vs after-hours (configurable hospital schedule) |
| `followup_status()` | Helpers for discharge / 24 h / 30-day in-hospital status when dates exist |

### 4. Clinical scores and physiology

| Function | Role |
|----------|------|
| `shock_index()`, `age_shock_index()` | HR/SBP; age-adjusted variant |
| `rts()` | Revised Trauma Score from GCS, SBP, RR (document coefficient source) |
| `iss()`, `niss()` | From AIS region scores when available; clear error if inputs incomplete |
| `triss_ps()` | Probability of survival (TRISS); blunt/penetrating coefficients; cite source |
| `iciss_*()` helpers | Support SRR lookup / multiplicative vs single-worst-injury variants used in prior papers |
| `gcs_total()` | From eye/verbal/motor with intubation/swelling rules |

Do **not** blindly duplicate all of CRAN `{traumar}`; wrap or document interoperability where their TRISS/W-score tools are sufficient, and implement only what our papers need that those packages do not cover cleanly (ICISS Indian SRRs, TITCO-specific GCS codes, etc.).

### 5. Cohort definition and outcomes

| Function | Role |
|----------|------|
| `define_cohort()` | Apply eligibility (age, blunt/penetrating, transfers, DOA exclusion, GCS thresholds, etc.) with an auditable attrition table |
| `attrition_table()` | CONSORT-style flow counts for Methods |
| `define_outcome()` | Standard outcomes: in-hospital death, death ≤24 h, death ≤30 days (while admitted), POMR flags |
| `temporal_split()` | Centre-stratified early/late split for temporal validation (as in TITCO protocols) |
| `centre_id()` utilities | Consistent centre labelling without exposing identifiable site comparisons unless permitted |

### 6. Descriptive analysis (“Table 1”)

Lab convention from protocols/papers: **medians + IQR** for continuous; **n (%)** for categorical; often stratified by outcome or exposure.

| Function | Role |
|----------|------|
| `describe_cohort()` | One-call Table 1 via `gtsummary` with lab defaults |
| `compare_groups()` | Bivariate tests with declared policy (or “descriptive only”) |
| `format_median_iqr()`, `format_n_pct()` | Inline reporting helpers for manuscript text |
| `export_table()` | Word/HTML/LaTeX via `gtsummary`/`gt` |

### 7. Modelling

| Function | Role |
|----------|------|
| `fit_logit()` | Thin wrapper returning tidy + glance objects; formula interface |
| `fit_logit_rcs()` | Logistic model with restricted cubic splines (`rms`) for age, SBP, time-to-event-style continuous predictors—matching common TITCO analysis plans |
| `fit_multilevel()` | Optional random intercept for hospital (`lme4` or `rms`) |
| `model_card()` | Capture formula, n, events, missing-data approach, spline knots for Methods |

### 8. Validation and performance

| Function | Role |
|----------|------|
| `auc_ci()` | Discrimination with CI |
| `calibration_plot()`, `calibration_slope()` | Visual + numeric calibration |
| `reclassify_summary()` | Optional; only if used in triage/prediction projects |
| `validate_temporal()` | Fit on test half → evaluate on validation half with standard metrics |

### 9. Missing data

| Function | Role |
|----------|------|
| `complete_case()` | Explicit complete-case filter + attrition note |
| `impute_mice()` | Opinionated `mice` wrapper with seed, predictor matrix helpers for common covariates |
| `pool_and_tidy()` | Pool multiply-imputed models into manuscript-ready tables |

### 10. Manuscript and reporting support

| Function / asset | Role |
|------------------|------|
| Quarto template: study plan | Align with [student-kit](https://github.com/martingerdin/student-kit) structure (Intro, Methods per STROBE/TRIPOD, etc.) |
| Quarto template: short report / paper | Methods → Results skeleton with code chunks calling `{noacs}` |
| `titco_acknowledgement()` | Standard funding/participants acknowledgement text from TITCO data policy |
| `reporting_checklist()` | Lightweight STROBE/TRIPOD item reminders (links + stubs, not a full GUI) |
| `inline_n()`, `inline_pct()`, `inline_or()` | Consistent inline Results language |
| `session_footer()` | Reproducibility chunk (package versions, dataset version) |

---

## Suggested analysis “recipes” (vignettes)

1. **Getting started** — install, load demo data, Table 1, simple mortality logistic model.
2. **Cleaning TITCO** — codebook validation, MOI collapse, GCS special codes, vital-sign zeroes, time intervals.
3. **Prediction / validation study** — temporal split, RCS logit, AUC + calibration (TRIPOD-oriented).
4. **Process / delay study** — time-to-CT or third-delay style workflow with adjusted models.
5. **Manuscript workflow** — from cleaned analysis object to Quarto paper with tables/figures.

---

## Implementation phases

### Phase 0 — Scaffold (first PR after this plan)

- `usethis::create_package()` structure, LICENSE (match lab preference; MIT or GPL-3), DESCRIPTION, README
- CI: `R CMD check` via GitHub Actions (`r-lib/actions`)
- `testthat`, `pkgdown` site skeleton
- Decide package title expansion for `{noacs}` (keep short name; document full name in DESCRIPTION)

### Phase 1 — Data cleaning + scores (highest reuse)

- Codebook validation, recodes, times, GCS/SI/RTS/(N)ISS/TRISS basics
- `define_cohort()` + `attrition_table()`
- Unit tests with synthetic fixtures
- Vignette: cleaning TITCO

### Phase 2 — Describe + core models

- `describe_cohort()`, formatting helpers
- `fit_logit_rcs()`, `temporal_split()`, `auc_ci()`, `calibration_plot()`
- Vignette: prediction study recipe

### Phase 3 — Missing data + multilevel

- MICE helpers, pooling
- Hospital random-intercept helpers
- Tests for seed stability and pool output shape

### Phase 4 — Manuscript tooling

- Quarto templates + acknowledgement helpers
- pkgdown articles; example paper repo or `inst/examples/`

### Phase 5 — Harden and release

- API freeze for 0.1.0
- News, versioning, optional CRAN prep (or GitHub-only release first)
- Student onboarding section in README linking student-kit ↔ `{noacs}`

---

## Testing strategy

- **Unit tests** for every score formula against published worked examples / hand calculations.
- **Snapshot tests** for attrition tables and Table 1 structure (not brittle p-values).
- **Fixture data**: small synthetic cohort mirroring TITCO variable names (do not ship the full 16k identifiable-adjacent dump inside the package; link to titco-I for full runs).
- **Skip-on-CRAN** heavy vignettes if needed; keep examples fast.

---

## Documentation and governance

- pkgdown reference organized by the themes above.
- Every user-facing function: `@examples`, lifecycle badge (`experimental` → `stable`).
- CONTRIBUTING.md: tidyverse style guide, `roxygen2`, conventional commits or clear messages.
- Code of collaboration: analyses that report centre-specific mortality need extra permission per TITCO data policy—package docs should warn when centre-stratified outcome exports are requested.

---

## Out of scope (for v0.1)

- Full AIS coding from free text / ICD automatic mapping pipelines (large project; integrate later or via external tools).
- Automatic journal submission portals.
- GUI / Shiny app (possible later as a separate package).
- Replacing domain packages wholesale (`rms`, `mice`, `{traumar}`).

---

## Open decisions (resolve in Phase 0)

1. **Full package name** — keep `{noacs}` as the install name; confirm subtitle (e.g. “Tools for reproducible trauma cohort analysis”).
2. **Default factor contrasts and reference levels** (e.g. male, blunt, not transferred).
3. **Office-hours definition** for multi-centre Indian hospitals (document default; allow override).
4. **TRISS coefficient set** (classic Boyd vs later updates)—pick one default, allow alternate.
5. **License and authorship** list for DESCRIPTION.
6. **Whether TITCO-specific functions live in `{noacs}` or a thin `{titco}` companion** — recommendation: keep TITCO helpers inside `{noacs}` under a clear `titco_*` prefix for v0.1 to avoid package sprawl.

---

## Immediate next steps

1. Confirm this plan (scope, naming, phase order).
2. Scaffold the package (Phase 0).
3. Implement Phase 1 functions with tests and a cleaning vignette using a synthetic TITCO-like fixture.
4. Iterate with one real student analysis as the dogfooding project.
