# Chapter 3: Map Raw Data to SDTM

*Last chapter we validated the SDTM. Now we understand where it came from — and build it.*

---

## Where We Left Off

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM domains
```

In the real world, SDTM doesn't arrive pre-built. It gets programmed from raw CRF data. This chapter adds a SDTM mapping program to PHARM-001.

The task: take `pharmaverseraw::dm_raw` and produce a CDISC-compliant DM domain. Start with two variables — `DOMAIN` and `SEX` — then build the full mapping.

---

## The Raw Data

```r
library(pharmaverseraw)

raw_dm <- pharmaverseraw::dm_raw
glimpse(raw_dm)
```

Raw data looks like what comes out of EDC — messy column names, non-CT values, dates in various formats. Key raw columns and their SDTM targets:

| Raw column | SDTM variable | Transformation needed |
|-----------|--------------|----------------------|
| `SUBJID` | `SUBJID` | direct |
| `GENDER` | `SEX` | `"Male"` → `"M"`, `"Female"` → `"F"` |
| `RFICDTC` | `RFICDTC` | date format conversion |
| — | `DOMAIN` | hard-coded `"DM"` |
| — | `USUBJID` | concatenate STUDYID + SITEID + SUBJID |

---

## The Naive Approach (and Why It Breaks)

**In SAS**, a SDTM mapping looks like this:

```sas
DATA dm;
  SET raw_dm;
  DOMAIN = "DM";

  /* CT mapping — written by hand each study */
  IF UPCASE(GENDER) = "MALE"   THEN SEX = "M";
  ELSE IF UPCASE(GENDER) = "FEMALE" THEN SEX = "F";
  ELSE SEX = "";  /* unmapped values become blank */

  /* Date conversion from EDC format to ISO8601 */
  IF INDEX(RFICDTC, "T") > 0 THEN
    RFICDTC_DT = INPUT(SUBSTR(RFICDTC, 1, 10), YYMMDD10.);
  ELSE
    RFICDTC_DT = INPUT(RFICDTC, YYMMDD10.);
  FORMAT RFICDTC_DT DATE9.;

  USUBJID = CATS(STUDYID, "-", SITEID, "-", SUBJID);
RUN;
```

**In R (naive approach):**

```r
dm_naive <- raw_dm %>%
  mutate(
    DOMAIN  = "DM",
    SEX     = case_when(
      GENDER == "Male"   ~ "M",
      GENDER == "Female" ~ "F",
      TRUE               ~ NA_character_
    )
  )
```

Both approaches work for this dataset. But what about the next study where raw values are `"MALE"`, `"male"`, `"1"`, `"2"`? You'd rewrite the mapping every time, and every programmer would write a slightly different version.

This is the problem `sdtm.oak` solves.

> ⚠️ **Key difference:** SAS `UPCASE(GENDER) = "MALE"` handles case variations automatically. In R, `GENDER == "Male"` only matches that exact case — `"MALE"` and `"male"` would fall through to `NA`. The `sdtm.oak` `assign_ct()` function uses standardised CDISC codelists that handle these variations consistently.

---

## What sdtm.oak Does

`sdtm.oak` provides **algorithm functions** — named transformations that move raw variables to SDTM variables according to CDISC rules. The algorithms:

| Algorithm | What it does |
|-----------|-------------|
| `hardcode_no_ct()` | Hard-code a value (e.g., DOMAIN = "DM") |
| `hardcode_ct()` | Hard-code a CT value |
| `assign_no_ct()` | Copy raw variable directly |
| `assign_ct()` | Map raw value to CT using a codelist |
| `assign_datetime()` | Convert to ISO8601 |

The key insight: CT mapping uses CDISC NCI codelist C-codes, not hand-coded `case_when`. This makes mappings reproducible across studies.

---

## Step 1: DOMAIN

Start simple. `DOMAIN` is always hard-coded:

**In SAS:**
```sas
DATA dm;
  SET raw_dm;
  DOMAIN = "DM";
