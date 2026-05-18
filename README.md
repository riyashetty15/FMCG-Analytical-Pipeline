# FMCG-Analytical-Pipeline

## Overview
 
A 3-year daily sales dataset covering two retail stores across
Germany and Italy. Contains 1,000,000 rows of transaction-level
FMCG data with rich contextual features including weather,
promotions, stock levels, and supplier information.
 
Designed for practising: data quality investigation, root cause
analysis, time series trend analysis, promotional effectiveness,
and statistical testing.
 
---
### Dataset: [FMCG Sales 3 Years 1M Rows - Kaggle](https://www.kaggle.com/datasets/robertocarlost/fmcg-multi-country-sales-dataset)
---
## What This Project Does
 
This project replicates the core analytical workflows of a
data quality and methods team working with large-scale retail
data. Rather than building a predictive model, the focus is on
the unglamorous but critical work that makes any downstream
analysis trustworthy, finding problems in data, diagnosing
why they exist, fixing them systematically, and producing
reliable metrics that reflect reality.
 
The pipeline moves through five logical stages:
 
```
Raw data → DQ audit → Cleaning → Core metrics → Investigation → Reporting
```
 
Each stage is implemented as a reproducible, reusable R pipeline
with defensive checks at every step.
 
---
 
## Implementation
 
### 1. Data Quality Audit
 
**What was done:**
Before touching any analysis, a structured audit was run to
document every data integrity issue, counts, percentages,
and severity, without removing a single row. The professional
principle here is: document first, fix second. Silently dropping
rows hides problems that may recur.
 
**How it was implemented:**
A `tibble()` was manually constructed where each row represents
one check and each column represents the result. The `Count`
column runs live calculations using vectorised logical operations:
 
- `sum(!complete.cases(df))` — counts rows with any missing value
- `sum(df$margin_pct < 0)` — flags below-cost selling
- `sum(df$stock_out_flag == 1 & df$units_sold > 0)` — detects
  logical contradictions using `&` to combine two conditions
- `sum(abs(gross  units * price) > 0.02)` — validates the
  mathematical relationship between columns
A `case_when()` then classifies each check as Pass, Review,
or Info — producing a structured audit table rather than
scattered `cat()` statements.
 
**Key R concepts:** `tibble()`, `complete.cases()`, vectorised
logical operations, `case_when()`, `paste0()` for formatting.
 
---
 
### 2. Data Cleaning and Feature Engineering
 
**What was done:**
Issues found in the audit were resolved systematically —
each step documented, each decision justified. The key
principle applied was **cap rather than remove** for outliers,
preserving data integrity while controlling their influence.
New analytical columns were derived from existing ones to
enable richer downstream analysis.
 
**How it was implemented:**
A single chained `mutate()` pipeline handles all transformations
in one pass through the data:
 
- `as.Date()` converts the character date column to a proper
  Date type, enabling all lubridate operations downstream
- `pmin(spend, cap_value)` caps outliers at the 99th percentile —
  `pmin()` is element-wise minimum, safer than `ifelse()` for this
- `isoweek()`, `quarter()`, `month()` extract time components
  for grouping and trend analysis
- `case_when()` builds categorical features: season, temperature
  band, and promotion type — all derived from existing columns
  rather than hardcoded lookups
A `stopifnot(nrow(clean) == nrow(raw))` check after cleaning
confirms no rows were accidentally dropped — a defensive
programming pattern that catches silent data loss.
 
**Key R concepts:** `mutate()`, `as.Date()`, `pmin()`,
`lubridate` date functions, `case_when()`, `stopifnot()`.
 
---
 
### 3. Core Metric Calculation
 
**What was done:**
Five standard retail metrics were calculated, revenue share,
brand market share, average margin, promotional rate, and
stock-out rate, broken down by category, brand, and store.
Each metric was calculated both at the raw level and as a
proportion of a meaningful denominator.
 
**How it was implemented:**
The core pattern throughout is a two-stage `group_by()`:
 
```r
# Stage 1: aggregate within groups
group_by(category, brand) %>%
summarise(net_sales = sum(net_sales), .groups = "drop") %>%
 
# Stage 2: calculate proportions across groups
group_by(category) %>%
mutate(share_pct = net_sales / sum(net_sales) * 100) %>%
ungroup()
```
 
The second `group_by()` after `summarise()` is the critical
pattern — it allows `sum(net_sales)` in `mutate()` to refer
to the category total rather than the grand total, giving
share within category not share of everything.
 
`across()` was used to apply the same outlier flagging function
to multiple columns simultaneously with `.names = "flag_{.col}"`
generating consistently named output columns automatically.
 
