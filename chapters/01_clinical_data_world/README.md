# Chapter 1: Study PHARM-001 Begins

*You have raw clinical data. Let's see what's in it.*

---

## The Problem

You've been handed Study PHARM-001 — a Phase III trial of an experimental treatment vs. placebo. The data is in R packages. Your first task from the SAP: count subjects and print a basic demographics summary.

Twenty lines. Let's go.

---

## The Data

Study PHARM-001 lives in `pharmaversesdtm`. The Demographics domain `dm` is your starting point — one row per subject.

```r
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm
nrow(dm)        # all subjects including screen failures
ncol(dm)        # variables
names(dm)       # what's in there
```

The columns that matter first: `USUBJID` (unique subject ID), `ACTARM` (actual treatment arm), `AGE`, `SEX`, `RACE`.

---

## SAS vs. R: The Translation Layer

You know this data. You've worked with it in SAS. Here's the mapping:

| SAS | R |
|-----|---|
| `PROC SORT` | `arrange()` |
| `PROC MEANS` | `summarise()` |
| `DATA` step merge | `left_join()` |
| `PROC FREQ` | `count()` |
| `WHERE` clause | `filter()` |
| `KEEP=` | `select()` |
| Assignment in DATA step | `mutate()` |

The pipe `%>%` (or `|>`) chains operations. Read it as "then":

**In SAS:**
```sas
/* Sort, keep 3 variables, filter */
PROC SORT DATA=dm(KEEP=USUBJID AGE SEX WHERE=(ACTARM NE "Screen Failure"))
          OUT=dm_filtered;
  BY USUBJID;
RUN;
```

**In R (pharmaverse):**
```r
dm %>%
  filter(ACTARM != "Screen Failure") %>%
  select(USUBJID, AGE, SEX) %>%
  arrange(USUBJID)
```

Same as sorting an input dataset, keeping three variables, and filtering in SAS — but in one readable chain.

> ⚠️ **Critical difference:** SAS string comparison is **case-insensitive** (`"Screen Failure"` = `"screen failure"` = `"SCREEN FAILURE"`). R is **case-sensitive**: `"Screen Failure" != "screen failure"`. Always verify the exact case of CT values in R.

---

## Reading XPT Files

In real studies, data arrives as SAS XPT transport files.

**In SAS:**
```sas
LIBNAME adam XPORT 'path/to/adsl.xpt';
DATA adsl;
  SET adam.adsl;
RUN;
```

**In R (pharmaverse):**
```r
library(haven)
adsl <- haven::read_xpt("path/to/adsl.xpt")
```

> **Note on dates:** SAS dates are stored as days since **January 1, 1960**. R `Date` objects count days since **January 1, 1970**. `haven::read_xpt()` handles this conversion automatically — you get proper R `Date` objects.

---

## Step 1: Count Subjects

**In SAS:**
```sas
/* Count all subjects */
PROC SQL;
  SELECT COUNT(*) AS N FROM dm;
QUIT;

/* Exclude screen failures */
DATA dm_enrolled;
  SET dm;
  WHERE ACTARM NE "Screen Failure";
RUN;

PROC SQL;
  SELECT COUNT(*) AS N_ENROLLED FROM dm_enrolled;
QUIT;
```

**In R (pharmaverse):**
```r
library(dplyr)
library(pharmaversesdtm)

dm <- pharmaversesdtm::dm

# All subjects
nrow(dm)

# Screen failures excluded from analysis
dm_enrolled <- dm %>% filter(ACTARM != "Screen Failure")
nrow(dm_enrolled)

cat(sprintf("Enrolled: %d (of %d total, %d screen failures)\n",
            nrow(dm_enrolled), nrow(dm), nrow(dm) - nrow(dm_enrolled)))
```

---

## Step 2: Count by Arm

**In SAS:**
```sas
PROC FREQ DATA=dm_enrolled;
  TABLES ACTARM / NOCUM;
RUN;
```

