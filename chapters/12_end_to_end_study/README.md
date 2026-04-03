# Chapter 12: The Complete Pipeline

*One script. Runs everything. Produces the complete PHARM-001 package.*

---

## Where We Left Off

Eleven chapters of incremental work. Every program is done, logged, and QC'd. Now we tie it together:

```
pharm001_ch11_adsl.R     — ADSL (with logging)
pharm001_ch06_adae.R     — ADAE
pharm001_ch09_export.R   — XPT export
pharm001_ch07_t14_1_1.R  — Table 14.1.1
pharm001_ch08_t14_3_1.R  — Table 14.3.1
pharm001_ch10_app.R      — teal explorer
```

One `run_all.R` script executes them all in the right order.

---

## The Pipeline

```
SDTM (pharmaversesdtm)
  ↓ pharm001_ch02_validate.R   — validate domains
  ↓ pharm001_ch03_dm.R         — map raw DM
ADaM
  ↓ pharm001_ch11_adsl.R       — ADSL
  ↓ pharm001_ch06_adae.R       — ADAE
  ↓ pharm001_ch09_export.R     — XPT files
TLG
  ↓ pharm001_ch07_t14_1_1.R    — Table 14.1.1
  ↓ pharm001_ch08_t14_3_1.R    — Table 14.3.1
  ↓ figure 14.2.1 (KM plot)
Submission Package
  ↓ output/adam/*.xpt
  ↓ output/tlg/*.txt, *.png
  ↓ logs/*.log
```

---

## SAS Autoexec / run_all Equivalent

**In SAS**, the equivalent of `run_all.R` is a master control program or autoexec:

```sas
/* run_all.sas — Master control program */
/* Setup: library references, macro library */
%INCLUDE "setup/autoexec.sas";           /* R equivalent: source() */

/* SDTM */
%INCLUDE "programs/sdtm/dm.sas";         /* R: source() or axecute() */
%INCLUDE "programs/sdtm/ae.sas";

/* ADaM */
%INCLUDE "programs/adam/adsl.sas";
%INCLUDE "programs/adam/adae.sas";

/* TLG */
%INCLUDE "programs/tlg/t14_1_1.sas";
%INCLUDE "programs/tlg/t14_3_1.sas";
```

| SAS | R |
|-----|---|
| `%INCLUDE "program.sas";` | `source("program.R")` |
| SAS autoexec.sas | `source("setup/options.R")` |
| Macro library (`%MACRO`, `SASAUTOS=`) | R package / `source()` |
| `LIBNAME adam "output/adam/";` | Working directory + `here::here()` |

---

## Metadata Catalog: SAS vs. metacore

**In SAS**, variable metadata (labels, types, formats) lives in a SAS catalog or dataset:
```sas
/* SAS metadata catalog approach */
PROC FORMAT CNTLOUT=fmtcat;   /* export format catalog */
RUN;

PROC CONTENTS DATA=adsl OUT=adsl_meta; /* export variable metadata */
RUN;
```

**In R (pharmaverse — metacore):**
```r
library(metacore)

# metacore loads metadata from a spec Excel file
spec <- spec_to_metacore(
  path              = "specs/ADaM_spec.xlsx",
  where_sep_sheet   = FALSE
)

# The metacore object contains:
# - Variable labels and types
# - Dataset-level metadata
# - Codelists for CT validation
# - Everything xportr needs for export

# Use it in the xportr pipeline
adsl %>%
  xportr_type(spec,   domain = "ADSL") %>%
  xportr_label(spec,  domain = "ADSL") %>%
  xportr_length(spec, domain = "ADSL") %>%
  xportr_format(spec, domain = "ADSL") %>%
  xportr_order(spec,  domain = "ADSL") %>%
  xportr_write("output/adam/adsl.xpt")
```

---

## run_all.R

