# Data Review - Validation Framework Structure

**Script:** `Data_Review`
**Total Lines:** 1065 lines
**Purpose:** Validate quarterly portfolio company data submissions against historical records
**Type:** R data quality validation script with automated change detection
**Primary Use:** SBIC quarterly reporting quality control

---

## Quick Summary

**What it does:** Compares current quarter portfolio company data against previous quarter to flag anomalies, missing data, significant changes, and compliance issues.

**Primary use:** Quality control for SBA SBIC quarterly reporting submissions before official filing.

**Complexity level:** ⚠️ **MEDIUM-HIGH** - 18+ distinct validation rules, period-over-period comparison logic, ESG metrics checking.

---

## File Structure Map

### Lines 1-24: Package Management
```
Lines 1-9:    Commented install commands (9 packages)
Lines 10-26:  Library imports (13 packages loaded)
              - dplyr, openxlsx, janitor, data.table, jrvFinance, etc.
Line 26:      options(scipen = 999) - disable scientific notation
```

### Lines 28-37: Global Variable Examples (Commented Out)
```
Lines 28-36:  Example parameter values for testing
              - as_of_date, prev_as_of_date, fund, investment_vertical, table
```

### Lines 38-224: Function 1 - ingest_data()
**Purpose:** Load reference dataset and current quarter template
**Parameters:**
- `quarterly_template` - File name (default: "Boathouse III - Everside Quarterly Report Template (2Q24)")
- `sheet_name` - Excel sheet name

**Input Files:**
- `~/R Data/Everside Dataset.xlsx` (historical reference data)
- `~/R Data/Quarterly Templates/{quarterly_template}.xlsx` (current quarter submission)

**Key Operations:**
- Lines 40-87: Load Everside Dataset (historical records)
- Lines 89-224: Load quarterly template with tryCatch error handling
- Date parsing from Excel serial format
- Column type conversions (numeric, date)

**Creates global variables:**
- `Everside_Dataset` - Historical reference data
- `quarterly_file` - Current quarter submission

---

### Lines 226-367: Function 2 - data_cleanse()
**Purpose:** Clean and standardize quarterly submission data
**Parameters:**
- `quarterly_file` - Raw quarterly data
- `table_name` - Portfolio company table name
- `as_of_date_input` - Quarter end date
- `update_denom_mill_flag` - Boolean flag for millions conversion (default: FALSE)

**Key Operations:**
- Lines 228-310: Conditional column standardization based on `update_denom_mill_flag`
  - If TRUE: Convert financial columns to millions (÷ 1,000,000)
  - Add calculated columns: `everside_exposure`, `esg_metrics`, `total_invested`
- Lines 312-365: Add metadata columns (fund, investment_vertical, table, as_of_date)

**Output:** Cleaned dataframe with standardized schema

---

### Lines 369-477: Function 3 - create_comparison_data()
**Purpose:** Prepare quarter-over-quarter comparison dataset
**Parameters:**
- `quarterly_data_esg_metrics_updated` - Cleaned current quarter data
- `kfund`, `kinvestment_vertical`, `ktable`, `kas_of_date` - Filter parameters

**Key Operations:**
- Filter historical data for matching fund/vertical/table
- Combine current + historical records
- Write intermediate file: `~/R Output/{table_name}-{as_of_date}_updated_template.xlsx`

**Output:** Combined dataset for comparison

---

### Lines 479-1027: Function 4 - compare_data() ⚠️ **CORE VALIDATION FUNCTION**
**Purpose:** Execute 18 validation checks and generate flagged items report
**Parameters:**
- `quarterly_data_for_comp` - Cleaned current quarter data
- `Everside_Dataset` - Historical reference data
- `kfund`, `kinvestment_vertical`, `ktable` - Identifiers
- `kas_of_date`, `kprev_as_of_date` - Current and previous quarter dates

