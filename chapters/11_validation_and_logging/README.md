# Chapter 11: Validation, Logging, and QC

*Regulatory-grade R: logrx for audit trails, diffdf for dataset comparison, valtools for validation.*

---

## Why Validation Matters

In pharma, your R programs are part of a regulated system. When you deliver an ADSL or a demographics table, the regulator needs confidence that:

1. The program ran correctly (logging / audit trail)
2. The output matches an independent QC check (double programming)
3. The package versions used are documented
4. Any errors during execution are captured

This chapter covers the tools that make R workflows submission-ready.

---

## {logrx} — Audit Logging

`logrx` creates a log file for every program execution — equivalent to the SAS `.log` file. It captures:

- Date/time of execution
- User and machine info
- R version + package versions used
- `sessionInfo()` output
- Any warnings or messages
- Completion status (success/error)

```r
library(logrx)
```

### The axecute() pattern

Wrap your entire analysis program in `axecute()`:

```r
logrx::axecute("adsl_derivation.R")
```

This runs the program and writes a `.log` file alongside it. The log looks like:

```
-----------------------------------------------------------------------------------------
Log file: adsl_derivation.R
Run date and time: 2026-04-02 14:32:01 CEST
Username: yosia
Filepath: /projects/study001/adam/adsl_derivation.R
-----------------------------------------------------------------------------------------

--- R Version ---
R version 4.4.2 (2024-10-31)
...

--- Packages Attached ---
admiral_1.1.0, dplyr_1.1.4, lubridate_1.9.3
...

--- Messages ---
[none]

--- Warnings ---
[none]

--- Program Status ---
Success
-----------------------------------------------------------------------------------------
```

### Inline logging

```r
# Log specific messages inside your program
log_print("Starting ADSL derivation")
log_print(sprintf("Input DM: %d rows", nrow(dm)))

# Log a dataset snapshot
log_print(head(adsl, 3))

# Log timing
start_time <- proc.time()
# ... your derivation ...
elapsed <- proc.time() - start_time
log_print(sprintf("Derivation complete in %.1f seconds", elapsed[3]))
```

---

## {diffdf} — Dataset Comparison

Double programming: two programmers independently derive the same dataset, then compare. `diffdf` does the comparison.

```r
library(diffdf)

# Compare two ADSL datasets
diff_result <- diffdf(
  base      = adsl_prog1,   # your derivation
  compare   = adsl_prog2,   # independent QC derivation
  keys      = "USUBJID",    # unique key
  tolerance = 1e-6          # numeric tolerance
)

# Summary
print(diff_result)         # shows all discrepancies
summary(diff_result)       # just counts

# Check: are they identical?
if (isTRUE(diffdf_issame(diff_result))) {
  cat("✓ Datasets are identical\n")
} else {
  cat("✗ Discrepancies found\n")
  print(diff_result)
}
```

`diffdf` output:

```
Differences found between the two DataFrames!

Not all Values Compared Equal:
> 3 Variables had non-identical values. 
    Variable  No of Differences
    TRTSDT             2
    TRTDURD            2
    AGEGR1             1

For the TRTSDT variable:
> USUBJID: 01-701-1015, Base=2014-01-02, Compare=2014-01-01
> USUBJID: 01-701-1023, Base=2014-03-17, Compare=2014-03-16
```

---

## Package Version Management with renv

In regulated environments, you need to ensure everyone runs the same package versions. `renv` locks your package dependencies:

```r
library(renv)

# Initialize renv for a project
renv::init()     # creates renv.lock

# Snapshot current packages
renv::snapshot() # updates renv.lock

# Restore to locked versions (on another machine / after update)
renv::restore()

# Check for version differences
renv::status()
```

The `renv.lock` file gets committed to git. Any team member who runs `renv::restore()` gets identical package versions.

---

## Program: QC Report Generator

