# Chapter 9: Export ADSL to XPT

*Chapter 8's exercise showed what breaks when you use `haven::write_xpt()` directly. Let's fix it.*

---

## Where We Left Off

```
pharm001_ch05_adsl.R: complete ADSL (all variables derived)
output/t14_1_1.txt, t14_3_1.txt, f14_2_1_km_os.png
```

Chapter 8's exercise: you wrote `haven::write_xpt(adsl, "adsl_naive.xpt")` and read it back. Here's what broke:

1. **Variable labels were lost** — `attr(adsl$USUBJID, "label")` might be set in R, but it doesn't always survive the round-trip correctly
2. **Date columns** — `TRTSDT` (R `Date`) came back as numeric without a SAS `DATE9.` format
3. **Variable lengths** — SAS XPT requires explicit lengths; `haven` picks defaults that may not match your spec
4. **Variable order** — XPT variables came out in R data frame column order, not spec order
5. **Variable types** — some integer columns came back as double

`xportr` fixes all of these.

---

## What xportr Does

`xportr` applies metadata (from a spec) to an R data frame before writing to XPT:

```
ADSL → xportr_type() → xportr_label() → xportr_length() →
       xportr_format() → xportr_order() → xportr_write()
```

Each step applies one aspect of the metadata. Chain them:

```r
library(xportr)
library(pharmaverseadam)
library(dplyr)

adsl <- pharmaverseadam::adsl
```

---

## The Metadata Spec

`xportr` needs a data frame with variable metadata: `dataset`, `variable`, `label`, `type`, `length`, `format`.

In a real project, this comes from a spec spreadsheet via `metacore`. Here we build it programmatically:

```r
var_spec_adsl <- tibble::tribble(
  ~dataset, ~variable,  ~label,                                   ~type,     ~length, ~format,
  "ADSL", "STUDYID",  "Study Identifier",                       "text",    200,     NA,
  "ADSL", "USUBJID",  "Unique Subject Identifier",               "text",    200,     NA,
  "ADSL", "SUBJID",   "Subject Identifier for the Study",        "text",    40,      NA,
  "ADSL", "SITEID",   "Study Site Identifier",                   "text",    20,      NA,
  "ADSL", "AGE",      "Age",                                     "float",   8,       NA,
  "ADSL", "AGEU",     "Age Units",                               "text",    8,       NA,
  "ADSL", "AGEGR1",   "Pooled Age Group 1",                     "text",    20,      NA,
  "ADSL", "AGEGR1N",  "Pooled Age Group 1 (N)",                 "integer", 8,       NA,
  "ADSL", "SEX",      "Sex",                                     "text",    2,       NA,
  "ADSL", "RACE",     "Race",                                    "text",    200,     NA,
  "ADSL", "ETHNIC",   "Ethnicity",                               "text",    200,     NA,
  "ADSL", "TRT01P",   "Planned Treatment for Period 01",         "text",    200,     NA,
  "ADSL", "TRT01PN",  "Planned Treatment for Period 01 (N)",     "integer", 8,       NA,
  "ADSL", "TRT01A",   "Actual Treatment for Period 01",          "text",    200,     NA,
  "ADSL", "TRT01AN",  "Actual Treatment for Period 01 (N)",      "integer", 8,       NA,
  "ADSL", "TRTSDT",   "Date of First Exposure to Treatment",     "integer", 8,       "DATE9.",
  "ADSL", "TRTEDT",   "Date of Last Exposure to Treatment",      "integer", 8,       "DATE9.",
  "ADSL", "TRTDURD",  "Total Treatment Duration (Days)",         "integer", 8,       NA,
  "ADSL", "ITTFL",    "Intent-To-Treat Population Flag",         "text",    2,       NA,
  "ADSL", "SAFFL",    "Safety Population Flag",                  "text",    2,       NA,
  "ADSL", "PPROTFL",  "Per-Protocol Population Flag",            "text",    2,       NA,
  "ADSL", "COMPLFL",  "Completers Population Flag",              "text",    2,       NA,
  "ADSL", "DTHFL",    "Subject Death Flag",                      "text",    2,       NA,
  "ADSL", "DTHDT",    "Date of Death",                           "integer", 8,       "DATE9.",
)
```

---

## The xportr Pipeline

```r
dir.create("output/adam", recursive = TRUE, showWarnings = FALSE)

adsl_xpt <- adsl %>%
  xportr_type(metadata   = var_spec_adsl, domain = "ADSL") %>%
  xportr_label(metadata  = var_spec_adsl, domain = "ADSL") %>%
  xportr_length(metadata = var_spec_adsl, domain = "ADSL",
                length_source = "metadata") %>%
  xportr_format(metadata = var_spec_adsl, domain = "ADSL") %>%
  xportr_order(metadata  = var_spec_adsl, domain = "ADSL") %>%
  xportr_write("output/adam/adsl.xpt",
               label = "Subject-Level Analysis Dataset")
```

Now verify:

```r
# Read back and check
adsl_check <- haven::read_xpt("output/adam/adsl.xpt")

cat(sprintf("Rows: %d  Cols: %d\n", nrow(adsl_check), ncol(adsl_check)))

# Labels survived?
attr(adsl_check$USUBJID, "label")   # "Unique Subject Identifier"
attr(adsl_check$TRTSDT,  "label")   # "Date of First Exposure to Treatment"

# Date format?
attr(adsl_check$TRTSDT, "format.sas")   # "DATE9."
```

---