**Structure:**
```
Lines 480-481:  Create Excel workbook for notes output
Lines 483-497:  Merge current vs previous quarter data (suffixes: _current_quarter, _previous_quarter)
Lines 499-1022: 18 VALIDATION CHECKS (detailed below)
Line 1023:      Save notes workbook to ~/R Output/
Line 1025:      Return comparison dataset
```

**Output Files:**
- `~/R Output/{fund} - {table} Notes - {as_of_date}.xlsx` (validation notes workbook)

---

## Validation Checks Catalog (18 Checks)

### Category 1: Reference Data Validation (Lines 499-539)

| Check # | Lines | Validation Name | Trigger Condition | Acceptable Values |
|---------|-------|-----------------|-------------------|-------------------|
| 1 | 499-518 | `industry_provided_incorrectly` | Industry not in whitelist | "Business Services", "Consumer", "Healthcare", "Technology", "Industrials", "Materials" |
| 2 | 520-539 | `geography_provided_incorrectly` | Geography not in whitelist | "Midwest", "Northeast", "Southeast", "Southwest", "West" |

**Purpose:** Ensure categorical fields use standardized values.

---

### Category 2: Realized Value Integrity (Lines 541-579)

| Check # | Lines | Validation Name | Trigger Condition |
|---------|-------|-----------------|-------------------|
| 3 | 541-559 | `realized_debt_decrease` | Realized debt decreased from prior quarter |
| 4 | 561-579 | `realized_equity_decrease` | Realized equity decreased from prior quarter |

**Purpose:** Realized values should never decrease (unless exit occurred). Flags potential data entry errors.

---

### Category 3: Investment Terms Changes (Lines 581-640)

| Check # | Lines | Validation Name | Trigger Condition |
|---------|-------|-----------------|-------------------|
| 5 | 581-600 | `maturity_date_problems` | Debt maturity date changed |
| 6 | 602-620 | `debt_type_missing` | Invested debt > 0 but no debt type specified |
| 7 | 622-640 | `equity_type_missing` | Invested equity > 0 but no equity type specified |

**Purpose:** Detect term amendments and ensure investment classifications are complete.

---

### Category 4: Significant Financial Changes (Lines 642-797)
**Threshold:** >20% increase or decrease

| Check # | Lines | Validation Name | Metric | Direction |
|---------|-------|-----------------|--------|-----------|
| 8 | 642-665 | `current_total_coupon_significant_change` | Debt total coupon | Increase >20% |
| 9 | 667-690 | `current_total_debt_significant_change` | Total debt | Increase >20% |
| 10 | 692-715 | `current_senior_debt_significant_change` | Senior debt | Increase >20% |
| 11 | 717-740 | `current_enterprise_value_significant_change` | Enterprise value | Decrease >20% |
| 12 | 742-772 | `current_ebitda_significant_change` | EBITDA | Decrease >20% (special handling for negative EBITDA) |
| 13 | 774-797 | `current_sales_significant_change` | Sales | Decrease >20% |

**Purpose:** Flag material operational or valuation changes requiring explanation.

**Special Logic (EBITDA):**
- Lines 744-751: Handles negative EBITDA scenarios differently
- If previous EBITDA < 0: Uses addition instead of subtraction for diff calculation

---

### Category 5: Investment Type Changes (Lines 800-824)

| Check # | Lines | Validation Name | Trigger Condition |
|---------|-------|-----------------|-------------------|
| 14 | 800-824 | `current_debt_security_change` | Type of debt security changed between quarters |

**Purpose:** Investment type should rarely change. Flags potential misclassification.

---

### Category 6: Credit Status Deterioration (Lines 826-884)

| Check # | Lines | Validation Name | Trigger Condition | Written to Notes File |
|---------|-------|-----------------|-------------------|----------------------|
| 15 | 826-855 | `companies_entered_covenant_breach` | Moved from compliant → breach | ✅ Yes (lines 847-848) |
| 16 | 857-884 | `companies_entered_not_current_on_interest` | Moved from current → late on interest | ✅ Yes (lines 876-877) |

