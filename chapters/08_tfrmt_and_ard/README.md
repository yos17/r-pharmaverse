# Chapter 8: ARDs and Formatting with {cards} and {tfrmt}

*Analysis Results Datasets, the emerging CDISC ARD standard, and publication-ready table formatting.*

---

## What Are ARDs?

An **Analysis Results Dataset (ARD)** separates the statistical computation from the table formatting. Instead of computing statistics and formatting them in the same step, you:

1. Compute statistics → store as an ARD (a tidy data frame)
2. Format the ARD → produce the final table

This maps onto the CDISC Analysis Results Standard (ARS) that's gaining traction for regulatory submissions.

```
ADaM dataset → {cards} → ARD → {tfrmt} → Formatted Table
```

In practice, this lets you:
- Reuse the same analysis results in multiple output formats (PDF, HTML, RTF, Word)
- Share analysis results across collaborators/reviewers
- Reproduce a table from just the ARD, without re-running the analysis

---

## {cards} — Creating ARDs

```r
library(cards)
library(pharmaverseadam)
library(dplyr)

adsl <- pharmaverseadam::adsl %>% filter(SAFFL == "Y")
```

### Continuous summaries

```r
ard_age <- ard_continuous(
  data      = adsl,
  variables = AGE,
  by        = TRT01A,
  statistic = ~ continuous_summary_fns(c("N", "mean", "sd", "median", "p25", "p75",
                                          "min", "max"))
)

ard_age
```

Output: a tidy long-format ARD with one row per statistic per group:

```
  group1  group1_level variable stat_name  stat
  <chr>   <chr>        <chr>    <chr>      <dbl>
  TRT01A  Drug A       AGE      N           86
  TRT01A  Drug A       AGE      mean        58.4
  TRT01A  Drug A       AGE      sd          11.6
  TRT01A  Drug A       AGE      median      60
  ...
```

### Categorical summaries

```r
ard_sex <- ard_categorical(
  data      = adsl,
  variables = SEX,
  by        = TRT01A,
  statistic = ~ categorical_summary_fns(c("n", "p"))
)

# Combine multiple ARDs
ard_demo <- bind_ard(ard_age, ard_sex)
```

### AE ARDs

```r
adae_teae <- pharmaverseadam::adae %>%
  filter(SAFFL == "Y", TRTEMFL == "Y")

ard_ae <- ard_categorical(
  data      = adae_teae,
  variables = AEDECOD,
  by        = TRT01A
)
```

---

## {tfrmt} — Table Formatting

`tfrmt` takes an ARD and a **table format specification** (a `tfrmt` object) and combines them into a publication-ready table. The format spec defines:

- Which rows map to which statistics
- How to display values (e.g., "mean (SD)", "n (%)")
- Spanning headers
- Column ordering
- Footnotes

```r
library(tfrmt)

# A simple tfrmt format spec
fmt_demo <- tfrmt(
  
  # Column structure
  group    = c(group1, group1_level),  # row grouping
  label    = label,                    # row label
  column   = column,                   # column variable (treatment arm)
  param    = param,                    # statistic name
  value    = value,                    # statistic value
  
  # Body plan: how to display each row type
  body_plan = body_plan(
    
    # Continuous: "mean (SD)"
    frmt_structure(
      group_val = list(group1 = "AGE"),
      label_val = ".default",
      frmt_combine(
        "{mean} ({sd})",
        mean = frmt("xx.x"),
        sd   = frmt("xx.x")
      )
    ),
    
    # Categorical: "n (%)"
    frmt_structure(
      group_val = list(group1 = "SEX"),
      label_val = ".default",
      frmt_combine(
        "{n} ({p}%)",
        n = frmt("xxx"),
        p = frmt("xx.x")
      )
    )
  ),
  
  # Column order
  col_plan = col_plan(
    "Drug A", "Drug B", "Placebo", "Total"
  ),
  
  # Spanning header
  col_style_plan = col_style_plan(
    col_style_structure(
      align = "center",
      col   = c("Drug A","Drug B","Placebo")
    )
  )
)
```

---

## Program: Demographics Table via ARD + tfrmt

