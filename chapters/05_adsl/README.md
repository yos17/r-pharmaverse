# Chapter 5: Building ADSL

*The complete subject-level analysis dataset from scratch.*

---

## ADSL Is the Backbone

Every other ADaM dataset joins back to ADSL. Get it right. ADSL has:

- One row per subject (enrolled + screen failures)
- Demographics (AGE, SEX, RACE, ETHNIC)
- Treatment assignment (TRT01P/A, TRT01PN/AN)
- Key dates (TRTSDT, TRTEDT, TRTDURD)
- Completion/withdrawal status
- Subgroup variables for analysis (AGEGR1, BMIBL, etc.)
- Flag variables (ITTFL, SAFFL, PPROTFL, COMPLFL)

---

## Setup

```r
library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)  # reference ADaM (for comparison)
library(metacore)
library(metatools)
library(dplyr)
library(lubridate)
library(stringr)

# SDTM inputs
dm  <- pharmaversesdtm::dm
ds  <- pharmaversesdtm::ds   # Disposition
ex  <- pharmaversesdtm::ex   # Exposure
sc  <- pharmaversesdtm::sc   # Subject Characteristics

# Reference ADSL (what we're building toward)
adsl_ref <- pharmaverseadam::adsl
```

---

## Step 1: Start from DM

```r
adsl <- dm %>%
  # Rename / restructure per ADaM conventions
  select(
    STUDYID, USUBJID, SUBJID, SITEID,
    RFICDTC, RFSTDTC, RFENDTC, RFXSTDTC, RFXENDTC,
    DTHDTC, DTHFL,
    AGE, AGEU, SEX, RACE, ETHNIC,
    ARMCD, ARM, ACTARMCD, ACTARM,
    COUNTRY
  ) %>%
  mutate(
    TRT01P  = ACTARM,
    TRT01A  = ACTARM,
    TRT01PN = as.integer(factor(ACTARM)),
    TRT01AN = TRT01PN
  )
```

---

## Step 2: Key Dates

```r
adsl <- adsl %>%
  
  # Reference start date (e.g., informed consent or first dose)
  derive_vars_dt(
    new_vars_prefix = "RANDDT",  # randomization date
    dtc             = RFICDTC,
    date_imputation = "none"     # no imputation
  ) %>%
  
  # Treatment start
  derive_vars_dt(
    new_vars_prefix = "TRTS",
    dtc             = RFXSTDTC,
    date_imputation = "first"
  ) %>%
  
  # Treatment end
  derive_vars_dt(
    new_vars_prefix = "TRTE",
    dtc             = RFXENDTC,
    date_imputation = "last"
  ) %>%
  
  # Death date
  derive_vars_dt(
    new_vars_prefix = "DTH",
    dtc             = DTHDTC,
    date_imputation = "first"
  ) %>%
  
  # Treatment duration
  mutate(
    TRTDURD = case_when(
      !is.na(TRTSDT) & !is.na(TRTEDT) ~ as.integer(TRTEDT - TRTSDT + 1),
      TRUE ~ NA_integer_
    )
  )
```

---

## Step 3: Demographics Subgroups

```r
adsl <- adsl %>%
  mutate(
    
    # Age groups
    AGEGR1 = case_when(
      AGE <  18             ~ "<18",
      AGE >= 18 & AGE < 40  ~ "18-39",
      AGE >= 40 & AGE < 65  ~ "40-64",
      AGE >= 65             ~ ">=65",
      TRUE                  ~ NA_character_
    ),
    AGEGR1N = case_when(
      AGEGR1 == "<18"   ~ 1L,
      AGEGR1 == "18-39" ~ 2L,
      AGEGR1 == "40-64" ~ 3L,
      AGEGR1 == ">=65"  ~ 4L
    ),
    
    # Sex numeric
    SEXN = case_when(
      SEX == "F" ~ 2L,
      SEX == "M" ~ 1L,
      TRUE       ~ NA_integer_
    ),
    
    # Race numeric
    RACEN = case_when(
      RACE == "WHITE"                         ~ 1L,
      RACE == "BLACK OR AFRICAN AMERICAN"     ~ 2L,
      RACE == "ASIAN"                         ~ 3L,
      RACE == "AMERICAN INDIAN OR ALASKA NATIVE" ~ 4L,
      RACE == "NATIVE HAWAIIAN OR OTHER PACIFIC ISLANDER" ~ 5L,
      RACE == "MULTIPLE"                      ~ 6L,
      RACE == "UNKNOWN"                       ~ 7L,
      TRUE                                    ~ 99L
    )
  )
```

