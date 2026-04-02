# Chapter 6: ADAE and Time-to-Event

*Building ADAE, ADTTE, and oncology TTE datasets with admiral.*

---

## ADAE — Adverse Events Analysis Dataset

ADAE has one row per AE occurrence. It extends the SDTM AE domain with:
- Study day variables (ASTDY, AENDY)
- Treatment-emergent flag (TRTEMFL)
- Pre-specified subgroups and analysis flags
- MedDRA hierarchy (SOC, HLGT, HLT, PT)

```r
library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(dplyr)
library(lubridate)

# SDTM inputs
ae   <- pharmaversesdtm::ae
adsl <- pharmaverseadam::adsl   # use reference ADSL
```

---

## Step 1: Merge ADSL Variables

```r
adae <- ae %>%
  # Join key ADSL variables
  left_join(
    adsl %>% select(STUDYID, USUBJID, TRTSDT, TRTEDT, TRT01A, TRT01AN,
                    SAFFL, AGE, SEX, RACE),
    by = c("STUDYID", "USUBJID")
  )
```

---

## Step 2: Analysis Dates

```r
adae <- adae %>%
  
  # Convert AE start/end dates from ISO8601 to Date
  derive_vars_dt(
    new_vars_prefix = "AST",      # AE start date → ASTDT
    dtc             = AESTDTC,
    date_imputation = "first"     # partial dates: impute to first
  ) %>%
  
  derive_vars_dt(
    new_vars_prefix = "AEN",      # AE end date → AENDT
    dtc             = AEENDTC,
    date_imputation = "last"
  ) %>%
  
  # Study days relative to reference (TRTSDT)
  derive_vars_dy(
    reference_date = TRTSDT,
    source_vars    = exprs(ASTDT, AENDT)
  )
# Creates: ASTDY, AENDY
```

---

## Step 3: Treatment-Emergent Flag (TRTEMFL)

An AE is treatment-emergent if it started on or after the first dose and no later than 30 days after last dose:

```r
adae <- adae %>%
  mutate(
    TRTEMFL = case_when(
      # Started on or after first dose
      !is.na(ASTDT) & !is.na(TRTSDT) & ASTDT >= TRTSDT &
      # Started within 30 days after last dose (or no end date = ongoing)
      (is.na(TRTEDT) | ASTDT <= TRTEDT + 30) ~ "Y",
      TRUE ~ NA_character_
    )
  )
```

Using admiral's `derive_var_trtemfl()` when available:

```r
adae <- adae %>%
  derive_var_trtemfl(
    new_var         = TRTEMFL,
    trt_start_date  = TRTSDT,
    trt_end_date    = TRTEDT,
    end_window      = 30    # days after treatment end
  )
```

---

## Step 4: Worst Severity Flag (AOCCIFL / AOCCPIFL)

For each subject + preferred term, flag the worst AE:

```r
adae <- adae %>%
  # AESEV numeric for sorting
  mutate(
    ASEVN = case_when(
      AESEV == "MILD"     ~ 1L,
      AESEV == "MODERATE" ~ 2L,
      AESEV == "SEVERE"   ~ 3L,
      TRUE                ~ NA_integer_
    )
  ) %>%
  
  # Flag worst severity per subject (overall)
  derive_var_extreme_flag(
    by_vars = exprs(USUBJID),
    order   = exprs(desc(ASEVN), ASTDT),
    new_var = AOCCIFL,
    mode    = "first"
  ) %>%
  
  # Flag worst severity per subject per PT
  derive_var_extreme_flag(
    by_vars = exprs(USUBJID, AEDECOD),
    order   = exprs(desc(ASEVN), ASTDT),
    new_var = AOCCPIFL,
    mode    = "first"
  )
```

---

## ADTTE — Time-to-Event Analysis Dataset

ADTTE has one row per subject per event type (PARAMCD). It's used for Kaplan-Meier and Cox regression analyses.

```r
library(admiral)

# Reference ADTTE
adtte_ref <- pharmaverseadam::adtte
```

### Structure

| Variable | Meaning |
|----------|---------|
| `PARAMCD` | Event type code (e.g., "OS", "PFS", "TTAE") |
| `PARAM` | Event type description |
| `AVAL` | Time to event or censoring (in `AVALU` units) |
| `CNSR` | Censoring indicator (0=event, 1=censored) |
| `EVNTDESC` | Event description |
| `CNSDTDSC` | Censoring date description |
| `STARTDT` | Start date |
| `ADT` | Event or censoring date |

---

## Building ADTTE: Time to First AE

```r
# Derive TTAE (Time to First Adverse Event)

# 1. Find first AE date per subject
first_ae <- adae %>%
  filter(TRTEMFL == "Y") %>%
  group_by(STUDYID, USUBJID) %>%
  summarise(AEDTFIRST = min(ASTDT, na.rm = TRUE), .groups = "drop")

# 2. Build ADTTE structure
adtte_ttae <- adsl %>%
  filter(SAFFL == "Y") %>%
  select(STUDYID, USUBJID, TRTSDT, TRTEDT, TRT01A, TRT01AN, AGE, SEX) %>%
  left_join(first_ae, by = c("STUDYID", "USUBJID")) %>%
  mutate(
    PARAMCD   = "TTAE",
    PARAM     = "Time to First Adverse Event",
    AVALU     = "DAYS",
    STARTDT   = TRTSDT,
    
    # Event date or censoring date
    ADT       = if_else(!is.na(AEDTFIRST), AEDTFIRST, TRTEDT),
    
    # Censoring indicator: 0 = event occurred, 1 = censored
    CNSR      = if_else(!is.na(AEDTFIRST), 0L, 1L),
    
    # Time to event (days)
    AVAL      = as.numeric(ADT - TRTSDT),
    
    EVNTDESC  = if_else(CNSR == 0L, "Adverse Event Occurred", ""),
    CNSDTDSC  = if_else(CNSR == 1L, "End of Treatment", "")
  )
```

