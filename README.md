# Data Quality Challenge — Analysis Repository

This repository contains my solution to the **HAMS Data Quality Challenge** based on the `challenge.db` SQLite dataset. The goal of the project is to analyze Company X’s e-commerce tracking data, identify data quality issues, and document the findings in a concise, reproducible way.

## Reference

Original challenge definition and source files:
- Challenge repository: https://github.com/haensel-ams/recruitment_challenge/tree/master/Data_Quality_202108

According to the original brief, the task is to analyze the following areas:
1. Coverage of AdWords costs in `api_adwords_costs` vs. `session_sources`
2. Stability of conversions over time in `conversions`
3. Reconciliation of `conversions` against `conversions_backend`
4. Consistency of attribution results in `attribution_customer_journey`
5. Bonus: channel stability over time
6. Bonus: any additional data quality issues

The official deliverable asks for a concise report that explains the analytical approach, assumptions, and findings, ideally supported by tables and charts.

## Repository Contents

Typical project contents:
- `main.ipynb` — the main Jupyter notebook with the full analysis
- `data_quality_challenge_report.docx` — the written report version of the findings
- `README.md` — this project overview
- `challenge.db` or extracted challenge data — local input data used by the notebook

## Approach

The analysis is organized around the challenge questions and follows a validation-first workflow:

### 1) Data loading and profiling
- Connected to the SQLite database
- Inspected schemas and row counts
- Compared date ranges across tables
- Checked missing values, uniqueness assumptions, and basic descriptive statistics

### 2) Cost coverage validation
- Aggregated `api_adwords_costs` by `event_date` and `campaign_id`
- Aggregated `session_sources` by the same keys using:
  - session count as a proxy for clicks
  - sum of `cpc` as session-side cost
- Compared API totals vs. session-side totals at both the overall and campaign level
- Investigated campaigns with incomplete or suspicious coverage

### 3) Conversion stability over time
- Compared daily frontend conversions (`conversions`) with backend truth (`conversions_backend`)
- Examined daily counts and daily revenue over time
- Calculated frontend/backend ratios and day-over-day changes
- Used robust spike checks to identify unusually high or low days

### 4) Frontend–backend reconciliation
- Performed a full comparison between `conversions` and `conversions_backend`
- Checked for:
  - missing conversion IDs in frontend
  - mismatched `user_id`
  - mismatched `conv_date`
  - mismatched `market`
  - mismatched `revenue`
- Reviewed the distribution and pattern of revenue gaps

### 5) Attribution consistency checks
- Validated whether attribution weights (`ihc`) sum to 1 for each conversion
- Checked for row-level invalid values such as `NULL`, negative values, or values greater than 1
- Looked for duplicate `(conv_id, session_id)` pairs
- Tested whether attributed sessions belong to the same user as the backend conversion
- Checked whether attributed sessions occur after the conversion date
- Verified whether attributed conversions and sessions exist in the underlying source tables

### 6) Channel stability analysis
- Standardized inconsistent channel labels before evaluating stability
- Built a daily complete channel-date grid so missing days are treated as zero sessions
- Evaluated channel stability using **daily share of sessions**, not only raw counts
- Segmented very small channels separately to avoid overinterpreting sparse data

## Key Assumptions

A few assumptions were necessary because they are not fully specified in the challenge text:

- `conversions_backend` is treated as the source of truth for conversion validation.
- Session count is used as a practical proxy for API clicks when comparing `session_sources` with `api_adwords_costs`.
- For attribution timing checks, only `conv_date` is available for conversions, so temporal validation is performed at the date level rather than timestamp level.
- Channel stability thresholds are heuristic and used for operational interpretation, not as universal statistical cutoffs.

## Main Findings

### 1) AdWords cost coverage is high but incomplete
- Most API spend is reflected in `session_sources`, but not perfectly.
- The main problematic campaign is **`campaign_id_79`**.
- Two campaigns, **`campaign_id_97`** and **`campaign_id_121`**, are present in the API table but missing from `session_sources`.
- Cost alignment is stronger than click/session alignment, which suggests either incomplete session capture or imperfect comparability between API clicks and tracked sessions.

### 2) Conversions are fairly stable over the observed time window, but the frontend under-records systematically
- Daily conversion counts do not show broad instability across the observed period.
- There are isolated spike days, but the larger pattern is that frontend counts and revenue are consistently below backend values.
- This points to a systematic frontend tracking or ingestion issue rather than a few isolated bad days.

### 3) Frontend conversions do not fully reconcile to backend truth
The comparison between `conversions` and `conversions_backend` identified several concrete defects:
- **345** conversion IDs exist in backend but are missing in frontend.
- **172** rows have revenue mismatches.
- **16** rows have `user_id` mismatches.
- `conv_date` and `market` match cleanly.
- The revenue mismatches follow a clear pattern: many affected frontend rows have `revenue = 0` while backend shows a positive value.

### 4) Attribution contains real integrity issues
The attribution table is not fully reliable as-is.
Observed problems include:
- conversions where `sum(ihc) != 1`
- cross-user journeys, where the conversion user differs from the attributed session user
- invalid row-level `ihc` values
- duplicate `(conv_id, session_id)` rows
- attributed sessions occurring after the conversion date
- missing conversion/session references in some joins

This suggests that both attribution weighting and customer-journey linkage should be treated as potentially corrupted.

### 5) Channel-level reporting has two issues
- **Label inconsistency:** some channels appear under multiple names, for example `Direct` vs `Direct Traffic`, `SEA - Brand` vs `SEA - Branded`, and `Social - Paid` vs `Social Paid`.
- **Actual instability:** after harmonizing labels and evaluating daily session share, several regular channels still appear unstable, especially:
  - `Email`
  - `Display`
  - `Influencers`
  - `SEA (no tagging)`
  - `Video Marketing`

## Practical Priorities

If this were a real production data stack, the first remediation priorities would be:

1. **Fix frontend conversion reconciliation**
   - investigate why backend conversions are missing or zero-valued in frontend
   - validate ETL, event ingestion, and revenue mapping logic

2. **Repair attribution integrity**
   - enforce same-user linkage rules
   - enforce `sum(ihc) = 1`
   - prevent sessions after conversion from entering the attributed journey

3. **Standardize campaign and channel labeling**
   - harmonize naming rules upstream
   - define canonical mappings for downstream reporting

4. **Re-check campaign ingestion completeness**
   - specifically review campaign coverage gaps for `campaign_id_79`, `campaign_id_97`, and `campaign_id_121`

## How to Run

A typical local workflow is:

```bash
# 1) create / activate your environment
# 2) install required packages
pip install jupyter pandas numpy matplotlib seaborn

# 3) open the notebook
jupyter notebook
```

Then open `main.ipynb`, ensure the SQLite database file is available at the path expected by the notebook, and run the notebook from top to bottom.

## Output

This repository is intended to provide two complementary deliverables:
- a reproducible **notebook analysis** (`main.ipynb`)
- a concise **written report** (`data_quality_challenge_report.docx`)

## Notes

This project is designed as a challenge solution, not as a production-grade data quality framework. The focus is on:
- clear analytical structure
- explicit assumptions
- targeted data validation checks
- concise communication of findings and business implications
