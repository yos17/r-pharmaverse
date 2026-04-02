# Chapter 13: Debugging Pharmaverse Pipelines

*browser() inside admiral chains, diagnosing join explosions, tracing rtables errors.*

---

## Yes, browser() Works Everywhere

`browser()` is just R — it works inside any function, any package, any pipe chain. The pharmaverse doesn't change that. What *is* different is *where* bugs typically hide in clinical programming:

- Silent NA propagation after a `left_join`
- Row count explosions (one-to-many join)
- Wrong date imputation
- admiral derivation producing unexpected values
- rtables crashing on a group with zero records
- xportr type coercion silently dropping values

Let's go through each.

---

## Debugging Inside a dplyr/admiral Pipeline

The pipe makes code readable but debugging harder — you can't easily stop mid-chain.

**Technique 1: Break the pipe**

```r
# This crashes somewhere — but where?
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC) %>%
  derive_vars_dt(new_vars_prefix="TRTE", dtc=RFXENDTC) %>%
  mutate(TRTDURD = as.integer(TRTEDT - TRTSDT + 1)) %>%
  left_join(ex_derived, by="USUBJID") %>%
  mutate(SAFFL = if_else(dosed, "Y", "N"))

# Break it at the suspected step:
step1 <- dm %>% filter(ACTARM != "Screen Failure")
step2 <- step1 %>% derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC)
step3 <- step2 %>% derive_vars_dt(new_vars_prefix="TRTE", dtc=RFXENDTC)
# → find the step that crashes, then browser() inside it
```

**Technique 2: Inject browser() mid-pipe with a helper**

```r
# A tap function: run a side-effect mid-pipe without changing the data
tap <- function(df, expr) {
  expr <- substitute(expr)
  eval(expr, envir = list(df = df), enclos = parent.frame())
  df   # return unchanged
}

# Use it:
adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  tap(cat("After filter:", nrow(df), "rows\n")) %>%
  derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC) %>%
  tap(cat("After TRTS derive, NAs:", sum(is.na(df$TRTSDT)), "\n")) %>%
  left_join(ex_derived, by="USUBJID") %>%
  tap(if (nrow(df) > 300) browser()) %>%   # stop if rows exploded
  mutate(SAFFL = if_else(dosed, "Y", "N"))
```

**Technique 3: `%T>%` (tee pipe)**

```r
library(magrittr)

adsl <- dm %>%
  filter(ACTARM != "Screen Failure") %>%
  derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC) %T>%
  { cat("TRTS NAs:", sum(is.na(.$TRTSDT)), "\n") } %>%    # side effect, passes df through
  derive_vars_dt(new_vars_prefix="TRTE", dtc=RFXENDTC)
```

---

## The Most Common Bug: Join Row Explosion

You have 200 subjects. After a `left_join`, you suddenly have 850 rows. This is the #1 silent bug in clinical programming.

```r
# Danger: ex has multiple rows per subject
adsl %>% left_join(ex, by = "USUBJID")   # 200 subjects × avg 4 doses = 800 rows!
```

**Diagnose:**

```r
# Before any join, always check the join key is unique on the right side
ex %>% count(USUBJID) %>% filter(n > 1)
# If this has rows, you need to aggregate ex first

# Add a row count assertion around every join
safe_join <- function(left, right, by, type = "left") {
  n_before <- nrow(left)
  
  result <- switch(type,
    left  = left_join(left, right, by = by),
    inner = inner_join(left, right, by = by),
    full  = full_join(left, right, by = by)
  )
  
  n_after <- nrow(result)
  if (n_after != n_before) {
    stop(sprintf(
      "Join on '%s' changed row count: %d → %d (+%d). Right side is not unique!",
      paste(by, collapse=", "), n_before, n_after, n_after - n_before
    ))
  }
  result
}

# Now use it:
adsl <- adsl %>% safe_join(ex_first_dose, by = "USUBJID")
# Crashes loudly if ex_first_dose has duplicates
```

