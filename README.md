# R for Clinical Statistical Programming — The Pharmaverse Way

*Learn to work as a statistical programmer in pharma using R and the pharmaverse ecosystem.*

---

## What This Book Is

You're a clinical SAS programmer. You know ADaM, SDTM, TLGs, submissions. Now you want to do the same work in R using open-source tools.

This book takes you through the **pharmaverse** — the open-source R ecosystem built specifically for clinical reporting in pharma. You'll learn by doing: every chapter builds something real, from raw collected data to a submission-ready package.

The pharmaverse pipeline:

```
Raw Data → SDTM → ADaM → TLG → eSub
{pharmaverseraw}  {sdtm.oak}  {admiral}  {rtables}/{tern}  {xportr}
```

---

## Who This Is For

- Clinical SAS programmers moving to R
- Statistical programmers joining pharma R projects
- Anyone who wants to understand how the pharmaverse stack fits together

**Assumed knowledge:**
- You know what SDTM and ADaM are
- You've worked with clinical trial data before
- You've read the R Programming book (or know R basics)

---

## Chapters

| # | Chapter | Key Packages |
|---|---------|-------------|
| 1 | The Clinical Data World in R | Base R, tidyverse orientation |
| 2 | CDISC Standards in Code | CDISC concepts, metadata |
| 3 | SDTM with {sdtm.oak} | sdtm.oak, pharmaverseraw |
| 4 | ADaM with {admiral} | admiral, pharmaversesdtm |
| 5 | Building ADSL | admiral, metacore, metatools |
| 6 | ADAE and Time-to-Event | admiral, admiralonco |
| 7 | Tables & Listings with {rtables} and {tern} | rtables, tern, rlistings |
| 8 | ARDs and Formatting with {cards} and {tfrmt} | cards, tfrmt, gtsummary |
| 9 | Submission Packages with {xportr} | xportr, metacore |
| 10 | Interactive Apps with {teal} | teal, teal.modules.clinical |
| 11 | Validation, Logging, and QC | logrx, valtools, diffdf |
| 12 | End-to-End Study | Full pipeline |

---

## Setup

```r
# Install pak (fast installer)
install.packages("pak")

# Core pharmaverse packages
pak::pak(c(
  "pharmaverse/pharmaverseraw",
  "pharmaverse/pharmaversesdtm",
  "pharmaverse/pharmaverseadam",
  "pharmaverse/admiral",
  "pharmaverse/admiralonco",
  "pharmaverse/sdtm.oak",
  "pharmaverse/metatools",
  "atorus-research/metacore",
  "atorus-research/xportr",
  "insightsengineering/rtables",
  "insightsengineering/tern",
  "insightsengineering/rlistings",
  "insightsengineering/cards",
  "gsk-biostatistics/tfrmt",
  "insightsengineering/teal",
  "pharmaverse/logrx"
))

# CRAN packages
install.packages(c(
  "tidyverse",
  "haven",
  "labelled",
  "gtsummary",
  "ggsurvfit",
  "survminer",
  "diffdf"
))
```

---

## Resources

- Pharmaverse: https://pharmaverse.org
- Examples: https://pharmaverse.github.io/examples/
- Admiral: https://pharmaverse.github.io/admiral/
- Posit Cloud (run examples in browser): https://posit.cloud/content/7279124