```r
# run_all.R
# Study PHARM-001 — Chapter 12
# Execute the complete pipeline in order

library(logrx)

# ---- Config ----
PROG_DIR <- "programs"
LOG_DIR  <- "logs"
OUT_DIR  <- "output"

dir.create(LOG_DIR,             recursive = TRUE, showWarnings = FALSE)
dir.create(file.path(OUT_DIR, "adam"), recursive = TRUE, showWarnings = FALSE)
dir.create(file.path(OUT_DIR, "tlg"),  recursive = TRUE, showWarnings = FALSE)

# ---- Run helper ----
run_program <- function(path, log_dir = LOG_DIR) {
  prog_name <- tools::file_path_sans_ext(basename(path))
  log_path  <- file.path(log_dir, paste0(prog_name, ".log"))
  
  cat(sprintf("  %-45s ... ", basename(path)))
  start <- proc.time()
  
  result <- tryCatch({
    logrx::axecute(path, log = log_path)
    elapsed <- (proc.time() - start)[3]
    cat(sprintf("OK  [%.1fs]\n", elapsed))
    TRUE
  }, error = function(e) {
    cat(sprintf("FAIL\n  → %s\n", conditionMessage(e)))
    FALSE
  })
  
  invisible(result)
}

# ---- Pipeline ----
cat("╔══════════════════════════════════════════╗\n")
cat("║  Study PHARM-001 — Full Pipeline Run     ║\n")
cat(sprintf("║  %s             ║\n", format(Sys.time(), "%Y-%m-%d %H:%M:%S")))
cat("╚══════════════════════════════════════════╝\n\n")

status <- logical(0)

cat("─── SDTM Validation ───\n")
status["validate"] <- run_program(
  file.path(PROG_DIR, "pharm001_ch02_validate.R"))

cat("\n─── SDTM Mapping ───\n")
status["dm"] <- run_program(
  file.path(PROG_DIR, "pharm001_ch03_dm.R"))

cat("\n─── ADaM ───\n")
status["adsl"] <- run_program(
  file.path(PROG_DIR, "pharm001_ch11_adsl.R"))
status["adae"] <- run_program(
  file.path(PROG_DIR, "pharm001_ch06_adae.R"))

cat("\n─── XPT Export ───\n")
status["export"] <- run_program(
  file.path(PROG_DIR, "pharm001_ch09_export.R"))

cat("\n─── TLG ───\n")
status["t14_1_1"] <- run_program(
  file.path(PROG_DIR, "pharm001_ch07_t14_1_1.R"))
status["t14_3_1"] <- run_program(
  file.path(PROG_DIR, "pharm001_ch08_t14_3_1.R"))

# ---- Summary ----
cat("\n╔══════════════════════════════════════════╗\n")
cat("║  Pipeline Summary                        ║\n")
cat("╚══════════════════════════════════════════╝\n")
n_ok   <- sum(status)
n_fail <- sum(!status)
cat(sprintf("  Passed: %d/%d programs\n", n_ok, length(status)))
if (n_fail > 0) {
  cat(sprintf("  Failed: %d — %s\n", n_fail,
              paste(names(status)[!status], collapse=", ")))
  cat("  STATUS: FAIL\n")
} else {
  cat("  STATUS: PASS\n")
}

# ---- Output inventory ----
cat("\nOutput files:\n")
for (f in c(
  list.files("output/adam", pattern = "*.xpt", full.names = TRUE),
  list.files("output/tlg",  pattern = "*",     full.names = TRUE)
)) {
  size <- round(file.info(f)$size / 1024, 1)
  cat(sprintf("  %-40s  %6.1f KB\n", basename(f), size))
}
```

---

## The Twelve Programs: Summary

| Program | What it produces |
|---------|-----------------|
| `pharm001_ch01.R` | Basic demographics printout |
| `pharm001_ch02_validate.R` | SDTM validation report |
| `pharm001_ch03_dm.R` | SDTM DM from raw |
| `pharm001_ch04.R` | TRTSDT/TRTEDT derivation |
| `pharm001_ch11_adsl.R` | `output/adam/adsl.xpt` |
| `pharm001_ch06_adae.R` | `output/adam/adae.xpt` |
| `pharm001_ch09_export.R` | All XPT files |
| `pharm001_ch07_t14_1_1.R` | `output/tlg/t14_1_1.txt` |
| `pharm001_ch08_t14_3_1.R` | `output/tlg/t14_3_1.txt` |
| `pharm001_ch10_app.R` | Interactive teal app |
| `run_all.R` | Runs everything in order |

---

## Study Folder Structure: SAS and R Are the Same

The folder structure for a regulatory submission is the same whether you use SAS or R. This is intentional — the ICH E3 and FDA submission guidelines are platform-agnostic.

**Both SAS and R studies use:**
```
study/
├── programs/          # programming files (.sas or .R)
│   ├── sdtm/
│   ├── adam/
│   └── tlg/
├── output/            # produced datasets and tables
│   ├── adam/         # XPT files (same format regardless of origin language)
│   └── tlg/
├── logs/              # one log per program run
├── specs/             # variable specs, define.xml
└── [renv.lock | SAS version file]   # version control
```

