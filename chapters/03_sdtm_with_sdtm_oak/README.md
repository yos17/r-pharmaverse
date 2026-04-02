# Chapter 3: SDTM with {sdtm.oak}

*Map raw collected data to CDISC SDTM using an algorithm-based approach.*

---

## What sdtm.oak Does

In SAS, you map raw data to SDTM with manual `DATA` steps and format statements. `sdtm.oak` provides a declarative, algorithm-based approach: you define *how* to assign each SDTM variable, and the package handles the mechanics.

```r
library(sdtm.oak)
library(pharmaverseraw)  # example raw datasets
library(dplyr)
library(lubridate)
```

---

## The Raw Data

`pharmaverseraw` provides realistic raw CRF datasets:

```r
# Raw adverse events
raw_ae <- pharmaverseraw::ae_raw
glimpse(raw_ae)

# Raw demographics
raw_dm <- pharmaverseraw::dm_raw
glimpse(raw_dm)
```

Raw data looks like what comes out of EDC (Electronic Data Capture) — messy column names, no controlled terminology, dates in various formats.

---

## The Algorithm Concept

`sdtm.oak` uses "algorithms" — named transformations that move raw variables to SDTM variables. Each algorithm is a function that assigns a value based on a rule.

The core algorithms:

| Algorithm | What it does |
|-----------|-------------|
| `assign_ct()` | Assign from controlled terminology codelist |
| `assign_no_ct()` | Assign value directly (no CT lookup) |
| `assign_datetime()` | Convert to ISO8601 datetime |
| `hardcode_ct()` | Hard-code a CT value (e.g., DOMAIN = "AE") |
| `hardcode_no_ct()` | Hard-code a non-CT value |
| `combine_dtc()` | Combine date + time into ISO8601 |

---

## Building a DM Domain

```r
# dm_mapping.R
# Map raw DM to SDTM DM using sdtm.oak

library(sdtm.oak)
library(pharmaverseraw)
library(dplyr)

# Load raw data
raw_dm <- pharmaverseraw::dm_raw

# Load the CDISC controlled terminology codelist
# (sdtm.oak ships with a built-in CT object)
ct <- sdtm.oak::ct_spec_example("ct-01-prog")

# --- Start the mapping ---
dm <- raw_dm %>%
  
  # Fixed domain variables
  hardcode_no_ct(oak_id_vars   = oak_id_vars(),
                 raw_dat       = raw_dm,
                 raw_var       = "DOMAIN",
                 tgt_var       = "DOMAIN",
                 tgt_val       = "DM") %>%
  
  # USUBJID — concatenate STUDYID + SITEID + SUBJID
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat     = raw_dm,
               raw_var     = "SUBJID",
               tgt_var     = "USUBJID",
               .fn         = function(x) paste(x$STUDYID, x$SITEID, x, sep="-")) %>%
  
  # AGE — direct assignment (numeric)
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat     = raw_dm,
               raw_var     = "AGE",
               tgt_var     = "AGE") %>%
  
  # SEX — CT assignment (maps raw "Male"/"Female" to "M"/"F")
  assign_ct(oak_id_vars = oak_id_vars(),
            raw_dat     = raw_dm,
            raw_var     = "GENDER",
            tgt_var     = "SEX",
            ct_spec     = ct,
            ct_clst     = "C66731") %>%   # C66731 = Sex codelist in NCI CT
  
  # RACE — CT
  assign_ct(oak_id_vars = oak_id_vars(),
            raw_dat     = raw_dm,
            raw_var     = "RACE",
            tgt_var     = "RACE",
            ct_spec     = ct,
            ct_clst     = "C74457") %>%
  
  # RFSTDTC — ISO8601 conversion
  assign_datetime(oak_id_vars = oak_id_vars(),
                  raw_dat     = raw_dm,
                  raw_var     = "RFICDTC",
                  tgt_var     = "RFICDTC",
                  raw_fmt     = "d-m-y") %>%
  
  # Select SDTM variables in correct order
  select(STUDYID, DOMAIN, USUBJID, SUBJID, SITEID,
         RFICDTC, RFSTDTC, RFENDTC,
         DTHDTC, DTHFL,
         AGE, AGEU, SEX, RACE, ETHNIC,
         ARMCD, ARM, ACTARMCD, ACTARM)
```

---

## Building an AE Domain

AE is more complex — it has repeat records per subject, start/end dates, severity grades, seriousness flags.