**Purpose:** Identify newly distressed investments requiring immediate attention.

---

### Category 7: Exit Flagging (Lines 886-906)

| Check # | Lines | Validation Name | Trigger Condition |
|---------|-------|-----------------|-------------------|
| 17 | 886-906 | `companies_to_remove_to_exit` | Both invested_debt = 0 AND invested_equity = 0 |

**Purpose:** Flag fully exited positions that should be removed from active dataset.

---

### Category 8: Portfolio-Level Credit Metrics (Lines 908-954)
**Note:** These are summary calculations, not validation checks

| Lines | Metric | Calculation | Written to Notes File |
|-------|--------|-------------|----------------------|
| 908-930 | Interest Delinquency Rate | (Debt with current_on_interest=FALSE) / Total Invested | ✅ Yes (line 928) |
| 932-954 | Covenant Breach Rate | (Debt with covenant_breach=TRUE) / Total Invested | ✅ Yes (line 952) |

**Purpose:** Portfolio-level risk metrics for SBIC reporting summary.

---

### Category 9: Default Flagging for SBIC (Lines 956-982)

| Lines | Validation Name | Trigger Condition | Written to Notes File |
|-------|-----------------|-------------------|----------------------|
| 956-982 | `companies_to_flag_for_default` | Covenant breach OR late on interest | ✅ Yes (lines 974-975) |

**Purpose:** Generate action item list for deal teams to communicate with SBIC.

---

### Category 10: ESG Data Completeness (Lines 984-1021)

| Lines | Validation Name | Logic |
|-------|-----------------|-------|
| 984-1021 | `list_of_esg_columns_fully_null` | Checks if any of 16 ESG columns are entirely NULL across portfolio |

**ESG Columns Checked (16 fields):**
- `net_income_2m_current_quarter`
- `tangible_net_worth_6m_current_quarter`
- `employees_at_close_current_quarter`
- `employees_at_recent_fte_current_quarter`
- `green_healthcare_education_job_training_etc_y_n_current_quarter`
- `low_mod_income_current_quarter`
- `hub_zone_current_quarter`
- `opportunity_zone_current_quarter`
- `rural_current_quarter`
- `minority_owned_current_quarter`
- `woman_owned_current_quarter`
- `veteran_owned_current_quarter`
- `woman_management_current_quarter`
- `minority_management_current_quarter`
- `veteran_management_current_quarter`
- `family_owned`
- `non_committed_fund_sponsor_current_quarter`

**Purpose:** Ensure ESG metrics required for SBIC reporting are populated.

---

### Lines 1029-1053: Function 5 - full_cleansing_and_comparison()
**Purpose:** Orchestrate full workflow (ingest → cleanse → compare)
**Parameters:**
- `date_input` - As-of date (default: "2025-06-30")
- `fund_input` - Fund name (default: "Fund II")
- `investment_vertical_input` - Primary/Secondary (default: "Primary")
- `table_input` - Portfolio company table (default: "Boathouse_III")
- `quarterly_template_input` - Template file name
- `quarterly_template_Sheet` - Sheet name (default: "Q2 2025")
- `update_denom_mill_flag_input` - Millions conversion flag (default: FALSE)

**Workflow:**
```
Lines 1030-1034: Calculate previous quarter date using rollback()
Lines 1036-1038: Call ingest_data()
Lines 1040-1042: Call data_cleanse()
Lines 1044-1046: Call create_comparison_data()
Lines 1048-1050: Call compare_data()
Line 1052:       Return comparison dataset
```

**Usage Example:**
```r
full_cleansing_and_comparison(
  date_input = "2024-06-30",
  fund_input = "Fund III",
  investment_vertical_input = "Primary",
  table_input = "Boathouse_V"
)
```

---

