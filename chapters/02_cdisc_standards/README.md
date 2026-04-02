# Chapter 2: CDISC Standards in Code

*SDTM and ADaM structure in R. Controlled terminology. Metadata with {metacore}.*

---

## CDISC in a Nutshell (for R Programmers)

You know what CDISC is. Here's the translation layer:

| CDISC concept | How it lives in R |
|--------------|-------------------|
| SDTM domain | A data frame with specific column names/types |
| Controlled terminology | Character constants (`"Y"`, `"N"`) + codelists |
| Variable attributes (label, length, type) | `haven::labelled` attributes |
| SAS XPT format | Read/write via `haven`, export via `xportr` |
| Define-XML / metadata spec | `metacore` objects |
| Codelist | Named character vector or a `codelist` in metacore |

---

## The SDTM Domains You'll Work With

```r
library(pharmaversesdtm)

# Core domains
dm  <- pharmaversesdtm::dm   # Demographics
ae  <- pharmaversesdtm::ae   # Adverse Events
ex  <- pharmaversesdtm::ex   # Exposure
vs  <- pharmaversesdtm::vs   # Vital Signs
lb  <- pharmaversesdtm::lb   # Laboratory
cm  <- pharmaversesdtm::cm   # Concomitant Medications
ds  <- pharmaversesdtm::ds   # Disposition
mh  <- pharmaversesdtm::mh   # Medical History
sc  <- pharmaversesdtm::sc   # Subject Characteristics
sv  <- pharmaversesdtm::sv   # Subject Visits

# Look at DM structure
glimpse(dm)
```

Key things to notice in `dm`:

```
$ STUDYID  <chr> — study identifier
$ DOMAIN   <chr> — always "DM"
$ USUBJID  <chr> — unique subject identifier (STUDYID-SITEID-SUBJID)
$ SUBJID   <chr> — subject identifier within site
$ RFSTDTC  <chr> — reference start date/time (ISO8601)
$ RFENDTC  <chr> — reference end date/time
$ RFXSTDTC <chr> — exposure start
$ RFXENDTC <chr> — exposure end
$ RFICDTC  <chr> — informed consent date
$ DTHDTC   <chr> — death date
$ DTHFL    <chr> — death flag
$ SITEID   <chr> — site
$ AGE      <dbl> — age in AGEDGU units
$ AGEU     <chr> — age units
$ SEX      <chr> — sex (controlled terminology: M/F)
$ RACE     <chr> — race (CT)
$ ETHNIC   <chr> — ethnicity (CT)
$ ARMCD    <chr> — planned arm code
$ ARM      <chr> — planned arm description
$ ACTARMCD <chr> — actual arm code
$ ACTARM   <chr> — actual arm description
```

---

## ISO8601 Dates

SDTM uses ISO8601 character dates (`"2021-03-15"`, `"2021-03-15T09:30:00"`). You'll convert them to R Date/POSIXct for calculations.

```r
library(dplyr)
library(lubridate)  # makes date arithmetic easy

dm_dates <- dm %>%
  mutate(
    # Convert ISO8601 to R Date
    RFSTDT = as.Date(RFSTDTC),
    RFENDT = as.Date(RFENDTC),
    
    # Calculate treatment duration (days)
    TRTDUR = as.integer(RFENDT - RFSTDT + 1),
    
    # Partial dates: "2021-03" (day unknown) → keep as NA date
    # Use admiral::convert_dtc_to_dt() for CDISC-compliant partial date handling
  )
```

**Admiral's approach** for ISO8601 partial dates:

```r
library(admiral)

# admiral provides derive_vars_dt() for proper ISO8601 handling
# including imputation rules (first/last day of month, etc.)
# See Chapter 5 for full usage
```

---

## Controlled Terminology

CDISC controlled terminology (CT) values are just character constants in R. The discipline is applying them consistently.

