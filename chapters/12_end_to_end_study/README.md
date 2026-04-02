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
