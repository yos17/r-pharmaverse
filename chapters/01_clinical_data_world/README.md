# Chapter 1: The Clinical Data World in R

*Why R for pharma? The tidyverse. How your SAS habits translate.*

---

## Why Pharmaverse?

You already know clinical programming. SDTM domains. ADaM conventions. TLGs. Output specifications. You've done all of it in SAS.

The question isn't *why learn R* — it's *how to map what you already know onto R and the pharmaverse toolchain.*

The short answer: everything you do in SAS has an equivalent in the pharmaverse. The concepts are identical. Only the syntax and tooling differ.

| SAS | R equivalent |
|-----|-------------|
| `PROC SORT` | `arrange()` (dplyr) |
| `PROC MEANS` / `PROC UNIVARIATE` | `summarise()`, `describe()` |
| `DATA` step merge | `left_join()`, `merge()` |
| `PROC FREQ` | `count()`, `table()` |
| `PROC REPORT` | `rtables` + `tern` |
| `PROC EXPORT` (XPT) | `xportr` |
| ADaM macros | `admiral` functions |
| SDTM programming | `sdtm.oak` |
| SAS formats/informats | `haven::labelled`, factor levels |

---

## The Tidyverse — Your Main Dialect

In SAS, you write `DATA` steps and `PROC` steps. In R pharmaverse work, you'll write **tidyverse** code — a set of packages with a consistent grammar.

The most important: **dplyr** (data manipulation) and **tidyr** (reshaping).

```r
library(dplyr)
library(tidyr)
```

### The Pipe

The pipe `%>%` (or `|>` in base R 4.1+) chains operations. Read it as "then":

```r
adsl %>%
  filter(TRT01P == "Drug A") %>%
  select(USUBJID, AGE, SEX, RACE) %>%
  arrange(USUBJID)
```

Compare to SAS:
```sas
PROC SORT DATA=adsl(WHERE=(TRT01P="Drug A")
                    KEEP=USUBJID AGE SEX RACE);
  BY USUBJID;
RUN;
```

Same logic. Different syntax.

---

## dplyr: The Six Verbs

```r
library(dplyr)
library(pharmaversesdtm)  # example SDTM data

dm <- pharmaversesdtm::dm   # Demographics domain

# filter() — like WHERE in SAS
dm %>% filter(ACTARM != "Screen Failure")

# select() — keep/drop columns (like KEEP=/DROP= in DATA step)
dm %>% select(USUBJID, SUBJID, SITEID, AGE, SEX, RACE, ETHNIC, ACTARM)

# mutate() — add/modify columns (like assignment in DATA step)
dm %>% mutate(
  AGE_GROUP = case_when(
    AGE < 40  ~ "<40",
    AGE < 65  ~ "40-64",
    TRUE      ~ ">=65"
  )
)

# arrange() — PROC SORT
dm %>% arrange(SITEID, USUBJID)

# summarise() + group_by() — PROC MEANS/FREQ by group
dm %>%
  group_by(ACTARM) %>%
  summarise(
    n       = n(),
    mean_age = mean(AGE, na.rm = TRUE),
    .groups = "drop"
  )

# rename() — RENAME= option
dm %>% rename(SUBJECT = USUBJID)
```

---

## Joining Datasets

You'll do this constantly — merging SDTM domains, adding variables from DM to ADSL, etc.

```r
# Left join (equivalent to DATA step merge + IF _MERGE_)
adsl <- left_join(dm_core, ex_derived, by = "USUBJID")

# Multiple by-variables
ae_with_demo <- left_join(ae, dm[, c("USUBJID","AGE","SEX")], 
                           by = "USUBJID")

# One-to-many: every AE row gets the subject's demographics
```

In SAS you'd sort both datasets and MERGE + BY. In dplyr you just join.

---

## Reading and Writing Clinical Data

```r
library(haven)   # SAS XPT + SAS7BDAT reader/writer

# Read SAS transport file (XPT)
adsl <- haven::read_xpt("path/to/adsl.xpt")

# Read SAS dataset
dm <- haven::read_sas("path/to/dm.sas7bdat")

# Write XPT (for submission — but use xportr for proper metadata)
haven::write_xpt(adsl, "adsl.xpt")

# Read/write CSV
readr::write_csv(adsl, "adsl.csv")
adsl <- readr::read_csv("adsl.csv")
```

---

## Labels in R

SAS datasets carry variable labels. R's `haven` package preserves them as attributes.

```r
library(haven)
library(labelled)

dm <- pharmaversesdtm::dm

# Variable labels are stored as attributes
attr(dm$USUBJID, "label")   # "Unique Subject Identifier"

# See all labels
labelled::var_label(dm)

# Set labels
dm$AGE_GROUP <- dm$AGE_GROUP %>% labelled::set_variable_labels(
  AGE_GROUP = "Age Group (Years)"
)
```

---

## Program: Your First Pharmaverse Script

