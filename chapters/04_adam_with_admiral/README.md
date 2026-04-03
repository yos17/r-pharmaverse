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

**In SAS:**
```sas
DATA adsl_naive;
  SET dm;
  WHERE ACTARM NE "Screen Failure";
  TRTSDT = INPUT(RFXSTDTC, YYMMDD10.);
  TRTEDT = INPUT(RFXENDTC, YYMMDD10.);
  FORMAT TRTSDT TRTEDT DATE9.;
  /* Problem: partial dates like "2014-03" cause INPUT to return missing */
RUN;
```

**In R (naive approach):**
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

**In SAS:**
```sas
/* SAS INPUT also fails on partial dates */
DATA test;
  dtc = "2014-03";
  dt  = INPUT(dtc, YYMMDD10.);  /* dt = . (missing) — silent failure */
  FORMAT dt DATE9.;
RUN;
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

## The BY-group Pattern: SAS vs. R

SAS programmers think in BY-groups — sorted data + `BY` statement in a `DATA` step.

**In SAS:**
```sas
/* BY-group processing to get first record per subject */
PROC SORT DATA=ex OUT=ex_sorted;
  BY USUBJID EXSTDTC;
RUN;

DATA ex_first;
  SET ex_sorted;
  BY USUBJID;
  IF FIRST.USUBJID;  /* keep only first record per subject */
RUN;
```

**In R (pharmaverse):**
```r
# group_by() is the R equivalent of "BY USUBJID"
# No sorting needed first — group_by works on unsorted data
ex_first <- ex %>%
  group_by(USUBJID) %>%
  arrange(EXSTDTC, .by_group = TRUE) %>%
  slice_head(n = 1) %>%      # equivalent of FIRST.USUBJID
  ungroup()
```

| SAS | R |
|-----|---|
| `PROC SORT; BY USUBJID;` | Not needed (group_by works unsorted) |
| `BY USUBJID;` in DATA step | `group_by(USUBJID)` |
| `IF FIRST.USUBJID` | `slice_head(n = 1)` |
| `IF LAST.USUBJID` | `slice_tail(n = 1)` |

---

## derive_vars_dt(): What It Actually Does

**In SAS (manual imputation):**
```sas
DATA adsl;
  SET dm;
  /* Complete dates: convert directly */
  IF LENGTH(TRIM(RFXSTDTC)) = 10 THEN
    TRTSDT = INPUT(RFXSTDTC, YYMMDD10.);
  /* Partial date (YYYY-MM): impute to first of month */
  ELSE IF LENGTH(TRIM(RFXSTDTC)) = 7 THEN DO;
    TRTSDT  = INPUT(CATS(RFXSTDTC, "-01"), YYMMDD10.);
    TRTSDTF = "D";  /* day imputed */
  END;
  /* Partial date (YYYY): impute to Jan 1 */
  ELSE IF LENGTH(TRIM(RFXSTDTC)) = 4 THEN DO;
    TRTSDT  = INPUT(CATS(RFXSTDTC, "-01-01"), YYMMDD10.);
    TRTSDTF = "MD";  /* month and day imputed */
  END;
  FORMAT TRTSDT DATE9.;
RUN;
```

**In R (pharmaverse — admiral):**
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

**In SAS:**
```sas
/* Manual CDISC study day calculation — no day 0 */
DATA adsl;
  SET adsl;
  IF TRTSDT >= RFSTDT THEN TRTSDY =  TRTSDT - RFSTDT + 1;
  ELSE                     TRTSDY =  TRTSDT - RFSTDT;
  /* Some CDISC programming environments use a company macro: */
  /* %STUDYDAY(STARTDT=RFSTDT, DT=TRTSDT, DY=TRTSDY) */
RUN;
```

**In R (pharmaverse — admiral):**

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

**In SAS:**
```sas
DATA adsl;
  SET adsl;
  IF      AGE < 40              THEN AGEGR1 = "<40";
  ELSE IF AGE >= 40 AND AGE < 65 THEN AGEGR1 = "40-64";
  ELSE IF AGE >= 65              THEN AGEGR1 = ">=65";
  ELSE                                AGEGR1 = "";

  IF      AGEGR1 = "<40"   THEN AGEGR1N = 1;
  ELSE IF AGEGR1 = "40-64" THEN AGEGR1N = 2;
  ELSE IF AGEGR1 = ">=65"  THEN AGEGR1N = 3;