**In R (pharmaverse):**
```r
dm_enrolled %>%
  count(ACTARM) %>%
  arrange(ACTARM)
```

---

## Step 3: Age Summary

**In SAS:**
```sas
PROC MEANS DATA=dm_enrolled N MEAN STD;
  CLASS ACTARM;
  VAR AGE;
RUN;
```

**In R (pharmaverse):**
```r
dm_enrolled %>%
  group_by(ACTARM) %>%
  summarise(
    n        = n(),
    mean_age = round(mean(AGE, na.rm = TRUE), 1),
    sd_age   = round(sd(AGE, na.rm = TRUE), 1),
    .groups  = "drop"
  )
```

> **Missing values:** SAS uses `.` for numeric missing and `" "` for character missing. R uses `NA` for all types. When reading SAS XPT files with `haven`, SAS `.` becomes `NA`. Always use `na.rm = TRUE` in summary functions.

---

## Variable Labels

SDTM and ADaM datasets carry variable labels. In SAS these are native; in R they're stored as attributes.

**In SAS:**
```sas
/* Labels are always visible in PROC CONTENTS, PROC PRINT, etc. */
PROC CONTENTS DATA=dm;
RUN;
/* AGE — Age (label from dataset) */

/* Setting labels: */
DATA dm;
  SET dm;
  LABEL AGE      = "Age"
        USUBJID  = "Unique Subject Identifier";
RUN;
```

**In R (pharmaverse):**
```r
# Read the label of a single variable
attr(dm$USUBJID, "label")   # "Unique Subject Identifier"

# Set labels with labelled package
library(labelled)
dm <- dm %>%
  set_variable_labels(
    AGE     = "Age",
    USUBJID = "Unique Subject Identifier"
  )
```

---

## The First PHARM-001 Program

```r
# pharm001_ch01.R
# Study PHARM-001 — Chapter 1
# Load DM, count subjects, print basic demographics

library(tidyverse)
library(pharmaversesdtm)

# ---- Load ----
dm  <- pharmaversesdtm::dm
ae  <- pharmaversesdtm::ae

# ---- Enrolled subjects ----
dm_enrolled <- dm %>% filter(ACTARM != "Screen Failure")

cat(sprintf("Study PHARM-001\n"))
cat(sprintf("Total screened:    %d\n", nrow(dm)))
cat(sprintf("Screen failures:   %d\n", nrow(dm) - nrow(dm_enrolled)))
cat(sprintf("Enrolled:          %d\n", nrow(dm_enrolled)))

# ---- Age by arm ----
cat("\nAge by treatment arm:\n")
age_summary <- dm_enrolled %>%
  group_by(ACTARM) %>%
  summarise(
    n        = n(),
    mean_age = round(mean(AGE, na.rm = TRUE), 1),
    sd_age   = round(sd(AGE, na.rm = TRUE), 1),
    min_age  = min(AGE, na.rm = TRUE),
    max_age  = max(AGE, na.rm = TRUE),
    .groups  = "drop"
  )

for (i in seq_len(nrow(age_summary))) {
  cat(sprintf("  %-35s n=%d  Mean(SD)=%.1f(%.1f)  Range=%d-%d\n",
              age_summary$ACTARM[i],
              age_summary$n[i],
              age_summary$mean_age[i],
              age_summary$sd_age[i],
              age_summary$min_age[i],
              age_summary$max_age[i]))
}

# ---- Sex by arm ----
cat("\nSex by arm:\n")
sex_table <- dm_enrolled %>%
  count(ACTARM, SEX) %>%
  group_by(ACTARM) %>%
  mutate(pct = round(100 * n / sum(n), 1)) %>%
  ungroup()

for (i in seq_len(nrow(sex_table))) {
  cat(sprintf("  %-35s %-2s n=%d (%.1f%%)\n",
              sex_table$ACTARM[i], sex_table$SEX[i],
              sex_table$n[i], sex_table$pct[i]))
}

# ---- AE count ----
ae_count <- ae %>%
  inner_join(dm_enrolled %>% select(USUBJID, ACTARM), by = "USUBJID") %>%
  group_by(ACTARM) %>%
  summarise(n_subjects = n_distinct(USUBJID), n_events = n(), .groups = "drop")

cat("\nAE overview:\n")
for (i in seq_len(nrow(ae_count))) {
  cat(sprintf("  %-35s %d subjects, %d events\n",
              ae_count$ACTARM[i], ae_count$n_subjects[i], ae_count$n_events[i]))
}
```

