# Chapter 7: Tables, Listings & Graphs with {rtables} and {tern}

*Build clinical submission tables: demographics, AEs, lab summaries.*

---

## The TLG Stack

In SAS you use `PROC REPORT`, `PROC TABULATE`, and custom macros to produce tables. The pharmaverse equivalent:

| Role | Package |
|------|---------|
| Table structure/layout | `rtables` |
| Clinical analysis functions | `tern` |
| Listings | `rlistings` |
| Summary tables (simpler) | `gtsummary` |
| Graphs | `ggplot2`, `ggsurvfit`, `visR` |

---

## rtables Concepts

`rtables` builds a table by defining a **layout** (the structure) and then **analyzing** data against it.

```r
library(rtables)
library(tern)
library(pharmaverseadam)

adsl <- pharmaverseadam::adsl
adsl_safe <- adsl %>% filter(SAFFL == "Y")
```

### Basic rtables

```r
# Simplest table: count by arm
lyt <- basic_table() %>%
  split_cols_by("TRT01A") %>%
  add_colcounts() %>%
  count_values("SEX", values = "F", .labels = c(count_fraction = "Female, n (%)"))

tbl <- build_table(lyt, adsl_safe)
tbl
```

Output:
```
                      Drug A (N=86)   Drug B (N=93)   Placebo (N=86)
——————————————————————————————————————————————————————————————————
Female, n (%)         43 (50%)        48 (51.6%)      42 (48.8%)
```

---

## Demographics Table (Table 14.1.1)

The standard demographics table: continuous (mean/SD/median/range) and categorical (n/%) by treatment arm. This is the most common table you'll build.

```r
library(rtables)
library(tern)
library(dplyr)
library(pharmaverseadam)

adsl      <- pharmaverseadam::adsl
adsl_safe <- adsl %>% filter(SAFFL == "Y")

# Analysis variables
cat_vars  <- c("SEX", "AGEGR1", "RACE", "ETHNIC")
cont_vars <- c("AGE", "BMIBL", "HEIGHTBL", "WEIGHTBL")

lyt <- basic_table(show_colcounts = TRUE) %>%
  
  # Column split: one column per treatment arm + Total
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  
  # ---- Continuous variables ----
  # Age: Mean (SD), Median, Range
  analyze_vars(
    vars    = "AGE",
    .stats  = c("mean_sd", "median", "range"),
    .labels = c(
      mean_sd = "Mean (SD)",
      median  = "Median",
      range   = "Min, Max"
    )
  ) %>%
  
  # ---- Categorical: Sex ----
  count_occurrences_by_grade(
    var    = "SEX",
    .stats = "count_fraction"
  ) %>%
  
  # ---- Categorical: Age group ----
  count_occurrences_by_grade(
    var    = "AGEGR1",
    .stats = "count_fraction"
  ) %>%
  
  # ---- Categorical: Race ----
  count_occurrences_by_grade(
    var    = "RACE",
    .stats = "count_fraction"
  )

demo_table <- build_table(lyt, adsl_safe)

# Print
demo_table

# Export to text (for RTF output see Chapter 8)
cat(export_as_txt(demo_table, lpp = 50))
```

---

## AE Incidence Table

Standard AE table: subjects with at least one event, by SOC and PT, sorted by descending frequency.

```r
library(rtables)
library(tern)
library(pharmaverseadam)

adae  <- pharmaverseadam::adae
adsl  <- pharmaverseadam::adsl

# Safety population, treatment-emergent AEs only
adae_safe <- adae %>%
  filter(SAFFL == "Y", TRTEMFL == "Y")

# Full ADSL safety population (denominator)
adsl_safe <- adsl %>% filter(SAFFL == "Y")

# ---- AE table layout ----
lyt_ae <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  
  # Overall any AE row
  count_patients_with_flags(
    var    = "USUBJID",
    flag_variables = "TRTEMFL",
    .labels = c(TRTEMFL = "Any TEAE")
  ) %>%
  
  # Serious AEs row
  count_patients_with_flags(
    var            = "USUBJID",
    flag_variables = "AESER",
    .labels        = c(AESER = "Any Serious TEAE")
  ) %>%
  
  # By SOC, then PT within SOC
  split_rows_by(
    "AEBODSYS",
    child_labels = "visible",
    nested       = FALSE,
    label_pos    = "topleft"
  ) %>%
  
  count_occurrences(
    vars       = "AEDECOD",
    .indent_mods = c(count_fraction = 1L)
  )

ae_table <- build_table(
  lyt_ae,
  df     = adae_safe,
  alt_counts_df = adsl_safe   # use full ADSL for denominators
)

ae_table
```

---

## Listings with rlistings

Listings are simpler — no statistics, just formatted rows:

