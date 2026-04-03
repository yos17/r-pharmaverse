# Chapter 6: Build ADAE

*The SAP requires a TEAE table. To produce it, we need ADAE. Let's build it.*

---

## Where We Left Off

```
pharm001_ch05_adsl.R: complete ADSL with TRTSDT, TRTEDT, flags
```

Chapter 5's exercise asked you to flag treatment-emergent AEs. That was the preview. Now we build the full ADAE dataset — one row per AE occurrence, with study days, the TEAE flag, and worst-severity flags.

---

## What ADAE Needs

The SAP's Table 14.3.1 (TEAE overview) and Table 14.3.2 (AE by SOC/PT) require:
- `TRTEMFL` — treatment-emergent flag
- `ASTDT` / `ASTDY` — AE start date and study day
- `ASEVN` — severity numeric (for worst-event analysis)
- `AOCCIFL` / `AOCCPIFL` — worst-severity flags per subject and per subject+PT

ADAE extends the SDTM AE domain with all of these.

---

## Step 1: Merge ADSL Variables

**In SAS:**
```sas
/* Sort both datasets first */
PROC SORT DATA=ae;    BY STUDYID USUBJID; RUN;
PROC SORT DATA=adsl;  BY STUDYID USUBJID; RUN;

DATA adae;
  MERGE ae(IN=inAE)
        adsl(IN=inADSL
             KEEP=STUDYID USUBJID TRTSDT TRTEDT
                  TRT01A TRT01AN SAFFL AGE SEX RACE);
  BY STUDYID USUBJID;
  IF inAE;   /* one row per AE, gets ADSL variables */
RUN;
```

**In R (pharmaverse):**
```r
library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(dplyr)

ae   <- pharmaversesdtm::ae
adsl <- pharmaverseadam::adsl   # or your Chapter 5 derivation

adae <- ae %>%
  left_join(
    adsl %>% select(STUDYID, USUBJID, TRTSDT, TRTEDT,
                    TRT01A, TRT01AN, SAFFL, AGE, SEX, RACE),
    by = c("STUDYID", "USUBJID")
  )
```

---

## Step 2: Analysis Dates

**In SAS:**
```sas
DATA adae;
  SET adae;
  /* Start date */
  IF LENGTH(TRIM(AESTDTC)) = 10 THEN
    ASTDT = INPUT(AESTDTC, YYMMDD10.);
  ELSE IF LENGTH(TRIM(AESTDTC)) = 7 THEN
    ASTDT = INPUT(CATS(AESTDTC, "-01"), YYMMDD10.);  /* impute to first */
  FORMAT ASTDT DATE9.;

  /* Study day — CDISC convention: no day 0 */
  IF ASTDT >= TRTSDT THEN ASTDY =  ASTDT - TRTSDT + 1;
  ELSE IF NOT MISSING(ASTDT) AND NOT MISSING(TRTSDT)
                     THEN ASTDY =  ASTDT - TRTSDT;
RUN;
```

**In R (pharmaverse — admiral):**

Convert ISO8601 `AESTDTC` / `AEENDTC` to Date, then compute study days:

```r
adae <- adae %>%
  derive_vars_dt(
    new_vars_prefix = "AST",
    dtc             = AESTDTC,
    date_imputation = "first"      # partial start dates → impute early
  ) %>%
  derive_vars_dt(
    new_vars_prefix = "AEN",
    dtc             = AEENDTC,
    date_imputation = "last"       # partial end dates → impute late
  ) %>%
  derive_vars_dy(
    reference_date = TRTSDT,
    source_vars    = exprs(ASTDT, AENDT)
  )
# Creates: ASTDT, AENDT, ASTDY, AENDY
```

---

## Step 3: Treatment-Emergent Flag

