# Chapter 2: Inspect PHARM-001's SDTM

*Last chapter we loaded DM and printed demographics. Now we look more carefully at what we have — and whether it's valid.*

---

## Where We Left Off

```r
# pharm001_ch01.R — what we have
library(dplyr)
library(pharmaversesdtm)

dm          <- pharmaversesdtm::dm
dm_enrolled <- dm %>% filter(ACTARM != "Screen Failure")
# 254 enrolled subjects, basic age/sex summary printed
```

Chapter 2's job: understand the full SDTM domain structure, validate one variable against controlled terminology, and add a `validate_sdtm.R` file to the pipeline.

---

## The SDTM Domains for PHARM-001

```r
library(pharmaversesdtm)

dm  <- pharmaversesdtm::dm   # Demographics — one row per subject
ae  <- pharmaversesdtm::ae   # Adverse Events
ex  <- pharmaversesdtm::ex   # Exposure
vs  <- pharmaversesdtm::vs   # Vital Signs
ds  <- pharmaversesdtm::ds   # Disposition
```

The DM structure matters most — everything else joins back to it.

**In SAS:**
```sas
/* Inspect structure of a SAS dataset */
PROC CONTENTS DATA=dm;
RUN;
```

**In R (pharmaverse):**
```r
glimpse(dm)
# or:
str(dm)
```

