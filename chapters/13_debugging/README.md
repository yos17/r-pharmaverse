# Chapter 13: Find the 3 Bugs

*Chapter 12 introduced 3 bugs into the PHARM-001 ADSL. Use `browser()` and `diffdf` to find them.*

---

## Where We Left Off

```
pharm001_ch11_adsl.R: ADSL with 3 deliberate bugs introduced
  Bug 1: TRTSDT wrong for subject "01-701-1015" (month typo in RFXENDTC)
  Bug 2: TRTSDT wrong for subject "01-701-1023" (one day off — imputation mismatch)
  Bug 3: AGEGR1 wrong for one subject (boundary condition off-by-one)
```

You know there are 3 bugs. You don't know exactly where the code is wrong. Let's find them.

---

## Debugging Mindset: SAS vs. R

In SAS, debugging usually means:
1. Looking at the `.log` for `ERROR:` and `WARNING:` messages
2. Adding `%PUT` statements to print variable values
3. Using `PROC PRINT` with `WHERE` to inspect specific observations
4. Running `PROC COMPARE` to find differences between datasets

In R, the toolkit is different but the strategy is identical: isolate where the data goes wrong, inspect values at that point.

| SAS | R |
|-----|---|
| Check `.log` for ERRORs | Check console errors / warnings |
| `%PUT &myvar;` | `cat(myvar, "\n")` / `print(myvar)` |
| `PROC PRINT DATA=adsl(WHERE=(USUBJID="01-701-1015") OBS=1);` | `adsl %>% filter(USUBJID=="01-701-1015") %>% print()` |
| `PROC COMPARE DATA=mine COMPARE=ref;` | `diffdf(base=mine, compare=ref, keys="USUBJID")` |
| `%PUT _ALL_;` in macro | `ls()` inside `browser()` |
| SAS debugger (rarely used) | `browser()` — powerful, interactive |

---

## Setup: The Buggy ADSL

```r
library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(dplyr)

dm <- pharmaversesdtm::dm

# ---- Introduce the 3 bugs deliberately ----

# Bug 1: Month typo in RFXENDTC (2014-07 → 2014-01)
dm_buggy <- dm %>%
  mutate(
    RFXENDTC = if_else(USUBJID == "01-701-1015",
                       "2014-01-02",   # was "2014-07-02"
                       RFXENDTC)
  )

# Bug 2: Wrong imputation rule — using "first" for end date (should be "last")
# Bug 3: Wrong age boundary — using < 65 instead of <= 65 for "40-64" group
```

Now derive ADSL from the buggy DM:

```r
adsl_buggy <- dm_buggy %>%
  filter(ACTARM != "Screen Failure") %>%
  mutate(TRT01P = ACTARM, TRT01A = ACTARM,
         TRT01PN = as.integer(factor(ACTARM, levels=sort(unique(ACTARM)))),
         TRT01AN = TRT01PN) %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC, date_imputation = "first") %>%
  # Bug 2: end date using "first" instead of "last"
  derive_vars_dt(new_vars_prefix = "TRTE", dtc = RFXENDTC, date_imputation = "first") %>%
  mutate(
    TRTDURD = as.integer(TRTEDT - TRTSDT + 1),
    # Bug 3: wrong boundary — >= 65 should catch subjects aged exactly 65
    AGEGR1  = case_when(
      AGE < 40              ~ "<40",
      AGE >= 40 & AGE < 65  ~ "40-64",   # subject aged exactly 65 goes here — wrong
      AGE > 65              ~ ">=65",     # BUG: should be >= 65
      TRUE                  ~ NA_character_
    )
  )
```

---

## Step 1: diffdf / PROC COMPARE to Locate the Bugs

**In SAS:**
```sas
/* Sort both datasets first */
PROC SORT DATA=adsl_buggy;  BY USUBJID; RUN;
PROC SORT DATA=adsl_ref;    BY USUBJID; RUN;

PROC COMPARE DATA=adsl_buggy COMPARE=adsl_ref LISTALL LISTVAR;
  ID USUBJID;
  /* Output shows:
     Variable    Number of Differences
     TRTEDT      2
     TRTDURD     2
     AGEGR1      1  */
RUN;
```

