# Chapter 7: Build the Demographics Table

*The SAP requires Table 14.1.1: Summary of Demographic Characteristics. We have the ADSL. Let's build the table — one row at a time.*

---

## Where We Left Off

```
pharm001_ch05_adsl.R: ADSL with AGE, AGEGR1, SEX, RACE, TRT01A, SAFFL
```

That's everything Table 14.1.1 needs. Now we learn `rtables` + `tern` to produce it.

---

## The SAS Mental Model

In SAS, you'd use `PROC REPORT` or `PROC TABULATE`. In R, the pharmaverse approach:

| SAS | R |
|-----|---|
| `PROC REPORT` | `rtables` (layout engine) |
| Clinical analysis macros | `tern` (analysis functions) |
| Listings | `rlistings` |
| Graphs | `ggplot2`, `ggsurvfit` |

`rtables` separates **layout** (what the table looks like) from **analysis** (what statistics to compute). Build the layout, then `build_table()` fills it with data.

---

## The Biggest Mental Shift: PROC REPORT vs. rtables

This is the conceptual leap from SAS to R for tables. Let's see both side by side.

**In SAS (PROC REPORT):**
```sas
PROC REPORT DATA=adsl_safe NOWD;
  COLUMN actarm, (age n mean std);
  DEFINE actarm / ACROSS "Treatment Arm";
  DEFINE age    / ANALYSIS;
  DEFINE n      / "N" FORMAT=8.;
  DEFINE mean   / "Mean" FORMAT=8.1;
  DEFINE std    / "SD"   FORMAT=8.2;
RUN;
```

**In R (pharmaverse — rtables/tern):**
```r
library(rtables)
library(tern)

lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%          # columns = treatment arms (like ACROSS in PROC REPORT)
  analyze_vars("AGE", .stats = c("n", "mean_sd"))  # rows = statistics

tbl <- build_table(lyt, adsl_safe)
tbl
```

The key difference: in `rtables`, you **declare the layout** and then `build_table()` fills it with data — similar to how `PROC REPORT` separates column definitions from the rendering step.

---

## Start With One Row: Age

Don't build the whole table first. Build one row. Make it work. Then add rows.

```r
library(rtables)
library(tern)
library(pharmaverseadam)
library(dplyr)

adsl_safe <- pharmaverseadam::adsl %>% filter(SAFFL == "Y")

# Simplest possible table: age statistics by treatment arm
lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  analyze_vars("AGE", .stats = c("mean_sd"))

tbl <- build_table(lyt, adsl_safe)
tbl
```

Output:
```
                Drug A (N=134)  Drug B (N=134)  Placebo (N=134)
AGE
  Mean (SD)    58.7 (8.2)      55.6 (9.2)      57.9 (8.9)
```

One row. It works. Now add more statistics:

```r
lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  analyze_vars("AGE", .stats = c("n", "mean_sd", "median", "range"))

tbl <- build_table(lyt, adsl_safe)
tbl
```

---

## Statistics Mapping: PROC MEANS vs. tern

**In SAS (PROC MEANS):**
```sas
PROC MEANS DATA=adsl_safe N MEAN STD MEDIAN MIN MAX;
  CLASS TRT01A;
  VAR AGE;
  OUTPUT OUT=age_stats N=n MEAN=mean STD=std MEDIAN=median MIN=min MAX=max;
RUN;

ODS RTF FILE="output/t14_1_1.rtf";
PROC REPORT DATA=age_stats NOWD;
  /* ... format for submission ... */
RUN;
ODS RTF CLOSE;
```

**In R (pharmaverse — tern):**
```r
lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  analyze_vars("AGE",
    .stats  = c("n", "mean_sd", "median", "range"),
    .labels = c(n="n", mean_sd="Mean (SD)", median="Median", range="Min, Max")
  )
```

| SAS (`PROC MEANS`) | R (`tern`) |
|---------------------|-----------|
| `N` | `.stats = "n"` |
| `MEAN` / `STD` | `.stats = "mean_sd"` |
| `MEDIAN` | `.stats = "median"` |
| `MIN` / `MAX` | `.stats = "range"` |
| `Q1` / `Q3` | `.stats = "quantiles"` |

---

## Add a Total Column

**In SAS:**
```sas
/* In PROC REPORT: add an ALL column */
PROC REPORT DATA=adsl_safe;
  COLUMN actarm all age ...;
  DEFINE all / COMPUTED "Total";
  /* OR use PROC TABULATE with ALL option */
RUN;
```