RUN;
```

**In R (pharmaverse):**
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

**In SAS:**
```sas
DATA adsl;
  SET adsl;
  TRT01P  = ACTARM;
  TRT01A  = ACTARM;
  /* Numeric codes must be hard-coded or derived from a format */
  IF      ACTARM = "Drug A" THEN TRT01PN = 1;
  ELSE IF ACTARM = "Drug B" THEN TRT01PN = 2;
  ELSE IF ACTARM = "Placebo" THEN TRT01PN = 3;
  TRT01AN = TRT01PN;
RUN;
```

**In R (pharmaverse):**
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

---

## Solutions

### Exercise 1

Add `SCRDT` (screening date from `RFICDTC`) and verify it precedes treatment start for all subjects.

```r
# SAS:
# DATA adsl;
#   SET adsl;
#   SCRDT = INPUT(RFICDTC, YYMMDD10.);
#   FORMAT SCRDT DATE9.;
#   IF SCRDT >= TRTSDT AND NOT MISSING(TRTSDT) THEN
#     PUT "WARNING: Screening not before treatment for " USUBJID=;
# RUN;

library(admiral)
library(pharmaversesdtm)
library(dplyr)

dm <- pharmaversesdtm::dm %>% filter(ACTARM != "Screen Failure")

adsl <- dm %>%
  # Derive TRTSDT from exposure start
  derive_vars_dt(
    new_vars_prefix = "TRTS",
    dtc             = RFXSTDTC,
    date_imputation = "first"
  ) %>%
  # Derive SCRDT from informed consent date
  derive_vars_dt(
    new_vars_prefix = "SCR",
    dtc             = RFICDTC,
    date_imputation = "first"
  )

# Check: screening before treatment?
bad_dates <- adsl %>%
  filter(!is.na(SCRDT), !is.na(TRTSDT), SCRDT >= TRTSDT)

cat(sprintf("Subjects with SCRDT >= TRTSDT: %d\n", nrow(bad_dates)))
if (nrow(bad_dates) > 0) {
  cat("Details:\n")
  print(bad_dates %>% select(USUBJID, SCRDT, TRTSDT, ACTARM))
}
# Expected: 0 violations (screening always precedes treatment)
```

### Exercise 2

Derive `TRTSDY` using `derive_vars_dy()` and verify subjects who started on their reference date have `TRTSDY = 1`.

```r
# SAS:
# DATA adsl;
#   SET adsl;
#   /* CDISC study day: no day 0 */
#   IF TRTSDT >= RFSTDT THEN TRTSDY =  TRTSDT - RFSTDT + 1;
#   ELSE IF NOT MISSING(TRTSDT) AND NOT MISSING(RFSTDT)
#                      THEN TRTSDY =  TRTSDT - RFSTDT;
# RUN;
# /* Verify: subjects where TRTSDT = RFSTDT have TRTSDY = 1 */
# PROC PRINT DATA=adsl(WHERE=(TRTSDT = RFSTDT));
#   VAR USUBJID TRTSDT RFSTDT TRTSDY;
# RUN;

library(admiral)
library(pharmaversesdtm)
library(dplyr)

dm <- pharmaversesdtm::dm %>% filter(ACTARM != "Screen Failure")

adsl <- dm %>%
  # Need a reference date: use RFSTDTC
  derive_vars_dt(
    new_vars_prefix = "RFST",
    dtc             = RFSTDTC,
    date_imputation = "first"
  ) %>%
  derive_vars_dt(
    new_vars_prefix = "TRTS",
    dtc             = RFXSTDTC,
    date_imputation = "first"
  ) %>%
  # Study day: CDISC convention (no day 0)
  derive_vars_dy(
    reference_date = RFSTDT,
    source_vars    = exprs(TRTSDT)
  )

# Verify: subjects starting on their reference date → TRTSDY = 1
same_day <- adsl %>%
  filter(!is.na(TRTSDT), !is.na(RFSTDT), TRTSDT == RFSTDT)

cat(sprintf("Subjects where TRTSDT == RFSTDT: %d\n", nrow(same_day)))
cat(sprintf("All have TRTSDY = 1: %s\n",
            all(same_day$TRTSDY == 1, na.rm = TRUE)))

# Range of study days
cat(sprintf("TRTSDY range: %d to %d\n",
            min(adsl$TRTSDY, na.rm = TRUE),
            max(adsl$TRTSDY, na.rm = TRUE)))
