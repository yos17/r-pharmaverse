# Chapter 8: Build the AE Table

*The demographics table is done. The AE table uses the same skeleton. We add it — and learn a cleaner approach for both.*

---

## Where We Left Off

```
pharm001_ch07_t14_1_1.R: Table 14.1.1 (demographics), KM plot
output/t14_1_1.txt, output/f14_2_1_km_os.png
```

The SAP also requires Table 14.3.1: Treatment-Emergent Adverse Events by SOC and Preferred Term. The column structure is identical to the demographics table. We reuse the skeleton.

---

## Reuse the Column Layout

Chapter 7's pattern:

```r
basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total")
```

The AE table uses this exactly. What changes: the row layout. Instead of `analyze_vars("AGE")`, we use `count_occurrences()` nested under `split_rows_by("AEBODSYS")`.

---

## The AE Table Layout

```r
library(rtables)
library(tern)
library(pharmaverseadam)
library(dplyr)

adae_teae <- pharmaverseadam::adae %>%
  filter(SAFFL == "Y", TRTEMFL == "Y")

adsl_safe <- pharmaverseadam::adsl %>%
  filter(SAFFL == "Y")

# Column layout: same as demographics table
lyt_ae <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  
  # Any TEAE row
  count_patients_with_flags(
    var            = "USUBJID",
    flag_variables = "TRTEMFL",
    .labels        = c(TRTEMFL = "Any TEAE, n (%)")
  ) %>%
  
  # Serious AEs row
  count_patients_with_flags(
    var            = "USUBJID",
    flag_variables = "AESER",
    .labels        = c(AESER = "Any Serious TEAE, n (%)")
  ) %>%
  
  # Nested: SOC → PT within SOC
  split_rows_by(
    "AEBODSYS",
    child_labels  = "visible",
    nested        = FALSE,
    label_pos     = "topleft"
  ) %>%
  count_occurrences(
    vars          = "AEDECOD",
    .indent_mods  = c(count_fraction = 1L)
  )

ae_table <- build_table(
  lyt_ae,
  df            = adae_teae,
  alt_counts_df = adsl_safe    # denominators from full safety population
)

ae_table
```

The key detail: `alt_counts_df`. The denominator for percentages should be the safety population (N per arm), not the number of AE records. This is a common source of errors — without `alt_counts_df`, percentages would be wrong.

---

## Sort by Frequency

Regulatory tables sort SOCs by descending subject count:

```r
ae_table_sorted <- ae_table %>%
  sort_at_path(
    path       = c("AEBODSYS"),
    scorefun   = cont_n_allcols,
    decreasing = TRUE
  )
```

---

## The Complete AE Program

```r
# pharm001_ch08_t14_3_1.R
# Study PHARM-001 — Chapter 8
# Table 14.3.1: Treatment-Emergent Adverse Events by SOC/PT
# Safety Population

library(rtables)
library(tern)
library(pharmaverseadam)
library(dplyr)

adae_teae <- pharmaverseadam::adae %>% filter(SAFFL == "Y", TRTEMFL == "Y")
adsl_safe <- pharmaverseadam::adsl %>% filter(SAFFL == "Y")

n_safety <- nrow(adsl_safe)
cat(sprintf("Safety population: %d subjects\n", n_safety))
cat(sprintf("TEAEs: %d events, %d subjects\n",
            nrow(adae_teae), n_distinct(adae_teae$USUBJID)))

lyt <- basic_table(
  title          = "Table 14.3.1: Treatment-Emergent Adverse Events by SOC and Preferred Term",
  subtitles      = "Safety Population",
  show_colcounts = TRUE
) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  
  count_patients_with_flags(
    "USUBJID",
    flag_variables = c("TRTEMFL", "AESER"),
    .labels = c(
      TRTEMFL = "Subjects with any TEAE",
      AESER   = "Subjects with any serious TEAE"
    )
  ) %>%
  
  split_rows_by("AEBODSYS", label_pos = "topleft", child_labels = "visible") %>%
  count_occurrences("AEDECOD", .indent_mods = c(count_fraction = 1L))

ae_tbl <- build_table(lyt, adae_teae, alt_counts_df = adsl_safe) %>%
  sort_at_path(path = c("AEBODSYS"), scorefun = cont_n_allcols, decreasing = TRUE)

print(ae_tbl)

writeLines(export_as_txt(ae_tbl, lpp = 70), "output/t14_3_1.txt")
cat("Saved: output/t14_3_1.txt\n")
```

---

## ARDs: The Cleaner Approach

The `rtables` approach computes statistics and formats them in one step. The ARD approach separates them:

```
ADaM → {cards} → ARD (statistics stored as tidy data) → {tfrmt} → formatted table
```

This matters when you need to reuse statistics across multiple outputs — e.g., the same age mean in both a table and a report.

```r
library(cards)
library(pharmaverseadam)
library(dplyr)

adsl_safe <- pharmaverseadam::adsl %>% filter(SAFFL == "Y")

# Build ARD: one row per statistic per group
ard_age <- ard_continuous(
  data      = adsl_safe,
  variables = AGE,
  by        = TRT01A,
  statistic = ~ continuous_summary_fns(c("N", "mean", "sd", "median", "min", "max"))
)

ard_sex <- ard_categorical(
  data      = adsl_safe,
  variables = SEX,
  by        = TRT01A,
  statistic = ~ categorical_summary_fns(c("n", "p"))
)

# Combine
ard_demo <- bind_ard(ard_age, ard_sex)

# Preview
ard_demo %>%
  filter(group1_level == "Drug A (N=134)", stat_name %in% c("mean","sd")) %>%
  select(variable, stat_name, stat)
```

The ARD is a tidy data frame — one row per statistic. You can inspect it, export it, pass it to `tfrmt` for formatting.

---

## gtsummary: The Quick Path

For exploratory or non-regulatory tables, `gtsummary` takes 5 lines:

```r
library(gtsummary)

adsl_safe %>%
  select(AGE, SEX, RACE, AGEGR1, TRT01A) %>%
  tbl_summary(
    by        = TRT01A,
    statistic = list(
      all_continuous()  ~ "{mean} ({sd})",
      all_categorical() ~ "{n} ({p}%)"
    )
  ) %>%
  add_overall() %>%
  bold_labels()
```

Beautiful output, almost no code. Tradeoff: less control over exact regulatory formatting. Use `rtables` for submission tables, `gtsummary` for reports and slides.

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
output/
  t14_1_1.txt
  t14_3_1.txt
  f14_2_1_km_os.png
```

---

## Exercises

**1.** The AE table currently shows all AEs. Add a filter for `AESEV == "SEVERE"` only, and produce a separate table of severe TEAEs. How does the column layout change? (It doesn't — same skeleton.)

**2.** Use `cards::ard_categorical()` to build an ARD for AE incidence by `AEDECOD` and `TRT01A`. Export it as a CSV. This is the beginning of an Analysis Results Dataset for the submission.

**3.** The `alt_counts_df` argument is critical. To see why: run `build_table()` *without* `alt_counts_df`. What happens to the percentages? Are they correct? Why or why not?

**4. (Sets up Chapter 9)** Try exporting your ADSL to XPT with just `haven::write_xpt(adsl, "adsl_naive.xpt")`. Then read it back with `haven::read_xpt("adsl_naive.xpt")`. Check: do variable labels survive the round trip? Do date columns have SAS date formats? Does the variable order match the ADaM spec? Note every problem you find — Chapter 9 fixes all of them.

---

*Next: Chapter 9 — we export ADSL (and all ADaM) to XPT with full metadata. We'll fix every problem from the exercise above.*
