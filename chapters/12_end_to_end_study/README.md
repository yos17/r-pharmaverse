# Chapter 12: End-to-End Study

*Everything together. A complete clinical reporting pipeline.*

---

## What You've Built

| Ch | Topic | Key Packages |
|----|-------|-------------|
| 1 | R + tidyverse in pharma context | dplyr, haven, labelled |
| 2 | CDISC standards in code | pharmaversesdtm, metacore |
| 3 | SDTM mapping | sdtm.oak, pharmaverseraw |
| 4 | admiral fundamentals | admiral |
| 5 | ADSL derivation | admiral, metatools |
| 6 | ADAE + ADTTE | admiral, admiralonco |
| 7 | Tables + listings + graphs | rtables, tern, rlistings, ggsurvfit |
| 8 | ARDs + formatting | cards, tfrmt, gtsummary |
| 9 | XPT export | xportr, haven |
| 10 | Interactive apps | teal, teal.modules.clinical |
| 11 | Validation + QC | logrx, diffdf, renv, testthat |

Now we tie it together with a complete study pipeline.

---

## The Pipeline

```
Raw CRF Data
     ↓ {sdtm.oak}
SDTM Domains (dm, ae, ex, vs, lb, ds)
     ↓ {admiral}
ADaM Datasets (adsl, adae, adtte, adlb)
     ↓ {xportr}
XPT Transport Files
     ↓ {rtables}/{tern}/{tfrmt}
TLGs (Tables, Listings, Graphs)
     ↓ {teal}
Interactive Explorer
     ↓ {logrx}/{diffdf}
Validated Output Package
```

---

## Folder Structure

```
study001/
├── data/
│   ├── raw/              # raw CRF data (CSV/SAS)
│   ├── sdtm/             # SDTM XPT outputs
│   └── adam/             # ADaM XPT outputs
├── programs/
│   ├── sdtm/
│   │   ├── dm.R
│   │   ├── ae.R
│   │   ├── ex.R
│   │   └── ...
│   ├── adam/
│   │   ├── adsl.R
│   │   ├── adae.R
│   │   ├── adtte.R
│   │   └── adlb.R
│   └── tlg/
│       ├── t_14_1_1_demographics.R
│       ├── t_14_3_1_ae_overview.R
│       ├── t_14_3_2_ae_by_soc_pt.R
│       ├── f_14_2_1_km_os.R
│       └── l_16_2_1_ae_listing.R
├── qc/
│   ├── adam/             # independent QC programs
│   └── tlg/
├── output/
│   ├── adam/             # XPT files
│   └── tlg/              # RTF/PDF outputs
├── logs/                 # logrx logs
├── specs/                # define.xml, variable specs
└── renv.lock             # package lock file
```

---

## Program: Master Run Script

```r
# run_all.R
# Execute the complete study pipeline in order

library(logrx)
library(dplyr)

# Configuration
STUDY_DIR   <- getwd()
LOG_DIR     <- file.path(STUDY_DIR, "logs")
OUTPUT_DIR  <- file.path(STUDY_DIR, "output")
ADAM_DIR    <- file.path(OUTPUT_DIR, "adam")
TLG_DIR     <- file.path(OUTPUT_DIR, "tlg")

dir.create(LOG_DIR,   recursive=TRUE, showWarnings=FALSE)
dir.create(ADAM_DIR,  recursive=TRUE, showWarnings=FALSE)
dir.create(TLG_DIR,   recursive=TRUE, showWarnings=FALSE)

# ---- Helper: run with logging ----
run_program <- function(path, log_path = NULL) {
  prog_name <- tools::file_path_sans_ext(basename(path))
  if (is.null(log_path))
    log_path <- file.path(LOG_DIR, paste0(prog_name, ".log"))
  
  cat(sprintf("Running: %-40s ... ", basename(path)))
  start <- proc.time()
  
  tryCatch({
    logrx::axecute(path, log = log_path)
    elapsed <- (proc.time() - start)[3]
    cat(sprintf("OK [%.1fs]\n", elapsed))
    return(invisible(TRUE))
  }, error = function(e) {
    cat(sprintf("FAILED: %s\n", conditionMessage(e)))
    return(invisible(FALSE))
  })
}

# ---- Run order ----
cat("=== Study 001 — Full Pipeline Run ===\n")
cat(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), "\n\n")

# 1. SDTM
cat("--- SDTM ---\n")
for (prog in list.files("programs/sdtm", pattern="*.R", full.names=TRUE)) {
  run_program(prog)
}

# 2. ADaM
cat("\n--- ADaM ---\n")
adam_programs <- c("adsl.R", "adae.R", "adtte.R", "adlb.R")
for (prog in adam_programs) {
  run_program(file.path("programs/adam", prog))
}

# 3. TLGs
cat("\n--- TLGs ---\n")
for (prog in list.files("programs/tlg", pattern="*.R", full.names=TRUE)) {
  run_program(prog)
}

cat("\n=== Run complete ===\n")
```