RUN;
```

**In R (pharmaverse — sdtm.oak):**
```r
library(sdtm.oak)
library(pharmaverseraw)
library(dplyr)

raw_dm <- pharmaverseraw::dm_raw

# oak_id_vars() identifies each raw record — by default STUDYID + USUBJID + raw sequence
dm <- raw_dm %>%
  hardcode_no_ct(
    oak_id_vars = oak_id_vars(),
    raw_dat     = raw_dm,
    raw_var     = "DOMAIN",
    tgt_var     = "DOMAIN",
    tgt_val     = "DM"
  )

table(dm$DOMAIN)   # should be all "DM"
```

---

## Step 2: SEX with CT Mapping

Now the interesting part. Load the CT spec and use `assign_ct()`:

**In SAS:**
```sas
PROC FORMAT;
  VALUE $SEXCT
    'Male'   = 'M'
    'Female' = 'F'
    OTHER    = ' ';   /* unmapped → blank (easy to miss!) */
RUN;

DATA dm;
  SET raw_dm;
  SEX = PUT(GENDER, $SEXCT.);
  /* Problem: blanks are silently wrong; no audit trail of unmapped values */
RUN;
```

**In R (pharmaverse — sdtm.oak):**
```r
ct <- sdtm.oak::ct_spec_example("ct-01-prog")

dm <- dm %>%
  assign_ct(
    oak_id_vars = oak_id_vars(),
    raw_dat     = raw_dm,
    raw_var     = "GENDER",    # raw column
    tgt_var     = "SEX",       # SDTM target
    ct_spec     = ct,
    ct_clst     = "C66731"     # NCI codelist: Sex (M/F/U/UNDIFFERENTIATED)
  )

table(raw_dm$GENDER, dm$SEX, useNA = "always")
```

The codelist `C66731` maps `"Male"` → `"M"`, `"Female"` → `"F"`, etc. Any raw value not in the codelist maps to `NA` — which surfaces as a problem, not a silent wrong value.

---

## The Full DM Mapping

```r
# pharm001_ch03_dm.R
# Study PHARM-001 — Chapter 3
# Map raw DM to SDTM DM using sdtm.oak

library(sdtm.oak)
library(pharmaverseraw)
library(pharmaversesdtm)
library(dplyr)

raw_dm <- pharmaverseraw::dm_raw
ct     <- sdtm.oak::ct_spec_example("ct-01-prog")

cat("=== PHARM-001 DM Mapping ===\n")
cat(sprintf("Raw rows: %d\n", nrow(raw_dm)))

dm <- raw_dm %>%
  
  # Fixed domain
  hardcode_no_ct(oak_id_vars = oak_id_vars(),
                 raw_dat = raw_dm,
                 raw_var = "DOMAIN",
                 tgt_var = "DOMAIN",
                 tgt_val = "DM") %>%
  
  # USUBJID: concatenate
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat     = raw_dm,
               raw_var     = "SUBJID",
               tgt_var     = "USUBJID",
               .fn         = function(x) paste(x$STUDYID, x$SITEID, x, sep = "-")) %>%
  
  # Demographics — direct copy
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat = raw_dm, raw_var = "AGE",    tgt_var = "AGE") %>%
  
  assign_no_ct(oak_id_vars = oak_id_vars(),
               raw_dat = raw_dm, raw_var = "AGEU",   tgt_var = "AGEU") %>%
  
  # Sex — CT mapping
  assign_ct(oak_id_vars = oak_id_vars(),
            raw_dat = raw_dm, raw_var = "GENDER", tgt_var = "SEX",
            ct_spec = ct, ct_clst = "C66731") %>%
  
  # Race — CT mapping
  assign_ct(oak_id_vars = oak_id_vars(),
            raw_dat = raw_dm, raw_var = "RACE",   tgt_var = "RACE",
            ct_spec = ct, ct_clst = "C74457") %>%
  
  # Dates — ISO8601 conversion
  assign_datetime(oak_id_vars = oak_id_vars(),
                  raw_dat = raw_dm, raw_var = "RFICDTC",
                  tgt_var = "RFICDTC", raw_fmt = "d-m-y") %>%
  
  # Select SDTM variables
  select(STUDYID, DOMAIN, USUBJID, SUBJID, SITEID,
         RFICDTC, RFSTDTC, RFENDTC, RFXSTDTC, RFXENDTC,
         DTHDTC, DTHFL, AGE, AGEU, SEX, RACE, ETHNIC,
         ARMCD, ARM, ACTARMCD, ACTARM)

