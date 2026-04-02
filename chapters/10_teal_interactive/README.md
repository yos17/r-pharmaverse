# Chapter 10: Interactive Apps with {teal}

*Build clinical data exploration apps using the teal Shiny framework.*

---

## What teal Is

`teal` is a Shiny-based framework for building interactive clinical trial data exploration apps. A teal app gives reviewers (statisticians, clinicians, regulators) an interactive interface to explore ADaM data without writing code.

Built by Roche's engineering team (now insightsengineering), it's widely used internally and increasingly in submissions for exploratory interactive analyses.

Features:
- **Filter panel** — global subject filters (treatment arm, population flag, demographics)
- **Analysis modules** — pluggable analysis views (tables, KM plots, scatterplots, listings)
- **Reproducibility code** — every analysis generates copy-pasteable R code
- **Reporter** — export results to a PDF/Word report in-app

```r
library(teal)
library(teal.modules.clinical)
library(pharmaverseadam)
library(dplyr)
```

---

## A Minimal teal App

```r
library(teal)
library(teal.modules.clinical)
library(pharmaverseadam)

# Load data
adsl <- pharmaverseadam::adsl
adae <- pharmaverseadam::adae

# Define data connectors (how teal loads the data)
data <- teal_data(
  ADSL = adsl,
  ADAE = adae
)

# Define app
app <- init(
  data = data,
  
  modules = modules(
    # Demographics summary
    tm_t_summary(
      label        = "Demographics",
      dataname     = "ADSL",
      arm_var      = choices_selected("TRT01A", selected = "TRT01A"),
      summarize_vars = choices_selected(
        c("AGE","SEX","RACE","AGEGR1"),
        selected = c("AGE","SEX","RACE")
      )
    ),
    
    # AE table
    tm_t_events(
      label       = "Adverse Events",
      dataname    = "ADAE",
      arm_var     = choices_selected("TRT01A", selected = "TRT01A"),
      hlt         = choices_selected("AEBODSYS"),
      llt         = choices_selected("AEDECOD"),
      add_total   = TRUE
    )
  ),
  
  title   = "Clinical Data Explorer",
  footer  = "Pharmaverse Example"
)

# Run
shinyApp(app$ui, app$server)
```

---

## The Filter Panel

The filter panel is teal's signature feature. Users can filter across all modules simultaneously:

```r
# You can pre-set default filters
data <- teal_data(
  ADSL = adsl,
  ADAE = adae
)

# Users can add filters like:
# ADSL: SAFFL == "Y"
# ADSL: TRT01A %in% c("Drug A", "Placebo")
# ADAE: AESER == "Y"
```

Filters propagate automatically to every module — if you filter to females only, every table and plot updates.

---

## teal.modules.clinical — Available Modules

```r
# Demographics / Summary
tm_t_summary()           # summary statistics table
tm_t_demographics()      # demographics table

# Adverse Events
tm_t_events()            # AE incidence table by SOC/PT
tm_t_events_summary()    # AE overview
tm_t_max_grade_tbl()     # maximum grade table

# Time-to-Event
tm_g_km()               # Kaplan-Meier plot
tm_t_tte()              # TTE summary table

# Labs
tm_g_lineplot()         # lab value over time
tm_t_shift_by_arm()     # shift table (baseline vs worst)
tm_t_summary_by()       # summary by visit

# Exploration
tm_t_crosstab()         # cross-tabulation
tm_g_scatterplot()      # scatterplot
tm_g_ipp()              # individual patient profile
```

---

## Program: Full Clinical Explorer App