---

## Program: Complete ADSL

This is the production-quality ADSL from the previous chapters assembled cleanly:

```r
# programs/adam/adsl.R
# ADSL derivation — Study 001
# Version: 1.0  Date: 2026-04-02
# Author: Yosia Hadisusanto

library(admiral)
library(pharmaversesdtm)
library(metatools)
library(xportr)
library(dplyr)
library(lubridate)
library(logrx)

log_print("=== ADSL Derivation — Study 001 ===")

# ---- Inputs ----
dm <- pharmaversesdtm::dm
ex <- pharmaversesdtm::ex
ds <- pharmaversesdtm::ds
sc <- pharmaversesdtm::sc

log_print(sprintf("Input DM: %d rows, %d subjects", nrow(dm), n_distinct(dm$USUBJID)))

# ---- Base ADSL from DM ----
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  mutate(
    TRT01P  = ACTARM,
    TRT01A  = ACTARM,
    TRT01PN = as.integer(factor(ACTARM, levels=sort(unique(ACTARM)))),
    TRT01AN = TRT01PN
  )

# ---- Dates ----
adsl <- adsl %>%
  derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC, date_imputation="first") %>%
  derive_vars_dt(new_vars_prefix="TRTE", dtc=RFXENDTC,  date_imputation="last") %>%
  derive_vars_dt(new_vars_prefix="DTH",  dtc=DTHDTC,    date_imputation="first") %>%
  mutate(
    TRTDURD = as.integer(TRTEDT - TRTSDT + 1),
    DTHFL   = if_else(!is.na(DTHDT), "Y", NA_character_)
  )

# ---- Demographics subgroups ----
adsl <- adsl %>%
  mutate(
    AGEGR1 = case_when(
      AGE < 40              ~ "<40",
      AGE >= 40 & AGE < 65  ~ "40-64",
      AGE >= 65              ~ ">=65",
      TRUE                   ~ NA_character_
    ),
    AGEGR1N = case_when(
      AGEGR1 == "<40"   ~ 1L,
      AGEGR1 == "40-64" ~ 2L,
      AGEGR1 == ">=65"  ~ 3L
    ),
    SEXN  = if_else(SEX=="M", 1L, if_else(SEX=="F", 2L, NA_integer_)),
    RACEN = case_when(
      RACE == "WHITE"                     ~ 1L,
      RACE == "BLACK OR AFRICAN AMERICAN" ~ 2L,
      RACE == "ASIAN"                     ~ 3L,
      TRUE                                ~ 99L
    )
  )

# ---- Analysis flags ----
ex_dosed <- ex %>%
  group_by(USUBJID) %>%
  summarise(DOSED = any(!is.na(EXSTDTC)), .groups="drop")

ds_last <- ds %>%
  arrange(USUBJID, DSSTDTC) %>%
  group_by(USUBJID) %>% slice_tail(n=1) %>% ungroup() %>%
  select(USUBJID, DSDECOD)

adsl <- adsl %>%
  left_join(ex_dosed, by="USUBJID") %>%
  left_join(ds_last,  by="USUBJID") %>%
  mutate(
    ITTFL    = "Y",
    SAFFL    = if_else(DOSED == TRUE, "Y", "N"),
    COMPLFL  = if_else(DSDECOD == "COMPLETED", "Y", "N"),
    PPROTFL  = if_else(SAFFL=="Y" & COMPLFL=="Y", "Y", "N"),
    EOSSTT   = case_when(
      DSDECOD == "COMPLETED"       ~ "COMPLETED",
      is.na(DSDECOD)               ~ "ONGOING",
      TRUE                          ~ "DISCONTINUED"
    )
  ) %>%
  select(-DOSED, -DSDECOD)

log_print(sprintf("ADSL rows: %d", nrow(adsl)))
log_print(sprintf("ITTFL=Y: %d", sum(adsl$ITTFL=="Y", na.rm=TRUE)))
log_print(sprintf("SAFFL=Y: %d", sum(adsl$SAFFL=="Y", na.rm=TRUE)))

# ---- Export XPT ----
adsl %>%
  xportr_write(
    path  = "output/adam/adsl.xpt",
    label = "Subject-Level Analysis Dataset"
  )

log_print("adsl.xpt written successfully")
log_print("=== ADSL derivation complete ===")
```

---

## Program: Demographics Table