---

## Debugging NA Propagation

You derive `TRTSDT`. Some subjects have NA. Why?

```r
# Step 1: find which subjects
na_subjects <- adsl %>% filter(is.na(TRTSDT)) %>% select(USUBJID, RFXSTDTC)
print(na_subjects)
# Likely: RFXSTDTC is NA or malformed

# Step 2: check the source
dm %>%
  filter(USUBJID %in% na_subjects$USUBJID) %>%
  select(USUBJID, RFXSTDTC, RFSTDTC, RFENDTC)
# RFXSTDTC is NA for these subjects → they never received treatment
# That's expected! But confirm it's intentional.

# Step 3: if unexpected, use browser() at the derivation
derive_and_check <- function(df) {
  result <- df %>%
    derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC, date_imputation="first")
  
  na_rows <- result %>% filter(is.na(TRTSDT), ACTARM != "Screen Failure")
  if (nrow(na_rows) > 0) {
    cat("Unexpected NAs in TRTSDT for non-screen-failure subjects:\n")
    print(na_rows %>% select(USUBJID, RFXSTDTC, ACTARM))
    browser()   # stop and investigate
  }
  result
}
```

---

## browser() Inside admiral derive Functions

You can debug into admiral's own source code:

```r
# Pause at the start of derive_vars_dt itself
debugonce(admiral::derive_vars_dt)

adsl <- adsl %>%
  derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC)
# → pauses inside derive_vars_dt
# Browse[1]> ls()   # see admiral's internal variables
# Browse[1]> n      # step through line by line
```

Or use `trace()` to inject `browser()` at a specific line inside an admiral function:

```r
# Inject browser at line 50 of derive_vars_dt
trace("derive_vars_dt", tracer = browser, at = 50, where = asNamespace("admiral"))
adsl %>% derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC)
untrace("derive_vars_dt", where = asNamespace("admiral"))
```

---

## Program: Debugging ADSL Derivation

Here's a realistic scenario: your ADSL has wrong `TRTDURD` for some subjects. Let's find it.