**In SAS (naive approach):**
```sas
DATA adae;
  SET adae;
  /* Naive: starts on or after treatment start */
  IF NOT MISSING(ASTDT) AND NOT MISSING(TRTSDT) AND
     ASTDT >= TRTSDT THEN TRTEMFL = "Y";
  /* This misses: AEs starting >30 days after last dose */
RUN;
```

**In SAS (SAP-compliant, 30-day window):**
```sas
DATA adae;
  SET adae;
  IF NOT MISSING(ASTDT) AND NOT MISSING(TRTSDT) AND
     ASTDT >= TRTSDT AND
     (MISSING(TRTEDT) OR ASTDT <= TRTEDT + 30)
  THEN TRTEMFL = "Y";
RUN;
```

An AE is treatment-emergent if it starts on or after the first dose date:

```r
# Naive approach — misses a subtlety:
adae <- adae %>%
  mutate(
    TRTEMFL = if_else(
      !is.na(ASTDT) & !is.na(TRTSDT) & ASTDT >= TRTSDT,
      "Y", NA_character_
    )
  )
```

The subtlety: what about AEs that started after treatment ended? The protocol usually allows a 30-day window. Let's show why the naive approach is incomplete:

```r
# An AE starting 45 days after last dose — should it be TEAE?
# The protocol says: within 30 days of last dose counts.
# Naive: ASTDT >= TRTSDT → TRUE. That's wrong for AEs well after treatment.
```

The correct definition per SAP: started on or after first dose AND within 30 days of last dose:

```r
adae <- adae %>%
  mutate(
    TRTEMFL = case_when(
      !is.na(ASTDT) & !is.na(TRTSDT) &
        ASTDT >= TRTSDT &
        (is.na(TRTEDT) | ASTDT <= TRTEDT + 30) ~ "Y",
      TRUE ~ NA_character_
    )
  )
```

Admiral provides `derive_var_trtemfl()` which implements this with parameter control:

**In R (pharmaverse — admiral):**
```r
adae <- adae %>%
  derive_var_trtemfl(
    new_var        = TRTEMFL,
    trt_start_date = TRTSDT,
    trt_end_date   = TRTEDT,
    end_window     = 30
  )
```

---

## Step 4: Worst Severity Flag

**In SAS:**
```sas
/* Numeric severity for sorting */
DATA adae;
  SET adae;
  IF      AESEV = "MILD"     THEN ASEVN = 1;
  ELSE IF AESEV = "MODERATE" THEN ASEVN = 2;
  ELSE IF AESEV = "SEVERE"   THEN ASEVN = 3;
RUN;

/* Worst per subject — PROC SORT + FIRST/LAST pattern */
PROC SORT DATA=adae OUT=adae_sev;
  BY USUBJID DESCENDING ASEVN ASTDT;
RUN;

DATA adae;
  SET adae_sev;
  BY USUBJID;
  IF FIRST.USUBJID THEN AOCCIFL = "Y";  /* worst = first after sort */
  ELSE AOCCIFL = "";
RUN;

/* Worst per subject per PT */
PROC SORT DATA=adae OUT=adae_pt;
  BY USUBJID AEDECOD DESCENDING ASEVN ASTDT;
RUN;

DATA adae;
  SET adae_pt;
  BY USUBJID AEDECOD;
  IF FIRST.AEDECOD THEN AOCCPIFL = "Y";
  ELSE AOCCPIFL = "";
RUN;
```

**In R (pharmaverse — admiral):**
```r
adae <- adae %>%
  mutate(
    ASEVN = case_when(
      AESEV == "MILD"     ~ 1L,
      AESEV == "MODERATE" ~ 2L,
      AESEV == "SEVERE"   ~ 3L,
      TRUE                ~ NA_integer_
    )
  ) %>%
  # Worst overall per subject
  derive_var_extreme_flag(
    by_vars = exprs(USUBJID),
    order   = exprs(desc(ASEVN), ASTDT),
    new_var = AOCCIFL,
    mode    = "first"
  ) %>%
  # Worst per subject per preferred term
  derive_var_extreme_flag(
    by_vars = exprs(USUBJID, AEDECOD),
    order   = exprs(desc(ASEVN), ASTDT),
    new_var = AOCCPIFL,
    mode    = "first"
  )
```