### Lines 1056-1065: Function 6 - getEversideFund()
**Purpose:** Helper function to extract fund name from table identifier
**Parameters:**
- `table_name` - Table name (e.g., "Boathouse_III")

**Logic:**
- Lines 1057-1060: Uses case_when to map table name to fund
- Returns fund name string

**Mapping Pattern:**
```r
case_when(
  grepl("Boathouse_III", table_name) ~ "Fund III",
  grepl("Boathouse_II", table_name) ~ "Fund II",
  # ... etc.
)
```

---

## Data Flow Diagram

```
INPUT FILES
├── ~/R Data/Everside Dataset.xlsx (historical reference)
└── ~/R Data/Quarterly Templates/{template}.xlsx (current quarter)
    ↓
[ingest_data()] - Load both files into memory
    ↓
[data_cleanse()] - Standardize schema, add calculated fields
    ↓
[create_comparison_data()] - Merge current + historical
    ↓ (intermediate output)
~/R Output/{table}-{date}_updated_template.xlsx
    ↓
[compare_data()] - Execute 18 validation checks
    ↓
CONSOLE OUTPUT: Print validation warnings (18 checks)
    ↓
OUTPUT FILE
└── ~/R Output/{fund} - {table} Notes - {date}.xlsx
    └── Sheet: "{fund} - {table}"
        ├── Covenant breach companies (if any)
        ├── Late interest companies (if any)
        ├── Interest delinquency rate (%)
        ├── Covenant breach rate (%)
        └── Default flagged companies (if any)
```

---

## Output Schema

### Console Output
- Prints validation results for all 18 checks
- Format: "There are {N} companies with {issue}, please validate!"
- If no issues: "There are NO issues due to {check}!"

### Excel Notes File
**File:** `~/R Output/{fund} - {table} Notes - {as_of_date}.xlsx`

**Sheet Name:** `{fund} - {table}` (e.g., "Fund II - Boathouse_III")

**Content Structure:**
```
Row 1:     "There are X companies that entered covenant breach status, please validate!"
Row 2+:    Table of companies in covenant breach
Row N:     "There are X companies that newly failed current on interest, please validate!"
Row N+1+:  Table of companies late on interest
Row M:     "Boathouse_III reported X.XX% of their debt as NOT current on interest"
Row M+2:   "Boathouse_III reported X.XX% of their debt IN covenant default"
Row M+4:   "Please have the deal team member responsible for {table} reach out..."
Row M+5+:  Table of companies requiring SBIC notification
```

**Dynamic Layout:** Uses `offset` variable to track current row position (lines 838, 849, 876, 878, 928, 930, 952, 954, 974, 976)

---

## Input Data Dependencies

### File 1: Everside Dataset.xlsx
**Path:** `~/R Data/Everside Dataset.xlsx`
**Sheet:** "Everside Dataset"

**Key Columns (50+ total):**
- `as_of_date` - Quarter end date
- `fund`, `investment_vertical`, `table` - Identifiers
- `company_name` - Portfolio company name
- `date_of_investment`, `exit_date` - Investment lifecycle dates
- `invested_debt`, `invested_equity` - Cost basis
- `debt_realized_value`, `equity_realized_value` - Realized proceeds
- `debt_pricing_total_coupon`, `debt_pricing_maturity` - Debt terms
- `current_total_debt`, `current_senior_debt` - Capital structure
- `current_enterprise_value`, `current_ebitda`, `current_sales` - Financials
- `current_covenants_breach`, `current_on_interest` - Credit status (boolean)
- `industry`, `geography` - Classification
- 16+ ESG columns (see Category 10 above)

**Source:** Cumulative historical dataset updated quarterly.

---

### File 2: Quarterly Template
**Path:** `~/R Data/Quarterly Templates/{quarterly_template}.xlsx`
**Sheet:** Variable (e.g., "Q2 2025")
**Start Row:** 3 (skips 2 header rows)

