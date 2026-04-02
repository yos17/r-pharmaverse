# Chapter 10: Add a teal App

*The clinicians want to explore PHARM-001 interactively. One demographics module. Then add KM.*

---

## Where We Left Off

```
output/adam/adsl.xpt, adae.xpt, adtte.xpt, adlb.xpt (Chapter 9)
```

The static tables are done. The SAP also calls for an interactive data explorer for internal review. We build it in teal.

---

## What teal Is

`teal` is a Shiny-based framework for clinical data exploration. A teal app gives reviewers — statisticians, clinicians, medical monitors — an interactive interface to explore ADaM data without writing code.

Features:
- **Filter panel** — global subject filters that propagate to every module
- **Analysis modules** — pluggable views (tables, KM plots, lab plots, listings)
- **Reproducibility code** — every analysis shows the R code behind it
- **Reporter** — export results to PDF/Word in-app

```r
library(teal)
library(teal.modules.clinical)
library(pharmaverseadam)
library(dplyr)
```

---

## No Direct SAS Equivalent — But Here's the Closest

SAS does not have a direct equivalent to `teal`. The closest SAS tools are:

| What teal provides | SAS equivalent / approximation |
|-------------------|-------------------------------|
| Interactive table explorer | SAS Visual Analytics (VA) / JMP |
| On-demand `PROC REPORT` | Manual re-running of programs |
| Dynamic `WHERE` clause across all analyses | Re-running with `WHERE` in each program |
| KM plot on filtered subset | `PROC LIFETEST` re-run |
| Point-and-click subgroup analysis | SAS Enterprise Guide / SAS Studio |

**The key conceptual insight:** `teal` is what you build when you want an interactive, browser-based version of all your `PROC REPORT` outputs in one place. Instead of running 10 SAS programs to check results for the female subgroup, you change one filter in teal and all tables update simultaneously.

The **filter panel** = a dynamic `WHERE` clause applied to **all** analyses simultaneously — something that would require re-running every SAS program manually.

---

## Start with One Module

Don't build the full app first. Start with one module.

```r
library(teal)
library(teal.modules.clinical)
library(pharmaverseadam)

# The data
adsl <- pharmaverseadam::adsl

data <- teal_data(ADSL = adsl)

# One module: demographics summary
app <- init(
  data = data,
  modules = modules(
    tm_t_summary(
      label        = "Demographics",
      dataname     = "ADSL",
      arm_var      = choices_selected("TRT01A", selected = "TRT01A"),
      summarize_vars = choices_selected(
        c("AGE", "SEX", "RACE", "AGEGR1"),
        selected = c("AGE", "SEX", "RACE")
      )
    )
  ),
  title = "PHARM-001 Clinical Data Explorer"
)

shinyApp(app$ui, app$server)
```

Run it. Click around. The filter panel appears automatically. Set `SAFFL == "Y"` in the filter panel — the demographics table updates. That's the key feature.

---

## Add the KM Module

```r
adsl  <- pharmaverseadam::adsl
adtte <- pharmaverseadam::adtte

data <- teal_data(
  ADSL  = adsl,
  ADTTE = adtte
)

app <- init(
  data = data,
  modules = modules(
    
    tm_t_summary(
      label          = "Demographics",
      dataname       = "ADSL",
      arm_var        = choices_selected("TRT01A", selected = "TRT01A"),
      summarize_vars = choices_selected(
        c("AGE", "SEX", "RACE", "AGEGR1", "BMIBL"),
        selected = c("AGE", "SEX", "RACE")
      )
    ),
    
    tm_g_km(
      label    = "Kaplan-Meier",
      dataname = "ADTTE",
      arm_var  = choices_selected("TRT01A", selected = "TRT01A"),
      paramcd  = choices_selected(
        choices  = value_choices("ADTTE", "PARAMCD", "PARAM"),
        selected = "OS"
      )
    )
  ),
  title = "PHARM-001 Clinical Data Explorer"
)

shinyApp(app$ui, app$server)
```

---

## The Full PHARM-001 Explorer