> **The PROC SORT + FIRST./LAST. pattern** is one of the most common SAS idioms. Admiral's `derive_var_extreme_flag()` replaces this entire pattern: no sorting needed first, no BY-group logic, no risk of forgetting to restore sort order.

---

## The Complete ADAE Program

```r
# pharm001_ch06_adae.R
# Study PHARM-001 — Chapter 6
# Build ADAE for TEAE tables

library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(dplyr)

ae   <- pharmaversesdtm::ae
adsl <- pharmaverseadam::adsl

# ---- Merge ADSL ----
adae <- ae %>%
  left_join(
    adsl %>% select(STUDYID, USUBJID, TRTSDT, TRTEDT,
                    TRT01A, TRT01AN, SAFFL, AGE, SEX, RACE),
    by = c("STUDYID", "USUBJID")
  )

# ---- Dates ----
adae <- adae %>%
  derive_vars_dt(new_vars_prefix = "AST", dtc = AESTDTC, date_imputation = "first") %>%
  derive_vars_dt(new_vars_prefix = "AEN", dtc = AEENDTC,  date_imputation = "last") %>%
  derive_vars_dy(reference_date  = TRTSDT, source_vars = exprs(ASTDT, AENDT))

# ---- TEAE flag ----
adae <- adae %>%
  derive_var_trtemfl(
    new_var        = TRTEMFL,
    trt_start_date = TRTSDT,
    trt_end_date   = TRTEDT,
    end_window     = 30
  )

# ---- Severity numeric + worst flags ----
adae <- adae %>%
  mutate(
    ASEVN = case_when(
      AESEV == "MILD"     ~ 1L,
      AESEV == "MODERATE" ~ 2L,
      AESEV == "SEVERE"   ~ 3L,
      TRUE                ~ NA_integer_
    )
  ) %>%
  derive_var_extreme_flag(
    by_vars = exprs(USUBJID),
    order   = exprs(desc(ASEVN), ASTDT),
    new_var = AOCCIFL,
    mode    = "first"
  ) %>%
  derive_var_extreme_flag(
    by_vars = exprs(USUBJID, AEDECOD),
    order   = exprs(desc(ASEVN), ASTDT),
    new_var = AOCCPIFL,
    mode    = "first"
  )

# ---- Sorting ----
adae <- adae %>%
  arrange(USUBJID, ASTDT)
# In SAS: PROC SORT DATA=adae BY USUBJID AESTDTC; RUN;

# ---- QC summary ----
n_safety <- sum(adsl$SAFFL == "Y", na.rm = TRUE)
teae     <- adae %>% filter(SAFFL == "Y", TRTEMFL == "Y")

cat(sprintf("Safety population: %d\n", n_safety))
cat(sprintf("TEAEs: %d events, %d subjects (%.1f%%)\n",
            nrow(teae),
            n_distinct(teae$USUBJID),
            100 * n_distinct(teae$USUBJID) / n_safety))
cat(sprintf("Serious TEAEs: %d events, %d subjects\n",
            sum(teae$AESER == "Y", na.rm = TRUE),
            teae %>% filter(AESER == "Y") %>% distinct(USUBJID) %>% nrow()))

# Top 5 PTs
cat("\nTop 5 preferred terms:\n")
teae %>%
  count(AEDECOD, sort = TRUE) %>%
  head(5) %>%
  print()
```

---

## ADTTE: Time-to-Event

For the KM plot in Chapter 7, we also need ADTTE. Structure: one row per subject per event type.