**In R (pharmaverse):**
```r
lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  analyze_vars("AGE", .stats = c("n", "mean_sd", "median", "range"))
```

---

## Categorical Variables: PROC FREQ vs. count_occurrences

**In SAS (PROC FREQ):**
```sas
PROC FREQ DATA=adsl_safe;
  TABLES TRT01A * SEX / NOCUM NOPERCENT;
  /* For denominator: DENOM= option or manual calculation */
RUN;
```

**In R (pharmaverse — tern):**
```r
lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  count_occurrences_by_grade("SEX",
    .labels = c(count_fraction="n (%)")
  )
```

---

## Add Age Group

```r
lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  analyze_vars("AGE", .stats = c("n", "mean_sd", "median", "range")) %>%
  count_occurrences_by_grade(
    var    = "AGEGR1",
    .stats = "count_fraction"
  )
```

---

## Add Sex and Race

Keep adding rows:

```r
lyt <- basic_table(show_colcounts = TRUE) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  analyze_vars("AGE", .stats = c("n", "mean_sd", "median", "range"),
               .labels = c(n="n", mean_sd="Mean (SD)",
                           median="Median", range="Min, Max")) %>%
  count_occurrences_by_grade("AGEGR1",
    .labels = c(count_fraction="Age Group, n (%)")) %>%
  count_occurrences_by_grade("SEX",
    .labels = c(count_fraction="Sex, n (%)")) %>%
  count_occurrences_by_grade("RACE",
    .labels = c(count_fraction="Race, n (%)"))

demo_table <- build_table(lyt, adsl_safe)
demo_table
```

---

## Exporting Output

**In SAS:**
```sas
ODS RTF FILE="output/t14_1_1.rtf";
PROC REPORT DATA=... NOWD;
  /* ... */
RUN;
ODS RTF CLOSE;
```

**In R (pharmaverse):**
```r
# Export to text (regulatory listings format)
out_txt <- export_as_txt(demo_table, lpp = 60)
writeLines(out_txt, "output/t14_1_1.txt")

# Export to RTF (via r2rtf package)
# library(r2rtf)
# ... see r2rtf vignette for RTF export
```

---

## Table 14.1.1 Complete

```r
# pharm001_ch07_t14_1_1.R
# Study PHARM-001 — Chapter 7
# Table 14.1.1: Summary of Demographic Characteristics
# Safety Population

library(rtables)
library(tern)
library(pharmaverseadam)
library(dplyr)

adsl_safe <- pharmaverseadam::adsl %>% filter(SAFFL == "Y")

cat(sprintf("Safety population: %d subjects\n", nrow(adsl_safe)))
cat(sprintf("Arms: %s\n", paste(sort(unique(adsl_safe$TRT01A)), collapse=", ")))

lyt <- basic_table(
  title          = "Table 14.1.1: Summary of Demographic Characteristics",
  subtitles      = "Safety Population",
  show_colcounts = TRUE
) %>%
  split_cols_by("TRT01A") %>%
  add_overall_col("Total") %>%
  
  # Age — continuous
  analyze_vars(
    "AGE",
    .stats  = c("n", "mean_sd", "median", "range", "quantiles"),
    .labels = c(n="n", mean_sd="Mean (SD)", median="Median",
                range="Min, Max", quantiles="Q1, Q3")
  ) %>%
  
  # Age group — categorical
  count_occurrences_by_grade(
    "AGEGR1",
    .labels = c(count_fraction="n (%)")
  ) %>%
  
  # Sex
  count_occurrences_by_grade(
    "SEX",
    .labels = c(count_fraction="n (%)")
  ) %>%
  
  # Race
  count_occurrences_by_grade(
    "RACE",
    .labels = c(count_fraction="n (%)")
  )

demo_table <- build_table(lyt, adsl_safe)
print(demo_table)

# Export to text
out_txt <- export_as_txt(demo_table, lpp = 60)
writeLines(out_txt, "output/t14_1_1.txt")
cat("\nSaved: output/t14_1_1.txt\n")
```

---

## Nested Rows: PROC TABULATE vs. split_rows_by