**In R (pharmaverse — diffdf):**
```r
library(diffdf)

adsl_ref <- pharmaverseadam::adsl

diff_result <- diffdf(
  base    = adsl_buggy,
  compare = adsl_ref,
  keys    = "USUBJID"
)

print(diff_result)
```

Output:
```
Differences found between the two DataFrames!

Not all Values Compared Equal:
  Variable  No of Differences
  TRTEDT             2
  TRTDURD            2
  AGEGR1             1
```

`diffdf` found 3 variable discrepancies. Now we know which variables are wrong. Get the subject IDs:

```r
# Which subjects have wrong TRTEDT?
diff_result$VarDiff_TRTEDT %>%
  select(USUBJID, BASE, COMPARE) %>%
  print()

# Which subject has wrong AGEGR1?
diff_result$VarDiff_AGEGR1 %>%
  select(USUBJID, BASE, COMPARE) %>%
  print()
```

---

## Step 2: Inspecting Specific Subjects

**In SAS:**
```sas
/* Inspect specific subject — SAS PROC PRINT with WHERE */
PROC PRINT DATA=adsl_buggy(WHERE=(USUBJID = "01-701-1015") OBS=1);
  VAR USUBJID RFXSTDTC RFXENDTC TRTSDT TRTEDT;
RUN;
/* Also inspect raw source data */
PROC PRINT DATA=dm_buggy(WHERE=(USUBJID = "01-701-1015") OBS=1);
  VAR USUBJID RFXSTDTC RFXENDTC;
RUN;
```

**In R (pharmaverse):**

Now we know subject `01-701-1015` has wrong `TRTEDT`. Use `filter()` to inspect the data at the moment `TRTEDT` is derived:

```r
# Break the pipe at the suspect step
step1 <- dm_buggy %>%
  filter(ACTARM != "Screen Failure") %>%
  mutate(TRT01P = ACTARM, TRT01A = ACTARM)

step2 <- step1 %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC, date_imputation = "first")

# Inspect the source variable for the suspect subject
step2 %>%
  filter(USUBJID == "01-701-1015") %>%
  select(USUBJID, RFXSTDTC, RFXENDTC, TRTSDT)
```

Output:
```
  USUBJID        RFXSTDTC   RFXENDTC   TRTSDT
  01-701-1015    2014-01-02  2014-01-02  2014-01-02
```

`RFXENDTC` is `"2014-01-02"` — same as start date. That's the month typo. The raw DM has the wrong value for this subject. Fix: correct the source data.

---

## Step 3: browser() to Trace Bug 2

`diffdf` showed a second subject with wrong `TRTEDT`. That comes from using `"first"` imputation for end dates:

**In SAS (equivalent investigation):**
```sas
/* Find partial dates in RFXENDTC */
DATA check;
  SET dm_buggy;
  len = LENGTH(TRIM(RFXENDTC));
  IF len = 7;   /* YYYY-MM only — partial date */
  KEEP USUBJID RFXENDTC len;
RUN;
PROC PRINT DATA=check; RUN;
```

**In R:**
```r
# Show subjects with partial RFXENDTC (missing day)
dm_buggy %>%
  filter(!is.na(RFXENDTC)) %>%
  filter(grepl("^\\d{4}-\\d{2}$", RFXENDTC)) %>%   # partial: YYYY-MM only
  select(USUBJID, RFXENDTC)
```

If subject `01-701-1023` has a partial end date `"2014-03"`, then:
- `date_imputation = "first"` → `2014-03-01`
- `date_imputation = "last"` → `2014-03-31`

The SAP says end dates use last-of-month imputation. Fix: change `"first"` to `"last"` in `derive_vars_dt(new_vars_prefix = "TRTE", ...)`.

### browser() for Interactive Inspection

In R you can also pause execution mid-pipe and inspect interactively:

```r
# Pause inside a pipe — no SAS equivalent!
adsl <- dm_buggy %>%
  filter(ACTARM != "Screen Failure") %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC) %>%
  { browser(); . } %>%        # pause here — . is the data so far
  derive_vars_dt(new_vars_prefix = "TRTE", dtc = RFXENDTC)
  # At the browser() prompt:
  # - Type . to see the data
  # - filter(., USUBJID=="01-701-1023") %>% select(RFXENDTC) to inspect
  # - ls() to list objects in scope
  # - n to step forward
  # - Q to quit
```