cat(sprintf("Mapped rows: %d\n", nrow(dm)))

# Verify against reference SDTM
ref_dm <- pharmaversesdtm::dm
cat(sprintf("\nReference SDTM rows: %d\n", nrow(ref_dm)))

# CT check
cat("\nSEX mapping check:\n")
print(table(raw_dm$GENDER, dm$SEX, useNA = "always"))
```

---

## Key sdtm.oak Concepts

### oak_id_vars()

Every function takes `oak_id_vars()` — the variables that uniquely identify a raw record. Defaults to `STUDYID` + `USUBJID` + the raw record sequence variable. Change it if your raw data has a different structure.

### Failed CT Mappings Produce NA

If a raw value has no match in the codelist, `assign_ct()` produces `NA`. This is intentional — it surfaces unmapped values as missing rather than wrong.

```r
# Find values that didn't map
dm %>% filter(is.na(SEX)) %>% select(USUBJID) %>%
  left_join(raw_dm %>% select(USUBJID, GENDER), by = "USUBJID")
# Shows: USUBJID + the raw GENDER value that failed
```

In SAS, an unmapped `PUT()` format returns blank — which looks like a valid missing value and hides the problem.

### NCI CT Codelist C-codes

The most common ones:

| Codelist | C-code | Contents |
|----------|--------|---------|
| Sex | C66731 | M, F, U, UNDIFFERENTIATED |
| No/Yes | C66742 | N, Y |
| Severity | C41338 | MILD, MODERATE, SEVERE |
| Race | C74457 | WHITE, BLACK OR AFRICAN AMERICAN, etc. |
| Route | C66729 | ORAL, INTRAVENOUS, etc. |

---

## Mapping Report: Raw vs. SDTM

```r
# Compare your mapping against reference
cat("\n=== Mapping Comparison vs Reference ===\n")

# Variables added by mapping (not in raw, present in SDTM)
added <- setdiff(names(ref_dm), names(raw_dm))
cat("Added by mapping:", paste(added, collapse = ", "), "\n")

# Subject count
cat(sprintf("Raw subjects:  %d\n", n_distinct(raw_dm$USUBJID)))
cat(sprintf("Mapped:        %d\n", n_distinct(dm$USUBJID)))
cat(sprintf("Reference:     %d\n", n_distinct(ref_dm$USUBJID)))

# Date format
cat("\nDate format check (RFICDTC, first 3):\n")
cat("  Raw:  ", paste(head(raw_dm$RFICDTC, 3), collapse=", "), "\n")
cat("  Mapped:", paste(head(dm$RFICDTC, 3), collapse=", "), "\n")
```

---

## What We Have Now

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM domains
pharm001_ch03_dm.R       — map raw DM to SDTM using sdtm.oak
```

---

## Exercises

**1.** The raw AE data is in `pharmaverseraw::ae_raw`. Map `AETERM`, `AESER` (use CT C66742), and `AESTDTC`. What percentage of `AESER` values fail CT mapping?

**2.** Add `AESEQ` as a sequence number within each `USUBJID`, ordered by `AESTDTC`.

**3.** Write `impute_date(dtc, rule)` where `rule = "first"` fills a partial date `"2014-03"` as `"2014-03-01"`, and `rule = "last"` fills it as `"2014-03-31"`.