Key fields:
- `USUBJID` — unique subject ID, format `STUDYID-SITEID-SUBJID`
- `RFSTDTC` — reference start (ISO8601 character, e.g. `"2014-01-02"`)
- `RFXSTDTC` — first exposure date/time (what we'll use for TRTSDT)
- `RFXENDTC` — last exposure date/time (what we'll use for TRTEDT)
- `SEX` — controlled terminology: `"M"` or `"F"`
- `ACTARM` — actual treatment arm

---

## ISO8601 Dates

SDTM stores all dates as ISO8601 character strings. That means `RFSTDTC` is `"2014-01-02"`, not a Date object.

```r
dm$RFSTDTC[1:3]
# [1] "2014-01-02" "2014-01-11" "2013-08-28"

class(dm$RFSTDTC)
# [1] "character"
```

**Date type comparison:**

| | SAS | R |
|-|-----|---|
| Storage | Numeric (days since **1960-01-01**) | `Date` class (days since **1970-01-01**) |
| SDTM representation | ISO8601 character `"2014-01-02"` | ISO8601 character `"2014-01-02"` |
| Conversion | `INPUT(dtc, YYMMDD10.)` | `as.Date(dtc)` |
| Missing | `.` | `NA` |

To do date arithmetic, convert:

```r
as.Date(dm$RFSTDTC[1:3])
# [1] "2014-01-02" "2014-01-11" "2013-08-28"
```

Partial dates are the complication: `"2014-03"` has no day. `as.Date("2014-03")` returns `NA`. CDISC defines imputation rules for this — Chapter 4 handles it properly with `admiral::derive_vars_dt()`.

---

## Printing the First Few Rows

**In SAS:**
```sas
PROC PRINT DATA=ae(OBS=5);
RUN;
```

**In R (pharmaverse):**
```r
head(ae, 5)
# or:
print(ae)   # tibbles auto-truncate to 10 rows
```

---

## Controlled Terminology

CDISC CT values are just character constants — the discipline is applying them consistently. PHARM-001 uses:

```r
# Sex: only M or F allowed
unique(dm$SEX)

# Treatment arms
unique(dm$ACTARM)

# AE seriousness: Y or N
unique(ae$AESER)
```

**In SAS**, CT compliance is often checked using SAS formats:
```sas
/* Defining CT as a SAS format */
PROC FORMAT;
  VALUE $SEXFMT
    'M' = 'Male'
    'F' = 'Female'
    'U' = 'Unknown'
    OTHER = 'INVALID';
RUN;

/* Using it to check */
PROC FREQ DATA=dm;
  TABLES SEX;
  FORMAT SEX $SEXFMT.;
RUN;
```

**In R (pharmaverse)**, CT values are just character constants — validation is explicit code:
```r
allowed_sex <- c("M", "F", "U", "UNDIFFERENTIATED")
bad_sex <- dm %>% filter(!SEX %in% allowed_sex)
nrow(bad_sex)   # should be 0
```

> ⚠️ **Case sensitivity:** SAS formats match case-insensitively. `"Male"` would match `'M'` in a well-written SAS format. In R, `"Male"` does **not** equal `"M"` — the comparison is exact. SDTM requires uppercase CT values; always verify.

If it's not zero, you have a problem. In real data, you'd see things like `"Male"`, `"MALE"`, `"male"` — all wrong.

---

## The Naive Validation (and Why It's Not Enough)

You might write:

```r
# This works, but it only prints — it doesn't fail loud
if (any(!dm$SEX %in% c("M","F","U","UNDIFFERENTIATED"))) {
  cat("Bad SEX values found\n")
}
```

The problem: silent failures. If validation fails and the script keeps running, you get wrong output. We want loud failures:

```r
# This stops execution with a clear message
stopifnot(
  "SEX contains non-CT values" = all(dm$SEX %in% c("M","F","U","UNDIFFERENTIATED"))
)
```

---

## PHARM-001 Validation Program

```r
# pharm001_ch02_validate.R
# Study PHARM-001 — Chapter 2
# Validate SDTM domains before any ADaM derivation

library(dplyr)
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm
ae <- pharmaversesdtm::ae
ex <- pharmaversesdtm::ex

cat("=== PHARM-001 SDTM Validation ===\n\n")

# ---- Helper ----
check <- function(label, condition) {
  status <- if (condition) "[PASS]" else "[FAIL]"
  cat(sprintf("%s %s\n", status, label))
  invisible(condition)
}

# ---- DM checks ----
check("DM: USUBJID is unique",
      !any(duplicated(dm$USUBJID)))

check("DM: SEX is controlled terminology",
      all(dm$SEX %in% c("M", "F", "U", "UNDIFFERENTIATED")))

check("DM: No missing STUDYID",
      !any(is.na(dm$STUDYID)))

check("DM: ACTARM populated",
      !any(is.na(dm$ACTARM)))

# Validate ISO8601 date format for RFSTDTC
iso_pattern <- "^\\d{4}(-\\d{2}(-\\d{2})?)?$"
rfstdtc_vals <- dm$RFSTDTC[!is.na(dm$RFSTDTC) & dm$RFSTDTC != ""]
check("DM: RFSTDTC is ISO8601 format",
      all(grepl(iso_pattern, rfstdtc_vals)))

# ---- AE checks ----
ae_subjects  <- unique(ae$USUBJID)
dm_subjects  <- unique(dm$USUBJID)
orphan_ae    <- setdiff(ae_subjects, dm_subjects)
check(sprintf("AE: All subjects in DM (%d AE subjects, %d in DM)",
              length(ae_subjects), length(dm_subjects)),
      length(orphan_ae) == 0)

check("AE: AESER is Y or N",
      all(ae$AESER %in% c("Y", "N", NA)))

check("AE: AESEQ is present",
      "AESEQ" %in% names(ae))

# ---- EX checks ----
ex_subjects <- unique(ex$USUBJID)
check("EX: All subjects in DM",
      length(setdiff(ex_subjects, dm_subjects)) == 0)

check("EX: EXDOSE is numeric",
      is.numeric(ex$EXDOSE))

# ---- Cross-domain: subjects with AE but no EX ----
ae_no_ex <- setdiff(ae_subjects, ex_subjects)
check(sprintf("Cross: AE subjects all have EX records (%d without EX)",
              length(ae_no_ex)),
      length(ae_no_ex) == 0)

# ---- Summary ----
cat("\nDomain sizes:\n")
for (nm in c("dm","ae","ex","vs","ds")) {
  df <- tryCatch(pharmaversesdtm::get(nm), error = function(e) NULL)
  if (is.null(df)) df <- get(nm, envir = .GlobalEnv)
  cat(sprintf("  %-5s: %d rows × %d cols\n", toupper(nm), nrow(df), ncol(df)))
}
```

---

## Metadata with {metacore}

In a real submission, variable metadata (labels, types, lengths, codelists) comes from a spec spreadsheet. `metacore` loads that spec:

```r
library(metacore)

# Load from Excel spec (see metacore vignette for format)
# spec <- spec_to_metacore("specs/ADaM_spec.xlsx", where_sep_sheet = FALSE)

# What metacore gives you:
# - Variable labels
# - Type (text/float/integer)
# - Length (for XPT)
# - Codelists (for CT validation)
```

We'll use metacore in Chapter 9 when we export to XPT. For now, know it exists.

---

## ADaM Structure Preview

Every ADaM dataset joins back to ADSL. Know the pattern before we build it:

| ADaM | Structure | Source SDTM |
|------|-----------|-------------|
| ADSL | One row per subject | DM + DS + EX |
| ADAE | One row per AE occurrence | AE + ADSL |
| ADTTE | One row per subject per event type | ADSL + AE/DS |

The hierarchy is always: SDTM → ADSL → analysis datasets.

---

## What We Have Now

```
pharm001_ch01.R          [done] — load DM, count subjects, demographics summary
pharm001_ch02_validate.R [done] — validate SDTM domains before ADaM work
```

Run validation first, then proceed:
```r
source("pharm001_ch02_validate.R")   # must all PASS
source("pharm001_ch01.R")
```

---

## Exercises

**1.** For `dm`, `ae`, `vs`, `ex`: list all `*DTC` variables (ISO8601 dates). Count how many rows have partial dates (missing day — pattern matches `^\d{4}-\d{2}$`).

**2.** Write `validate_ct(df, var, allowed)` that returns a data frame of violations: `USUBJID`, the bad value, and the variable name.

**3.** Load `ds` (disposition). Map `DSDECOD` to a completion status: `"COMPLETED"`, `"WITHDRAWN"`, or `"OTHER"`.

**4. (Sets up Chapter 3)** Look at `pharmaverseraw::dm_raw` (install `pharmaverseraw` if needed). Compare its `SEX`/`GENDER` column to `pharmaversesdtm::dm$SEX`. The raw data has values like `"Male"` and `"Female"`. The SDTM has `"M"` and `"F"`. What's the mapping rule? Write a function `map_sex(raw_val)` that does the conversion.

---

*Next: Chapter 3 — we take raw data and use sdtm.oak to produce SDTM-compliant domains for PHARM-001.*

---

## Solutions

### Exercise 1

Find all `*DTC` variables and count partial dates (YYYY-MM pattern) in each domain.

```r
# SAS equivalent:
# /* Find all date variables */
# PROC CONTENTS DATA=dm OUT=dm_meta; RUN;
# DATA dtc_vars; SET dm_meta; WHERE INDEX(NAME, "DTC") > 0; RUN;
# /* Count partial dates — LENGTH = 7 means "YYYY-MM" */
# DATA dm2; SET dm; len_rfstdtc = LENGTH(TRIM(RFSTDTC)); RUN;
# PROC FREQ DATA=dm2; TABLES len_rfstdtc; RUN;

library(dplyr)
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm
ae <- pharmaversesdtm::ae
vs <- pharmaversesdtm::vs
ex <- pharmaversesdtm::ex

count_partial_dates <- function(df, domain_name) {
  dtc_vars <- names(df)[grepl("DTC$", names(df))]
  if (length(dtc_vars) == 0) {
    cat(sprintf("%-5s: no *DTC variables\n", domain_name))
    return(invisible(NULL))
  }
  partial_pattern <- "^\\d{4}-\\d{2}$"   # YYYY-MM only
  for (v in dtc_vars) {
    vals     <- df[[v]][!is.na(df[[v]]) & df[[v]] != ""]
    n_total  <- length(vals)
    n_partial <- sum(grepl(partial_pattern, vals))
    cat(sprintf("  %-25s: %4d total, %3d partial (%.1f%%)\n",
                paste0(domain_name, "$", v),
                n_total, n_partial,
                if (n_total > 0) 100 * n_partial / n_total else 0))
  }
}

cat("=== Partial Date Summary (YYYY-MM pattern) ===\n")
count_partial_dates(dm, "DM")
count_partial_dates(ae, "AE")
count_partial_dates(vs, "VS")
count_partial_dates(ex, "EX")
```

### Exercise 2

Write a generic CT validation function that returns a data frame of violations.

```r
# SAS equivalent:
# PROC FORMAT; VALUE $SEXCT 'M'='ok' 'F'='ok' 'U'='ok' OTHER='VIOLATION'; RUN;
# DATA violations; SET dm;
#   IF PUT(SEX,$SEXCT.) = 'VIOLATION' THEN OUTPUT;
# RUN;

library(dplyr)
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm
ae <- pharmaversesdtm::ae

# Function: returns data frame of CT violations
validate_ct <- function(df, var, allowed) {
  if (!var %in% names(df)) {
    warning(sprintf("Variable '%s' not found in data frame", var))
    return(data.frame())
  }
  df %>%
    filter(!is.na(.data[[var]]), !.data[[var]] %in% allowed) %>%
    mutate(variable = var, bad_value = as.character(.data[[var]])) %>%
    select(USUBJID, variable, bad_value)
}

# Check SEX in DM
sex_violations <- validate_ct(dm, "SEX", c("M","F","U","UNDIFFERENTIATED"))
cat(sprintf("SEX violations: %d\n", nrow(sex_violations)))

# Check AESER in AE
aeser_violations <- validate_ct(ae, "AESER", c("Y","N"))
cat(sprintf("AESER violations: %d\n", nrow(aeser_violations)))

# If violations exist, print them
if (nrow(sex_violations) > 0) print(sex_violations)
```

### Exercise 3

Map `DSDECOD` to a three-level completion status.

```r
# SAS:
# DATA ds2;
#   SET ds;
#   IF      DSDECOD = "COMPLETED"  THEN STATUS = "COMPLETED";
#   ELSE IF DSDECOD IN ("WITHDRAWAL BY SUBJECT","ADVERSE EVENT",
#                       "LACK OF EFFICACY") THEN STATUS = "WITHDRAWN";
#   ELSE                                         STATUS = "OTHER";
# RUN;

library(dplyr)
library(pharmaversesdtm)

ds <- pharmaversesdtm::ds

ds_status <- ds %>%
  mutate(
    COMPLETION_STATUS = case_when(
      DSDECOD == "COMPLETED"                              ~ "COMPLETED",
      DSDECOD %in% c("WITHDRAWAL BY SUBJECT",
                     "ADVERSE EVENT",
                     "LACK OF EFFICACY",
                     "LOST TO FOLLOW-UP",
                     "DEATH",
                     "PHYSICIAN DECISION")               ~ "WITHDRAWN",
      TRUE                                                ~ "OTHER"
    )
  )

cat("Disposition status breakdown:\n")
print(count(ds_status, COMPLETION_STATUS))
```

### Exercise 4

Compare `pharmaverseraw::dm_raw$GENDER` to `pharmaversesdtm::dm$SEX` and write a mapping function.

```r
# SAS:
# /* Check raw vs SDTM values */
# PROC FREQ DATA=raw_dm; TABLES GENDER; RUN;
# PROC FREQ DATA=dm;     TABLES SEX;    RUN;

library(dplyr)

# Note: pharmaverseraw may need to be installed separately
# install.packages("pharmaverseraw")
library(pharmaverseraw)
library(pharmaversesdtm)

raw_dm <- pharmaverseraw::dm_raw
sdtm_dm <- pharmaversesdtm::dm

cat("Raw GENDER values:\n")
print(table(raw_dm$GENDER, useNA = "always"))

cat("\nSDTM SEX values:\n")
print(table(sdtm_dm$SEX, useNA = "always"))

# Mapping function — case-insensitive for robustness
# SAS: automatically case-insensitive; R: must handle explicitly
map_sex <- function(raw_val) {
  val_up <- toupper(trimws(raw_val))
  dplyr::case_when(
    val_up == "MALE"             ~ "M",
    val_up == "FEMALE"           ~ "F",
    val_up == "UNKNOWN"          ~ "U",
    val_up %in% c("1", "M")     ~ "M",
    val_up %in% c("2", "F")     ~ "F",
    TRUE                          ~ NA_character_
  )
}

# Verify mapping
cat("\nMapping check:\n")
test_vals <- c("Male","Female","MALE","FEMALE","male","Unknown","")
mapped    <- map_sex(test_vals)
data.frame(raw = test_vals, sdtm = mapped) %>% print()
```