```r
library(rlistings)
library(pharmaverseadam)

adae_safe <- pharmaverseadam::adae %>%
  filter(SAFFL == "Y", AESER == "Y") %>%
  select(USUBJID, AEDECOD, AEBODSYS, AESEV, AESTDTC, AEENDTC, AEOUT) %>%
  arrange(USUBJID, AESTDTC)

lst <- as_listing(
  adae_safe,
  key_cols   = c("USUBJID", "AEBODSYS"),
  disp_cols  = c("AEDECOD", "AESEV", "AESTDTC", "AEENDTC", "AEOUT"),
  main_title = "Listing of Serious Adverse Events",
  subtitles  = c("Safety Population", "Treatment-Emergent"),
  main_footer = "Note: Sorted by USUBJID and AE start date."
)

lst
cat(export_as_txt(lst, lpp = 60))
```

---

## Graphs: KM Curve

```r
library(ggsurvfit)
library(survival)
library(dplyr)

adtte <- pharmaverseadam::adtte

# Overall survival
os_data <- adtte %>%
  filter(PARAMCD == "OS") %>%
  mutate(
    AVAL_MONTHS = AVAL / 30.4375,   # days to months
    ARM = factor(TRT01A)
  )

km_fit <- survfit(Surv(AVAL_MONTHS, 1 - CNSR) ~ ARM, data = os_data)

ggsurvfit(km_fit, linewidth = 1) +
  add_confidence_interval() +
  add_risktable(risktable_stats = "n.risk") +
  add_pvalue(caption = "Log-rank {p.value}") +
  scale_ggsurvfit() +
  labs(
    title    = "Figure 14.2.1: Kaplan-Meier Plot of Overall Survival",
    subtitle = "Intent-to-Treat Population",
    x        = "Time (Months)",
    y        = "Overall Survival Probability"
  ) +
  theme_classic()
```

---

## Program: Complete TLG Suite

```r
# tlg_suite.R
# Produce Table 14.1.1 (demographics), Table 14.3.1 (AE overview),
# and Figure 14.2.1 (OS KM) as text output

library(rtables)
library(tern)
library(rlistings)
library(ggsurvfit)
library(survival)
library(pharmaverseadam)
library(dplyr)

adsl  <- pharmaverseadam::adsl
adae  <- pharmaverseadam::adae
adtte <- pharmaverseadam::adtte

# Populations
adsl_safe <- adsl %>% filter(SAFFL == "Y")
adae_teae <- adae %>% filter(SAFFL=="Y", TRTEMFL=="Y")

# ---- Table 14.1.1: Demographics ----
cat("\n", strrep("=", 60), "\n")
cat("TABLE 14.1.1 — SUMMARY OF DEMOGRAPHIC CHARACTERISTICS\n")
cat("SAFETY POPULATION\n")
cat(strrep("=", 60), "\n")

lyt_demo <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  analyze_vars("AGE", .stats = c("n","mean_sd","median","range")) %>%
  count_occurrences_by_grade("SEX") %>%
  count_occurrences_by_grade("AGEGR1") %>%
  count_occurrences_by_grade("RACE")

build_table(lyt_demo, adsl_safe) %>% print()

# ---- Table 14.3.1: AE Summary ----
cat("\n", strrep("=", 60), "\n")
cat("TABLE 14.3.1 — OVERVIEW OF ADVERSE EVENTS\n")
cat("SAFETY POPULATION\n")
cat(strrep("=", 60), "\n")

lyt_ae_overview <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  count_patients_with_flags(
    "USUBJID",
    flag_variables = c("TRTEMFL","AESER"),
    .labels = c(
      TRTEMFL = "Subjects with any TEAE",
      AESER   = "Subjects with any Serious TEAE"
    )
  )

build_table(lyt_ae_overview, adae_teae, alt_counts_df = adsl_safe) %>% print()

# ---- AE by PT ----
cat("\n--- By System Organ Class and Preferred Term ---\n")

lyt_ae_pt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  split_rows_by("AEBODSYS", label_pos = "topleft") %>%
  count_occurrences("AEDECOD")

build_table(lyt_ae_pt, adae_teae, alt_counts_df = adsl_safe) %>%
  sort_at_path(
    path     = c("AEBODSYS"),
    scorefun = cont_n_allcols,
    decreasing = TRUE
  ) %>%
  print()
```

---

## Exercises

**1. p-values in demographics**

Add p-values to the demographics table using `tern::test_summary_vector`. Use chi-square for categorical and t-test/ANOVA for continuous variables.

**2. Forest plot**

Using `tern::g_forest()` or `ggplot2`, produce a forest plot of odds ratios for any AE by subgroup (SEX, AGEGR1, RACE).

**3. Lab shift table**

Using `adlb` (lab analysis dataset), produce a shift table: baseline NCI CTCAE grade (rows) × worst post-baseline grade (columns), for each treatment arm.

**4. Volcano plot**

For the AE data: produce a volcano plot where x = risk difference (Drug vs Placebo) and y = -log10(p-value from Fisher's exact test). Use `ggplot2`.

---

*Next: Chapter 8 — ARDs and Formatting with {cards} and {tfrmt}*