**4. (Sets up Chapter 4)** In `pharmaversesdtm::dm`, find subjects where `RFXSTDTC` is not `NA`. Convert `RFXSTDTC` to a Date. Now find subjects where `RFXSTDTC` is missing — what's their `ACTARM`? This tells you why the derivation of `TRTSDT` in the next chapter needs special handling.

---

*Next: Chapter 4 — we use admiral to derive TRTSDT and TRTEDT from the SDTM we now understand.*

---

## Solutions

### Exercise 1

Map `AETERM`, `AESER` (CT C66742), and `AESTDTC` from raw AE data; calculate the percentage of `AESER` CT failures.

```r
# SAS:
# DATA ae;
#   SET raw_ae;
#   DOMAIN = "AE";
#   AETERM = AETERM_RAW;    /* direct copy */
#   IF      UPCASE(AESER) = "YES" THEN AESER = "Y";
#   ELSE IF UPCASE(AESER) = "NO"  THEN AESER = "N";
#   ELSE                               AESER = "";   /* unmapped */
#   AESTDTC = PUT(INPUT(AESTDTC_RAW, DATE9.), YYMMDD10.);
# RUN;

library(dplyr)
library(pharmaverseraw)

# pharmaverseraw::ae_raw contains the raw AE data
ae_raw <- pharmaverseraw::ae_raw

cat(sprintf("Raw AE rows: %d\n", nrow(ae_raw)))
cat("Raw AESER values:\n")
print(table(ae_raw$AESER, useNA = "always"))

# Note: sdtm.oak API may change — this pattern uses dplyr as fallback
# For sdtm.oak, use: assign_ct(raw_dat=ae_raw, raw_var="AESER", ct_clst="C66742")
# Here we show the dplyr approach as a safe alternative
ae_mapped <- ae_raw %>%
  mutate(
    DOMAIN  = "AE",
    AETERM  = AETERM,          # direct copy (no CT)
    AESER   = case_when(
      toupper(AESER) %in% c("YES","Y") ~ "Y",
      toupper(AESER) %in% c("NO", "N") ~ "N",
      TRUE                              ~ NA_character_   # unmapped
    ),
    AESTDTC = AESTDTC          # keep ISO8601 as-is if already formatted
  )

# Percentage that failed CT mapping
n_total   <- nrow(ae_mapped)
n_fail    <- sum(is.na(ae_mapped$AESER))
cat(sprintf("\nAESER CT failures: %d / %d (%.1f%%)\n",
            n_fail, n_total, 100 * n_fail / n_total))
```

### Exercise 2

Add `AESEQ` as a sequence number within each `USUBJID`, ordered by `AESTDTC`.

```r
# SAS:
# /* PROC SORT + RETAIN pattern */
# PROC SORT DATA=ae OUT=ae_sorted; BY USUBJID AESTDTC; RUN;
# DATA ae;
#   SET ae_sorted;
#   BY USUBJID;
#   RETAIN AESEQ 0;
#   IF FIRST.USUBJID THEN AESEQ = 0;
#   AESEQ + 1;
# RUN;

library(dplyr)
library(pharmaversesdtm)

ae <- pharmaversesdtm::ae

ae_with_seq <- ae %>%
  arrange(USUBJID, AESTDTC) %>%        # sort before numbering
  group_by(USUBJID) %>%
  mutate(AESEQ = row_number()) %>%      # equivalent to RETAIN AESEQ + 1
  ungroup()

# Verify: max sequence per subject
cat("AESEQ range:\n")
ae_with_seq %>%
  group_by(USUBJID) %>%
  summarise(max_seq = max(AESEQ)) %>%
  summarise(min_maxseq = min(max_seq), max_maxseq = max(max_seq)) %>%
  print()

# First 3 AEs for one subject
ae_with_seq %>%
  filter(USUBJID == first(USUBJID)) %>%
  select(USUBJID, AESEQ, AESTDTC, AETERM) %>%
  head(3) %>%
  print()
```

### Exercise 3

Write `impute_date(dtc, rule)` that imputes partial dates using first or last day of month.