```r
# Time to First Adverse Event
first_ae <- adae %>%
  filter(TRTEMFL == "Y") %>%
  group_by(STUDYID, USUBJID) %>%
  summarise(AEDTFIRST = min(ASTDT, na.rm = TRUE), .groups = "drop")

adtte_ttae <- adsl %>%
  filter(SAFFL == "Y") %>%
  select(STUDYID, USUBJID, TRTSDT, TRTEDT, TRT01A, TRT01AN) %>%
  left_join(first_ae, by = c("STUDYID", "USUBJID")) %>%
  mutate(
    PARAMCD  = "TTAE",
    PARAM    = "Time to First Adverse Event",
    AVALU    = "DAYS",
    STARTDT  = TRTSDT,
    ADT      = if_else(!is.na(AEDTFIRST), AEDTFIRST, TRTEDT),
    CNSR     = if_else(!is.na(AEDTFIRST), 0L, 1L),
    AVAL     = as.numeric(ADT - TRTSDT),
    EVNTDESC = if_else(CNSR == 0L, "Adverse Event Occurred", ""),
    CNSDTDSC = if_else(CNSR == 1L, "End of Treatment", "")
  )
```

---

## Sequence Numbers

**In SAS:**
```sas
/* Sequence number per subject — the RETAIN + sum statement pattern */
DATA adae;
  SET adae;
  BY USUBJID;
  RETAIN AESEQ 0;
  IF FIRST.USUBJID THEN AESEQ = 0;
  AESEQ + 1;
RUN;
```

**In R (pharmaverse):**
```r
adae <- adae %>%
  arrange(USUBJID, ASTDT) %>%
  group_by(USUBJID) %>%
  mutate(AESEQ = row_number()) %>%
  ungroup()
```

---

## What We Have Now

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM
pharm001_ch03_dm.R       — map raw DM to SDTM
pharm001_ch04.R          — derive TRTSDT, TRTEDT, TRTDURD
pharm001_ch05_adsl.R     — complete ADSL
pharm001_ch06_adae.R     — complete ADAE (TEAE flag, severity flags, study days)
```

---

## Exercises

**1.** Add NCI CTCAE grade to ADAE: `ATOXGR` (numeric 1–5), `ATOXGRN`. Flag subjects with Grade 3+ events: `GR3FL = "Y"`.

**2.** The naive TEAE definition `ASTDT >= TRTSDT` differs from the 30-day window definition. How many AEs are flagged by the naive approach but not by the correct one? Show the subjects and their AE dates.

**3.** Build ADTTE for overall survival: `PARAMCD = "OS"`. Event = death (`DTHFL == "Y"` in ADSL), censoring = last known alive date (use `TRTEDT` as proxy).

**4. (Sets up Chapter 7)** The demographics table (Chapter 7) will be built from ADSL. The AE table (Chapter 8) will be built from ADAE. But both need the same column structure for treatment arms. Write a quick check: do `unique(adsl$TRT01A)` and `unique(adae$TRT01A)` match? If not, which arm is missing from ADAE?

---

*Next: Chapter 7 — we build the demographics table. Start with one row (age N/mean/SD), add rows one by one.*

---

## Solutions

### Exercise 1

Add NCI CTCAE grade as `ATOXGR`, numeric `ATOXGRN`, and flag subjects with Grade 3+ events.

```r
# SAS:
# DATA adae;
#   SET adae;
#   /* CTCAE grade from AETOXGR (numeric 1-5 in pharmaversesdtm) */
#   ATOXGR  = AETOXGR;
#   ATOXGRN = INPUT(AETOXGR, 8.);
#   IF ATOXGRN >= 3 THEN GR3FL = "Y";
#   ELSE                 GR3FL = "";
# RUN;

library(dplyr)
library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)

ae   <- pharmaversesdtm::ae
adsl <- pharmaverseadam::adsl

