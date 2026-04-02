# R for Clinical Statistical Programming — The Pharmaverse Way

*Build a complete clinical reporting pipeline from scratch, one chapter at a time.*

---

## Study PHARM-001

Every chapter in this book works on the same fictional study. You start in Chapter 1 by loading raw data and counting subjects. By Chapter 12 you have a complete submission package: validated SDTM, ADaM datasets, tables, listings, figures, an interactive explorer, and a single `run_all.R` that produces all of it.

The pipeline you build:

```
Raw CRF data
  ↓ Ch 3 {sdtm.oak}       map raw DM to SDTM
SDTM (pharmaversesdtm)
  ↓ Ch 4 {admiral}        derive TRTSDT, TRTEDT
  ↓ Ch 5 {admiral}        build complete ADSL
  ↓ Ch 6 {admiral}        build ADAE with TEAE flag
ADaM datasets
  ↓ Ch 7 {rtables/tern}   Table 14.1.1: demographics + KM plot
  ↓ Ch 8 {rtables/tern}   Table 14.3.1: TEAE by SOC/PT
Tables, Listings, Graphs
  ↓ Ch 9 {xportr}         export to XPT with full metadata
  ↓ Ch 10 {teal}          interactive data explorer
  ↓ Ch 11 {logrx}         logging and QC for every program
  ↓ Ch 12                 run_all.R — the complete pipeline
  ↓ Ch 13                 find 3 bugs using browser() and diffdf
Submission package
```

---

## Who This Is For

Clinical SAS programmers moving to R. You know ADaM, SDTM, and TLGs. This book shows you how the pharmaverse does the same work in R — through a single growing example rather than isolated demos.

**Assumed:** You know what SDTM and ADaM are. You've read the R Programming book (or know R basics).

---

## Chapters

| # | Chapter | What PHARM-001 gains |
|---|---------|---------------------|
| 1 | Study PHARM-001 Begins | Load DM, count subjects, basic demographics |
| 2 | Inspect PHARM-001's SDTM | Validate domains, understand CT |
| 3 | Map Raw Data to SDTM | sdtm.oak DM mapping from pharmaverseraw |
| 4 | First ADaM Variables | Derive TRTSDT, TRTEDT with admiral |
| 5 | Build ADSL | Complete ADSL — all variables for Table 14.1.1 |
| 6 | Build ADAE | TEAE flag, severity flags, ADTTE |
| 7 | Build the Demographics Table | Table 14.1.1 + KM plot |
| 8 | Build the AE Table | Table 14.3.1 — reuse Ch 7 skeleton |
| 9 | Export to XPT | Submission-ready XPT with metadata |
| 10 | Add a teal App | Interactive explorer (demographics + KM) |
| 11 | Add Logging | logrx audit trail on every program |
| 12 | The Complete Pipeline | run_all.R — everything in order |
| 13 | Find the 3 Bugs | browser() + diffdf to locate 3 ADSL errors |

---

## Setup

```r
install.packages("pak")

pak::pak(c(
  "pharmaverse/pharmaversesdtm",
  "pharmaverse/pharmaverseadam",
  "pharmaverse/pharmaverseraw",
  "pharmaverse/admiral",
  "pharmaverse/sdtm.oak",
  "pharmaverse/logrx",
  "atorus-research/metacore",
  "atorus-research/xportr",
  "insightsengineering/rtables",
  "insightsengineering/tern",
  "insightsengineering/rlistings",
  "insightsengineering/cards",
  "insightsengineering/teal",
  "insightsengineering/teal.modules.clinical"
))

install.packages(c(
  "tidyverse", "haven", "labelled", "diffdf",
  "gtsummary", "ggsurvfit", "tfrmt"
))
```

---

## Resources

- Pharmaverse: https://pharmaverse.org
- Admiral: https://pharmaverse.github.io/admiral/
- Pharmaverse examples: https://pharmaverse.github.io/examples/
- R in Pharma conference: https://rinpharma.com