---

## Step 4: Disposition and Analysis Flags

```r
# Disposition: derive completion status
ds_disposition <- ds %>%
  filter(DSCAT == "DISPOSITION EVENT" | is.na(DSCAT)) %>%
  arrange(USUBJID, DSSTDTC) %>%
  group_by(USUBJID) %>%
  slice_tail(n = 1) %>%
  ungroup() %>%
  select(USUBJID, DSDECOD, DSSTDTC)

adsl <- adsl %>%
  left_join(ds_disposition, by = "USUBJID") %>%
  mutate(
    COMPLFL = if_else(DSDECOD == "COMPLETED", "Y", "N"),
    EOSSTT  = case_when(
      DSDECOD == "COMPLETED"                           ~ "COMPLETED",
      str_starts(DSDECOD, "WITHDRAWAL")                ~ "DISCONTINUED",
      DSDECOD %in% c("DEATH", "LOST TO FOLLOW-UP")    ~ "DISCONTINUED",
      TRUE                                              ~ "ONGOING"
    )
  )

# Analysis flags from EX (subject must have had at least one dose)
ex_dosed <- ex %>%
  group_by(USUBJID) %>%
  summarise(dosed = n() > 0, .groups = "drop")

adsl <- adsl %>%
  left_join(ex_dosed, by = "USUBJID") %>%
  mutate(
    # ITT: all randomized subjects
    ITTFL  = if_else(ACTARM != "Screen Failure", "Y", "N"),
    # Safety: all subjects who received at least one dose
    SAFFL  = if_else(ITTFL == "Y" & dosed == TRUE, "Y", "N"),
    # Per protocol: approximate (usually more complex criteria)
    PPROTFL = if_else(COMPLFL == "Y" & SAFFL == "Y", "Y", "N")
  ) %>%
  select(-dosed)
```

---

## Step 5: Body Characteristics (from SC or VS)

```r
# If body measurements come from SC domain
sc_bmi <- sc %>%
  filter(SCTESTCD %in% c("HEIGHT", "WEIGHT")) %>%
  select(USUBJID, SCTESTCD, SCSTRESN) %>%
  tidyr::pivot_wider(names_from = SCTESTCD, values_from = SCSTRESN) %>%
  rename(HEIGHTBL = HEIGHT, WEIGHTBL = WEIGHT) %>%
  mutate(
    BMIBL = round(WEIGHTBL / (HEIGHTBL / 100)^2, 1),
    BMIGRP = case_when(
      BMIBL < 18.5             ~ "Underweight",
      BMIBL >= 18.5 & BMIBL < 25 ~ "Normal",
      BMIBL >= 25 & BMIBL < 30   ~ "Overweight",
      BMIBL >= 30                 ~ "Obese"
    )
  )

adsl <- adsl %>%
  left_join(sc_bmi, by = "USUBJID")
```

---

## Step 6: Variable Labels with {metatools}