**Expected Columns:** Same schema as Everside Dataset (cleaned column names applied).

**Source:** Submitted by portfolio companies or sourced from accounting/fund admin system.

---

## Validation Logic Patterns

### Pattern 1: Categorical Validation
```r
data %>%
  filter(!column %in% c("Allowed", "Values")) %>%
  select(company_name, column_current, column_previous)
```
**Used in:** Checks 1-2 (Industry, Geography)

---

### Pattern 2: Decrease Detection
```r
data %>%
  filter(metric_current < metric_previous) %>%
  transmute(company_name, metric_current, metric_previous, diff = metric_current - metric_previous)
```
**Used in:** Checks 3-4 (Realized values)

---

### Pattern 3: Significant Change Detection (Percentage)
```r
data %>%
  mutate(
    diff = metric_current - metric_previous,
    diff_perc = (metric_current - metric_previous) / metric_previous
  ) %>%
  filter(diff_perc > 0.2 | diff_perc < -0.2)
```
**Used in:** Checks 8-13 (Financial metrics)

---

### Pattern 4: Status Change Detection
```r
data %>%
  filter(
    status_current == TRUE,
    status_previous != TRUE
  )
```
**Used in:** Checks 15-16 (Covenant breach, Interest delinquency)

---

### Pattern 5: Missing Required Data
```r
data %>%
  filter(
    investment_amount > 0,
    is.na(classification) | classification %in% c("<N/A>", "NA", "N/A", "")
  )
```
**Used in:** Checks 6-7 (Debt type, Equity type)

---

## Known Issues and Considerations

### 1. Hardcoded Categorical Values (Lines 501, 522)
**Issue:** Whitelists for Industry and Geography are hardcoded in function.
**Impact:** Adding new categories requires code change.
**Better Approach:** Store in configuration file or lookup table.

---

### 2. No Validation for Missing Historical Data
**Issue:** If previous quarter data is missing, merge will have all `_previous_quarter` columns as NA.
**Risk:** False positives on change detection checks.
**Missing Logic:** Should validate that previous quarter data exists before comparing.

---

### 3. 20% Threshold is Universal (Lines 647, 672, 697, 722, 754, 779)
**Issue:** All significant change checks use fixed 20% threshold.
**Consideration:** Some metrics may warrant different thresholds (e.g., coupon vs. sales).
**Recommendation:** Make threshold configurable per check.

---

### 4. ESG Column List Hardcoded (Lines 989-1007)
**Issue:** 16 ESG columns listed manually in function.
**Impact:** Adding/removing ESG fields requires code update.
**Better Approach:** Define ESG column list in configuration.

---

### 5. No Validation of Date Rollback Logic (Line 1034)
**Issue:** Uses `rollback()` three times to get previous quarter, assumes quarterly cadence.
**Risk:** If irregular reporting periods, may not match correct previous quarter.
**Missing:** Should validate that `prev_as_of_date` actually exists in historical data.

---

### 6. Error Handling Limited (Lines 89-224)
**Issue:** Only `ingest_data()` has tryCatch; other functions have no error handling.
**Risk:** Missing files, malformed data, or schema mismatches will cause unhandled crashes.

---

### 7. Global Variable Assignment (Lines 40, 90, 156, 491)
**Issue:** Uses `<<-` to assign global variables (`Everside_Dataset`, `quarterly_file`, `comp_data`).
**Impact:** Side effects make testing difficult; variables persist in global environment.
**Best Practice:** Return values instead of global assignment.

---

### 8. Notes File Overwrite (Line 1023)
**Issue:** `overwrite = TRUE` means re-running for same quarter/fund/table will erase previous notes.
**Risk:** Lose historical validation results if script re-run.
**Recommendation:** Add timestamp to filename or version control.

---

## Execution Pattern