> **No SAS equivalent for `browser()`:** SAS has no interactive mid-execution debugger. The closest approach in SAS is a `%PUT` statement after each step, then re-running. R's `browser()` lets you pause at any point, inspect the environment, and continue without re-running from scratch.

---

## Step 4: The Age Boundary Bug

```r
# Subject with wrong AGEGR1
diff_result$VarDiff_AGEGR1 %>% select(USUBJID, BASE, COMPARE)
# USUBJID=01-xyz   BASE="40-64"   COMPARE=">=65"
# BASE is buggy (="40-64"), COMPARE is reference (=">=65")

# What's this subject's age?
adsl_buggy %>%
  filter(USUBJID == "01-xyz") %>%
  select(USUBJID, AGE, AGEGR1)
# AGE=65, AGEGR1="40-64"

# The bug: AGE > 65 should be AGE >= 65
# Subject aged exactly 65 falls into "40-64" (because 65 < 65 is FALSE → AGE > 65 is FALSE)
```

**In SAS this is equivalent to:**
```sas
/* The SAS bug would look like: */
ELSE IF AGE > 65 THEN AGEGR1 = ">=65";  /* should be >= 65 */

/* To find it:
PROC FREQ DATA=adsl_buggy;
  TABLES AGE * AGEGR1 / LIST;
  WHERE AGE IN (39, 40, 64, 65, 66);   /* check boundary ages */
RUN;
```

This is the most common age-group bug. Always use `>=` not `>` for the upper boundary.

Fix:
```r
AGEGR1 = case_when(
  AGE < 40              ~ "<40",
  AGE >= 40 & AGE < 65  ~ "40-64",
  AGE >= 65             ~ ">=65",   # fixed: >= not >
  TRUE                  ~ NA_character_
)
```

---

## The Complete Bug-Finding Program

```r
# pharm001_ch13_debug.R
# Study PHARM-001 — Chapter 13
# Find the 3 bugs in adsl_buggy using diffdf + browser()

library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(dplyr)
library(diffdf)

# ---- Build buggy ADSL (from Chapter 12 exercise) ----
dm <- pharmaversesdtm::dm

dm_buggy <- dm %>%
  mutate(RFXENDTC = if_else(USUBJID == "01-701-1015", "2014-01-02", RFXENDTC))

adsl_buggy <- dm_buggy %>%
  filter(ACTARM != "Screen Failure") %>%
  mutate(TRT01P = ACTARM, TRT01A = ACTARM,
         TRT01PN = as.integer(factor(ACTARM, levels=sort(unique(ACTARM)))),
         TRT01AN = TRT01PN) %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC, date_imputation = "first") %>%
  derive_vars_dt(new_vars_prefix = "TRTE", dtc = RFXENDTC, date_imputation = "first") %>%
  mutate(
    TRTDURD = as.integer(TRTEDT - TRTSDT + 1),
    AGEGR1  = case_when(
      AGE < 40              ~ "<40",
      AGE >= 40 & AGE < 65  ~ "40-64",
      AGE > 65              ~ ">=65",   # BUG
      TRUE                  ~ NA_character_
    )
  )

adsl_ref <- pharmaverseadam::adsl

# ---- Step 1: diffdf (equivalent to PROC COMPARE) ----
cat("=== PHARM-001 ADSL Bug Hunt ===\n\n")
diff_result <- diffdf(base = adsl_buggy, compare = adsl_ref, keys = "USUBJID")

if (diffdf_issame(diff_result)) {
  cat("No differences found — no bugs to find!\n")
  stop()
}

print(diff_result)

# ---- Step 2: Per-variable diagnosis ----
cat("\n--- Variable discrepancies ---\n")
for (var in unique(diff_result$NumDiff$Variable)) {
  n_diff <- diff_result$NumDiff$`No of Differences`[diff_result$NumDiff$Variable == var]
  cat(sprintf("\n%s: %d discrepancy/ies\n", var, n_diff))
  
  detail_key <- paste0("VarDiff_", var)
  if (detail_key %in% names(diff_result)) {
    detail <- diff_result[[detail_key]]
    for (i in seq_len(min(5, nrow(detail)))) {
      cat(sprintf("  USUBJID=%-20s  buggy=%-15s  ref=%s\n",
                  detail$USUBJID[i],
                  as.character(detail$BASE[i]),
                  as.character(detail$COMPARE[i])))
    }
  }
}

# ---- Step 3: Root cause for TRTEDT ----
cat("\n--- TRTEDT: inspect source data for discrepant subjects ---\n")
if ("VarDiff_TRTEDT" %in% names(diff_result)) {
  bad_ids <- diff_result$VarDiff_TRTEDT$USUBJID
  # Equivalent to: PROC PRINT DATA=dm_buggy(WHERE=(USUBJID IN (&bad_ids)));
  dm_buggy %>%
    filter(USUBJID %in% bad_ids) %>%
    select(USUBJID, RFXSTDTC, RFXENDTC) %>%
    print()
  cat("FIX: Check RFXENDTC for month typo (Bug 1) and date_imputation rule (Bug 2)\n")
}

# ---- Step 4: Root cause for AGEGR1 ----
cat("\n--- AGEGR1: inspect age for discrepant subjects ---\n")
if ("VarDiff_AGEGR1" %in% names(diff_result)) {
  bad_ids <- diff_result$VarDiff_AGEGR1$USUBJID
  adsl_buggy %>%
    filter(USUBJID %in% bad_ids) %>%
    select(USUBJID, AGE, AGEGR1) %>%
    print()
  cat("FIX: AGE > 65 should be AGE >= 65 in case_when (Bug 3)\n")
}

# ---- Summary ----
cat("\n=== Summary ===\n")
cat("Bug 1: RFXENDTC month typo for 01-701-1015 (2014-07 → 2014-01)\n")
cat("Bug 2: TRTEDT using date_imputation='first' (should be 'last')\n")
cat("Bug 3: AGEGR1 boundary: AGE > 65 should be AGE >= 65\n")
cat("\nAll 3 bugs identified. Fix and re-run.\n")
```