```r
# debug_adsl.R
library(admiral)
library(pharmaversesdtm)
library(dplyr)

dm <- pharmaversesdtm::dm
ex <- pharmaversesdtm::ex

# ---- Deliberately introduce a bug ----
# Pretend one subject's RFXENDTC was entered wrong (year typo)
dm_buggy <- dm %>%
  mutate(
    RFXENDTC = if_else(USUBJID == "01-701-1015",
                        "2014-01-02",   # should be 2014-07-02 — month typo!
                        RFXENDTC)
  )

# ---- Derive ADSL ----
adsl <- dm_buggy %>%
  filter(ACTARM != "Screen Failure") %>%
  derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC, date_imputation="first") %>%
  derive_vars_dt(new_vars_prefix="TRTE", dtc=RFXENDTC,  date_imputation="last") %>%
  mutate(TRTDURD = as.integer(TRTEDT - TRTSDT + 1))

# ---- Detect the bug ----
cat("=== TRTDURD Sanity Checks ===\n")

# Check 1: no negative durations
neg_dur <- adsl %>% filter(!is.na(TRTDURD) & TRTDURD <= 0)
if (nrow(neg_dur) > 0) {
  cat(sprintf("[FAIL] %d subjects with TRTDURD <= 0:\n", nrow(neg_dur)))
  print(neg_dur %>% select(USUBJID, TRTSDT, TRTEDT, TRTDURD))
} else {
  cat("[PASS] No negative durations\n")
}

# Check 2: durations are plausible (< 2 years for this study)
long_dur <- adsl %>% filter(!is.na(TRTDURD) & TRTDURD > 730)
if (nrow(long_dur) > 0) {
  cat(sprintf("\n[WARN] %d subjects with TRTDURD > 730 days:\n", nrow(long_dur)))
  print(long_dur %>% select(USUBJID, TRTSDT, TRTEDT, TRTDURD))
}

# Check 3: compare with expected range from protocol
# Protocol says treatment 24-48 weeks → 168-336 days
out_of_range <- adsl %>%
  filter(ACTARM != "Screen Failure", !is.na(TRTDURD)) %>%
  filter(TRTDURD < 50 | TRTDURD > 400)   # flag obvious outliers

if (nrow(out_of_range) > 0) {
  cat(sprintf("\n[WARN] %d subjects outside expected duration range:\n",
              nrow(out_of_range)))
  print(out_of_range %>% select(USUBJID, ACTARM, TRTSDT, TRTEDT, TRTDURD))
  
  # Now use browser() to investigate the first one
  suspect <- out_of_range$USUBJID[1]
  cat(sprintf("\nInvestigating %s...\n", suspect))
  
  # What's in raw DM?
  cat("Raw DM for this subject:\n")
  dm_buggy %>% filter(USUBJID == suspect) %>%
    select(USUBJID, RFXSTDTC, RFXENDTC) %>% print()
  
  # What's in EX?
  cat("EX records:\n")
  ex %>% filter(USUBJID == suspect) %>%
    select(USUBJID, EXTRT, EXSTDTC, EXENDTC) %>% print()
  
  # Uncomment to drop into interactive debugger:
  # browser()
}

# ---- Cross-check against EX ----
cat("\n=== Cross-check TRTSDT vs first EX record ===\n")

ex_first <- ex %>%
  mutate(EXSTDT = as.Date(substr(EXSTDTC, 1, 10))) %>%
  group_by(USUBJID) %>%
  summarise(EXSTDT_FIRST = min(EXSTDT, na.rm=TRUE), .groups="drop")

mismatch <- adsl %>%
  left_join(ex_first, by="USUBJID") %>%
  filter(!is.na(TRTSDT), !is.na(EXSTDT_FIRST)) %>%
  filter(abs(as.integer(TRTSDT - EXSTDT_FIRST)) > 1)  # allow 1-day imputation diff

if (nrow(mismatch) > 0) {
  cat(sprintf("[WARN] %d subjects where TRTSDT doesn't match first EX date:\n",
              nrow(mismatch)))
  print(mismatch %>% select(USUBJID, TRTSDT, EXSTDT_FIRST) %>%
    mutate(diff = as.integer(TRTSDT - EXSTDT_FIRST)))
} else {
  cat("[PASS] TRTSDT matches first EX record for all subjects\n")
}
```

Run this — it will catch the injected bug (RFXENDTC month typo makes TRTDURD very small) and show you exactly which subject and what the raw values were.

---

## Debugging rtables Errors

rtables errors are often cryptic. Common ones:

```r
# "Error: 'by' variable has 0 levels in this subset"
# → a treatment arm has been completely filtered out before build_table

# Fix: check your data going in
cat("Arms in data:", unique(adsl_safe$TRT01A), "\n")
cat("N per arm:\n"); print(table(adsl_safe$TRT01A))
# → one arm might have 0 rows after filter

# "Error in split_cols_by: column variable not found"
# → column name typo, or wrong dataname

# Debug: build piece by piece
lyt <- basic_table() %>% split_cols_by("TRT01A")
# Does this work?
test_tbl <- build_table(lyt, adsl_safe)
# Add one row at a time
lyt <- lyt %>% analyze_vars("AGE")
test_tbl <- build_table(lyt, adsl_safe)   # crash? → AGE might be wrong type
```

**For rtables: always check the data *before* passing to build_table:**

