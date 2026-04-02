# Chapter 4: First ADaM Variables

*The SAP requires treatment start and end dates. Let's derive them — and understand what admiral is actually doing.*

---

## Where We Left Off

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM
pharm001_ch03_dm.R       — map raw DM to SDTM
```

The SAP for PHARM-001 requires `TRTSDT` (treatment start date) and `TRTEDT` (treatment end date) in ADSL. These come from `RFXSTDTC` and `RFXENDTC` in DM. This chapter: derive them in 15 lines, understand what admiral does.

---

## The Problem with Naive Date Conversion

Last chapter's exercise showed that `RFXSTDTC` is missing for some subjects. Let's try the naive approach first:

```r
library(dplyr)
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm %>% filter(ACTARM != "Screen Failure")

# Naive: just convert the string
dm_naive <- dm %>%
  mutate(
    TRTSDT = as.Date(RFXSTDTC),
    TRTEDT = as.Date(RFXENDTC)
  )

# How many NAs?
sum(is.na(dm_naive$TRTSDT))   # expected: some subjects have no exposure
sum(is.na(dm_naive$TRTEDT))
```

That works for complete dates. Now try it with a partial date:

```r
as.Date("2014-03")    # NA — partial dates fail silently
```

CDISC specifies imputation rules for partial dates. A date like `"2014-03"` (month known, day missing) should be imputed as `"2014-03-01"` (first of month) or `"2014-03-31"` (last of month) depending on context. `as.Date()` doesn't do this. Admiral does.

---

## What admiral Is

`admiral` is the pharmaverse package for ADaM derivation. It provides functions that implement CDISC ADaM conventions:

- `derive_vars_dt()` — ISO8601 character → R Date + imputation + imputation flag
- `derive_vars_dy()` — study day calculation (CDISC convention: no day 0)
- `derive_var_trtdurd()` — treatment duration
- `derive_var_extreme_flag()` — first/last occurrence flags
- `derive_vars_joined()` — join from another dataset with ordering

The philosophy: each function is **additive** — it returns the full dataset with new variables appended. Chain with `%>%`:

```r
adsl <- dm %>%
  derive_vars_dt(...) %>%
  derive_vars_dt(...) %>%
  mutate(...)
```

---

## The exprs() Pattern

Admiral uses `exprs()` from `rlang` for column references. You'll see it everywhere:

```r
# In SAS: BY USUBJID PARAMCD
# In admiral:
by_vars = exprs(USUBJID, PARAMCD)

# In SAS: ORDER BY ADT VISITNUM
# In admiral:
order = exprs(ADT, VISITNUM)
```

This is how admiral passes unquoted column names into functions — the same mechanism `dplyr` uses for `select()` and `filter()`.

---

## derive_vars_dt(): What It Actually Does

```r
library(admiral)
library(pharmaversesdtm)
library(dplyr)

dm <- pharmaversesdtm::dm %>% filter(ACTARM != "Screen Failure")

adsl <- dm %>%
  derive_vars_dt(
    new_vars_prefix = "TRTS",      # prefix for new variables
    dtc             = RFXSTDTC,   # source ISO8601 column
    date_imputation = "first"      # partial dates → first of month
  )
```

This creates three new variables:
- `TRTSDT` — treatment start date (Date class)
- `TRTSDTM` — treatment start datetime (POSIXct, if time present)
- `TRTSDTF` — imputation flag: `"D"` if day was imputed, `"M"` if month was, `NA` if complete

```r
# Verify
head(adsl %>% select(USUBJID, RFXSTDTC, TRTSDT, TRTSDTF))
```

---

## The 15-Line PHARM-001 Derivation

```r
# pharm001_ch04.R
# Study PHARM-001 — Chapter 4
# Derive TRTSDT, TRTEDT using admiral

library(admiral)
library(pharmaversesdtm)
library(dplyr)

dm <- pharmaversesdtm::dm %>% filter(ACTARM != "Screen Failure")

adsl <- dm %>%
  # Treatment start: from first exposure date
  derive_vars_dt(
    new_vars_prefix = "TRTS",
    dtc             = RFXSTDTC,
    date_imputation = "first"
  ) %>%
  # Treatment end: from last exposure date
  derive_vars_dt(
    new_vars_prefix = "TRTE",
    dtc             = RFXENDTC,
    date_imputation = "last"
  ) %>%
  # Treatment duration
  mutate(
    TRTDURD = as.integer(TRTEDT - TRTSDT + 1)
  )