adae <- ae %>%
  left_join(
    adsl %>% select(STUDYID, USUBJID, TRTSDT, TRTEDT, TRT01A, SAFFL),
    by = c("STUDYID", "USUBJID")
  ) %>%
  derive_vars_dt(new_vars_prefix = "AST", dtc = AESTDTC, date_imputation = "first") %>%
  derive_var_trtemfl(
    new_var        = TRTEMFL,
    trt_start_date = TRTSDT,
    trt_end_date   = TRTEDT,
    end_window     = 30
  ) %>%
  mutate(
    # CTCAE grade — AETOXGR is stored as character "1"-"5" in pharmaversesdtm
    ATOXGR  = AETOXGR,
    ATOXGRN = suppressWarnings(as.integer(AETOXGR)),
    GR3FL   = if_else(!is.na(ATOXGRN) & ATOXGRN >= 3, "Y", NA_character_)
  )

# Summary
n_gr3_subj <- adae %>%
  filter(SAFFL == "Y", TRTEMFL == "Y", GR3FL == "Y") %>%
  distinct(USUBJID) %>%
  nrow()

cat("Grade 3+ TEAE summary:\n")
cat(sprintf("  Subjects with Grade 3+ TEAE: %d\n", n_gr3_subj))

adae %>%
  filter(SAFFL == "Y", TRTEMFL == "Y", GR3FL == "Y") %>%
  count(ATOXGR, sort = TRUE) %>%
  print()
```

### Exercise 2

Compare naive TEAE definition vs. 30-day window — show which AEs differ.

```r
# SAS:
# /* Naive */
# IF ASTDT >= TRTSDT THEN TRTEMFL_NAIVE = "Y";
# /* Correct with 30-day window */
# IF ASTDT >= TRTSDT AND (MISSING(TRTEDT) OR ASTDT <= TRTEDT + 30)
#    THEN TRTEMFL_CORRECT = "Y";

library(dplyr)
library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)

ae   <- pharmaversesdtm::ae
adsl <- pharmaverseadam::adsl

adae_compare <- ae %>%
  left_join(
    adsl %>% select(USUBJID, TRTSDT, TRTEDT, SAFFL),
    by = "USUBJID"
  ) %>%
  filter(SAFFL == "Y") %>%
  mutate(
    ASTDT = as.Date(substr(AESTDTC, 1, 10)),

    # Naive: on or after first dose (no end window)
    TRTEMFL_NAIVE = case_when(
      !is.na(ASTDT) & !is.na(TRTSDT) & ASTDT >= TRTSDT ~ "Y",
      TRUE ~ NA_character_
    ),

    # Correct: on or after first dose AND within 30 days of last dose
    TRTEMFL_CORRECT = case_when(
      !is.na(ASTDT) & !is.na(TRTSDT) &
        ASTDT >= TRTSDT &
        (is.na(TRTEDT) | ASTDT <= TRTEDT + 30) ~ "Y",
      TRUE ~ NA_character_
    )
  )

# Compare
n_naive   <- sum(adae_compare$TRTEMFL_NAIVE    == "Y", na.rm = TRUE)
n_correct <- sum(adae_compare$TRTEMFL_CORRECT  == "Y", na.rm = TRUE)
n_diff    <- n_naive - n_correct

cat(sprintf("Naive TEAE:   %d events\n", n_naive))
cat(sprintf("Correct TEAE: %d events\n", n_correct))
cat(sprintf("Difference:   %d events flagged by naive only\n", n_diff))

# Show the extra events (flagged naive but not correct)
extra <- adae_compare %>%
  filter(TRTEMFL_NAIVE == "Y", is.na(TRTEMFL_CORRECT)) %>%
  select(USUBJID, AEDECOD, AESTDTC, AEENDTC, ASTDT, TRTSDT, TRTEDT) %>%
  mutate(DAYS_AFTER_TRTEDT = as.integer(ASTDT - TRTEDT))