**In SAS (PROC TABULATE):**
```sas
PROC TABULATE DATA=adae_teae;
  CLASS TRT01A AEBODSYS AEDECOD;
  TABLE AEBODSYS * AEDECOD,
        TRT01A * (N PCTN<TRT01A>) / BOX="System Organ Class/Preferred Term";
RUN;
```

**In R (pharmaverse — rtables/tern):**
```r
lyt_ae <- basic_table() %>%
  split_cols_by("TRT01A") %>%
  split_rows_by("AEBODSYS",              # rows split by SOC (like AEBODSYS row in PROC TABULATE)
    child_labels = "visible",
    nested       = FALSE) %>%
  count_occurrences("AEDECOD")           # leaf rows = preferred terms
```

---

## Denominator Control

**In SAS:**
```sas
/* DENOM= option controls the denominator */
PROC FREQ DATA=adae_teae;
  TABLES TRT01A * AEDECOD / NOCUM;
  WEIGHT / DENOM = adsl_safe_n;  /* % based on safety population */
RUN;
```

**In R (pharmaverse):**
```r
# alt_counts_df provides denominators from full safety population
ae_table <- build_table(
  lyt_ae,
  df            = adae_teae,
  alt_counts_df = adsl_safe    # denominators from full safety population
)
```

---

## Listings with rlistings

Listings are simpler — no statistics, just formatted rows:

```r
library(rlistings)
library(pharmaverseadam)

adae_serious <- pharmaverseadam::adae %>%
  filter(SAFFL == "Y", AESER == "Y") %>%
  select(USUBJID, AEDECOD, AEBODSYS, AESEV, AESTDTC, AEENDTC, AEOUT) %>%
  arrange(USUBJID, AESTDTC)

lst <- as_listing(
  adae_serious,
  key_cols   = c("USUBJID", "AEBODSYS"),
  disp_cols  = c("AEDECOD", "AESEV", "AESTDTC", "AEENDTC", "AEOUT"),
  main_title = "Listing of Serious Adverse Events",
  subtitles  = "Safety Population — Treatment-Emergent"
)

print(lst)
```

---

## KM Plot

The SAP requires a Kaplan-Meier plot (Figure 14.2.1). Use `ggsurvfit`:

```r
library(ggsurvfit)
library(survival)
library(pharmaverseadam)
library(dplyr)

os_data <- pharmaverseadam::adtte %>%
  filter(PARAMCD == "OS") %>%
  mutate(AVAL_MONTHS = AVAL / 30.4375)

km_fit <- survfit(Surv(AVAL_MONTHS, 1 - CNSR) ~ TRT01A, data = os_data)

ggsurvfit(km_fit, linewidth = 1) +
  add_confidence_interval() +
  add_risktable(risktable_stats = "n.risk") +
  add_pvalue(caption = "Log-rank {p.value}") +
  scale_ggsurvfit() +
  labs(
    title    = "Figure 14.2.1: Kaplan-Meier Plot of Overall Survival",
    subtitle = "Intent-to-Treat Population",
    x        = "Time (Months)",
    y        = "Probability"
  ) +
  theme_classic()

ggsave("output/f14_2_1_km_os.png", width = 10, height = 7)
```

> **SAS equivalent for KM plots:** `PROC LIFETEST` produces the Kaplan-Meier curve. `ggsurvfit` is the R pharmaverse equivalent. Both use the same underlying Kaplan-Meier estimator — the results should be numerically identical.

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
output/
  t14_1_1.txt
  f14_2_1_km_os.png
```

---

## Exercises

**1.** Add p-values to the demographics table. Use `test_summary_vector()` from `tern` for continuous variables and chi-square for categorical.

**2.** Produce a simple listing of all screen failures from `dm`: `USUBJID`, `SITEID`, `RFICDTC`, `ACTARM` — sorted by `SITEID` then `USUBJID`.

**3.** The `rtables` function `sort_at_path()` sorts rows by frequency. Add it to the race rows in Table 14.1.1 so that the most common race appears first.

**4. (Sets up Chapter 8)** The AE table (Table 14.3.1) has the same column structure as the demographics table: arms as columns, a Total column. The pattern is identical: `split_cols_by("TRT01A") %>% add_overall_col("Total")`. But the rows are different — AE counts by SOC and PT, not continuous statistics. Sketch the `rtables` layout for the AE table: what functions would you use for the rows? Look at `count_occurrences()` in the `tern` documentation.

---

*Next: Chapter 8 — we reuse the demographics table skeleton to build the AE table.*