```r
pre_flight_check <- function(df, col_var, analyze_vars) {
  # Column variable exists and has levels
  stopifnot(col_var %in% names(df))
  levels <- unique(df[[col_var]])
  cat(sprintf("Column split '%s': %d levels — %s\n",
              col_var, length(levels), paste(levels, collapse=", ")))
  stopifnot(length(levels) > 0)
  
  # Analyze variables exist and have correct type
  for (v in analyze_vars) {
    stopifnot(v %in% names(df))
    cat(sprintf("  Var '%s': %s, %d NAs\n",
                v, class(df[[v]]), sum(is.na(df[[v]]))))
  }
}

pre_flight_check(adsl_safe, "TRT01A", c("AGE","SEX","RACE"))
tbl <- build_table(lyt, adsl_safe)
```

---

## Debugging xportr

```r
# "Error: Variable X has type 'character' but spec says 'float'"
# → a variable that should be numeric was read as character

# Find the problem:
adsl %>%
  select(where(is.character)) %>%
  names() %>%
  intersect(var_spec %>% filter(type == "float") %>% pull(variable))
# → shows which "float" variables are currently character

# Fix: coerce before xportr
adsl <- adsl %>%
  mutate(AGE = as.numeric(AGE))

# "Warning: Variable label truncated to 200 chars"
# → check your label lengths
var_spec %>%
  mutate(label_len = nchar(label)) %>%
  filter(label_len > 200) %>%
  select(variable, label, label_len)
```

---

## The Quick Reference

```r
# ── STOP AND LOOK ──────────────────────────────
browser()                        # pause here, open REPL
browser(expr = condition)        # pause only when condition is TRUE
debugonce(some_function)         # pause at start of function, once
options(error = recover)         # after any error, pick a frame to explore
options(error = NULL)            # reset

# ── FIND WHERE IT CRASHED ──────────────────────
traceback()                      # call stack after error

# ── INJECT INTO A PIPE ─────────────────────────
df %>% { browser(); . }          # pause mid-pipe (. is the data)
df %>% (function(x) { cat(nrow(x), "\n"); x })()  # print mid-pipe

# ── HANDLE ERRORS ──────────────────────────────
tryCatch(expr, error = function(e) ...)
withCallingHandlers(expr, warning = function(w) ...)

# ── CRASH LOUDLY ───────────────────────────────
stopifnot(nrow(adsl) == 254)
stop("Expected 254 subjects, got ", nrow(adsl))

# ── CLINICAL-SPECIFIC ──────────────────────────
# Check join uniqueness before joining
before <- nrow(left_df)
result <- left_join(left_df, right_df, by = "USUBJID")
stopifnot("Join multiplied rows" = nrow(result) == before)

# Check no unexpected NAs in flag variables
stopifnot(all(adsl$ITTFL %in% c("Y", NA)))

# Check date ordering
bad_dates <- adsl %>% filter(!is.na(TRTSDT) & !is.na(TRTEDT) & TRTEDT < TRTSDT)
if (nrow(bad_dates) > 0) stop(nrow(bad_dates), " subjects with TRTEDT < TRTSDT")
```

---

## Exercises

**1. Inject browser() mid-admiral-chain**

Take the ADSL derivation from Chapter 5. After `derive_vars_dt(TRTS)`, inject `browser()` using the `{ browser(); . }` trick. Confirm you can inspect `TRTSDT` at that point.

**2. Build safe_join()**

Implement `safe_join(left, right, by, type="left")` that:
- Performs the join
- Crashes with a useful message if row count changes
- Returns the joined dataset if counts match

Use it for every join in your ADSL.

**3. rtables pre-flight**

Build `preflight_check(df, lyt)` that extracts all variables referenced in a layout and checks they exist in `df` with the right types.

**4. diffdf on a buggy ADSL**

Introduce 3 bugs into `pharmaverseadam::adsl` (swap TRTSDT for two subjects, change one AGEGR1, set one SAFFL to "N" wrongly). Run `diffdf()` to detect all three.

---

*Debugging is the same skill everywhere. The pharmaverse doesn't change the tools — just the typical bugs.*
