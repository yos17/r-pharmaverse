# Chapter 5: Build ADSL

*The SAP requires a demographics table (Table 14.1.1). To produce it, we need ADSL. Let's build it completely.*

---

## Where We Left Off

```
pharm001_ch04.R: TRTSDT, TRTEDT, TRTDURD derived from DM
```

That's the skeleton. The SAP's Table 14.1.1 requires: age (N/mean/SD/range), age group, sex, race, and treatment duration. It needs subgroup variables, analysis flags, and disposition status. Each of those drives a specific ADSL derivation. This chapter builds all of them.

---

## ADSL Is the Backbone

Every other ADaM dataset joins back to ADSL. One row per subject. Get it right.

```r
library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(metatools)
library(dplyr)
library(lubridate)
library(stringr)

# SDTM inputs
dm  <- pharmaversesdtm::dm
ds  <- pharmaversesdtm::ds   # Disposition — needed for completion flags
ex  <- pharmaversesdtm::ex   # Exposure — needed for SAFFL
sc  <- pharmaversesdtm::sc   # Subject Characteristics (height, weight)

# Reference — what we're building toward
adsl_ref <- pharmaverseadam::adsl
```

---

## Step 1: Start from DM

Last chapter gave us `TRTSDT` and `TRTEDT`. Now we add the structure:

```r
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
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
    TRT01PN = as.integer(factor(ACTARM, levels = sort(unique(ACTARM)))),
    TRT01AN = TRT01PN
  )
```

---

## Step 2: Dates

We did this in Chapter 4. Now the complete set:

```r
adsl <- adsl %>%
  derive_vars_dt(new_vars_prefix = "TRTS",  dtc = RFXSTDTC, date_imputation = "first") %>%
  derive_vars_dt(new_vars_prefix = "TRTE",  dtc = RFXENDTC,  date_imputation = "last") %>%
  derive_vars_dt(new_vars_prefix = "DTH",   dtc = DTHDTC,    date_imputation = "first") %>%
  mutate(
    TRTDURD = case_when(
      !is.na(TRTSDT) & !is.na(TRTEDT) ~ as.integer(TRTEDT - TRTSDT + 1),
      TRUE ~ NA_integer_
    ),
    DTHFL = if_else(!is.na(DTHDT), "Y", NA_character_)
  )
```

---

## Step 3: Demographics Subgroups

The SAP requires `AGEGR1` for the age group row in Table 14.1.1. Add numeric versions for sorting:

```r
adsl <- adsl %>%
  mutate(
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
    SEXN = case_when(
      SEX == "M" ~ 1L,
      SEX == "F" ~ 2L,
      TRUE       ~ NA_integer_
    ),
    RACEN = case_when(
      RACE == "WHITE"                             ~ 1L,
      RACE == "BLACK OR AFRICAN AMERICAN"         ~ 2L,
      RACE == "ASIAN"                             ~ 3L,
      RACE == "AMERICAN INDIAN OR ALASKA NATIVE"  ~ 4L,
      RACE == "MULTIPLE"                          ~ 6L,
      RACE == "UNKNOWN"                           ~ 7L,
      TRUE                                        ~ 99L
    )
  )
```

---

## Step 4: Disposition and Completion

The SAP requires `COMPLFL` and `EOSSTT`. These come from the DS domain.

```r
# Get each subject's last disposition record
ds_last <- ds %>%
  arrange(USUBJID, DSSTDTC) %>%
  group_by(USUBJID) %>%
  slice_tail(n = 1) %>%
  ungroup() %>%
  select(USUBJID, DSDECOD, DSSTDTC)

adsl <- adsl %>%
  left_join(ds_last, by = "USUBJID") %>%
  mutate(
    COMPLFL = if_else(DSDECOD == "COMPLETED", "Y", "N"),
    EOSSTT  = case_when(
      DSDECOD == "COMPLETED"                           ~ "COMPLETED",
      str_starts(DSDECOD, "WITHDRAWAL") |
        DSDECOD %in% c("DEATH", "LOST TO FOLLOW-UP")  ~ "DISCONTINUED",
      TRUE                                              ~ "ONGOING"
    )
  )
```

---

## Step 5: Analysis Flags

The SAP defines three analysis populations. Each flag is derived from explicit rules.