`quantile(x, 0.80)` was used to define performance tier
thresholds dynamically — the top 20% cutoff adapts to the
actual data distribution rather than a hardcoded number.
 
**Key R concepts:** two-stage `group_by()`, `summarise()`,
`mutate()` after summarise for proportions, `across()`,
`quantile()`, `ungroup()`.
 
---
 
### 4. Joins and Table Construction
 
**What was done:**
A separate SKU-level reference table was built from the
transaction data, enriched with a derived price tier, then
joined back onto transactions. Three join types were used
for three different purposes.
 
**How it was implemented:**
`distinct()` collapsed 1M transaction rows to 84 unique SKUs
by specifying the columns that define product identity —
excluding transactional columns like `units_sold` and `date`
which vary by row. The result was validated immediately:
 
```r
stopifnot(nrow(sku_master) == n_distinct(fmcg$sku_id))
```
 
If this fails, a price change created duplicate SKU records,
a many-to-many join risk that this check catches before it
silently inflates downstream row counts.
 
Three joins were used with different intentions:
- `left_join()` — enrich all transactions with price tier,
  keeping every row regardless of match
- `anti_join()` — find transactions with no matching SKU
  record, surfacing data gaps
- `inner_join()` — restrict analysis to Premium/Luxury SKUs
  only, intentionally dropping unmatched rows
`relationship = "many-to-one"` was declared on the left join
to make R enforce key uniqueness in the lookup table — it
throws an error rather than silently inflating rows if the
assumption is violated.
 
**Key R concepts:** `distinct()`, `left_join()`, `anti_join()`,
`inner_join()`, `relationship` argument, `stopifnot()` for
join validation.
 
---
 
### 5. Reshaping
 
**What was done:**
Data was reshaped between long and wide formats for different
analytical purposes, long for time series operations and
visualisation, wide for client-facing output tables and
side-by-side comparisons.
 
**How it was implemented:**
`pivot_wider()` was used to build a brand × month matrix
(6 brands × 36 months = 216 cells) for a high-level revenue
overview. `values_fill = 0` was specified to replace NA with
zero for months with no sales, preventing downstream
calculation failures.
 
`pivot_longer()` was then applied to reshape the wide matrix
back to long format, not as a reversal, but because long
format is required for `lag()` to compute month-on-month
growth. This demonstrates the key principle: **wide format
for display, long format for analysis**.
 
The `lag()` function computes month-on-month growth by
looking at the previous row's value. This only works correctly
after `arrange(brand, yr_month)` sorts chronologically, and
inside `group_by(brand)` so the lag restarts at NA for each
new brand rather than crossing brand boundaries.
 
**Key R concepts:** `pivot_wider()`, `pivot_longer()`,
`values_fill`, `lag()`, `arrange()` before position-dependent
operations, `group_by()` to contain `lag()`.
 
---
 
### 6. Reusable Functions with Tidy Evaluation
 
**What was done:**
Instead of writing separate pipelines for each category,
brand, or metric, two generalised functions were built that
accept any column as an argument, enabling the same analysis
to run across any segment or metric with a single call.
 
**How it was implemented:**
The `{{ }}` (curly-curly) operator from rlang enables tidy
evaluation — passing column names as function arguments to
dplyr verbs. Without it, `group_by(group_col)` would look
for a column literally called "group_col". With it, R unwraps
the argument and uses whatever column name was passed:
 
```r
segment_report <- function(df, group_col, min_rows = 100) {
  df %>%
    group_by({{ group_col }}) %>%
    summarise(
      total_net_sales = sum(net_sales),
      avg_margin      = mean(margin_pct),
      .groups = "drop"
    )
}
 
# Same function, any grouping:
segment_report(fmcg_clean, category)
segment_report(fmcg_clean, brand)
segment_report(fmcg_clean, season)
```
 
Default argument values (`min_rows = 100`) allow the function
to work with sensible defaults while remaining overridable.
`pull()` is used to extract single computed values from
pipelines as plain vectors rather than 1×1 tibbles.
 
**Key R concepts:** `{{ }}` tidy evaluation, default arguments,
`pull()`, function composition with `%>%`.
 
---
 
### 7. Rolling Window Anomaly Detection
 
**What was done:**
Weekly sales trends were modelled using rolling averages and
rolling standard deviations. Data points falling outside ±2
standard deviations of the rolling mean were flagged as
anomalies, distinguishing genuine spikes from normal
week-to-week variation.
 
**How it was implemented:**
`slide_dbl()` from the `slider` package computes rolling
window statistics. Two critical setup steps must precede it:
 
1. `arrange(category, date)` — `slide_dbl()` is
   position-dependent, operating on physically adjacent rows.
   Without sorting, it averages non-consecutive dates producing
   meaningless results.