### Typical Usage:
```r
# Load packages (implied - script doesn't auto-run this)
# source("Data_Review")

# Run full validation workflow
results <- full_cleansing_and_comparison(
  date_input = "2024-06-30",
  fund_input = "Fund III",
  investment_vertical_input = "Primary",
  table_input = "Boathouse_V",
  quarterly_template_input = "Boathouse V - Everside Quarterly Report Template (2Q24)",
  quarterly_template_Sheet = "Q2 2024",
  update_denom_mill_flag_input = FALSE
)

# Outputs:
# 1. Console: Prints 18 validation check results
# 2. File: ~/R Output/Fund III - Boathouse_V Notes - 2024-06-30.xlsx
# 3. File: ~/R Output/Boathouse_V-2024-06-30_updated_template.xlsx
# 4. Return: comp_data dataframe with current vs previous quarter columns
```

---

## Validation Check Summary Table

| # | Check Name | Category | Trigger | Output Location |
|---|------------|----------|---------|-----------------|
| 1 | Industry whitelist | Reference Data | Non-standard industry | Console |
| 2 | Geography whitelist | Reference Data | Non-standard geography | Console |
| 3 | Realized debt decrease | Realized Values | Debt proceeds decreased | Console |
| 4 | Realized equity decrease | Realized Values | Equity proceeds decreased | Console |
| 5 | Maturity date change | Investment Terms | Maturity date changed | Console |
| 6 | Debt type missing | Investment Terms | Debt investment without type | Console |
| 7 | Equity type missing | Investment Terms | Equity investment without type | Console |
| 8 | Total coupon increase | Financial Change | >20% increase | Console |
| 9 | Total debt increase | Financial Change | >20% increase | Console |
| 10 | Senior debt increase | Financial Change | >20% increase | Console |
| 11 | Enterprise value decrease | Financial Change | >20% decrease | Console |
| 12 | EBITDA decrease | Financial Change | >20% decrease | Console |
| 13 | Sales decrease | Financial Change | >20% decrease | Console |
| 14 | Debt security type change | Investment Type | Type changed | Console |
| 15 | Covenant breach (new) | Credit Status | Entered breach | Console + Notes File |
| 16 | Late on interest (new) | Credit Status | Became late | Console + Notes File |
| 17 | Exit flagging | Exit Detection | Invested amounts = 0 | Console |
| 18 | Default flagging | SBIC Alert | Breach OR late | Console + Notes File |
| - | Interest delinquency rate | Portfolio Metric | (Summary calc) | Console + Notes File |
| - | Covenant breach rate | Portfolio Metric | (Summary calc) | Console + Notes File |
| - | ESG completeness | Data Quality | Missing ESG columns | Console |

**Total Validations:** 18 checks + 2 portfolio metrics + 1 ESG check = **21 quality controls**

---

## Related Scripts

**Upstream Producers:**
- `468_Excel_Processor.ipynb` - May generate quarterly template data
- Accounting/fund admin exports

**Downstream Consumers:**
- Manual review by fund managers
- SBIC quarterly filing preparation

**Data Flow:**
```
Quarterly Templates (Excel)
  ↓
[Data_Review] → Validation Notes (Excel)
  ↓
Manual Review & SBIC Filing
```

---

## Documentation Status

✅ **Structure mapped** - All 6 functions identified
✅ **Validation catalog created** - 21 quality controls documented
✅ **Data flow diagrammed** - Input → Processing → Output
✅ **Known issues flagged** - 8 considerations identified
⏳ **Detailed line-by-line logic** - Deferred (per instructions)

**Next steps for deeper documentation (if needed):**
- Part 2: Detailed logic for each validation check
- Part 3: ESG column definitions and SBA requirements
- Part 4: Quarter-over-quarter merge logic edge cases

---

**End of Part 1: Structure Overview**

**Created:** 2025-11-12
**Purpose:** Validation framework roadmap for non-technical audiences
**Time to complete:** ~15 minutes
**Validation checks identified:** 21 total controls