```r
# qc_report.R
# Generate a complete QC comparison report

library(diffdf)
library(dplyr)
library(pharmaverseadam)

# Simulate two independent derivations (in practice, two separate programs)
adsl_prod <- pharmaverseadam::adsl

# Simulate QC with one deliberate difference
adsl_qc   <- pharmaverseadam::adsl %>%
  mutate(
    # Simulate QC programmer using different date imputation
    TRTSDT = if_else(USUBJID == "01-701-1015",
                     TRTSDT - 1L,   # one day off
                     TRTSDT),
    # Simulate typo in one AGEGR1
    AGEGR1 = if_else(USUBJID == "01-701-1023", "<40", AGEGR1)
  )

# ---- Run comparison ----
cat("=== Dataset QC Comparison ===\n")
cat(sprintf("Production: %d rows × %d cols\n", nrow(adsl_prod), ncol(adsl_prod)))
cat(sprintf("QC:         %d rows × %d cols\n", nrow(adsl_qc),   ncol(adsl_qc)))

# Row count
if (nrow(adsl_prod) != nrow(adsl_qc)) {
  cat(sprintf("\n[FAIL] Row count mismatch: prod=%d, qc=%d\n",
              nrow(adsl_prod), nrow(adsl_qc)))
} else {
  cat(sprintf("\n[PASS] Row counts match: %d\n", nrow(adsl_prod)))
}

# Key variable: check USUBJID alignment
prod_ids <- sort(adsl_prod$USUBJID)
qc_ids   <- sort(adsl_qc$USUBJID)
if (!identical(prod_ids, qc_ids)) {
  missing_in_qc   <- setdiff(prod_ids, qc_ids)
  extra_in_qc     <- setdiff(qc_ids, prod_ids)
  cat(sprintf("[FAIL] USUBJID mismatch: %d missing in QC, %d extra in QC\n",
              length(missing_in_qc), length(extra_in_qc)))
} else {
  cat("[PASS] USUBJID sets match\n")
}

# Full comparison
diff_result <- diffdf(
  base    = adsl_prod,
  compare = adsl_qc,
  keys    = "USUBJID"
)

cat("\n--- Variable Comparison ---\n")
if (diffdf_issame(diff_result)) {
  cat("[PASS] All variables match\n")
} else {
  # Report discrepancies
  vars_diff <- unique(diff_result$NumDiff$Variable)
  cat(sprintf("[FAIL] %d variable(s) with differences:\n", length(vars_diff)))
  
  for (var in vars_diff) {
    n_diff <- diff_result$NumDiff$`No of Differences`[diff_result$NumDiff$Variable == var]
    cat(sprintf("  %-20s: %d discrepancies\n", var, n_diff))
  }
  
  # Show details for each discrepant variable
  cat("\nDiscrepancy details:\n")
  for (var in vars_diff) {
    cat(sprintf("\n  %s:\n", var))
    detail_name <- paste0("VarDiff_", var)
    if (detail_name %in% names(diff_result)) {
      detail <- diff_result[[detail_name]]
      for (i in seq_len(min(5, nrow(detail)))) {
        cat(sprintf("    USUBJID=%-20s  Prod: %-15s  QC: %s\n",
                    detail$USUBJID[i],
                    as.character(detail$BASE[i]),
                    as.character(detail$COMPARE[i])))
      }
    }
  }
}

# ---- Overall QC status ----
cat("\n=== QC RESULT ===\n")
if (diffdf_issame(diff_result) && nrow(adsl_prod) == nrow(adsl_qc)) {
  cat("STATUS: PASS — Production and QC match exactly\n")
} else {
  cat("STATUS: FAIL — Differences found, requires investigation\n")
}
```

---

## Validation Documentation

Beyond running the code, submission-ready R requires:

**1. User Requirements Specification (URS)**
- What the program is supposed to do
- Input datasets, output datasets
- Variable derivation rules

**2. Functional Specification**
- How the program implements the URS
- Which admiral functions are used and why

**3. Testing Evidence**
- diffdf comparison vs. independent QC
- logrx log file showing successful execution
- Unit test results (if using `testthat`)

```r
# testthat for unit testing analysis functions
library(testthat)

test_that("TRTEMFL is Y only for post-baseline AEs", {
  result <- derive_trtemfl(test_ae, test_adsl)
  trtemfl_dates <- result %>% filter(TRTEMFL == "Y")
  expect_true(all(!is.na(trtemfl_dates$ASTDT)))
  expect_true(all(trtemfl_dates$ASTDT >= trtemfl_dates$TRTSDT))
})

test_that("TRTDURD is never negative", {
  expect_true(all(adsl$TRTDURD >= 0, na.rm = TRUE))
})

test_that("SAFFL subjects have at least one dose", {
  safe_subjects <- adsl %>% filter(SAFFL == "Y") %>% pull(USUBJID)
  dosed_subjects <- ex %>% distinct(USUBJID) %>% pull(USUBJID)
  expect_true(all(safe_subjects %in% dosed_subjects))
})
```

---

## Exercises

**1. Log your ADSL**

Wrap your ADSL derivation from Chapter 5 in `logrx::axecute()`. Review the generated log file. Does it capture all warnings?

**2. diffdf on ADAE**

Introduce 3 deliberate errors into a copy of `pharmaverseadam::adae` (one in TRTEMFL, one in ASTDY, one in ASEVN). Run `diffdf`. Does it catch all three?

**3. renv lockfile**

Create an `renv` project, install the core pharmaverse packages, and snapshot. Send the `renv.lock` file to a colleague — they should be able to exactly reproduce your environment with `renv::restore()`.

**4. testthat for analysis rules**

Write 5 unit tests for your ADSL derivation:
- Flag variables are only "Y" or NA
- Date variables are Date class
- TRTDURD = TRTEDT - TRTSDT + 1
- All ITTFL subjects are in DM
- No duplicate USUBJIDs

---

*Next: Chapter 12 — End-to-End Study: the complete pipeline from raw to submission*