```r
# pharm001_ch10_app.R
# Study PHARM-001 — Chapter 10
# Interactive clinical data explorer

library(teal)
library(teal.modules.clinical)
library(pharmaverseadam)
library(dplyr)

# ---- Load data ----
adsl  <- pharmaverseadam::adsl
adae  <- pharmaverseadam::adae
adtte <- pharmaverseadam::adtte
adlb  <- pharmaverseadam::adlb

data <- teal_data(
  ADSL  = adsl,
  ADAE  = adae,
  ADTTE = adtte,
  ADLB  = adlb
)

# ---- Build app ----
app <- init(
  data = data,
  
  modules = modules(
    
    # Tab 1: Demographics
    tm_t_summary(
      label          = "Demographics & Baseline",
      dataname       = "ADSL",
      arm_var        = choices_selected(
        choices  = variable_choices("ADSL", c("TRT01A", "TRT01P")),
        selected = "TRT01A"
      ),
      summarize_vars = choices_selected(
        choices  = variable_choices("ADSL",
                     c("AGE","AGEGR1","SEX","RACE","ETHNIC","BMIBL","HEIGHTBL","WEIGHTBL")),
        selected = c("AGE", "AGEGR1", "SEX", "RACE")
      )
    ),
    
    # Tab 2: AE Incidence
    tm_t_events(
      label      = "AE Incidence by SOC/PT",
      dataname   = "ADAE",
      arm_var    = choices_selected("TRT01A", selected = "TRT01A"),
      hlt        = choices_selected(
        choices  = variable_choices("ADAE", c("AEBODSYS", "AEHLT")),
        selected = "AEBODSYS"
      ),
      llt        = choices_selected(
        choices  = variable_choices("ADAE", "AEDECOD"),
        selected = "AEDECOD"
      ),
      add_total  = TRUE,
      event_type = "adverse events"
    ),
    
    # Tab 3: AE Overview
    tm_t_events_summary(
      label    = "AE Overview",
      dataname = "ADAE",
      arm_var  = choices_selected("TRT01A", selected = "TRT01A"),
      flag_var_anl = choices_selected(
        choices  = variable_choices("ADAE", c("TRTEMFL", "AESER")),
        selected = c("TRTEMFL", "AESER")
      )
    ),
    
    # Tab 4: Kaplan-Meier
    tm_g_km(
      label    = "Kaplan-Meier Plot",
      dataname = "ADTTE",
      arm_var  = choices_selected("TRT01A", selected = "TRT01A"),
      paramcd  = choices_selected(
        choices  = value_choices("ADTTE", "PARAMCD", "PARAM"),
        selected = "OS"
      )
    ),
    
    # Tab 5: Lab Values
    tm_g_lineplot(
      label    = "Lab Value by Visit",
      dataname = "ADLB",
      arm_var  = choices_selected("TRT01A", selected = "TRT01A"),
      paramcd  = choices_selected(
        choices  = value_choices("ADLB", "PARAMCD", "PARAM"),
        selected = "ALT"
      ),
      visit_var = choices_selected(
        choices  = variable_choices("ADLB", "AVISIT"),
        selected = "AVISIT"
      )
    )
  ),
  
  title  = "PHARM-001 Clinical Data Explorer",
  footer = paste("pharmaversesdtm + pharmaverseadam | teal", packageVersion("teal"))
)

cat("Launching PHARM-001 explorer at http://localhost:3838/\n")
shinyApp(app$ui, app$server, options = list(port = 3838, launch.browser = TRUE))
```

---

## The Filter Panel

The filter panel is teal's key feature. Users can filter across all modules simultaneously. To pre-set defaults:

```r
# Pre-filter safety population
# In the app, users see this filter already applied
# They can remove or add more filters
```

Filters propagate automatically — if you filter to females only in the panel, every table and plot updates. This replaces the manual re-running of SAS programs for subgroup reviews.

> **SAS mental model:** Imagine adding `WHERE SEX = "F"` to every single SAS program — `PROC REPORT`, `PROC LIFETEST`, `PROC MEANS` — and re-running them all. That's what you do in SAS for a subgroup review. In teal, you change the filter panel once and everything updates instantly.

---

## Available Modules

```r
# Demographics
tm_t_summary()           # summary statistics
tm_t_demographics()      # demographics table

# Adverse Events
tm_t_events()            # AE incidence by SOC/PT
tm_t_events_summary()    # AE overview
tm_t_max_grade_tbl()     # max grade table

# Time-to-Event
tm_g_km()               # Kaplan-Meier plot
tm_t_tte()              # TTE summary table

# Labs
tm_g_lineplot()         # lab value over time
tm_t_shift_by_arm()     # shift table

# Exploration
tm_t_crosstab()         # cross-tabulation
tm_g_scatterplot()      # scatterplot
tm_g_ipp()             # individual patient profile
```

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
pharm001_ch09_export.R   — export all ADaM to XPT
pharm001_ch10_app.R      — interactive teal explorer (5 modules)
output/adam/
  adsl.xpt, adae.xpt, adtte.xpt, adlb.xpt
```

---

## Exercises

**1.** Add `tm_t_crosstab()` to cross-tabulate AE incidence by sex and treatment arm.

**2.** Add `tm_g_ipp()` (individual patient profile) for lab values. This shows one subject's values over time — useful for medical review.

**3.** Write a custom teal module that shows a barplot of AE frequency by body system:

```r
my_ae_barplot <- module(
  label    = "AE Barplot",
  dataname = "ADAE",
  server   = function(id, data, filter_panel_api, reporter) {
    moduleServer(id, function(input, output, session) {
      output$plot <- renderPlot({
        df <- data[["ADAE"]]()
        df %>%
          filter(TRTEMFL == "Y") %>%
          count(AEBODSYS) %>%
          ggplot2::ggplot(ggplot2::aes(x = reorder(AEBODSYS, n), y = n)) +
          ggplot2::geom_col() +
          ggplot2::coord_flip() +
          ggplot2::labs(x = NULL, y = "Subject count", title = "TEAEs by System Organ Class")
      })
    })
  },
  ui = function(id) {
    ns <- NS(id)
    ggplot2::plotOutput(ns("plot"))
  }
)
```

**4. (Sets up Chapter 11)** Run `pharm001_ch10_app.R` and interact with it. Use the "Show R code" button on a module. Copy that code and run it outside the app. Does it reproduce the same output? This is the reproducibility feature. Chapter 11 adds logging so every program run — including the app — leaves an audit trail.

---

*Next: Chapter 11 — we add logrx logging to every PHARM-001 program. Show what the log catches.*