```r
# ae_mapping.R

library(sdtm.oak)
library(pharmaverseraw)
library(dplyr)

raw_ae <- pharmaverseraw::ae_raw
ct     <- sdtm.oak::ct_spec_example("ct-01-prog")

ae <- raw_ae %>%
  
  # DOMAIN
  hardcode_no_ct(oak_id_vars = oak_id_vars(),
                 raw_dat     = raw_ae,
                 raw_var     = "DOMAIN",
                 tgt_var     = "DOMAIN",
                 tgt_val     = "AE") %>%
  
  # AETERM — verbatim term (no CT)
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat     = raw_ae,
               raw_var     = "AETERM",
               tgt_var     = "AETERM") %>%
  
  # AEDECOD — MedDRA preferred term (comes from coding)
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat     = raw_ae,
               raw_var     = "MEDDRA_PT",
               tgt_var     = "AEDECOD") %>%
  
  # AEBODSYS — MedDRA SOC
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat     = raw_ae,
               raw_var     = "MEDDRA_SOC",
               tgt_var     = "AEBODSYS") %>%
  
  # AESER — seriousness (Y/N CT)
  assign_ct(oak_id_vars = oak_id_vars(),
            raw_dat     = raw_ae,
            raw_var     = "SERIOUS",
            tgt_var     = "AESER",
            ct_spec     = ct,
            ct_clst     = "C66742") %>%   # No/Yes
  
  # AESEV — severity (MILD/MODERATE/SEVERE)
  assign_ct(oak_id_vars = oak_id_vars(),
            raw_dat     = raw_ae,
            raw_var     = "SEVERITY",
            tgt_var     = "AESEV",
            ct_spec     = ct,
            ct_clst     = "C41338") %>%   # Severity
  
  # AESTDTC — start date/time
  assign_datetime(oak_id_vars = oak_id_vars(),
                  raw_dat     = raw_ae,
                  raw_var     = "AESTDTC",
                  tgt_var     = "AESTDTC",
                  raw_fmt     = "d-m-y") %>%
  
  # AEENDTC — end date
  assign_datetime(oak_id_vars = oak_id_vars(),
                  raw_dat     = raw_ae,
                  raw_var     = "AEENDTC",
                  tgt_var     = "AEENDTC",
                  raw_fmt     = "d-m-y") %>%
  
  # AESEQ — sequence number (derive after all records are built)
  group_by(USUBJID) %>%
  mutate(AESEQ = row_number()) %>%
  ungroup() %>%
  
  select(STUDYID, DOMAIN, USUBJID, AESEQ, AETERM, AEDECOD, AEBODSYS,
         AESEV, AESER, AEACN, AESTDTC, AEENDTC, AEOUT, AETOXGR)
```

---

## Key sdtm.oak Concepts

### oak_id_vars()

Every `sdtm.oak` function takes `oak_id_vars()` — these are the variables that uniquely identify a record in the raw data. By default: `STUDYID`, `USUBJID`, and the raw record sequence variable.

```r
# Default oak_id_vars — usually fine for standard mappings
oak_id_vars()  
```

### Algorithms Are Composable

You chain them with `%>%`. Each step adds a variable to the dataset. Failed mappings (no matching CT value, missing raw) produce `NA`.

### CT Codes

CDISC NCI thesaurus codelists have C-codes. The most common ones:

| Codelist | C-code | Contents |
|----------|--------|---------|
| Sex | C66731 | M, F, U, UNDIFFERENTIATED |
| No/Yes | C66742 | N, Y |
| Severity | C41338 | MILD, MODERATE, SEVERE |
| Race | C74457 | WHITE, BLACK OR AFRICAN AMERICAN, etc. |
| Route | C66729 | ORAL, INTRAVENOUS, etc. |
| Action Taken | C66767 | DOSE NOT CHANGED, etc. |

---

## Program: SDTM Mapping Report

```r
# sdtm_report.R
# Map raw data and produce a comparison report

library(sdtm.oak)
library(pharmaverseraw)
library(pharmaversesdtm)   # reference SDTM
library(dplyr)

raw_dm <- pharmaverseraw::dm_raw
ref_dm <- pharmaversesdtm::dm    # what it should look like

cat("=== Raw DM vs SDTM DM ===\n\n")

# Compare column coverage
raw_vars <- names(raw_dm)
sdtm_vars <- names(ref_dm)
added     <- setdiff(sdtm_vars, raw_vars)  # derived in mapping
dropped   <- setdiff(raw_vars, sdtm_vars)  # raw-only, not in SDTM

cat("Variables added by mapping:\n")
for (v in added) cat(sprintf("  + %s\n", v))

cat("\nRaw variables not carried to SDTM:\n")
for (v in dropped) cat(sprintf("  - %s\n", v))

# Compare subject counts
cat(sprintf("\nRaw subjects:  %d\n", n_distinct(raw_dm$USUBJID)))
cat(sprintf("SDTM subjects: %d\n", n_distinct(ref_dm$USUBJID)))

# Date format comparison
cat("\nDate format check (first 3 RFSTDTC values):\n")
cat("  Raw:  ", paste(head(raw_dm$RFSTDTC, 3), collapse=", "), "\n")
cat("  SDTM: ", paste(head(ref_dm$RFSTDTC, 3), collapse=", "), "\n")

# Sex CT mapping
raw_sex  <- table(raw_dm$SEX)
sdtm_sex <- table(ref_dm$SEX)
cat("\nSex mapping:\n")
cat("  Raw: "); print(raw_sex)
cat("  SDTM:"); print(sdtm_sex)
```

---

## Exercises

**1. Map VS domain**

Load `pharmaverseraw::vs_raw`. Map it to SDTM VS including:
- `VSTESTCD` / `VSTEST` (test code and name)
- `VSORRES` / `VSSTRESC` (original result and standardized)
- `VSORRESU` / `VSSTRESU` (original and standard units)
- `VSSTDTC` (date/time)
- `VISITNUM` / `VISIT`

**2. Sequence numbers**

Add `VSSEQ` as a sequence number within each USUBJID, ordered by `VSTESTCD` then `VSSTDTC`.

**3. Date imputation**

Some SDTM dates are partial — the day is missing. Write a function `impute_date(dtc, rule)` where `rule = "first"` fills day=01 and `rule = "last"` fills the last day of the month.

**4. CT validation**

After mapping, check that all `AESER` values are in `c("Y", "N")`. How would you flag rows that couldn't be mapped?

---

*Next: Chapter 4 — ADaM with {admiral}: the core package for building analysis datasets*