The XPT format is binary and identical whether produced by SAS or R + `xportr`. An FDA reviewer cannot tell the difference.

---

## What Real Statistical Programmers Do Differently

The tools are the same. The judgment is different.

**Read the SAP.** Every derivation rule in admiral has study-specific variations. The SAP overrides the default. Admiral functions implement common rules — the SAP may have tweaks.

**Handle edge cases.** What if a subject has no exposure records? What if `RFXSTDTC` is missing but `RFSTDTC` exists? What if `AESTDTC` is partial and the imputed date precedes treatment start? These cases don't appear in vignettes.

**Version everything.** Every program has a version header. Every dataset has a derivation date variable. Every run has a log.

**Know the submission requirements.** Pinnacle 21 validator, define.xml, reviewer's guide. The xportr export is one piece — the full submission package requires more.

---

## The Complete PHARM-001 Folder

```
pharm001/
├── programs/
│   ├── pharm001_ch02_validate.R
│   ├── pharm001_ch03_dm.R
│   ├── pharm001_ch11_adsl.R
│   ├── pharm001_ch06_adae.R
│   ├── pharm001_ch09_export.R
│   ├── pharm001_ch07_t14_1_1.R
│   ├── pharm001_ch08_t14_3_1.R
│   └── pharm001_ch10_app.R
├── output/
│   ├── adam/
│   │   ├── adsl.xpt
│   │   ├── adae.xpt
│   │   ├── adtte.xpt
│   │   └── adlb.xpt
│   └── tlg/
│       ├── t14_1_1.txt
│       ├── t14_3_1.txt
│       └── f14_2_1_km_os.png
├── logs/
│   └── (one .log per program run)
├── run_all.R
└── renv.lock
```

---

## What We Have Now

The complete PHARM-001 pipeline. Raw data → submission package.

---

## Exercises

**1.** Add a dependency check to `run_all.R`: before running ADAE, verify that `output/adam/adsl.xpt` exists. If ADSL failed, skip ADAE and all downstream programs.

**2.** Add a timing report at the end of `run_all.R`: show total elapsed time and time per program.

**3.** Add `renv::status()` as the first check in `run_all.R`. If packages have drifted from `renv.lock`, warn before running.

**4. (Sets up Chapter 13)** Your QA colleague runs `run_all.R` and gets 3 discrepancies when comparing the output ADSL against the reference. The differences are in `TRTSDT` for 2 subjects, `AGEGR1` for 1 subject. You need to reproduce the bug and find the root cause. Introduce those 3 bugs deliberately into `pharm001_ch11_adsl.R` (without fixing them yet). Save the buggy ADSL. Chapter 13 teaches you how to find them.

---

*Next: Chapter 13 — debugging. We find the 3 bugs using `browser()` and `diffdf`.*

---

## Solutions

### Exercise 1

Add dependency checks to `run_all.R`: skip downstream programs if an upstream output is missing.