2. `group_by(category)` before `mutate()` — makes the rolling
   window restart at the beginning of each category rather than
   running continuously across category boundaries.
```r
mutate(
  rolling_avg = slide_dbl(net_sales, mean, .before = 6),
  rolling_sd  = slide_dbl(net_sales, sd,   .before = 6),
  is_anomaly  = abs(net_sales - rolling_avg) > 2 * rolling_sd
)
```
 
The anomaly flag compares each week's distance from the
rolling mean against the rolling standard deviation, a
self-calibrating threshold that adapts to each category's
natural volatility rather than using a fixed absolute value.
 
**Key R concepts:** `slider::slide_dbl()`, `arrange()` before
rolling operations, `group_by()` to contain window functions,
self-calibrating thresholds.
 
---
 
### 8. Root Cause Analysis — 4-Step Workflow
 
**What was done:**
A structured investigation was conducted into a Beverages
revenue anomaly, following a systematic 4-step methodology:
isolate the anomaly, decompose by dimension, distinguish
cause from symptom, document findings and recommend action.
 
**How it was implemented:**
 
**Step 1 — Isolate:** Monthly totals were computed and
compared against their own mean ± 1.5 SD. `format(date, "%Y-%m")`
groups days into months without needing a separate date
truncation step.
 
**Step 2 — Decompose:** The anomaly month was compared to
the prior month across subcategories using `pivot_wider()` —
long format with two periods becomes a wide comparison table,
enabling `mutate(pct_chg = (target - prev) / prev * 100)`.
`format(as.Date(paste0(month, "-01")) - 1, "%Y-%m")` derives
the prior month without a lookup table.
 
**Step 3 — Distinguish cause:** `n_distinct(date)` counts
active trading days, `mean(promo_flag)` gives the promotional
rate, and `mean(stock_out_flag)` gives the stock-out rate —
all within the same `summarise()` call. Comparing these
between the anomaly month and the prior month distinguishes
a supply problem from a demand problem.
 
**Step 4 — Document:** `glue()` builds the summary message
by interpolating computed values directly into text,
producing a readable, data-driven conclusion rather than
a hardcoded string.
 
**Key R concepts:** `format()` for date grouping, `pivot_wider()`
for period comparison, `n_distinct()`, `glue()` for
parameterised conclusions.
 
---
 
### 9. Statistical Testing
 
**What was done:**
Three statistical tests were applied to answer specific
business questions — always reporting both statistical
significance (p-value) and practical significance (effect
size) together, since large datasets make almost everything
statistically significant regardless of practical relevance.
 
**How it was implemented:**
 
**Kruskal-Wallis** tested whether daily sales differ across
seasons. Non-parametric was chosen over ANOVA because sales
data is right-skewed, a few high-value days pull the mean
upward, violating ANOVA's normality assumption. Kruskal-Wallis
converts values to ranks first, making it robust to skew
and outliers.
 
**Pairwise Wilcoxon** followed up to identify which specific
season pairs differ. `p.adjust.method = "bonferroni"` corrects
for multiple comparisons, running 6 pairwise tests without
correction inflates the chance of false positives.
 
**Two-proportion z-test** was implemented from scratch to test
whether brand market share changed significantly between years.
The pooled proportion, standard error, z-statistic, and
two-tailed p-value were all calculated manually, demonstrating
the underlying maths rather than relying on a black-box function.
 
For all three tests, the median difference (not mean) was
reported as the effect size measure, appropriate for skewed
distributions where the mean is misleading.
 
**Key R concepts:** `kruskal.test()`, `pairwise.wilcox.test()`,
`p.adjust.method`, `pnorm()` for manual p-value calculation,
effect size alongside p-value.
 
---
 
### 10. Promotion Effectiveness Analysis
 
**What was done:**
Promotional uplift was quantified at SKU level, how much
more did each product sell on promotional days versus normal
days, and did the revenue uplift justify the margin compression?
 
**How it was implemented:**
Daily average units and revenue were calculated separately
for promo and non-promo days per SKU using `group_by(sku_id,
promo_flag)`. The results were then reshaped with `pivot_wider()`
to place promo and non-promo metrics side by side on the same
row, enabling a direct comparison with a single `mutate()`:
 
```r
mutate(
  unit_uplift_pct = (avg_units_promo - avg_units_base) /
                     avg_units_base * 100
)
```
 
`rename_with()` with `str_replace()` cleaned the auto-generated
column names from `pivot_wider()` into readable labels.
 
**Key R concepts:** two-condition `group_by()`, `pivot_wider()`
for side-by-side comparison, `rename_with()`, `str_replace()`.
 
---
 
### 11. Weather Impact Analysis
 
