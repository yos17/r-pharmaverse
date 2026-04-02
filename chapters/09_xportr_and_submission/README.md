# Chapter 9: Submission Packages with {xportr}

*Apply SAS XPT metadata and export submission-ready transport files.*

---

## What xportr Does

When you submit to FDA/EMA, your ADaM and SDTM datasets must be:
- In SAS Version 5 XPT format (`.xpt`)
- With correct variable types (numeric/character)
- With variable labels matching the define.xml
- With variable lengths set (not left at R's default)
- With numeric formats applied (e.g., date variables as SAS date integers)
- Variables in the correct order

`xportr` automates all of this. It reads metadata from a `metacore` object (your spec) and applies it before writing the XPT.

```r
library(xportr)
library(metacore)
library(metatools)
library(pharmaverseadam)
library(dplyr)
library(haven)
```

---

## The Metadata Spec

`xportr` needs variable metadata: type, label, length, format, order. In a real project this comes from your Define-XML or spec spreadsheet. Here we'll build it programmatically.

```r
# Build a metacore spec for ADSL
adsl_meta <- metacore::spec_to_metacore(
  path            = "specs/ADaM_spec.xlsx",
  where_sep_sheet = FALSE
)

# Or build it manually:
var_spec_adsl <- tibble::tribble(
  ~dataset, ~variable, ~label,                                  ~type,     ~length, ~format,
  "ADSL", "STUDYID",  "Study Identifier",                      "text",    200,     NA,
  "ADSL", "USUBJID",  "Unique Subject Identifier",              "text",    200,     NA,
  "ADSL", "SUBJID",   "Subject Identifier for the Study",       "text",    40,      NA,
  "ADSL", "SITEID",   "Study Site Identifier",                  "text",    20,      NA,
  "ADSL", "AGE",      "Age",                                    "float",   8,       NA,
  "ADSL", "AGEU",     "Age Units",                              "text",    8,       NA,
  "ADSL", "AGEGR1",   "Pooled Age Group 1",                    "text",    20,      NA,
  "ADSL", "AGEGR1N",  "Pooled Age Group 1 (N)",                "integer", 8,       NA,
  "ADSL", "SEX",      "Sex",                                    "text",    2,       NA,
  "ADSL", "RACE",     "Race",                                   "text",    200,     NA,
  "ADSL", "ETHNIC",   "Ethnicity",                              "text",    200,     NA,
  "ADSL", "TRT01P",   "Planned Treatment for Period 01",        "text",    200,     NA,
  "ADSL", "TRT01PN",  "Planned Treatment for Period 01 (N)",    "integer", 8,       NA,
  "ADSL", "TRT01A",   "Actual Treatment for Period 01",         "text",    200,     NA,
  "ADSL", "TRT01AN",  "Actual Treatment for Period 01 (N)",     "integer", 8,       NA,
  "ADSL", "TRTSDT",   "Date of First Exposure to Treatment",    "integer", 8,       "DATE9.",
  "ADSL", "TRTEDT",   "Date of Last Exposure to Treatment",     "integer", 8,       "DATE9.",
  "ADSL", "TRTDURD",  "Total Treatment Duration (Days)",        "integer", 8,       NA,
  "ADSL", "ITTFL",    "Intent-To-Treat Population Flag",        "text",    2,       NA,
  "ADSL", "SAFFL",    "Safety Population Flag",                 "text",    2,       NA,
  "ADSL", "PPROTFL",  "Per-Protocol Population Flag",           "text",    2,       NA,
  "ADSL", "COMPLFL",  "Completers Population Flag",             "text",    2,       NA,
  "ADSL", "DTHFL",    "Subject Death Flag",                     "text",    2,       NA,
  "ADSL", "DTHDT",    "Date of Death",                          "integer", 8,       "DATE9.",
)
```

---

## The xportr Pipeline

Apply metadata step by step, then export:

```r
adsl <- pharmaverseadam::adsl

adsl_xpt <- adsl %>%
  
  # 1. Variable type coercion (numeric/character per spec)
  xportr_type(metadata = var_spec_adsl, domain = "ADSL") %>%
  
  # 2. Variable labels
  xportr_label(metadata = var_spec_adsl, domain = "ADSL") %>%
  
  # 3. Variable lengths (SAS XPT requires explicit lengths)
  xportr_length(metadata = var_spec_adsl, domain = "ADSL",
                length_source = "metadata") %>%
  
  # 4. Format (SAS date format → DATE9.)
  xportr_format(metadata = var_spec_adsl, domain = "ADSL") %>%
  
  # 5. Variable order per spec
  xportr_order(metadata = var_spec_adsl, domain = "ADSL") %>%
  
  # 6. Drop variables not in spec (optional — depends on policy)
  # xportr_drop_variables(metadata = var_spec_adsl, domain = "ADSL") %>%
  
  # 7. Write XPT
  xportr_write("adsl.xpt", label = "Subject-Level Analysis Dataset")
```

---

## Validation Checks

Before exporting, validate the dataset:

```r
# Check variable names conform to SAS (≤8 chars, uppercase, no special chars)
bad_names <- names(adsl)[!grepl("^[A-Z][A-Z0-9]{0,7}$", names(adsl))]
if (length(bad_names) > 0)
  cat("Non-conformant variable names:", paste(bad_names, collapse=", "), "\n")

# Check character lengths don't exceed spec
for (var in var_spec_adsl$variable) {
  if (var %in% names(adsl)) {
    spec_len  <- var_spec_adsl$length[var_spec_adsl$variable == var]
    spec_type <- var_spec_adsl$type[var_spec_adsl$variable == var]
    if (spec_type == "text") {
      max_len <- max(nchar(as.character(adsl[[var]]), type="bytes"), na.rm=TRUE)
      if (max_len > spec_len)
        cat(sprintf("WARNING: %s max length %d exceeds spec %d\n",
                    var, max_len, spec_len))
    }
  }
}

# Verify XPT was written correctly
adsl_check <- haven::read_xpt("adsl.xpt")
cat(sprintf("Written XPT: %d rows × %d cols\n", nrow(adsl_check), ncol(adsl_check)))
cat("Labels check:\n")
for (v in c("USUBJID","AGE","TRTSDT","TRT01P")) {
  if (v %in% names(adsl_check))
    cat(sprintf("  %s: '%s'\n", v, attr(adsl_check[[v]], "label")))
}
```

---

## Submission Folder Structure

```
submission/
├── adam/
│   ├── adsl.xpt
│   ├── adae.xpt
│   ├── adtte.xpt
│   ├── adlb.xpt
│   └── define.xml
├── sdtm/
│   ├── dm.xpt
│   ├── ae.xpt
│   ├── vs.xpt
│   ├── ex.xpt
│   └── define.xml
├── analysis/
│   ├── programs/          # your R scripts
│   ├── output/            # tables, listings, figures
│   └── validation/        # QC programs
└── define2-0-0.xsl
```

---

## Program: Export All ADaM Datasets

```r
# export_adam.R
# Export all ADaM datasets to XPT with metadata

library(xportr)
library(pharmaverseadam)
library(dplyr)

# Create output directory
dir.create("submission/adam", recursive = TRUE, showWarnings = FALSE)

# Define datasets to export
datasets <- list(
  ADSL  = pharmaverseadam::adsl,
  ADAE  = pharmaverseadam::adae,
  ADTTE = pharmaverseadam::adtte,
  ADLB  = pharmaverseadam::adlb
)

# Basic metadata for all datasets (in practice: from spec)
# Here we just demonstrate the xportr_write step with minimal metadata
for (ds_name in names(datasets)) {
  df   <- datasets[[ds_name]]
  path <- file.path("submission/adam", paste0(tolower(ds_name), ".xpt"))
  
  # Apply labels from column attributes if they exist
  df %>%
    xportr_write(
      path  = path,
      label = switch(ds_name,
        ADSL  = "Subject-Level Analysis Dataset",
        ADAE  = "Adverse Events Analysis Dataset",
        ADTTE = "Time-to-Event Analysis Dataset",
        ADLB  = "Laboratory Test Results Analysis Dataset"
      )
    )
  
  # Verify
  written <- haven::read_xpt(path)
  cat(sprintf("%-8s: %4d rows × %2d cols  → %s\n",
              ds_name, nrow(written), ncol(written), path))
}

cat("\nExport complete.\n")
cat("XPT file sizes:\n")
xpt_files <- list.files("submission/adam", pattern = "*.xpt", full.names = TRUE)
for (f in xpt_files) {
  size <- file.info(f)$size
  cat(sprintf("  %-30s  %s KB\n", basename(f), formatC(size/1024, digits=1, format="f")))
}
```

---

## define.xml

A proper submission needs a `define.xml` — the machine-readable metadata for the XPT files. Generating define.xml is complex. Tools:

- **`pharmaverse/defineR`** — in development
- **Pinnacle 21 Community** — free validation + define.xml generator
- **Medidata Rave** / company SAS macros — often used for the final define.xml

For now, document your variables in the spec spreadsheet and let the define.xml tooling pick it up.

---

## Exercises

**1. xportr full pipeline**

Apply the complete xportr pipeline (type → label → length → format → order → write) to ADAE. Define at minimum 10 variables in your metadata spec.

**2. XPT round-trip test**

Write a function `xpt_roundtrip_check(df, path)` that:
1. Writes `df` to `path`
2. Reads it back
3. Compares row counts, column counts, and values
4. Reports any discrepancies

Where can round-trips fail? (Hint: date types, character encoding)

**3. SAS variable name check**

Write a function that flags any variable name that would be invalid in SAS:
- More than 8 characters
- Contains lowercase letters
- Starts with a number
- Contains special characters

**4. Submission checklist**

Write a `submission_check(xpt_path, spec)` function that validates an XPT file against a spec, checking: all required variables present, types match, lengths match, labels match.

---

*Next: Chapter 10 — Interactive Apps with {teal}: clinical data exploration*