if (nrow(extra) > 0) {
  cat(sprintf("\nAEs in naive but NOT in correct definition (%d events):\n", nrow(extra)))
  print(head(extra, 10))
}
```

### Exercise 3

Build ADTTE for overall survival (`PARAMCD = "OS"`).

```r
# SAS:
# DATA adtte_os;
#   SET adsl;
#   PARAMCD = "OS";
#   PARAM   = "Overall Survival";
#   AVALU   = "DAYS";
#   /* Event: death */
#   IF DTHFL = "Y" THEN DO;
#     CNSR = 0; ADT = DTHDT; EVNTDESC = "Death";
#   END;
#   ELSE DO;
#     CNSR = 1; ADT = TRTEDT; CNSDTDSC = "End of Treatment";
#   END;
#   AVAL = ADT - TRTSDT;
#   FORMAT ADT DTHDT DATE9.;
# RUN;

library(dplyr)
library(admiral)
library(pharmaverseadam)

adsl <- pharmaverseadam::adsl

adtte_os <- adsl %>%
  filter(SAFFL == "Y") %>%
  mutate(
    # Death date
    DTHDT = as.Date(DTHDTC)
  ) %>%
  mutate(
    PARAMCD  = "OS",
    PARAM    = "Overall Survival",
    AVALU    = "DAYS",
    STARTDT  = TRTSDT,

    # Event indicator: 0 = event (death), 1 = censored
    CNSR     = if_else(!is.na(DTHFL) & DTHFL == "Y", 0L, 1L),

    # Analysis date: death date if event, last contact if censored
    ADT      = if_else(CNSR == 0L, DTHDT, TRTEDT),

    # Analysis value in days from treatment start
    AVAL     = as.numeric(ADT - TRTSDT),

    EVNTDESC = if_else(CNSR == 0L, "Death", ""),
    CNSDTDSC = if_else(CNSR == 1L, "End of Treatment", "")
  ) %>%
  select(STUDYID, USUBJID, TRT01A, TRT01AN, PARAMCD, PARAM, AVALU,
         STARTDT, ADT, AVAL, CNSR, EVNTDESC, CNSDTDSC)

cat("OS ADTTE summary:\n")
cat(sprintf("  Rows: %d\n", nrow(adtte_os)))
cat(sprintf("  Deaths (CNSR=0): %d\n", sum(adtte_os$CNSR == 0)))
cat(sprintf("  Censored:        %d\n", sum(adtte_os$CNSR == 1)))
cat(sprintf("  AVAL range: %.0f – %.0f days\n",
            min(adtte_os$AVAL, na.rm=TRUE), max(adtte_os$AVAL, na.rm=TRUE)))
```

### Exercise 4

Check that `TRT01A` arms match between ADSL and ADAE.

```r
# SAS:
# PROC SQL; SELECT DISTINCT TRT01A FROM adsl; QUIT;
# PROC SQL; SELECT DISTINCT TRT01A FROM adae; QUIT;

library(dplyr)
library(pharmaverseadam)

adsl <- pharmaverseadam::adsl
adae <- pharmaverseadam::adae

arms_adsl <- sort(unique(adsl$TRT01A))
arms_adae <- sort(unique(adae$TRT01A))

cat("ADSL arms:\n"); print(arms_adsl)
cat("ADAE arms:\n"); print(arms_adae)

missing_from_adae <- setdiff(arms_adsl, arms_adae)
extra_in_adae     <- setdiff(arms_adae, arms_adsl)

if (length(missing_from_adae) == 0 && length(extra_in_adae) == 0) {
  cat("\n✓ Arms match exactly between ADSL and ADAE\n")
} else {
  if (length(missing_from_adae) > 0)
    cat("Arms in ADSL missing from ADAE:", paste(missing_from_adae, collapse=", "), "\n")
  if (length(extra_in_adae) > 0)
    cat("Extra arms in ADAE:", paste(extra_in_adae, collapse=", "), "\n")
}
# Note: an arm present in ADSL but not ADAE means no AEs were reported
# for subjects in that arm — not a data error, but worth investigating
```