```r
# clinical_explorer.R
# A complete teal clinical data exploration app

library(teal)
library(teal.modules.clinical)
library(pharmaverseadam)
library(dplyr)

# ---- Load and prepare data ----
adsl  <- pharmaverseadam::adsl
adae  <- pharmaverseadam::adae
adtte <- pharmaverseadam::adtte
adlb  <- pharmaverseadam::adlb

# Define data
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
    
    # ---- Tab: Subject Overview ----
    tm_t_summary(
      label        = "Demographics & Baseline",
      dataname     = "ADSL",
      arm_var      = choices_selected(
        choices  = variable_choices("ADSL", c("TRT01A","TRT01P")),
        selected = "TRT01A"
      ),
      summarize_vars = choices_selected(
        choices  = variable_choices("ADSL", c("AGE","AGEGR1","SEX","RACE","ETHNIC",
                                              "BMIBL","HEIGHTBL","WEIGHTBL")),
        selected = c("AGE","AGEGR1","SEX","RACE")
      ),
      useNA = "ifany"
    ),
    
    # ---- Tab: Adverse Events ----
    tm_t_events(
      label     = "AE Incidence by SOC/PT",
      dataname  = "ADAE",
      arm_var   = choices_selected("TRT01A", selected = "TRT01A"),
      hlt       = choices_selected(
        choices = variable_choices("ADAE", c("AEBODSYS","AEHLT")),
        selected = "AEBODSYS"
      ),
      llt = choices_selected(
        choices  = variable_choices("ADAE", "AEDECOD"),
        selected = "AEDECOD"
      ),
      add_total  = TRUE,
      event_type = "adverse events"
    ),
    
    # ---- Tab: AE Overview ----
    tm_t_events_summary(
      label    = "AE Overview",
      dataname = "ADAE",
      arm_var  = choices_selected("TRT01A", selected = "TRT01A"),
      flag_var_anl = choices_selected(
        choices  = variable_choices("ADAE", c("TRTEMFL","AESER")),
        selected = c("TRTEMFL","AESER")
      )
    ),
    
    # ---- Tab: Survival ----
    tm_g_km(
      label      = "Kaplan-Meier Plot",
      dataname   = "ADTTE",
      arm_var    = choices_selected("TRT01A", selected = "TRT01A"),
      paramcd    = choices_selected(
        choices  = value_choices("ADTTE", "PARAMCD", "PARAM"),
        selected = "OS"
      ),
      xticks     = NULL,
      conf_level = teal.transform::arm_var_choices_selected(list(ADSL = adsl)),
      time_unit  = "AVALU"
    ),
    
    # ---- Tab: Lab Values ----
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
  
  title  = "Clinical Data Explorer — Phase III Study",
  footer = paste("Data: pharmaverseadam | Built with teal", packageVersion("teal"))
)

# Launch
cat("Launching app at http://localhost:3838/\n")
cat("Press Ctrl+C to stop.\n")
shinyApp(app$ui, app$server, options = list(port = 3838, launch.browser = TRUE))
```

---

## Deploying teal Apps

```r
# Deploy to Posit Connect (shinyapps.io or internal server)
library(rsconnect)

rsconnect::deployApp(
  appDir    = ".",
  appFiles  = c("clinical_explorer.R"),
  appName   = "clinical-explorer",
  account   = "your-account",
  server    = "your-server"
)
```

For internal deployment on Posit Connect, ask your IT team for the server URL and API token.

---

## Exercises

**1. Subgroup analysis module**

Add `tm_t_crosstab()` to the app: cross-tabulate AE incidence by sex and treatment arm.

**2. Individual patient profile**

Add `tm_g_ipp()` (individual patient profile) for lab values. This shows one subject's values over time.

**3. Custom module**

Write a custom teal module using `module()` that shows a barplot of AE frequency by body system. Use `renderPlot()` and `plotOutput()` from Shiny.

```r
my_ae_barplot <- module(
  label    = "AE Barplot",
  dataname = "ADAE",
  server   = function(id, data, filter_panel_api, reporter) {
    moduleServer(id, function(input, output, session) {
      output$plot <- renderPlot({
        # your ggplot code here
      })
    })
  },
  ui = function(id) {
    ns <- NS(id)
    plotOutput(ns("plot"))
  }
)
```

**4. Pre-filter populations**

Modify the app so the filter panel defaults to:
- `ADSL`: `SAFFL == "Y"`
- `ADAE`: `TRTEMFL == "Y"`

---

*Next: Chapter 11 — Validation, Logging, and QC*