## Validation Before Export

Before writing the XPT, run these checks:

```r
# SAS variable name rules: ≤8 chars, uppercase, alphanumeric + underscore, starts with letter
check_varnames <- function(df) {
  bad <- names(df)[!grepl("^[A-Z][A-Z0-9_]{0,7}$", names(df))]
  if (length(bad) > 0)
    cat("Non-conformant names:", paste(bad, collapse=", "), "\n")
  else
    cat("All variable names SAS-conformant\n")
}

check_varnames(adsl)

# Character length vs. spec
for (var in var_spec_adsl$variable) {
  if (var %in% names(adsl)) {
    spec_row  <- var_spec_adsl[var_spec_adsl$variable == var, ]
    if (spec_row$type == "text") {
      max_len <- max(nchar(as.character(adsl[[var]]), type = "bytes"), na.rm = TRUE)
      if (max_len > spec_row$length)
        cat(sprintf("WARNING: %s max=%d > spec=%d\n", var, max_len, spec_row$length))
    }
  }
}
```

---

## The Complete Export Program

```r
# pharm001_ch09_export.R
# Study PHARM-001 — Chapter 9
# Export all ADaM datasets to XPT with metadata

library(xportr)
library(pharmaverseadam)
library(dplyr)
library(haven)

dir.create("output/adam", recursive = TRUE, showWarnings = FALSE)

# Define datasets and labels
datasets <- list(
  ADSL  = pharmaverseadam::adsl,
  ADAE  = pharmaverseadam::adae,
  ADTTE = pharmaverseadam::adtte,
  ADLB  = pharmaverseadam::adlb
)

ds_labels <- c(
  ADSL  = "Subject-Level Analysis Dataset",
  ADAE  = "Adverse Events Analysis Dataset",
  ADTTE = "Time-to-Event Analysis Dataset",
  ADLB  = "Laboratory Test Results Analysis Dataset"
)

cat("=== PHARM-001 ADaM XPT Export ===\n")

for (ds_name in names(datasets)) {
  df   <- datasets[[ds_name]]
  path <- file.path("output/adam", paste0(tolower(ds_name), ".xpt"))
  
  # Write (in production: apply full xportr pipeline with spec)
  df %>%
    xportr_write(path = path, label = ds_labels[ds_name])
  
  # Verify round-trip
  written <- haven::read_xpt(path)
  size_kb  <- round(file.info(path)$size / 1024, 1)
  
  cat(sprintf("  %-8s: %4d rows × %2d cols  %6.1f KB  → %s\n",
              ds_name, nrow(written), ncol(written), size_kb,
              basename(path)))
}

# Check ADSL key variables
adsl_back <- haven::read_xpt("output/adam/adsl.xpt")
cat("\nADSL variable label check:\n")
for (v in c("USUBJID", "AGE", "TRTSDT", "TRT01P", "SAFFL")) {
  if (v %in% names(adsl_back)) {
    lbl <- attr(adsl_back[[v]], "label")
    cat(sprintf("  %-12s: %s\n", v, if (!is.null(lbl)) lbl else "(no label)"))
  }
}

cat("\nExport complete.\n")
```

---

## Submission Folder Structure

```
pharm001/
├── programs/
│   ├── sdtm/        dm.R, ae.R, ex.R, ...
│   ├── adam/        adsl.R, adae.R, adtte.R, adlb.R
│   └── tlg/         t_14_1_1.R, t_14_3_1.R, f_14_2_1.R, ...
├── output/
│   ├── adam/        adsl.xpt, adae.xpt, adtte.xpt, adlb.xpt
│   └── tlg/         t14_1_1.txt, t14_3_1.txt, f14_2_1.png
├── logs/            one .log per program (Chapter 11)
├── specs/           variable spec, define.xml
└── renv.lock        package versions
```

---

## What We Have Now

```
pharm001_ch01.R          — load DM, count subjects, demographics
pharm001_ch02_validate.R — validate SDTM
pharm001_ch03_dm.R       — map raw DM to SDTM
pharm001_ch04.R          — derive TRTSDT, TRTEDT, TRTDURD
pharm001_ch05_adsl.R     — complete ADSL
pharm001_ch06_adae.R     — complete ADAE
pharm001_ch07_t14_1_1.R  — Table 14.1.1 (demographics) + KM plot
pharm001_ch08_t14_3_1.R  — Table 14.3.1 (TEAE by SOC/PT)
pharm001_ch09_export.R   — export all ADaM to XPT
output/adam/
  adsl.xpt, adae.xpt, adtte.xpt, adlb.xpt
```

---

## Exercises

**1.** Apply the complete xportr pipeline (type → label → length → format → order → write) to ADAE. Define at minimum 10 variables in a `var_spec_adae` spec.

**2.** Write `xpt_roundtrip_check(df, path)` that writes a dataset, reads it back, and compares row counts, column counts, and all values. Where can the round-trip fail? Try it on ADSL.

**3.** The SAS variable name check: write a function that flags any variable name that is more than 8 characters, contains lowercase, starts with a number, or contains special characters.

**4. (Sets up Chapter 10)** The teal app in Chapter 10 needs your ADaM data. Load back `output/adam/adsl.xpt` with `haven::read_xpt()`. Does it work in a basic teal `teal_data()` call? Are the variable types correct for `split_cols_by("TRT01A")`? Fix any issues you find before proceeding.

---

*Next: Chapter 10 — we build an interactive teal app for PHARM-001, starting with one module.*