```r
# Define your CT values as constants (or use a codelist)
SEX_CT     <- c(F = "F", M = "M", U = "U")
RACE_CT    <- c(
  WHITE    = "WHITE",
  BLACK    = "BLACK OR AFRICAN AMERICAN",
  ASIAN    = "ASIAN",
  OTHER    = "OTHER",
  UNKNOWN  = "UNKNOWN"
)
YESNO_CT   <- c(Y = "Y", N = "N")

# Validate
ae %>%
  filter(!AESER %in% c("Y", "N", NA)) %>%
  nrow()   # should be 0
```

---

## Metadata with {metacore}

In a real submission, your metadata lives in a spec (Excel or XML). `metacore` loads that spec into R so you can programmatically apply labels, types, lengths, and codelists.

```r
library(metacore)

# Load from a spec (Excel format — see metacore vignette for format)
spec <- spec_to_metacore("path/to/spec.xlsx", where_sep_sheet = FALSE)

# Or build programmatically for demonstration:
var_spec <- tibble::tribble(
  ~dataset, ~variable, ~label,                            ~type,  ~length,
  "ADSL",   "USUBJID", "Unique Subject Identifier",       "text",  200,
  "ADSL",   "TRT01P",  "Planned Treatment for Period 01", "text",  200,
  "ADSL",   "AGE",     "Age",                             "float",  8,
  "ADSL",   "AGEU",    "Age Units",                       "text",   8,
  "ADSL",   "SEX",     "Sex",                             "text",   2,
  "ADSL",   "RACE",    "Race",                            "text",  200,
)
```

We'll use `metacore` extensively in Chapter 5 (ADSL) and Chapter 9 (xportr).

---

## Program: Validate SDTM Domains