```

### Exercise 3

Derive `EXSTDT_FIRST` from the `ex` domain and compare to `TRTSDT`.

```r
# SAS:
# PROC SORT DATA=ex OUT=ex_first; BY USUBJID EXSTDTC; RUN;
# DATA ex_first;
#   SET ex_first;
#   BY USUBJID;
#   IF FIRST.USUBJID;
#   EXSTDT = INPUT(EXSTDTC, YYMMDD10.);
#   FORMAT EXSTDT DATE9.;
# RUN;
# DATA compare;
#   MERGE adsl(KEEP=USUBJID TRTSDT) ex_first(KEEP=USUBJID EXSTDT);
#   BY USUBJID;
#   DIFF = TRTSDT - EXSTDT;
# RUN;
# PROC MEANS DATA=compare; VAR DIFF; RUN;

library(admiral)
library(pharmaversesdtm)
library(dplyr)

dm <- pharmaversesdtm::dm %>% filter(ACTARM != "Screen Failure")
ex <- pharmaversesdtm::ex

# Derive TRTSDT
adsl <- dm %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC, date_imputation = "first")

# First dose date per subject from EX
ex_first_dose <- ex %>%
  mutate(EXSTDT = as.Date(substr(EXSTDTC, 1, 10))) %>%
  group_by(USUBJID) %>%
  summarise(EXSTDT_FIRST = min(EXSTDT, na.rm = TRUE), .groups = "drop")

# Compare
comparison <- adsl %>%
  select(USUBJID, TRTSDT) %>%
  left_join(ex_first_dose, by = "USUBJID") %>%
  filter(!is.na(TRTSDT), !is.na(EXSTDT_FIRST)) %>%
  mutate(DIFF_DAYS = as.integer(TRTSDT - EXSTDT_FIRST))

cat(sprintf("Subjects compared: %d\n", nrow(comparison)))
cat(sprintf("Subjects where TRTSDT == EXSTDT_FIRST: %d\n",
            sum(comparison$DIFF_DAYS == 0, na.rm = TRUE)))
cat(sprintf("Subjects where TRTSDT != EXSTDT_FIRST: %d\n",
            sum(comparison$DIFF_DAYS != 0, na.rm = TRUE)))

# Show discrepancies if any
discrepancies <- comparison %>% filter(DIFF_DAYS != 0)
if (nrow(discrepancies) > 0) {
  cat("Discrepant subjects (TRTSDT ≠ EXSTDT_FIRST):\n")
  print(discrepancies)
}
```

### Exercise 4

Derive `AGEGR1`, `SEXN`, and `RACEN` for the demographics table.

```r
# SAS:
# DATA adsl;
#   SET adsl;
#   IF      AGE < 40              THEN AGEGR1 = "<40";
#   ELSE IF AGE >= 40 AND AGE < 65 THEN AGEGR1 = "40-64";
#   ELSE IF AGE >= 65              THEN AGEGR1 = ">=65";
#   IF      AGEGR1 = "<40"   THEN AGEGR1N = 1;
#   ELSE IF AGEGR1 = "40-64" THEN AGEGR1N = 2;
#   ELSE IF AGEGR1 = ">=65"  THEN AGEGR1N = 3;
#   IF SEX = "M" THEN SEXN = 1; ELSE IF SEX = "F" THEN SEXN = 2;
#   IF      RACE = "WHITE"                             THEN RACEN = 1;
#   ELSE IF RACE = "BLACK OR AFRICAN AMERICAN"         THEN RACEN = 2;
#   ELSE IF RACE = "ASIAN"                             THEN RACEN = 3;
#   ELSE                                                    RACEN = 99;
# RUN;

library(dplyr)
library(admiral)
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm %>% filter(ACTARM != "Screen Failure")

adsl <- dm %>%
  mutate(
    AGEGR1 = case_when(
      AGE < 40              ~ "<40",
      AGE >= 40 & AGE < 65  ~ "40-64",
      AGE >= 65             ~ ">=65",
      TRUE                  ~ NA_character_
    ),
    AGEGR1N = case_when(
      AGEGR1 == "<40"   ~ 1L,
      AGEGR1 == "40-64" ~ 2L,
      AGEGR1 == ">=65"  ~ 3L
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

# Verify
cat("AGEGR1 distribution:\n")
print(count(adsl, AGEGR1, AGEGR1N))
cat("\nSEX / SEXN check (should be 1-to-1):\n")
print(table(adsl$SEX, adsl$SEXN))
cat("\nRACEN distribution:\n")
print(count(adsl, RACE, RACEN) %>% arrange(RACEN))
```