```r
# SAS: /* No built-in dependency management — typically done with custom macro logic */
# %MACRO CHECK_EXIST(path);
#   %IF NOT %SYSFUNC(FILEEXIST(&path)) %THEN %DO;
#     %PUT ERROR: Required file not found: &path;
#     %ABORT CANCEL;
#   %END;
# %MEND;
# %CHECK_EXIST(output/adam/adsl.xpt);

library(logrx)

# Dependency-aware run function
run_with_deps <- function(prog_path, log_dir, required_files = character(0)) {
  prog_name <- tools::file_path_sans_ext(basename(prog_path))
  log_file  <- file.path(log_dir, paste0(prog_name, ".log"))

  # Check dependencies first
  missing_deps <- required_files[!file.exists(required_files)]
  if (length(missing_deps) > 0) {
    cat(sprintf("  [SKIP] %-45s — missing upstream: %s\n",
                basename(prog_path), paste(basename(missing_deps), collapse=", ")))
    return(FALSE)
  }

  if (!file.exists(prog_path)) {
    cat(sprintf("  [SKIP] %-45s — program not found\n", basename(prog_path)))
    return(FALSE)
  }

  cat(sprintf("  %-47s ... ", basename(prog_path)))
  start <- proc.time()
  result <- tryCatch({
    logrx::axecute(prog_path, log = log_file)
    elapsed <- (proc.time() - start)[3]
    cat(sprintf("OK [%.1fs]\n", elapsed))
    TRUE
  }, error = function(e) {
    cat(sprintf("FAIL: %s\n", conditionMessage(e)))
    FALSE
  })
  result
}

# Pipeline with explicit dependencies
LOG_DIR <- "logs"
dir.create(LOG_DIR, showWarnings = FALSE)

cat("=== PHARM-001 Pipeline with Dependency Checks ===\n\n")

status <- list()

# SDTM (no dependencies)
status$validate <- run_with_deps(
  "programs/pharm001_ch02_validate.R", LOG_DIR
)

# ADSL depends on nothing (reads from pharmaversesdtm directly)
status$adsl <- run_with_deps(
  "programs/pharm001_ch11_adsl.R", LOG_DIR
)

# ADAE depends on ADSL XPT
status$adae <- run_with_deps(
  "programs/pharm001_ch06_adae.R", LOG_DIR,
  required_files = c("output/adam/adsl.xpt")
)

# Export depends on ADaM programs
status$export <- run_with_deps(
  "programs/pharm001_ch09_export.R", LOG_DIR,
  required_files = c("output/adam/adsl.xpt", "output/adam/adae.xpt")
)

# Tables depend on export
status$t14_1_1 <- run_with_deps(
  "programs/pharm001_ch07_t14_1_1.R", LOG_DIR,
  required_files = c("output/adam/adsl.xpt")
)

status$t14_3_1 <- run_with_deps(
  "programs/pharm001_ch08_t14_3_1.R", LOG_DIR,
  required_files = c("output/adam/adae.xpt")
)

# Summary
n_ok   <- sum(unlist(status))
n_total <- length(status)
cat(sprintf("\nPipeline complete: %d/%d programs succeeded.\n", n_ok, n_total))
```

### Exercise 2

Add a timing report to `run_all.R`.