```r
# validate_sdtm.R
# Check SDTM domains against common rules

library(tidyverse)
library(pharmaversesdtm)

# ---- Load domains ----
dm <- pharmaversesdtm::dm
ae <- pharmaversesdtm::ae
ex <- pharmaversesdtm::ex

# ---- Validation functions ----

# Check required variables exist
check_required_vars <- function(df, domain, required) {
  missing <- setdiff(required, names(df))
  if (length(missing) > 0) {
    cat(sprintf("[FAIL] %s: Missing required variables: %s\n",
                domain, paste(missing, collapse=", ")))
    return(FALSE)
  }
  cat(sprintf("[PASS] %s: All required variables present\n", domain))
  TRUE
}

# Check USUBJID uniqueness (for DM)
check_dm_uniqueness <- function(dm) {
  dupes <- dm %>% count(USUBJID) %>% filter(n > 1)
  if (nrow(dupes) > 0) {
    cat(sprintf("[FAIL] DM: %d duplicate USUBJID(s)\n", nrow(dupes)))
    return(FALSE)
  }
  cat("[PASS] DM: USUBJID is unique\n")
  TRUE
}

# Check all AE subjects are in DM
check_ae_in_dm <- function(ae, dm) {
  ae_subjects <- unique(ae$USUBJID)
  dm_subjects <- unique(dm$USUBJID)
  orphans     <- setdiff(ae_subjects, dm_subjects)
  if (length(orphans) > 0) {
    cat(sprintf("[FAIL] AE: %d USUBJID(s) not in DM\n", length(orphans)))
    return(FALSE)
  }
  cat("[PASS] AE: All subjects in DM\n")
  TRUE
}

# Check ISO8601 date format
check_dtc_format <- function(df, domain, dtc_vars) {
  iso_pattern <- "^\\d{4}(-\\d{2}(-\\d{2}(T\\d{2}:\\d{2}(:\\d{2})?)?)?)?$"
  for (var in dtc_vars) {
    if (!var %in% names(df)) next
    vals <- df[[var]][!is.na(df[[var]]) & df[[var]] != ""]
    bad  <- vals[!grepl(iso_pattern, vals)]
    if (length(bad) > 0) {
      cat(sprintf("[FAIL] %s.%s: %d non-ISO8601 values: %s\n",
                  domain, var, length(bad),
                  paste(head(bad, 3), collapse=", ")))
    } else {
      cat(sprintf("[PASS] %s.%s: ISO8601 format OK\n", domain, var))
    }
  }
}

# Check controlled terminology
check_ct <- function(df, domain, var, allowed_values) {
  vals <- unique(df[[var]][!is.na(df[[var]])])
  bad  <- setdiff(vals, allowed_values)
  if (length(bad) > 0) {
    cat(sprintf("[FAIL] %s.%s: Non-CT values: %s\n",
                domain, var, paste(bad, collapse=", ")))
    return(FALSE)
  }
  cat(sprintf("[PASS] %s.%s: Controlled terminology OK\n", domain, var))
  TRUE
}

# ---- Run validations ----
cat("=== SDTM Domain Validation ===\n\n")

# DM
check_required_vars(dm, "DM", c("STUDYID","DOMAIN","USUBJID","SUBJID",
                                  "RFSTDTC","SEX","AGE","ACTARM"))
check_dm_uniqueness(dm)
check_dtc_format(dm, "DM", c("RFSTDTC", "RFENDTC", "RFICDTC"))
check_ct(dm, "DM", "SEX", c("M","F","U","UNDIFFERENTIATED"))
check_ct(dm, "DM", "ACTARM", unique(dm$ACTARM))  # at minimum not NA

cat("\n")

# AE
check_required_vars(ae, "AE", c("STUDYID","DOMAIN","USUBJID","AESEQ",
                                   "AETERM","AEDECOD","AEBODSYS",
                                   "AESTDTC","AESER"))
check_ae_in_dm(ae, dm)
check_dtc_format(ae, "AE", c("AESTDTC","AEENDTC"))
check_ct(ae, "AE", "AESER", c("Y","N"))
check_ct(ae, "AE", "AEACN", c("DOSE NOT CHANGED","DRUG WITHDRAWN",
                               "DOSE REDUCED","DOSE INCREASED",
                               "NOT APPLICABLE","UNKNOWN","N/A",NA))

cat("\n")

# EX
check_required_vars(ex, "EX", c("STUDYID","DOMAIN","USUBJID","EXSEQ",
                                   "EXTRT","EXDOSE","EXDOSU",
                                   "EXSTDTC","EXENDTC"))
check_dtc_format(ex, "EX", c("EXSTDTC","EXENDTC"))

cat("\n=== Validation complete ===\n")
```

---

## ADaM Structure

ADaM datasets in R are just data frames following the CDISC ADaM conventions. The key datasets you'll build:

| ADaM | Contents | Key structure |
|------|---------|---------------|
| ADSL | Subject-level analysis | One row per subject |
| ADAE | Adverse event analysis | One row per AE occurrence |
| ADTTE | Time-to-event analysis | One row per subject per event type |
| ADLB | Lab analysis | One row per subject per visit per parameter |
| ADVS | Vital signs analysis | Same as ADLB pattern |
| ADPC | PK concentrations | One row per sample |

---

## Exercises

**1. Domain inspection**

For each of `dm`, `ae`, `vs`, `ex` from `pharmaversesdtm`:
- List all `*DTC` variables (ISO8601 dates)
- Count how many have partial dates (missing day or month)

**2. Controlled terminology check**

Write a function `validate_ct(df, domain, var, codelist)` that returns a tidy data frame of violations (USUBJID + bad value + variable name).

**3. Subject disposition**

Load `ds` (disposition domain). Map `DSDECOD` values to derive a "completion status" for each subject:
- `"COMPLETED"` if DSDECOD = "COMPLETED"
- `"WITHDRAWN"` if DSDECOD starts with "WITHDRAWAL"
- `"OTHER"` otherwise

**4. Cross-domain consistency**

Check: every subject in `ae` has an exposure record in `ex`. Find subjects with AEs but no exposure.

---

*Next: Chapter 3 — SDTM with {sdtm.oak}: converting raw data to CDISC SDTM domains*