```r
# Which subjects received at least one dose?
ex_dosed <- ex %>%
  group_by(USUBJID) %>%
  summarise(DOSED = any(!is.na(EXSTDTC)), .groups = "drop")

adsl <- adsl %>%
  left_join(ex_dosed, by = "USUBJID") %>%
  mutate(
    ITTFL    = "Y",                                       # all enrolled
    SAFFL    = if_else(DOSED == TRUE, "Y", "N"),          # received ≥1 dose
    PPROTFL  = if_else(SAFFL == "Y" & COMPLFL == "Y",    # dosed + completed
                       "Y", "N")
  ) %>%
  select(-DOSED)
```

---

## Step 6: Body Characteristics

The SAP needs BMI for Table 14.1.1:

```r
sc_bmi <- sc %>%
  filter(SCTESTCD %in% c("HEIGHT", "WEIGHT")) %>%
  select(USUBJID, SCTESTCD, SCSTRESN) %>%
  tidyr::pivot_wider(names_from = SCTESTCD, values_from = SCSTRESN) %>%
  rename(HEIGHTBL = HEIGHT, WEIGHTBL = WEIGHT) %>%
  mutate(
    BMIBL  = round(WEIGHTBL / (HEIGHTBL / 100)^2, 1),
    BMIGR1 = case_when(
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

## Step 7: Labels

In SAS, labels are in the dataset. In R, they're attributes. Use `labelled`:

```r
library(labelled)

adsl <- adsl %>%
  set_variable_labels(
    USUBJID  = "Unique Subject Identifier",
    TRT01P   = "Planned Treatment for Period 01",
    TRT01A   = "Actual Treatment for Period 01",
    TRTSDT   = "Date of First Exposure to Treatment",
    TRTEDT   = "Date of Last Exposure to Treatment",
    TRTDURD  = "Total Treatment Duration (Days)",
    AGE      = "Age",
    AGEGR1   = "Pooled Age Group 1",
    SEX      = "Sex",
    RACE     = "Race",
    ITTFL    = "Intent-To-Treat Population Flag",
    SAFFL    = "Safety Population Flag",
    PPROTFL  = "Per-Protocol Population Flag",
    COMPLFL  = "Completers Population Flag"
  )
```

---

## The Complete Program

```r
# pharm001_ch05_adsl.R
# Study PHARM-001 — Chapter 5
# Complete ADSL derivation

library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(labelled)
library(dplyr)
library(stringr)

dm  <- pharmaversesdtm::dm
ds  <- pharmaversesdtm::ds
ex  <- pharmaversesdtm::ex
sc  <- pharmaversesdtm::sc

# ---- Base ----
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  mutate(
    TRT01P  = ACTARM,
    TRT01A  = ACTARM,
    TRT01PN = as.integer(factor(ACTARM, levels = sort(unique(ACTARM)))),
    TRT01AN = TRT01PN
  )

# ---- Dates ----
adsl <- adsl %>%
  derive_vars_dt(new_vars_prefix = "TRTS",  dtc = RFXSTDTC, date_imputation = "first") %>%
  derive_vars_dt(new_vars_prefix = "TRTE",  dtc = RFXENDTC,  date_imputation = "last") %>%
  derive_vars_dt(new_vars_prefix = "DTH",   dtc = DTHDTC,    date_imputation = "first") %>%
  mutate(
    TRTDURD = as.integer(TRTEDT - TRTSDT + 1),
    DTHFL   = if_else(!is.na(DTHDT), "Y", NA_character_)
  )

# ---- Demographics subgroups ----
adsl <- adsl %>%
  mutate(
    AGEGR1  = case_when(
      AGE < 40              ~ "<40",
      AGE >= 40 & AGE < 65  ~ "40-64",
      AGE >= 65             ~ ">=65",
      TRUE                  ~ NA_character_
    ),
    AGEGR1N = case_when(AGEGR1 == "<40" ~ 1L, AGEGR1 == "40-64" ~ 2L, AGEGR1 == ">=65" ~ 3L),
    SEXN    = case_when(SEX == "M" ~ 1L, SEX == "F" ~ 2L),
    RACEN   = case_when(
      RACE == "WHITE"                     ~ 1L,
      RACE == "BLACK OR AFRICAN AMERICAN" ~ 2L,
      RACE == "ASIAN"                     ~ 3L,
      TRUE                                ~ 99L
    )
  )

