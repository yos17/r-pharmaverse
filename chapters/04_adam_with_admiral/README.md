# Chapter 4: ADaM with {admiral}

*The core admiral package. How it works. Derive your first analysis variables.*

---

## What admiral Is

`admiral` (ADaM in R) is the backbone of pharmaverse ADaM programming. It provides:

- Functions to **derive** ADaM variables from SDTM (e.g., `derive_vars_dt()`, `derive_var_age_years()`)
- **Dataset templates** as starting points
- A consistent, pipe-friendly API
- Clinical-trial-specific logic baked in (CDISC imputation rules, flag derivations, etc.)

Think of it as replacing your company's internal ADaM macro library, but open-source, reviewed, and validated.

```r
library(admiral)
library(pharmaversesdtm)
library(dplyr)
library(lubridate)
```

---

## The admiral Philosophy

Admiral functions are **additive** — each call derives one or more variables and returns the full dataset. Chain them with `%>%`:

```r
adsl <- dm %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC) %>%
  derive_vars_dt(new_vars_prefix = "TRTE", dtc = RFXENDTC) %>%
  derive_var_age_years(AGE, new_var = AGE)
```

This replaces the equivalent `DATA` step pattern in SAS where you'd manually convert dates, compute ages, etc.

---

## Key admiral Functions by Category

### Date/Time Derivation

```r
# derive_vars_dt() — ISO8601 character → SAS-style date + imputation flag
adsl <- adsl %>%
  derive_vars_dt(
    new_vars_prefix = "TRTS",      # creates TRTSDTM + TRTSDT + TRTSDTF
    dtc             = RFXSTDTC,   # source DTC variable
    date_imputation = "first",     # when day is missing: use 01
    time_imputation = "first"      # when time missing: use 00:00:00
  )
```

After this, your dataset has:
- `TRTSDT` — treatment start date (Date class)
- `TRTSDTM` — treatment start datetime (POSIXct)
- `TRTSDTF` — imputation flag (if day/month was imputed)

### Flag Derivation

```r
# derive_var_extreme_flag() — LSTALVFL, ABLFL etc.
adlb_flagged <- adlb %>%
  restrict_derivation(
    derivation = derive_var_extreme_flag,
    args = params(
      by_vars     = exprs(USUBJID, PARAMCD),
      order       = exprs(ADT, VISITNUM),
      new_var     = ABLFL,   # baseline flag
      mode        = "last"
    ),
    filter = ABLFL_CRIT  # pre-computed baseline criterion
  )
```

### Event Date (for TTE)

```r
# derive_vars_joined() — join dates from another dataset
adtte <- adtte %>%
  derive_vars_joined(
    dataset_add  = adae,
    by_vars      = exprs(STUDYID, USUBJID),
    new_vars     = exprs(AVAL = ADT),   # event date → AVAL
    filter_add   = AESER == "Y",         # only serious AEs
    order        = exprs(ADT),
    mode         = "first",              # first occurrence
    new_vars_prefix = "DTHD"
  )
```

---

## The exprs() Pattern

Admiral uses `exprs()` (from `rlang`) for non-standard evaluation. You'll see it everywhere:

```r
# In SAS: BY USUBJID PARAMCD
# In admiral:
by_vars = exprs(USUBJID, PARAMCD)

# In SAS: WHERE ABLFL = "Y"
# In admiral (filter argument):
filter = ABLFL == "Y"

# In SAS: ORDER = ADT VISITNUM
# In admiral:
order = exprs(ADT, VISITNUM)
```

---

## params() and restrict_derivation()

`restrict_derivation()` applies a derivation to a subset of records. `params()` bundles the derivation arguments:

```r
# Apply extreme_flag only to post-baseline records for ABLFL
adlb %>%
  restrict_derivation(
    derivation = derive_var_extreme_flag,
    args = params(
      by_vars = exprs(USUBJID, PARAMCD),
      order   = exprs(ADT),
      new_var = ABLFL,
      mode    = "last"
    ),
    filter = AVISIT == "Baseline"   # ← only to baseline visits
  )
```

---

## Common Derivations

```r
# ----- merge from DM -----
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  mutate(TRTP = ACTARM, TRTPN = as.integer(factor(ACTARM)))

# ----- treatment start/end dates -----
adsl <- adsl %>%
  derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC, date_imputation="first") %>%
  derive_vars_dt(new_vars_prefix="TRTE", dtc=RFXENDTC,  date_imputation="last")

# ----- treatment duration -----
adsl <- adsl %>%
  derive_var_trtdurd()   # TRTDURD = TRTEDT - TRTSDT + 1

# ----- age in years -----
adsl <- adsl %>%
  derive_var_age_years(AGE, new_var = AGE)   # or with unit conversion

# ----- death flag -----
adsl <- adsl %>%
  derive_var_extreme_dt(
    new_var    = DTHDT,
    dataset_add = dm,
    by_vars    = exprs(STUDYID, USUBJID),
    dtc        = DTHDTC,
    date_imputation = "first"
  )

adsl <- adsl %>%
  mutate(DTHFL = if_else(!is.na(DTHDT), "Y", NA_character_))
```