```r
# SAS: /* PROC MEANS on elapsed times from LOG or manual timing with macros */
# OPTIONS FULLSTIMER;   /* adds detailed timing to SAS log */
# data _null_; start = datetime(); /* manual timing */

library(logrx)

# Timing-aware run helper
run_timed <- function(prog_path, log_dir) {
  if (!file.exists(prog_path)) return(list(ok=FALSE, elapsed=0))
  log_file <- file.path(log_dir, paste0(
    tools::file_path_sans_ext(basename(prog_path)), ".log"))

  start <- proc.time()
  ok <- tryCatch({
    logrx::axecute(prog_path, log = log_file)
    TRUE
  }, error = function(e) FALSE)

  elapsed <- (proc.time() - start)[3]
  list(ok = ok, elapsed = elapsed, prog = basename(prog_path))
}

LOG_DIR    <- "logs"
PROG_DIR   <- "programs"
pipeline_start <- proc.time()
dir.create(LOG_DIR, showWarnings = FALSE)

programs <- c(
  "pharm001_ch02_validate.R",
  "pharm001_ch11_adsl.R",
  "pharm001_ch06_adae.R",
  "pharm001_ch09_export.R",
  "pharm001_ch07_t14_1_1.R",
  "pharm001_ch08_t14_3_1.R"
)

results <- lapply(programs, function(p) {
  path <- file.path(PROG_DIR, p)
  cat(sprintf("  %-47s ... ", p))
  res  <- run_timed(path, LOG_DIR)
  cat(sprintf("%s [%.1fs]\n", if (res$ok) "OK  " else "FAIL", res$elapsed))
  res
})

# Timing report
total_elapsed <- (proc.time() - pipeline_start)[3]

cat("\n=== Timing Report ===\n")
cat(sprintf("%-47s  %8s  %s\n", "Program", "Elapsed", "Status"))
cat(strrep("-", 65), "\n")
for (res in results) {
  cat(sprintf("%-47s  %6.1fs   %s\n",
              res$prog,
              res$elapsed,
              if (res$ok) "PASS" else "FAIL"))
}
cat(strrep("-", 65), "\n")
cat(sprintf("%-47s  %6.1fs   %s\n", "TOTAL",
            total_elapsed,
            if (all(sapply(results, `[[`, "ok"))) "ALL PASS" else "FAILURES"))
```

### Exercise 3

Add `renv::status()` as the first check in `run_all.R`.

```r
# SAS: /* No equivalent — SAS version documented in header comment or PROC SETINIT */
# /* At best: OPTIONS VERBOSE; /* prints SAS version to log */

library(renv)

# Add this block at the top of run_all.R before any programs run:
cat("=== PHARM-001 Environment Check ===\n")
cat(sprintf("R version: %s\n", R.version$version.string))
cat(sprintf("Date:      %s\n", format(Sys.time(), "%Y-%m-%d %H:%M:%S")))

# Check renv status
if (file.exists("renv.lock")) {
  cat("\nChecking renv lockfile...\n")
  renv_status <- tryCatch({
    renv::status(verbose = FALSE)
    NULL
  }, error = function(e) e)

  if (!is.null(renv_status)) {
    cat(sprintf("[WARN] renv::status() error: %s\n", conditionMessage(renv_status)))
    cat("[WARN] Package versions may differ from renv.lock — proceed with caution\n")
  } else {
    cat("[INFO] renv::status() completed — check output above for drift warnings\n")
  }
} else {
  cat("[WARN] renv.lock not found — package versions not pinned\n")
  cat("[WARN] Run renv::snapshot() to create renv.lock\n")
}

# Key package versions
cat("\nCore package versions:\n")
for (pkg in c("admiral","pharmaversesdtm","pharmaverseadam","rtables","tern","xportr","logrx")) {
  ver <- tryCatch(as.character(packageVersion(pkg)), error = function(e) "not installed")
  cat(sprintf("  %-25s: %s\n", pkg, ver))
}

cat("\nEnvironment check complete. Proceeding with pipeline...\n\n")
```

### Exercise 4

Introduce 3 deliberate bugs into ADSL for the Chapter 13 debugging exercise.

```r
# This script creates a buggy ADSL for Chapter 13 debugging practice.
# SAS equivalent: intentionally introduce bugs and use PROC COMPARE to find them.

library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(dplyr)

dm <- pharmaversesdtm::dm
ds <- pharmaversesdtm::ds
ex <- pharmaversesdtm::ex

cat("=== Creating buggy ADSL for Chapter 13 debugging exercise ===\n\n")

# BUG 1: Month typo in RFXENDTC for one subject
# SAS equivalent: corrupted value in raw data
dm_buggy <- dm %>%
  mutate(
    RFXENDTC = if_else(USUBJID == "01-701-1015",
                       "2014-01-02",   # was "2014-07-02" — month changed 07→01
                       RFXENDTC)
  )
cat("Bug 1 introduced: RFXENDTC month typo for 01-701-1015\n")

# BUG 2: Wrong imputation rule — "first" instead of "last" for end date
# SAS equivalent: INPUT(dtc, YYMMDD10.) with wrong partial date handling

# BUG 3: Wrong age boundary — > instead of >= for age group cutoff
# SAS equivalent: wrong > vs >= in IF-THEN-ELSE

adsl_buggy <- dm_buggy %>%
  filter(ACTARM != "Screen Failure") %>%
  mutate(TRT01P = ACTARM, TRT01A = ACTARM,
         TRT01PN = as.integer(factor(ACTARM, levels=sort(unique(ACTARM)))),
         TRT01AN = TRT01PN) %>%
  derive_vars_dt(new_vars_prefix = "TRTS", dtc = RFXSTDTC, date_imputation = "first") %>%
  # BUG 2: end date should use "last" imputation, not "first"
  derive_vars_dt(new_vars_prefix = "TRTE", dtc = RFXENDTC, date_imputation = "first") %>%
  mutate(
    TRTDURD = as.integer(TRTEDT - TRTSDT + 1),
    # BUG 3: should be AGE >= 65, not AGE > 65
    AGEGR1  = case_when(
      AGE < 40              ~ "<40",
      AGE >= 40 & AGE < 65  ~ "40-64",
      AGE > 65              ~ ">=65",   # BUG: subjects aged exactly 65 get wrong group
      TRUE                  ~ NA_character_
    )
  )

# Save the buggy ADSL
dir.create("output/debug", showWarnings = FALSE)
saveRDS(adsl_buggy, "output/debug/adsl_buggy.rds")

cat("\n3 bugs introduced:\n")
cat("  Bug 1: RFXENDTC month typo → wrong TRTEDT and TRTDURD\n")
cat("  Bug 2: TRTEDT imputation 'first' instead of 'last'\n")
cat("  Bug 3: AGEGR1 age >= 65 boundary uses > instead of >=\n")
cat("\nSaved: output/debug/adsl_buggy.rds\n")
cat("Chapter 13 will find all 3 bugs using diffdf() and browser().\n")
```