---

## Key R Concepts for SAS Programmers

**Missing values:** R uses `NA` for all types. SAS uses `.` for numeric. After reading XPT with `haven`, SAS `.` becomes `NA`. Always use `na.rm = TRUE` in summary functions.

**Character case:** R is case-sensitive. `"MALE"` ≠ `"Male"`. SDTM uses uppercase CT values.

**Dates:** SAS dates are days since Jan 1, 1960. R dates are days since Jan 1, 1970. `haven::read_xpt()` converts automatically. SDTM stores dates as ISO8601 character strings — we'll convert them in Chapter 4.

**Labels:** SAS variable labels live as attributes in R. `attr(dm$USUBJID, "label")` gives you `"Unique Subject Identifier"`. Package `labelled` lets you set and view them.

---

## What We Have Now

```
pharm001_ch01.R       # 20 lines: load dm, count subjects, demographics summary
```

Run it:
```r
source("pharm001_ch01.R")
```

Output: subject counts by arm, age mean/SD/range, sex breakdown, AE count.

---

## Joining Datasets

You'll do this constantly. The pattern:

**In SAS:**
```sas
/* One AE row gets the subject's demographics */
PROC SORT DATA=ae;       BY USUBJID; RUN;
PROC SORT DATA=dm_enrolled; BY USUBJID; RUN;

DATA ae_with_demo;
  MERGE ae(IN=inAE)
        dm_enrolled(IN=inDM KEEP=USUBJID AGE SEX ACTARM);
  BY USUBJID;
  IF inAE;   /* keep all AE records */
RUN;
```

> ⚠️ SAS `MERGE` requires sorted datasets. If `dm_enrolled` had duplicate `USUBJID` rows, the merge result can be unpredictable. The R `left_join` similarly multiplies AE rows if `dm_enrolled` had duplicates.

**In R (pharmaverse):**
```r
# One AE row gets the subject's demographics — same as SAS MERGE + BY
ae_with_demo <- ae %>%
  left_join(dm_enrolled %>% select(USUBJID, AGE, SEX, ACTARM),
            by = "USUBJID")
```

Warning: if `dm_enrolled` had duplicate `USUBJID` rows, this would multiply AE rows. We'll deal with that in Chapter 13.

---

## Exercises

**1.** Load `dm`, `ae`, `vs`, `ex` from `pharmaversesdtm`. For each: how many rows? How many distinct USUBJIDs? Are all AE subjects in DM?

**2.** Translate this to dplyr:
```sas
DATA want;
  SET dm;
  WHERE SEX = "F" AND AGE >= 50;
  KEEP USUBJID AGE RACE ACTARM;
RUN;
```

**3.** Find the subject with the most AEs. What arm are they in?

**4. (Sets up Chapter 2)** The variable `RFSTDTC` in DM holds the reference start date as an ISO8601 character string. Look at its values. How would you convert it to an R `Date`? What happens to subjects where it's `NA`? Write a function that converts it and counts how many subjects have a valid date.

---

## What's Next

Chapter 2: we inspect the SDTM domains more carefully, validate one variable against controlled terminology, and understand why character constants matter.

Current pipeline:

```
pharm001_ch01.R    [done] — load DM, count subjects, basic demographics
```

*Next chapter adds the first real quality check to that foundation.*