**What was done:**
Pearson correlation was calculated between daily temperature
and Beverages sales, with results broken down by temperature
band and season. This provides contextual explanation for
anomalies, a sales dip in August may reflect cold weather
rather than a real demand change.
 
**How it was implemented:**
`cor()` computes the Pearson correlation coefficient between
two numeric vectors. The result was interpreted against a
standard threshold (|r| > 0.3 = meaningful correlation)
and used to generate an automated plain-English conclusion
with `case_when()`.
 
A scatter plot with `geom_smooth(method = "lm")` overlaid the
linear trend, visually confirming or contradicting the
correlation coefficient.
 
**Key R concepts:** `cor()`, `geom_smooth()`, automated
interpretation with `case_when()`.
 
---
 
### 12. Visualisation Dashboard
 
**What was done:**
Four publication-ready charts were produced covering the
key analytical themes, revenue trends, promo margin impact,
weather correlation, and brand share evolution — combined
into a single dashboard image.
 
**How it was implemented:**
Each chart was built independently as a `ggplot2` object
and stored as a variable. `patchwork` then combines them
into a 2×2 grid with a shared title and caption using the
`(p1 | p2) / (p3 | p4)` layout syntax.
 
Specific design choices made deliberately:
- `scale_y_log10()` for spend distributions, log scale
  compresses the right tail, making the bulk of the
  distribution readable rather than squashed at the bottom
- `geom_ribbon()` for rolling average confidence bands,
  shows the ±2 SD range without obscuring the trend line
- `coord_flip()` for bar charts, long category names
  render horizontally without rotation or truncation
- `scale_fill_brewer(palette = "Set2")` for brand colours,
  colour-blind accessible palette
**Key R concepts:** `ggplot2` layering, `patchwork` for
multi-panel layouts, `scale_y_log10()`, `geom_ribbon()`,
`coord_flip()`, colour-blind accessible palettes.
 
---
 
### 13. Automated Reporting
 
**What was done:**
A `generate_category_report()` function was built that
produces a complete analytical summary for any category,
DQ check, KPIs, brand breakdown, with a single function
call. This mirrors the kind of repeatable, scheduled
reporting a data methods team produces weekly.
 
**How it was implemented:**
The function accepts a category name as a string argument,
filters the data internally, and computes all metrics within
the function scope. `gt()` formats the brand breakdown table
with `data_color()` applying a colour gradient to the margin
column, making high and low margin visually immediate.
 
`invisible()` returns the computed results silently, the
function's primary output is the printed summary, but the
results remain accessible if needed downstream.
 
```r
# Run for any category:
generate_category_report(fmcg_clean, "Beverages")
generate_category_report(fmcg_clean, "Snacks")
 
# Or loop across all categories:
for (cat in unique(fmcg_clean$category)) {
  generate_category_report(fmcg_clean, cat)
}
```
 
**Key R concepts:** `gt()`, `data_color()`, `invisible()`,
function encapsulation, loop-driven report generation.
 
---
 
## R Packages Used
 
| Package | Purpose |
|---|---|
| dplyr | Core data manipulation — group_by, joins, mutate |
| tidyr | Reshaping — pivot_longer, pivot_wider |
| lubridate | Date parsing and feature extraction |
| stringr | String cleaning and pattern matching |
| slider | Rolling window functions — slide_dbl() |
| janitor | Duplicate detection — get_dupes() |
| ggplot2 | Visualisation |
| patchwork | Multi-panel chart layouts |
| gt | Formatted output tables |
| glue | String interpolation for dynamic messages |
| scales | Number formatting in ggplot2 |
 
---
 
## Key Concepts Demonstrated
 
```
Data Quality        → Audit before fix, flag don't remove,
                      stopifnot() validation at every step
 
Root Cause Analysis → 4-step: isolate → decompose →
                      distinguish → document
 
Joins               → left_join (enrich), anti_join (gaps),
                      inner_join (restrict), row count validation
 
Reshaping           → Wide for display, long for analysis,
                      lag() requires sorted long format
 
Tidy Evaluation     → {{ }} for reusable dplyr functions,
                      one function replaces many pipelines
 
Rolling Windows     → arrange() before slide_dbl(),
                      group_by() to contain window per group
 
Statistical Testing → Kruskal-Wallis for group comparison,
                      Wilcoxon for two groups, z-test for
                      proportions, always report effect size
 
Defensive Programming → stopifnot(), relationship= in joins,
                        row count checks, change logging
```
 
---
 
## Project Structure
 
```
fmcg-data-quality-analysis/
│
├── fmcg_analysis.R          # Main analysis — all 15 sections
├── FMCG_README.md           # This file
└── data/
    └── fmcg_sales_3years_1M_rows.csv   # Add dataset from Kaggle
```