```r
# programs/tlg/t_14_1_1_demographics.R
# Table 14.1.1: Summary of Demographic Characteristics
# Safety Population

library(rtables)
library(tern)
library(pharmaverseadam)
library(dplyr)
library(logrx)

log_print("Building Table 14.1.1")

adsl_safe <- pharmaverseadam::adsl %>% filter(SAFFL == "Y")
log_print(sprintf("Safety population: %d subjects", nrow(adsl_safe)))

lyt <- basic_table(
  title         = "Table 14.1.1: Summary of Demographic Characteristics",
  subtitles     = "Safety Population",
  show_colcounts = TRUE
) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  
  # Age
  analyze_vars("AGE", .stats=c("n","mean_sd","median","range","quantiles"),
               .labels=c(n="n", mean_sd="Mean (SD)", median="Median",
                         range="Min, Max", quantiles="Q1, Q3")) %>%
  
  # Age group
  count_occurrences_by_grade("AGEGR1",
    .labels = c(count_fraction="Age Group, n (%)")) %>%
  
  # Sex
  count_occurrences_by_grade("SEX",
    .labels = c(count_fraction="Sex, n (%)")) %>%
  
  # Race
  count_occurrences_by_grade("RACE",
    .labels = c(count_fraction="Race, n (%)"))

tbl <- build_table(lyt, adsl_safe)
print(tbl)

# Save as text
sink("output/tlg/t_14_1_1.txt")
cat(export_as_txt(tbl, lpp=60))
sink()

log_print("Table 14.1.1 complete → output/tlg/t_14_1_1.txt")
```

---

## What Real Statistical Programmers Do Differently

After learning the tools, the real work is about **judgment**:

**1. Read the SAP carefully**
The Statistical Analysis Plan defines every derivation rule. Admiral functions implement common rules, but the SAP may have study-specific tweaks.

**2. Handle edge cases**
What if a subject has no exposure records? What if RFXSTDTC is missing but RFSTDTC exists? What if AESTDTC is partial and the imputed date precedes treatment start? These cases don't appear in vignettes.

**3. Know when to ask**
Statisticians write SAS specs in SAS. When transitioning to R, there will be ambiguities. Ask early.

**4. Version everything**
Every program has a version header. Every dataset has a derivation date variable. Every run has a log.

**5. Document derivations**
If a variable isn't in admiral's standard functions, write a comment explaining the derivation logic. Future you (and auditors) will thank you.

---

## The Packages You Now Know

| Package | Role |
|---------|------|
| `pharmaversesdtm` | Example SDTM data |
| `pharmaverseadam` | Example ADaM data |
| `pharmaverseraw` | Example raw CRF data |
| `sdtm.oak` | SDTM programming |
| `admiral` | ADaM derivation |
| `admiralonco` | Oncology ADaM extensions |
| `metacore` | Variable metadata/spec |
| `metatools` | Metadata utilities |
| `xportr` | XPT export with metadata |
| `rtables` | Table layout engine |
| `tern` | Clinical analysis functions |
| `rlistings` | Regulatory listings |
| `cards` | Analysis Results Datasets |
| `tfrmt` | Table formatting |
| `gtsummary` | Publication tables |
| `ggsurvfit` | KM plots |
| `teal` | Interactive apps |
| `logrx` | Audit logging |
| `diffdf` | Dataset comparison/QC |
| `renv` | Package version management |

---

## Your Next Steps

**Practice:**
- Run all the pharmaverse examples: https://pharmaverse.github.io/examples/
- Clone the pharmaverse examples repo and modify them
- Build a complete pipeline for one of the open-source ADaM templates

**Community:**
- pharmaverse Slack: https://join.slack.com/t/pharmaverse
- admiral GitHub: https://github.com/pharmaverse/admiral/discussions
- R in Pharma conference (annual): https://rinpharma.com

**Deeper:**
- CDISC ADaM Implementation Guide (free at cdisc.org)
- FDA CDER pilot with R submissions: https://github.com/RConsortium/submissions-pilot2
- R Validation Hub: https://www.pharmar.org

---

## The Twelve Programs

| Ch | Program | What it does |
|----|---------|-------------|
| 1 | first_pharmaverse.R | Explore SDTM data, demographics summary |
| 2 | validate_sdtm.R | Validate SDTM against CDISC rules |
| 3 | sdtm_report.R | SDTM mapping comparison |
| 4 | first_adam.R | Derive basic ADaM variables |
| 5 | adsl.R | Complete ADSL derivation |
| 6 | ae_summary.R | TEAE incidence summary |
| 7 | tlg_suite.R | Demographics + AE tables + KM plot |
| 8 | ard_demo_table.R | Demographics via ARD pipeline |
| 9 | export_adam.R | Export all ADaM to XPT |
| 10 | clinical_explorer.R | Full teal interactive app |
| 11 | qc_report.R | QC comparison with diffdf |
| 12 | run_all.R + adsl.R + t_14_1_1.R | Complete pipeline |

Every program runs. Every concept demonstrated through clinical trial data.