---

## Debugging Techniques for the Pipeline

### Technique 1: Break the Pipe

**In SAS (equivalent step isolation):**
```sas
/* Run steps one at a time, check intermediate results */
DATA step1;
  SET dm(WHERE=(ACTARM NE "Screen Failure"));
RUN;
PROC CONTENTS DATA=step1; RUN;   /* check step1 */

DATA step2;
  SET step1;
  /* ... date conversion ... */
RUN;
PROC PRINT DATA=step2(OBS=5); RUN;  /* check step2 */
```

**In R:**
```r
# This crashes somewhere — where?
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  derive_vars_dt(...) %>%
  left_join(ex_derived, by="USUBJID") %>%
  mutate(SAFFL = if_else(dosed, "Y", "N"))

# Break it:
step1 <- dm %>% filter(ACTARM != "Screen Failure")
step2 <- step1 %>% derive_vars_dt(...)
step3 <- step2 %>% left_join(ex_derived, by="USUBJID")
# Check nrow() at each step — find where rows change unexpectedly
```

### Technique 2: Mid-Pipe browser()

```r
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC) %>%
  { browser(); . } %>%        # pause here, . is the data
  derive_vars_dt(new_vars_prefix = "TRTE", dtc = RFXENDTC)
```

At the `browser()` prompt: type `.` to see the data, `ls()` to list objects, `n` to step forward.

### Technique 3: The Join Row Explosion

You have 254 subjects. After a `left_join`, you suddenly have 800 rows. This is the #1 silent bug.

**In SAS — equivalent problem:**
```sas
/* SAS MERGE requires sorted, unique BY variables */
/* If dm has multiple rows per USUBJID, MERGE multiplies records */
/* You'd see: NOTE: Multiple lengths were specified for the BY variable USUBJID */
/* Or worse: wrong results without any note */

/* Diagnose: */
PROC FREQ DATA=ex NOPRINT;
  TABLES USUBJID / OUT=ex_counts;
RUN;
PROC PRINT DATA=ex_counts(WHERE=(COUNT > 1)); RUN;
```

**In R:**
```r
# Danger: ex has multiple rows per subject
adsl %>% left_join(ex, by = "USUBJID")   # 254 × avg 4 doses = ~800 rows!
```

**Diagnose:**
```r
ex %>% count(USUBJID) %>% filter(n > 1)
# Shows every subject with multiple EX records
```