---

## Building ADTTE: Overall Survival

```r
# Time to death (or censoring at last known alive)

# Death date from ADSL
adtte_os <- adsl %>%
  filter(ITTFL == "Y") %>%
  select(STUDYID, USUBJID, TRTSDT, DTHDT, DTHFL, TRT01P, TRT01PN) %>%
  mutate(
    PARAMCD  = "OS",
    PARAM    = "Overall Survival",
    AVALU    = "DAYS",
    STARTDT  = TRTSDT,
    
    # Event: death; Censoring: last known alive
    CNSR     = if_else(!is.na(DTHDT), 0L, 1L),
    ADT      = if_else(!is.na(DTHDT), DTHDT, as.Date(NA)),
    AVAL     = as.numeric(ADT - TRTSDT),
    
    EVNTDESC = if_else(CNSR == 0L, "Death", ""),
    CNSDTDSC = if_else(CNSR == 1L, "Last Known Alive Date", "")
  )
```

---

## Program: AE Summary Report

```r
# ae_summary.R

library(admiral)
library(pharmaversesdtm)
library(pharmaverseadam)
library(dplyr)

ae   <- pharmaversesdtm::ae
adsl <- pharmaverseadam::adsl

# Build minimal ADAE
adae <- ae %>%
  left_join(adsl %>% select(USUBJID, TRTSDT, TRTEDT, TRT01A, SAFFL),
            by = "USUBJID") %>%
  derive_vars_dt(new_vars_prefix="AST", dtc=AESTDTC, date_imputation="first") %>%
  mutate(
    TRTEMFL = if_else(!is.na(ASTDT) & !is.na(TRTSDT) & ASTDT >= TRTSDT, "Y", NA_character_),
    ASEVN   = case_when(AESEV=="MILD"~1L, AESEV=="MODERATE"~2L, AESEV=="SEVERE"~3L)
  )

# Safety population TEAE
teae <- adae %>%
  filter(SAFFL == "Y", TRTEMFL == "Y")

n_safety <- sum(adsl$SAFFL == "Y", na.rm=TRUE)

cat("=== Treatment-Emergent Adverse Events ===\n")
cat(sprintf("Safety population: %d subjects\n\n", n_safety))

# Overall incidence
n_any_ae <- teae %>% distinct(USUBJID) %>% nrow()
cat(sprintf("Any TEAE: %d/%d (%.1f%%)\n", n_any_ae, n_safety, 100*n_any_ae/n_safety))

n_serious <- teae %>% filter(AESER=="Y") %>% distinct(USUBJID) %>% nrow()
cat(sprintf("Any serious TEAE: %d/%d (%.1f%%)\n\n", n_serious, n_safety, 100*n_serious/n_safety))

# By SOC and PT — top 10 PTs
pt_summary <- teae %>%
  group_by(AEBODSYS, AEDECOD) %>%
  summarise(
    n_subj = n_distinct(USUBJID),
    n_ev   = n(),
    .groups = "drop"
  ) %>%
  mutate(pct = round(100 * n_subj / n_safety, 1)) %>%
  arrange(desc(n_subj)) %>%
  head(10)

cat("Top 10 TEAEs by preferred term:\n")
cat(sprintf("  %-35s  %-45s  %4s  %5s\n", "SOC", "Preferred Term", "n", "%"))
cat(strrep("-", 95), "\n")
for (i in seq_len(nrow(pt_summary))) {
  cat(sprintf("  %-35s  %-45s  %4d  %5.1f%%\n",
              substr(pt_summary$AEBODSYS[i], 1, 35),
              substr(pt_summary$AEDECOD[i], 1, 45),
              pt_summary$n_subj[i],
              pt_summary$pct[i]))
}

# By arm
cat("\nTEAE incidence by arm:\n")
arm_summary <- teae %>%
  group_by(TRT01A) %>%
  summarise(n_subj = n_distinct(USUBJID), .groups = "drop")

n_per_arm <- adsl %>% filter(SAFFL=="Y") %>% count(TRT01A, name="total")
arm_summary <- arm_summary %>%
  left_join(n_per_arm, by="TRT01A") %>%
  mutate(pct = round(100*n_subj/total, 1))

for (i in seq_len(nrow(arm_summary))) {
  cat(sprintf("  %-30s: %d/%d (%.1f%%)\n",
              arm_summary$TRT01A[i],
              arm_summary$n_subj[i],
              arm_summary$total[i],
              arm_summary$pct[i]))
}
```

---

## Exercises

**1. ADAE severity grades**

Add NCI CTCAE grade columns to ADAE:
- `ATOXGR` — toxicity grade (1–5)
- `ATOXGRN` — numeric
- Flag subjects with Grade 3+ events: `GR3FL`

**2. Concomitant medications check**

Using `cm` domain: is any AE possibly related to a concomitant medication? Identify subjects who started a new concomitant medication within 7 days before an AE started.

**3. Build ADTTE for PFS**

Define Progression-Free Survival as first of: progression event in `rs` (response) domain OR death. Subjects without events are censored at last assessment date.

**4. Kaplan-Meier plot**

Install `ggsurvfit`. Using your `adtte_os`, produce a basic KM plot:
```r
library(ggsurvfit)
library(survival)

fit <- survfit(Surv(AVAL, 1-CNSR) ~ TRT01P, data = adtte_os)
ggsurvfit(fit) + add_risktable()
```

---

*Next: Chapter 7 — Tables, Listings, Graphs with {rtables} and {tern}*