# Summary
cat(sprintf("Enrolled subjects: %d\n", nrow(adsl)))
cat(sprintf("TRTSDT missing: %d\n", sum(is.na(adsl$TRTSDT))))
cat(sprintf("TRTEDT missing: %d\n", sum(is.na(adsl$TRTEDT))))
cat(sprintf("TRTDURD range: %d – %d days\n",
            min(adsl$TRTDURD, na.rm = TRUE),
            max(adsl$TRTDURD, na.rm = TRUE)))

# Check: end date always >= start date
bad_dates <- adsl %>% filter(!is.na(TRTSDT) & !is.na(TRTEDT) & TRTEDT < TRTSDT)
if (nrow(bad_dates) > 0) {
  cat(sprintf("WARNING: %d subjects have TRTEDT < TRTSDT\n", nrow(bad_dates)))
  print(bad_dates %>% select(USUBJID, TRTSDT, TRTEDT, TRTDURD))
} else {
  cat("Date check: TRTEDT >= TRTSDT for all subjects\n")
}
```

---

## Other Key admiral Functions

### Study Day

CDISC study day has no day 0: the day before the reference date is day -1, the reference date is day 1.

```r
adsl <- adsl %>%
  mutate(
    RFSTDT = as.Date(RFSTDTC),
    TRTSDY = case_when(
      TRTSDT >= RFSTDT ~ as.integer(TRTSDT - RFSTDT) + 1L,
      TRTSDT  < RFSTDT ~ as.integer(TRTSDT - RFSTDT),
      TRUE             ~ NA_integer_
    )
  )
```

Admiral provides `derive_vars_dy()` which handles this correctly for all variables at once:

```r
adsl <- adsl %>%
  derive_vars_dy(
    reference_date = RFSTDT,
    source_vars    = exprs(TRTSDT, TRTEDT)
  )
# Creates: TRTSDY, TRTEDT_DY
```

### Age Groups

```r
adsl <- adsl %>%
  mutate(
    AGEGR1 = case_when(
      AGE < 40              ~ "<40",
      AGE >= 40 & AGE < 65  ~ "40-64",
      AGE >= 65             ~ ">=65",
      TRUE                   ~ NA_character_
    ),
    AGEGR1N = case_when(
      AGEGR1 == "<40"   ~ 1L,
      AGEGR1 == "40-64" ~ 2L,
      AGEGR1 == ">=65"  ~ 3L
    )
  )
```

### Treatment Arms

```r
adsl <- adsl %>%
  mutate(
    TRT01P  = ACTARM,
    TRT01A  = ACTARM,
    TRT01PN = as.integer(factor(ACTARM, levels = sort(unique(ACTARM)))),
    TRT01AN = TRT01PN
  )
```

---

## restrict_derivation() and params()

Admiral has a pattern for applying derivations to subsets of records:

```r
# Apply extreme_flag only to baseline records
adlb %>%
  restrict_derivation(
    derivation = derive_var_extreme_flag,
    args = params(
      by_vars = exprs(USUBJID, PARAMCD),
      order   = exprs(ADT),
      new_var = ABLFL,
      mode    = "last"
    ),
    filter = AVISIT == "Baseline"
  )
```

`params()` bundles the arguments to the derivation function. `restrict_derivation()` applies it only to rows where `filter` is TRUE, passes through all other rows unchanged.

You'll use this in Chapter 6 for baseline flags in ADAE and ADLB.

---

## What We Have Now

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM
pharm001_ch03_dm.R       — map raw DM to SDTM
pharm001_ch04.R          — derive TRTSDT, TRTEDT, TRTDURD (15 lines)
```

The pipeline grows: `pharm001_ch04.R` produces the first ADaM-style variables.

---

## Exercises

**1.** Add `SCRDT` (screening date) from `RFICDTC` in DM. Check: is every subject's screening date before their treatment start date?

**2.** Using `derive_vars_dy()`, derive `TRTSDY` (study day of treatment start). Verify: subjects who started on their reference start date have `TRTSDY = 1`.

**3.** Using the `ex` domain, derive `EXSTDT_FIRST` (first dose date per subject) and compare to `TRTSDT`. Are they always the same?

**4. (Sets up Chapter 5)** The SAP requires a demographics table with subgroup variables: `AGEGR1`, `SEXN`, `RACEN`. Derive all three for the enrolled subjects. For `RACEN`, map `WHITE` → 1, `BLACK OR AFRICAN AMERICAN` → 2, `ASIAN` → 3, everything else → 99. These will be needed in Chapter 5's full ADSL build.

---

*Next: Chapter 5 — we build the complete ADSL. Every variable is motivated by the demographics table we'll produce in Chapter 7.*