```r
# first_pharmaverse.R
# Load CDISC example data and do basic exploration

library(tidyverse)
library(haven)
library(labelled)
library(pharmaversesdtm)

# ---- Load SDTM data ----
dm  <- pharmaversesdtm::dm
ae  <- pharmaversesdtm::ae
vs  <- pharmaversesdtm::vs
ex  <- pharmaversesdtm::ex

cat("=== Dataset Overview ===\n")
datasets <- list(dm=dm, ae=ae, vs=vs, ex=ex)
for (nm in names(datasets)) {
  d <- datasets[[nm]]
  cat(sprintf("%-5s: %4d rows × %2d cols\n", 
              toupper(nm), nrow(d), ncol(d)))
}

# ---- Screen failures ----
# In clinical trials, screen failures are excluded from analysis
dm_enrolled <- dm %>% filter(ACTARM != "Screen Failure")
cat(sprintf("\nEnrolled subjects: %d (of %d total)\n",
            nrow(dm_enrolled), nrow(dm)))

# ---- Demographics summary ----
cat("\n=== Demographics by Treatment Arm ===\n")

# Age by arm
age_summary <- dm_enrolled %>%
  group_by(ACTARM) %>%
  summarise(
    n        = n(),
    mean_age = round(mean(AGE, na.rm=TRUE), 1),
    sd_age   = round(sd(AGE,  na.rm=TRUE), 1),
    min_age  = min(AGE, na.rm=TRUE),
    max_age  = max(AGE, na.rm=TRUE),
    .groups  = "drop"
  )

cat("\nAge:\n")
for (i in seq_len(nrow(age_summary))) {
  cat(sprintf("  %-30s n=%d  Mean(SD)=%.1f(%.1f)  Range=%d-%d\n",
              age_summary$ACTARM[i],
              age_summary$n[i],
              age_summary$mean_age[i],
              age_summary$sd_age[i],
              age_summary$min_age[i],
              age_summary$max_age[i]))
}

# Sex by arm
sex_table <- dm_enrolled %>%
  count(ACTARM, SEX) %>%
  group_by(ACTARM) %>%
  mutate(pct = round(100 * n / sum(n), 1)) %>%
  ungroup()

cat("\nSex:\n")
for (i in seq_len(nrow(sex_table))) {
  cat(sprintf("  %-30s %-2s n=%d (%.1f%%)\n",
              sex_table$ACTARM[i], sex_table$SEX[i],
              sex_table$n[i], sex_table$pct[i]))
}

# ---- Adverse events summary ----
cat("\n=== Adverse Events ===\n")
ae_with_arm <- ae %>%
  inner_join(dm_enrolled %>% select(USUBJID, ACTARM), by="USUBJID")

ae_summary <- ae_with_arm %>%
  group_by(ACTARM) %>%
  summarise(
    n_subjects = n_distinct(USUBJID),
    n_events   = n(),
    n_serious  = sum(AESER == "Y", na.rm=TRUE),
    .groups    = "drop"
  )

n_per_arm <- dm_enrolled %>% count(ACTARM, name="total_n")
ae_summary <- ae_summary %>%
  left_join(n_per_arm, by="ACTARM") %>%
  mutate(ae_pct = round(100 * n_subjects / total_n, 1))

for (i in seq_len(nrow(ae_summary))) {
  cat(sprintf("  %-30s: %d/%d subjects (%s%%) with AEs, %d serious\n",
              ae_summary$ACTARM[i],
              ae_summary$n_subjects[i],
              ae_summary$total_n[i],
              ae_summary$ae_pct[i],
              ae_summary$n_serious[i]))
}
```

---

## Key Differences from SAS

1. **Observations vs. variables**: In R, `nrow()` = number of records, `ncol()` = number of variables. In SAS it's "observations" and "variables."

2. **Missing values**: R uses `NA` for all types. SAS uses `.` for numeric and `" "` for character. After reading XPT with `haven`, SAS `.` becomes `NA`. Be aware: `haven` also has `NA_integer_`, `NA_real_`, `NA_character_`, `NA_complex_` typed NAs.

3. **Character case**: R is case-sensitive. `"MALE"` ≠ `"Male"`. SAS functions like `UPCASE` → use `toupper()` in R.

4. **Factors vs. formats**: SAS formats are like R's factors but different. `haven` reads SAS formats as labelled vectors. Use `haven::as_factor()` to convert.

5. **Date handling**: SAS dates are integers (days since Jan 1, 1960). R dates are integers (days since Jan 1, 1970). `haven` converts automatically.

---

## Exercises

**1. Load and explore**

Install `pharmaversesdtm`. Load `dm`, `ae`, `vs`, `ex`. For each:
- How many rows and columns?
- What are the key variables?
- Are there any `NA` values? Where?

**2. Rewrite in dplyr**

Translate this pseudo-SAS to dplyr:
```sas
DATA want;
  SET dm;
  WHERE SEX = "F" AND AGE >= 50;
  KEEP USUBJID AGE RACE ACTARM;
  IF ACTARM = "Placebo" THEN ARM_LABEL = "PBO";
  ELSE ARM_LABEL = "ACTIVE";
RUN;
```

**3. Join practice**

Create a dataset with one row per AE, including the subject's age, sex, and treatment arm. How many AEs does the oldest subject have?

**4. First summary**

Produce a frequency table of `AEDECOD` (adverse event preferred term) sorted by descending count.

---

*Next: Chapter 2 — CDISC Standards in Code: metadata, codelists, and the controlled terminology mindset*