# ---- Disposition ----
ds_last <- ds %>%
  arrange(USUBJID, DSSTDTC) %>%
  group_by(USUBJID) %>% slice_tail(n=1) %>% ungroup() %>%
  select(USUBJID, DSDECOD)

adsl <- adsl %>%
  left_join(ds_last, by = "USUBJID") %>%
  mutate(
    COMPLFL = if_else(DSDECOD == "COMPLETED", "Y", "N"),
    EOSSTT  = case_when(
      DSDECOD == "COMPLETED"       ~ "COMPLETED",
      is.na(DSDECOD)               ~ "ONGOING",
      TRUE                          ~ "DISCONTINUED"
    )
  )

# ---- Analysis flags ----
ex_dosed <- ex %>%
  group_by(USUBJID) %>%
  summarise(DOSED = any(!is.na(EXSTDTC)), .groups = "drop")

adsl <- adsl %>%
  left_join(ex_dosed, by = "USUBJID") %>%
  mutate(
    ITTFL   = "Y",
    SAFFL   = if_else(DOSED == TRUE, "Y", "N"),
    PPROTFL = if_else(SAFFL == "Y" & COMPLFL == "Y", "Y", "N")
  ) %>%
  select(-DOSED, -DSDECOD)

# ---- BMI ----
sc_bmi <- sc %>%
  filter(SCTESTCD %in% c("HEIGHT", "WEIGHT")) %>%
  select(USUBJID, SCTESTCD, SCSTRESN) %>%
  tidyr::pivot_wider(names_from = SCTESTCD, values_from = SCSTRESN) %>%
  rename(HEIGHTBL = HEIGHT, WEIGHTBL = WEIGHT) %>%
  mutate(BMIBL = round(WEIGHTBL / (HEIGHTBL / 100)^2, 1))

adsl <- adsl %>% left_join(sc_bmi, by = "USUBJID")

# ---- QC ----
ref <- pharmaverseadam::adsl
cat(sprintf("Rows: mine=%d  ref=%d\n",     nrow(adsl),  nrow(ref)))
for (fl in c("ITTFL","SAFFL","PPROTFL","COMPLFL")) {
  my_y  <- sum(adsl[[fl]] == "Y", na.rm = TRUE)
  ref_y <- sum(ref[[fl]]  == "Y", na.rm = TRUE)
  cat(sprintf("  %s: mine=%d  ref=%d  %s\n",
              fl, my_y, ref_y, if (my_y == ref_y) "✓" else "✗"))
}
```

---

## QC: What If Your ADSL Disagrees with Reference?

```r
library(diffdf)

diff_result <- diffdf(
  base    = adsl,
  compare = pharmaverseadam::adsl,
  keys    = "USUBJID"
)

if (!diffdf_issame(diff_result)) {
  print(diff_result)
} else {
  cat("Datasets match exactly\n")
}
```

`diffdf` shows you exactly which rows differ and what the values are. We'll use this heavily in Chapter 13.

---

## What We Have Now

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM
pharm001_ch03_dm.R       — map raw DM to SDTM
pharm001_ch04.R          — derive TRTSDT, TRTEDT, TRTDURD
pharm001_ch05_adsl.R     — complete ADSL (all variables for Table 14.1.1)
```

`pharm001_ch05_adsl.R` is the production ADSL. Every subsequent chapter reads it.

---

## Exercises

**1.** Add `RANDFL` (randomized flag) and `ENRLFL` (enrolled flag — everyone in DM including screen failures). What's the difference between enrolled and ITT?

**2.** Derive `DCSREAS` (discontinuation reason) from DS. Categories: `"ADVERSE EVENT"`, `"LACK OF EFFICACY"`, `"WITHDRAWAL BY SUBJECT"`, `"COMPLETED"`, `"OTHER"`.

**3.** Run `diffdf()` on your ADSL vs `pharmaverseadam::adsl`. How many variables match exactly? Which differ?

**4. (Sets up Chapter 6)** The SAP requires a TEAE (treatment-emergent adverse event) table. For an AE to be treatment-emergent, it must start on or after `TRTSDT`. Write code that takes `ae` from SDTM, joins `TRTSDT` from your ADSL, and flags each AE as treatment-emergent. How many TEAEs are there? How many subjects have at least one?

---

*Next: Chapter 6 — ADAE. The SAP requires a TEAE table. We build it.*