---

## Program: First ADaM Variables

```r
# first_adam.R
# Derive basic ADaM variables using admiral

library(admiral)
library(pharmaversesdtm)
library(dplyr)
library(lubridate)

# ---- Load SDTM ----
dm <- pharmaversesdtm::dm
ex <- pharmaversesdtm::ex

# ---- Enrolled subjects only ----
dm_enrolled <- dm %>%
  filter(ACTARM != "Screen Failure")

cat(sprintf("Enrolled subjects: %d\n", nrow(dm_enrolled)))

# ---- Derive treatment start/end dates ----
adsl <- dm_enrolled %>%
  derive_vars_dt(
    new_vars_prefix = "TRTS",
    dtc             = RFXSTDTC,
    date_imputation = "first"
  ) %>%
  derive_vars_dt(
    new_vars_prefix = "TRTE",
    dtc             = RFXENDTC,
    date_imputation = "last"
  )

# ---- Treatment duration ----
adsl <- adsl %>%
  mutate(TRTDURD = as.integer(TRTEDT - TRTSDT + 1))

# ---- AGE already numeric; add AGEGR1 ----
adsl <- adsl %>%
  mutate(
    AGEGR1 = case_when(
      AGE < 40          ~ "<40",
      AGE >= 40 & AGE < 65 ~ "40-64",
      AGE >= 65          ~ ">=65",
      TRUE               ~ NA_character_
    ),
    AGEGR1N = case_when(
      AGEGR1 == "<40"   ~ 1L,
      AGEGR1 == "40-64" ~ 2L,
      AGEGR1 == ">=65"  ~ 3L
    )
  )

# ---- Planned treatment arms ----
adsl <- adsl %>%
  mutate(
    TRT01P  = ACTARM,
    TRT01PN = as.integer(factor(ACTARM, levels = sort(unique(ACTARM))))
  )

# ---- Death flag from DTHFL in DM ----
adsl <- adsl %>%
  mutate(DTHFL = if_else(DTHFL == "Y", "Y", NA_character_))

# ---- Print summary ----
cat("\n=== ADSL Variables Derived ===\n")
new_vars <- c("TRTSDT","TRTEDT","TRTDURD","AGEGR1","AGEGR1N","TRT01P","TRT01PN")
for (v in new_vars) {
  vals <- adsl[[v]]
  if (is.numeric(vals)) {
    cat(sprintf("  %-12s [numeric] range: %s – %s, NAs: %d\n",
                v, min(vals,na.rm=T), max(vals,na.rm=T), sum(is.na(vals))))
  } else {
    cat(sprintf("  %-12s [char]  n_unique: %d, NAs: %d\n",
                v, n_distinct(vals[!is.na(vals)]), sum(is.na(vals))))
  }
}

# ---- Summary by arm ----
cat("\n=== Treatment Summary by Arm ===\n")
arm_summary <- adsl %>%
  group_by(TRT01P) %>%
  summarise(
    n        = n(),
    mean_age = round(mean(AGE, na.rm=TRUE), 1),
    mean_dur = round(mean(TRTDURD, na.rm=TRUE), 1),
    n_death  = sum(DTHFL == "Y", na.rm=TRUE),
    .groups  = "drop"
  )

cat(sprintf("%-30s  %4s  %8s  %8s  %7s\n",
            "Arm", "n", "Avg Age", "Avg Dur", "Deaths"))
cat(strrep("-", 65), "\n")
for (i in seq_len(nrow(arm_summary))) {
  cat(sprintf("%-30s  %4d  %8.1f  %8.1f  %7d\n",
              arm_summary$TRT01P[i],
              arm_summary$n[i],
              arm_summary$mean_age[i],
              arm_summary$mean_dur[i],
              arm_summary$n_death[i]))
}
```

---

## Exercises

**1. Add COUNTRY**

Derive `COUNTRY` from `dm$COUNTRY` (or from `SITEID` if country isn't available). Map to ISO 3166-1 alpha-3 codes.

**2. Screening dates**

Derive `SCRDT` (screening date) from `RFICDTC` in DM. Check: is screening date always before treatment start?

**3. Exposure-based treatment flags**

Using the `ex` domain, derive:
- `EXSTDT` — first dose date per subject
- `EXENDT` — last dose date per subject
- Compare to `TRTSDT` / `TRTEDT` from DM

**4. Study day**

Derive `TRTSDY` (study day of treatment start relative to reference start):
```
TRTSDY = TRTSDT - RFSTDT + 1 (if TRTSDT >= RFSTDT)
TRTSDY = TRTSDT - RFSTDT     (if TRTSDT < RFSTDT)
```
(CDISC study day calculation — no day 0)

---

*Next: Chapter 5 — Building ADSL: the complete subject-level analysis dataset*