```r
# ard_demo_table.R
# Build demographics table using cards + tfrmt

library(cards)
library(tfrmt)
library(pharmaverseadam)
library(dplyr)
library(tidyr)

adsl      <- pharmaverseadam::adsl
adsl_safe <- adsl %>% filter(SAFFL == "Y")

# ---- 1. Build ARDs ----

# Continuous
ard_cont <- ard_continuous(
  data      = adsl_safe,
  variables = c(AGE, BMIBL),
  by        = TRT01A,
  statistic = ~ continuous_summary_fns(c("N","mean","sd","median","p25","p75","min","max"))
)

# Categorical
ard_cat <- ard_categorical(
  data      = adsl_safe,
  variables = c(SEX, AGEGR1, RACE),
  by        = TRT01A,
  statistic = ~ categorical_summary_fns(c("n","p"))
)

# Overall column
ard_cont_total <- ard_continuous(
  data      = adsl_safe,
  variables = c(AGE, BMIBL),
  statistic = ~ continuous_summary_fns(c("N","mean","sd","median","p25","p75","min","max"))
) %>% mutate(group1 = "TRT01A", group1_level = "Total")

ard_cat_total <- ard_categorical(
  data      = adsl_safe,
  variables = c(SEX, AGEGR1, RACE),
  statistic = ~ categorical_summary_fns(c("n","p"))
) %>% mutate(group1 = "TRT01A", group1_level = "Total")

# Bind all
ard_full <- bind_ard(ard_cont, ard_cat, ard_cont_total, ard_cat_total)

cat(sprintf("ARD rows: %d\n", nrow(ard_full)))
cat(sprintf("ARD statistics: %s\n", paste(unique(ard_full$stat_name), collapse=", ")))

# ---- 2. Preview ARD ----
cat("\nARD sample (Age, Drug A):\n")
ard_full %>%
  filter(group1_level == "Drug A", variable == "AGE") %>%
  select(group1_level, variable, stat_name, stat) %>%
  print()

# ---- 3. Summary statistics as a quick table ----
cat("\n=== Demographics Summary (from ARD) ===\n")

# Extract key stats and pivot
arm_summary <- ard_full %>%
  filter(stat_name %in% c("mean","sd","N")) %>%
  select(group1_level, variable, stat_name, stat) %>%
  pivot_wider(names_from = stat_name, values_from = stat) %>%
  filter(!is.na(mean)) %>%
  mutate(
    display = sprintf("%.1f (%.1f)", mean, sd),
    N       = as.integer(N)
  )

cat("\nMean (SD) by variable and treatment arm:\n")
cat(sprintf("  %-12s  %-30s  %s\n", "Variable", "Arm", "Mean (SD)"))
cat(strrep("-", 58), "\n")
for (i in seq_len(nrow(arm_summary))) {
  cat(sprintf("  %-12s  %-30s  %s  (n=%d)\n",
              arm_summary$variable[i],
              arm_summary$group1_level[i],
              arm_summary$display[i],
              arm_summary$N[i]))
}
```

---

## {gtsummary} — The Simpler Path

For standard tables where you don't need full ARD control, `gtsummary` is faster:

```r
library(gtsummary)

# Demographics table in 5 lines
adsl_safe %>%
  select(AGE, SEX, RACE, AGEGR1, TRT01A) %>%
  tbl_summary(
    by         = TRT01A,
    statistic  = list(
      all_continuous()  ~ "{mean} ({sd})",
      all_categorical() ~ "{n} ({p}%)"
    ),
    digits     = all_continuous() ~ 1
  ) %>%
  add_overall() %>%
  add_p() %>%
  bold_labels()
```

`gtsummary` produces beautiful HTML/Word/PDF tables with almost no code. The tradeoff: less control over exact regulatory formatting.

---

## RTF Output

For regulatory submissions, tables typically need RTF (Rich Text Format). Both `rtables` and `tfrmt` support this:

```r
# rtables → RTF (via formatR)
library(rtables)
library(formatR)   # or r2rtf

tbl <- build_table(lyt, adsl_safe)
# Use r2rtf for final RTF
# r2rtf::write_rtf(tbl, "table_14_1_1.rtf")

# tfrmt → GT → RTF (via officer/flextable)
# tfrmt tables can be converted to gt objects, then saved
```

---

## Exercises

**1. Full ARD for AEs**

Build an ARD for AE incidence by PT and treatment arm. Include:
- `n` (subjects with that PT)
- `p` (proportion of safety population)
- `N` (safety population denominator)

**2. tfrmt AE table**

Using your AE ARD, define a `tfrmt` format spec that produces:
```
                   Drug A (N=86)  Drug B (N=93)  Placebo (N=86)
SOC / Preferred Term
  Eye disorders
    Dry eye          3 (3.5%)      4 (4.3%)       2 (2.3%)
```

**3. gtsummary lab summary**

Using `adlb` (filter to PARAMCD == "ALT", "AST", "BILI"), produce a gtsummary table of baseline lab values by treatment arm.

**4. ARD to Excel**

Write code to export an ARD to Excel with one sheet per variable:
```r
library(openxlsx)
wb <- createWorkbook()
for (var in unique(ard$variable)) {
  addWorksheet(wb, var)
  writeData(wb, var, ard %>% filter(variable == var))
}
saveWorkbook(wb, "ard_export.xlsx")
```

---

*Next: Chapter 9 — Submission Packages with {xportr}: export to XPT with full metadata*