**Fix: always aggregate before joining**
```r
ex_first_dose <- ex %>%
  mutate(EXSTDT = as.Date(substr(EXSTDTC, 1, 10))) %>%
  group_by(USUBJID) %>%
  summarise(EXSTDT_FIRST = min(EXSTDT, na.rm = TRUE), .groups = "drop")
# Now ex_first_dose has one row per subject — safe to join

adsl %>% left_join(ex_first_dose, by = "USUBJID")
```

**Safe join wrapper — crashes loudly if rows change:**

```r
safe_join <- function(left, right, by, type = "left") {
  n_before <- nrow(left)
  result <- switch(type,
    left  = left_join(left, right, by = by),
    inner = inner_join(left, right, by = by)
  )
  n_after <- nrow(result)
  if (n_after != n_before)
    stop(sprintf(
      "Join on '%s' changed row count: %d → %d (+%d). Right side not unique!",
      paste(by, collapse=", "), n_before, n_after, n_after - n_before
    ))
  result
}
```

---

## Debugging Quick Reference

```r
# ── STOP AND LOOK ───────────────────────────────
browser()                       # pause, open REPL
browser(expr = condition)       # pause only when TRUE
debugonce(admiral::derive_vars_dt)  # pause at function entry, once
options(error = recover)        # post-error: pick a stack frame

# ── FIND THE CRASH ──────────────────────────────
traceback()                     # call stack after error

# ── INJECT INTO A PIPE ──────────────────────────
df %>% { browser(); . }         # pause mid-pipe (. is the data)
df %>% tap(cat("rows:", nrow(df), "\n"))   # print mid-pipe

# ── CRASH LOUDLY ────────────────────────────────
stopifnot(nrow(adsl) == 254)
stop("Expected 254 subjects, got ", nrow(adsl))

# ── CLINICAL-SPECIFIC ───────────────────────────
# Check join uniqueness
before <- nrow(left_df)
result <- left_join(left_df, right_df, by = "USUBJID")
stopifnot("Join expanded rows" = nrow(result) == before)

# Date ordering
bad <- adsl %>% filter(!is.na(TRTSDT) & !is.na(TRTEDT) & TRTEDT < TRTSDT)
if (nrow(bad) > 0) stop(nrow(bad), " subjects with TRTEDT < TRTSDT")

# Flag integrity
stopifnot(all(adsl$ITTFL %in% c("Y", NA)))
stopifnot(!any(duplicated(adsl$USUBJID)))
```

---

## What We Have Now: The Complete Pipeline

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM
pharm001_ch03_dm.R       — map raw DM to SDTM
pharm001_ch04.R          — derive TRTSDT, TRTEDT, TRTDURD
pharm001_ch11_adsl.R     — complete ADSL (logged)
pharm001_ch06_adae.R     — complete ADAE
pharm001_ch09_export.R   — XPT export
pharm001_ch07_t14_1_1.R  — Table 14.1.1
pharm001_ch08_t14_3_1.R  — Table 14.3.1
pharm001_ch10_app.R      — teal explorer
pharm001_ch13_debug.R    — bug-finding script
run_all.R                — run the complete pipeline
output/adam/*.xpt        — submission-ready XPT files
output/tlg/*.txt, *.png  — tables and figures
logs/*.log               — audit trail
renv.lock                — locked package versions
```

---

## Exercises

**1.** Implement `safe_join(left, right, by, type="left")`. Use it to replace every `left_join` in `pharm001_ch11_adsl.R`. Do any of them fire?

**2.** Introduce 3 more bugs in ADAE (swap `TRTEMFL` for two subjects, wrong `ASTDY` for one). Run `diffdf()` against `pharmaverseadam::adae`. Do you catch all three?

**3.** Use `debugonce(admiral::derive_vars_dt)` to step through the imputation logic for a subject with a partial `RFXENDTC`. Confirm you understand what the `date_imputation` argument does.

**4.** The final exercise: run `run_all.R` from a clean R session (`Sys.setenv(R_LIBS_USER="")`) with only packages from `renv.lock`. Does everything still work? This is how submission QA works.

---

*The pharmaverse doesn't change the debugging tools. It just gives you better things to debug.*