```r
library(metatools)

# Define variable labels (in practice, from metacore/spec)
var_labels <- list(
  USUBJID  = "Unique Subject Identifier",
  TRT01P   = "Planned Treatment for Period 01",
  TRT01PN  = "Planned Treatment for Period 01 (N)",
  TRT01A   = "Actual Treatment for Period 01",
  TRTSDT   = "Date of First Exposure to Treatment",
  TRTEDT   = "Date of Last Exposure to Treatment",
  TRTDURD  = "Total Treatment Duration (Days)",
  AGE      = "Age",
  AGEGR1   = "Pooled Age Group 1",
  SEX      = "Sex",
  SEXN     = "Sex (N)",
  RACE     = "Race",
  RACEN    = "Race (N)",
  ITTFL    = "Intent-To-Treat Population Flag",
  SAFFL    = "Safety Population Flag",
  PPROTFL  = "Per-Protocol Population Flag",
  COMPLFL  = "Completers Population Flag"
)

adsl <- adsl %>%
  labelled::set_variable_labels(.labels = var_labels, .strict = FALSE)

# Verify
labelled::var_label(adsl$TRTSDT)   # "Date of First Exposure to Treatment"
```

---

## Program: ADSL QC

```r
# adsl_qc.R
# Compare derived ADSL against reference

library(admiral)
library(pharmaverseadam)
library(dplyr)
library(diffdf)  # for dataset comparison

adsl_ref <- pharmaverseadam::adsl

# (assume adsl is your derived dataset from above)

cat("=== ADSL Quality Check ===\n\n")

# 1. Row count
cat(sprintf("Your ADSL: %d rows\n", nrow(adsl)))
cat(sprintf("Reference: %d rows\n", nrow(adsl_ref)))

# 2. Column coverage
my_vars  <- names(adsl)
ref_vars <- names(adsl_ref)
missing_vars <- setdiff(ref_vars, my_vars)
extra_vars   <- setdiff(my_vars, ref_vars)

if (length(missing_vars) > 0)
  cat("Missing vars:", paste(missing_vars, collapse=", "), "\n")
if (length(extra_vars) > 0)
  cat("Extra vars:", paste(extra_vars, collapse=", "), "\n")

# 3. Flag variable frequencies
cat("\nFlag variable comparison:\n")
flags <- c("ITTFL","SAFFL","PPROTFL","COMPLFL")
for (fl in flags) {
  if (fl %in% names(adsl) && fl %in% names(adsl_ref)) {
    my_y  <- sum(adsl[[fl]] == "Y", na.rm=TRUE)
    ref_y <- sum(adsl_ref[[fl]] == "Y", na.rm=TRUE)
    match <- if_else(my_y == ref_y, "✓", "✗")
    cat(sprintf("  %s: yours=%d, ref=%d %s\n", fl, my_y, ref_y, match))
  }
}

# 4. Date comparison
cat("\nDate variable check:\n")
date_vars <- c("TRTSDT","TRTEDT")
for (dv in date_vars) {
  if (dv %in% names(adsl) && dv %in% names(adsl_ref)) {
    joined <- inner_join(
      adsl[c("USUBJID", dv)],
      adsl_ref[c("USUBJID", dv)],
      by = "USUBJID",
      suffix = c("_mine", "_ref")
    )
    n_diff <- sum(joined[[paste0(dv,"_mine")]] != joined[[paste0(dv,"_ref")]], 
                  na.rm = TRUE)
    cat(sprintf("  %s: %d mismatches\n", dv, n_diff))
  }
}
```

---

## Exercises

**1. Complete ADSL**

Add to your ADSL:
- `ENRLFL` — enrolled flag (everyone in DM including screen failures = "Y" for enrolled)
- `RANDFL` — randomized flag
- Study day variables: `TRTSDY`, `TRTEDSDY`

**2. Withdrawal reason**

Derive `DCSREAS` (discontinuation reason) from the DS domain. Categories: "ADVERSE EVENT", "LACK OF EFFICACY", "WITHDRAWAL BY SUBJECT", "OTHER", "COMPLETED".

**3. Stratification**

If the study protocol defines a stratification factor (e.g., disease severity), derive it from `SC` domain. Create both character and numeric versions.

**4. Compare against reference**

Install `diffdf` and use `diffdf::diffdf()` to compare your ADSL row-by-row against `pharmaverseadam::adsl`. How many variables match exactly?

---

*Next: Chapter 6 — ADAE and Time-to-Event: adverse event analysis datasets*