```r
# SAS equivalent:
# /* Impute first */
# IF LENGTH(TRIM(DTC)) = 7 THEN DT_IMP = INPUT(CATS(DTC,"-01"), YYMMDD10.);
# /* Impute last */
# IF LENGTH(TRIM(DTC)) = 7 THEN DO;
#   mo  = INPUT(SUBSTR(DTC,6,2), 2.);
#   yr  = INPUT(SUBSTR(DTC,1,4), 4.);
#   DT_IMP = INTNX("MONTH", MDY(mo,1,yr), 0, "END");
# END;

library(dplyr)
library(lubridate)

impute_date <- function(dtc, rule = "first") {
  # dtc: ISO8601 character vector (complete "YYYY-MM-DD" or partial "YYYY-MM")
  # rule: "first" → first of month, "last" → last of month
  stopifnot(rule %in% c("first", "last"))

  result <- character(length(dtc))

  for (i in seq_along(dtc)) {
    d <- dtc[i]
    if (is.na(d) || d == "") {
      result[i] <- NA_character_
    } else if (grepl("^\\d{4}-\\d{2}-\\d{2}$", d)) {
      result[i] <- d           # complete date — no imputation needed
    } else if (grepl("^\\d{4}-\\d{2}$", d)) {
      # Partial: YYYY-MM
      yr <- as.integer(substr(d, 1, 4))
      mo <- as.integer(substr(d, 6, 7))
      if (rule == "first") {
        result[i] <- sprintf("%04d-%02d-01", yr, mo)
      } else {
        last_day <- lubridate::days_in_month(as.Date(sprintf("%04d-%02d-01", yr, mo)))
        result[i] <- sprintf("%04d-%02d-%02d", yr, mo, last_day)
      }
    } else {
      result[i] <- NA_character_   # unrecognised format
    }
  }
  as.Date(result)
}

# Test
test_dtcs <- c("2014-03", "2014-12", "2014-03-15", NA, "2014-02")
cat("First imputation:\n")
print(impute_date(test_dtcs, "first"))
cat("Last imputation:\n")
print(impute_date(test_dtcs, "last"))
# Expected:
#  "2014-03-01" "2014-12-01" "2014-03-15"  NA  "2014-02-01"
#  "2014-03-31" "2014-12-31" "2014-03-15"  NA  "2014-02-28"
```

### Exercise 4

Find subjects with `RFXSTDTC` present vs. missing, and check their treatment arm.

```r
# SAS:
# DATA dm2;
#   SET dm;
#   RFXSTDT = INPUT(RFXSTDTC, YYMMDD10.);
#   MISSING_TRT = (MISSING(RFXSTDTC));
#   FORMAT RFXSTDT DATE9.;
# RUN;
# PROC FREQ DATA=dm2; TABLES MISSING_TRT * ACTARM; RUN;

library(dplyr)
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm

# Convert RFXSTDTC to Date
dm_dates <- dm %>%
  mutate(
    RFXSTDT = as.Date(RFXSTDTC),
    HAS_RFXSTDT = !is.na(RFXSTDT)
  )

cat("RFXSTDTC presence by arm:\n")
dm_dates %>%
  count(ACTARM, HAS_RFXSTDT) %>%
  tidyr::pivot_wider(names_from = HAS_RFXSTDT, values_from = n,
                     names_prefix = "has_rfxstdt_") %>%
  print()

# Subjects missing RFXSTDTC — what arm?
missing_trt <- dm_dates %>%
  filter(!HAS_RFXSTDT) %>%
  select(USUBJID, ACTARM, RFXSTDTC)

cat(sprintf("\nSubjects missing RFXSTDTC: %d\n", nrow(missing_trt)))
cat("Their arms:\n")
print(table(missing_trt$ACTARM))
# Explanation: screen failures have ACTARM = "Screen Failure" and no exposure date
# → TRTSDT derivation in Ch4 must handle these gracefully (result: NA TRTSDT)
```
