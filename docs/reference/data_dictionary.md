# Data Dictionary - Quick Reference

**Purpose:** Quick scan showing inputs, outputs, and sample columns for each script.
**Status:** Lightweight reference - detailed schema requires manual review of actual data files.
**Last Updated:** 2025-11-12

---

## Parse investment template to deal cloud file.py

**Type:** Python script (81 lines)

**Inputs:**
- `Investment Template_Farragut v4.xlsx` (sheet: 'eFile', skiprows=7)
- `deal_cloud_dataset_template.xlsx` (template)

**Outputs:**
- `output_file_farragut_v4-v7.xlsx`

**Sample Column Mappings Observed:**
```python
column_mapping = {
    'Company Name': 'E',
    'Industry': 'H',
    'Date of Investment Memo': 'D',
    'Revenue': ['AZ', 'BO'],
    'Adj. EBITDA ': ['BA', 'BP'],
    'Implied EV': ['AX', 'BM'],
    'Description': 'DE',
    'Sourcing': 'J',
    'Investment': 'U',
    'Total Investment': 'R',
    'Senior Debt.1': ['AR', 'BD'],
    'Sr. Subordinated debt.1': ['AT', 'BF'],
    'Cash': 'AJ',
    'Dividend/PIK': 'AK',
    'Ownership.1': 'AQ',
    'Tenor': 'AN'
}
```

**Notes:** Maps investment template columns to Deal Cloud dataset format using letter-based Excel column references.

---

## Parsing investment file into everside dataset template.py

**Type:** Python notebook (similar to above)

**Status:** Duplicate functionality - notebook version of Parse investment template script

**Inputs:** Same as Parse investment template to deal cloud file.py

**Outputs:** Same mapping structure

**Notes:** Interactive notebook version for exploration/testing.

---

## 468_Excel_Processor.ipynb

**Type:** Jupyter Notebook (~19 cells)

**Inputs:**
- `2024.06.30 - St. Cloud Fund III Form 468.xlsx`
  - Sheet: 'S1Inv' (skiprows=19)
  - Sheet: 'S4Del' (skiprows=9)
  - Sheet: 'S12pc'
- `Book1.xlsx` (output template)

**Outputs:**
- `v1.xlsx` (intermediate)
- `v3x.xlsx` (final processed)

**Input Columns Observed (S1Inv sheet):**
- Portfolio Company Name
- Financing Type (values: 'Equity', 'Loan', 'Debt')
- Investment Type (values: includes 'Warrants (Debt)', 'Warrants (Equity)', 'Other')
- Initial Financing Date
- First Investment Date
- Total Cash Invested (A)
- Cost at Beginning Period
- Cost at End of Period
- SBA Reported Value (B)
- GAAP Reported Value (C)
- Cum. Cash Proceeds (D)
- Ownership % (Fully Diluted)
- Employer ID
- Loan/ Debt Status
- Interest or Dividend Rate

**Columns Dropped:**
- "Critical Technology\n(if applicable)"
- "Restructured?"
- "Class 1"
- "Class 2"
- "Class 2 Date of Up Round"
- "Prior SBA Mult"

**Key Transformations:**
```python
# Separation logic:
equity_df = df[(df['Financing Type'] == 'Equity') | (df["Investment Type"].isin(warrants))]
loan_df = df[(df['Financing Type'].isin(['Loan', 'Debt'])) & (~df["Investment Type"].isin(warrants))]

# Aggregation by company - sums for numeric, min for dates:
- Total Cash Invested (A): sum
- Cost at Beginning/End Period: sum
- SBA/GAAP Reported Values: sum
- Initial/First Investment Date: min (earliest date)
- Ownership %: sum of non-null values
```

**Output Columns Created:**
- Loan dataframe: 'Debt Pricing - Maturity', 'Status Compliance of Credit - Current Status', 'Status Compliance of Credit - Covenants Breach', 'Status Compliance of Credit - Current on Interest'
- Equity dataframe: Renamed to 'Company Name', 'Equity Pricing - Equity Ownership (FD)', 'Invested Dollars - Equity Cost', 'Invested Dollars - Equity FMV'

**Notes:** Processes SBA Form 468 regulatory data, separates equity vs. debt investments, aggregates by portfolio company.

---

## Champlain_Capital468.ipynb

**Type:** Jupyter Notebook

**Inputs:**
- Client-specific Form 468 Excel file (name varies)

**Outputs:**
- Excel workbook (to_excel)
- CSV export (to_csv)

**Status:** Client-specific version of 468_Excel_Processor.ipynb

**Notes:** Similar structure to 468_Excel_Processor.ipynb but customized for Champlain Capital client.

---

## Historic_Everside_Deal_Cloud_parsing.ipynb

**Type:** Jupyter Notebook

**Inputs:**
- Historical deal data (file name not hardcoded)

**Outputs:**
- Deal Cloud formatted Excel

**Status:** Possibly deprecated - one-time migration script

**Notes:** Historical data migration to Deal Cloud format.

---

## Parse_investment_template_to_deal_cloud_file.ipynb

**Type:** Jupyter Notebook

**Inputs:**
- `Investment Template - Oxer.xlsx` (sheet: 'eFile', skiprows=7)
- `deal_cloud_dataset_template_for_python.xlsx`

**Outputs:**
- Output Excel file (name parameterized)

**Status:** Variant of Parse investment template script

**Notes:** Client-specific (Oxer) version of investment template parser.

---

## Capital_Call_Distribution_data.R

**Type:** R script (815 lines)

**Inputs:**
- `~/R Data/Aggregated Capital Calls and Distributions.xlsx`

**Outputs:**
- ggplot visualizations (not saved to file by default)

**Input Columns Expected:**
- fund (values: "Direct I", "Direct II", "Fund I Founders", "Fund I Suncap", "Fund II", "Fund III", "Fund IV")
- entity
- call_dist (values: "Call", "Distribution")
- date
- amount
- days
- years
- fund_max_age
- fund_max_days
- cumulative_called
- cumulative_distribution

**Calculated Columns:**
- facility_max (hardcoded by fund)
- cumulative_called_percent = (-1 * cumulative_called) / facility_max
- cumulative_distribution_percent = cumulative_distribution / (-1 * cumulative_called)

**Function:** `everside_capital_plot(input_fund = "All", x_axis = "years")`

**Notes:** Generates time-series visualizations of capital calls and distributions across funds.

---

## Capital_For_Filtered_Funds.R

**Type:** R script (similar to Capital_Call_Distribution_data.R)

**Inputs:**
- `~/R Data/Aggregated Capital Calls and Distributions.xlsx`

**Outputs:**
- ggplot visualizations

**Function:** `everside_capital_plot_selected_funds(input_fund = c("All"), x_axis = "years")`

**Notes:** Filtered version allowing selection of specific funds for comparison.

---

## IRR_Data_Tab_Creation.R

**Type:** R script (979 lines)

**Inputs:**
- `~/R Data/{gl_file_input}.xlsx` (GL transaction export)
  - Expected: "LTD Investment Transactions - {date}.xlsx"

**Outputs:**
- `~/R Output/IRR Data Output/{fund}-{start_date}-IRR_data_tab_output.csv`

**Input Columns Expected (GL file):**
- legal_entity (filtered by fund entities)
- gl_date
- position
- deal_name
- trans_type (values: "Cash Out - Investment", "Investment - Cost", "Interest Income - Investment", "Dividend Income - Investment", "Realized Gain", "Interest Expense", "Deferred Income", "Cash In - Investment", etc.)
- dr_cr_amount
- batch_id
- comments_batch
- comments_transaction

**Output Schema:**
```r
vintage_year           # year(gl_date)
transaction_date       # gl_date
position_name          # position or deal_name
investment_type        # "Debt", "Equity", or "Partnership" (derived from position name patterns)
commitment             # NA_real_ (placeholder)
unfunded_commitment    # NA_real_ (placeholder)
gap                    # NA_real_ (placeholder)
investment_amount      # dr_cr_amount * -1 for Investment - Cost
investment_expense     # dr_cr_amount * -1 for Interest Expense
proceeds_return_of_capital
proceeds_interest_income
proceeds_dividend_income
proceeds_realized_gain
proceeds_mfee_rebate
proceeds_closing_fee
proceeds_due_to_manager  # NA_real_ (TODO: verify with Hao)
realized_proceeds      # sum of all proceeds columns
fund_name              # legal_entity
```

**Key Transformations:**
- Separates cash out vs. cash in transactions by batch_id
- Categorizes investment_type using regex patterns:
  - Debt: "Promissory Note|Term Note|Subordinated Debt|Secured Note|Senior|Subordinated|Debt|Note|Loan"
  - Equity: "Preferred|Common|Units|Stock|Equity|LLC Interest"
  - Partnership: "(Primary)|(Secondary)"
- Aggregates transactions by position/deal
- Filters by fund entities (hardcoded lists for each fund)

**Function:** `create_irr_data_tab(fund, start_date, gl_file_input, excel_flag = TRUE)`

**Notes:** Core IRR calculation engine. Processes GL transactions into standardized cash flow format.

---

## IRR_Front_Page_Calcs.R

**Type:** R script (199 lines)

**Inputs:**
- `~/R Data/{irr_data_file}.xlsx` (sheet: "Data", startRow=4)
  - Example: "Everside Fund III IRR Model 3Q24.xlsx"
- `~/R Data/Aggregated Capital Calls and Distributions.xlsx`
- `~/R Data/Fund_Alias_Lookup.xlsx` (fund-specific sheets)
- **SOURCES:** `~/R Data/IRR_Data_Tab_Creation.R` (line 53)

**Outputs:**
- Returns data.frame (not written to file by default)

**Input Columns from IRR Model:**
- vintage_year
- transaction_date
- position_name
- investment_type
- commitment
- unfunded_commitment
- investment_amount
- investment_expenses
- proceeds_return_of_capital
- proceeds_interest_income
- proceeds_realized_gain
- proceeds_mfee_rebate
- proceeds_closing_fee
- proceeds_due_to_manager
- realized_proceeds
- fund_name
- fair_market_value

**Output Schema:**
```r
alias                      # from Fund_Alias_Lookup
position_name
investment_type
total_invested_called      # cumsum(investment_amount)*-1 - cumsum(investment_expenses)
total_realized_proceeds    # cumsum(realized_proceeds)
total_unrealized_value     # cumsum(fair_market_value)
total_value                # total_realized + total_unrealized
gross_moic                 # total_value / total_invested_called
net_moic                   # adjusted for fund-level fees
```

**Key Calculations:**
```r
tvpi = (net_asset_value + actual_cash_distributions) / capital_called_to_date
gross_moic_multiplier = (fund_gross_moic - tvpi) * capital_called_to_date
net_moic = (total_value - (total_value / (total_fund_value * gross_moic_multiplier))) / total_invested_called
```

**Function:** `create_irr_front_page_view(irr_data_file, fund_filter, date_filter)`

**Notes:** Generates summary IRR metrics for investor report front pages. Combines existing IRR data with new GL transactions.

---

## Data_Review

**Type:** R script (1,066 lines)

**Inputs:**
- `~/R Data/Everside Dataset.xlsx` (sheet: "Everside Dataset")
- `~/R Data/Quarterly Templates/{quarterly_template}.xlsx` (startRow=3)

**Outputs:**
- `~/R Output/{table_name}-{as_of_date}_updated_template.xlsx` (cleansed data)
- `~/R Output/{fund} - {table} Notes - {as_of_date}.xlsx` (validation notes)

**Input Schema (Quarterly Template):**
All columns prefixed with `invested_dollars_`, `debt_pricing_`, `equity_pricing_`, `at_close_`, `current_`, `status_compliance_of_credit_`, ESG metrics, etc.

**Output Schema (standardized as):**
```r
fund
investment_vertical
table
as_of_date
company_name
date_of_investment
industry                    # validated against: Business Services, Consumer, Healthcare, Technology, Industrials, Materials
geography                   # validated against: Midwest, Northeast, Southeast, Southwest, West
use_of_proceeds
sponsor_name
sponsor_type
name_of_senior_lender
type_of_debt_security
type_of_equity_security
invested_debt               # from invested_dollars_debt_cost
debt_realized_value
debt_fmv
invested_equity
equity_realized_value
equity_fmv
debt_pricing_cash
debt_pricing_pik
debt_pricing_total_coupon
debt_pricing_maturity
equity_pricing_cash
equity_pricing_pik
equity_pricing_equity_ownership_fd
at_close_senior_debt
at_close_sr_debt_turns      # calculated: at_close_senior_debt / at_close_ebitda
at_close_total_debt
at_close_total_debt_turns   # calculated
at_close_liquidity_cash_available_liquidity
at_close_fccr_covenants
at_close_enterprise_value
at_close_ev_multiple        # calculated: at_close_enterprise_value / at_close_ebitda
at_close_sales
at_close_ebitda
at_close_capex
at_close_ebitda_capex
current_senior_debt
current_sr_debt_turns       # calculated
current_total_debt
current_total_debt_turns    # calculated
current_liquidity_cash_available_liquidity
current_fccr_current
current_enterprise_value
current_ev_multiple         # calculated
current_sales
current_ebitda
current_capex
current_ebitda_capex        # calculated: current_ebitda - current_capex (adjusted for sign)
gross_profit_ttm
interest_charges_ttm
net_income_ttm
cashflow_from_ops_ttm
cash_balance_ttm
current_assets_ttm
fixed_assets_ttm
total_assets_ttm
current_liabilities_ttm
total_liabilities_ttm
eo_y_equity_value_market
federal_taxes_paid
state_taxes_paid
current_status              # from status_compliance_of_credit_current_status
current_type_of_debt
current_covenants_breach    # boolean
current_on_interest         # boolean
status_compliance_of_credit_date_interest_off
status_compliance_of_credit_days_w_o_interest
net_income_2m               # boolean (< $2M threshold)
tangible_net_worth_6m       # boolean (< $6M threshold)
employees_at_close
employees_at_recent_fte
green_healthcare_education_job_training_etc_y_n  # boolean
company_address
low_mod_income              # boolean
hub_zone                    # boolean
opportunity_zone            # boolean
rural                       # boolean
minority_owned              # boolean
minority_owned_percent
woman_owned                 # boolean
woman_owned_percent
veteran_owned               # boolean
veterans_owned_percent
woman_management            # boolean
minority_management         # boolean
veteran_management          # boolean
family_owned                # boolean
non_committed_fund_sponsor  # boolean
short_one_sentence_description_of_impact_related_comment
exit_date
exit_multiple
exit_leverage
exit_returns_net_irr
exit_returns_net_moic
exit_scale
exit_type
exit_buyer
```

**Data Validation Checks:**
- Industry/geography value validation
- Realized value decrease detection (should never decrease)
- Maturity date changes
- Debt/equity type presence validation
- Covenant breach entry detection
- Interest payment status changes
- Significant changes (>20%): total coupon, debt levels, enterprise value, EBITDA, sales
- Debt security type changes
- ESG data completeness

**Validation Metrics Calculated:**
```r
# % of debt not current on interest
interest_non_current_percent = sum(invested_debt where current_on_interest=FALSE) / sum(total_invested)

# % of debt in covenant default
covenant_breach_percent = sum(invested_debt where current_covenants_breach=TRUE) / sum(total_invested)
```

**Functions:**
- `ingest_data(quarterly_template, sheet_name)`
- `data_cleanse(quarterly_file, table_name, as_of_date_input, update_denom_mill_flag)`
- `create_comparison_data(quarterly_data_esg_metrics_updated, ...)`
- `compare_data(quarterly_data_for_comp, Everside_Dataset, ...)` - generates validation report
- `full_cleansing_and_comparison(...)` - wrapper function

**Notes:** Comprehensive data quality control system. Compares current quarter data against previous quarter to detect anomalies.

---

## high_level_metrics_review

**Type:** R script (140 lines)

**Inputs:**
- `~/R Data/Everside Dataset.xlsx`
  - Sheet: "Everside Dataset"
  - Sheet: "IRR Dataset"

**Outputs:**
- Returns data.frame objects (not written to file)
- `View(full_grouped_metrics)` displays in R console

**Calculated Metrics:**

**From Everside Dataset:**
```r
fund_level_metrics_dataset:
  leverage                 # weighted.mean(current_leverage, current_leverage_everside_exposure)
  coupon                   # weighted.mean(debt_coupon_all_in_rate, debt_coupon_everside_exposure)
  total_exposure           # sum(everside_exposure)
  total_debt_exposure      # sum(everside_exposure_debt)
  total_equity_exposure    # sum(everside_exposure_equity)
  active_company_count     # n()
  debt_percentage          # total_debt_exposure / total_exposure
  equity_percentage        # total_equity_exposure / total_exposure
```

**From IRR Dataset:**
```r
IRR_fund_level_data:
  total_commitments        # sum(commitment_total)
  total_invested           # sum(invested_total)
  total_realized           # sum(realized_value_total)
  total_unrealized         # sum(unrealized_value_total)
  investment_count         # n()
```

**Final Output:**
```r
full_grouped_metrics (merged, grouped by fund/investment_vertical/table):
  fund
  investment_vertical
  table
  as_of_date
  leverage
  coupon
  invested_total
  commitment_total
```

**Filter Applied:**
- as_of_date == "2025-03-31"
- investment_vertical != "Directd" (filters out typo?)

**Notes:** Portfolio-wide metrics aggregation. Combines exposure data with IRR metrics. Uses weighted averages for leverage and coupon calculations.

---

## radius_monthly_financials_data_pull

**Type:** R script (157 lines)

**Inputs:**
- `~/R Data/Monthly Financials/{file_name}.xlsx`
  - Sheet: {balance_sheet_tab} (e.g., "RH Balance Sheet")
  - Sheet: {cash_flow_tab} (e.g., "RH Cash Flow")
  - Sheet: {income_statement_tab} (e.g., "9m July, Aug, Sept  Q3")

**Outputs:**
- Returns data.table with calculated metrics (not written to file)

**Input Columns Expected:**

**Balance Sheet:**
- Assets (and nested hierarchy x2-x6)
- x7 (amount column)
- "WSFS Loan Payable" (senior debt)
- "Everside Loan" or "Merion Loan" (sub debt)
- " - Revolver" (revolver balance)

**Cash Flow:**
- radius_holdings_llc (field name)
- x2 (amount column)
- "Cash and cash equivalents - end of year"

**Income Statement:**
- x1-x5 (nested category hierarchy)
- {current_month_column} (parameterized, e.g., "september")
- "Revenue"
- "Payroll and Related's"
- "Adjusted EBITDA"

**Output Schema:**
```r
Field                         Amount (in thousands)
--------------------------------------------
Revenue                       revenue/1000
Gross Profit                  (revenue - payroll_expense)/1000
GP Margin %                   (revenue - payroll_expense) / revenue
Adj. EBITDA                   adjusted_ebitda/1000
EBITDA Margin %               adjusted_ebitda / revenue
Rolling LTM Revenue           "" (TODO)
Rolling LTM Adj. EBITDA       "" (TODO)
skip_row                      ""
Senior Debt                   wsfs_loan/1000
Sub Debt                      (everside_loan + merion_loan)/1000
Total Debt                    (senior + sub)/1000
Cash                          cash_balance/1000
Revolver Availability         (10000000 - revolver_amount)/1000
Liquidity                     (cash + revolver_remaining)/1000
```

**Key Calculations:**
```r
gross_profit = revenue - payroll_expense
gp_margin = gross_profit / revenue
ebitda_margin = adjusted_ebitda / revenue
senior_debt = sum(WSFS Loan Payable entries)
sub_debt = sum(Everside Loan + Merion Loan)
revolver_remaining = 10000000 - revolver_amount  # $10M max hardcoded
liquidity = cash + revolver_remaining
```

**Function:** `radius_monthly_financials_data_pull(as_of_date, file_name, balance_sheet_tab, cash_flow_tab, income_statement_tab, current_month_column)`

**Notes:** Client-specific (Radius Holdings) monthly financial extraction. Hardcoded assumptions: $10M revolver max, specific account names.

---

## Risk_Rating_Function

**Type:** R script (29,084 lines - very large)

**Status:** Complex - needs manual review

**Notes:** File too large for quick scan. Likely contains portfolio company risk scoring algorithms and matrices. Requires detailed analysis.

---

## Summary Statistics

| Category | Count |
|----------|-------|
| Python Scripts Documented | 2 |
| Jupyter Notebooks Documented | 5 |
| R Scripts Documented | 7 |
| Files Needing Deep Review | 1 (Risk_Rating_Function) |
| Total Input Files Referenced | ~15 unique |
| Total Output File Types | ~10 |

---

## Common Data Sources

### Shared Input Files (Used by Multiple Scripts):
1. **Everside Dataset.xlsx** - Central data warehouse
   - Used by: Data_Review, high_level_metrics_review
   - Sheets: "Everside Dataset", "IRR Dataset"

2. **Aggregated Capital Calls and Distributions.xlsx**
   - Used by: Capital_Call_Distribution_data.R, Capital_For_Filtered_Funds.R, IRR_Front_Page_Calcs.R
   - Purpose: Fund-level capital activity tracking

3. **GL Transaction Exports** (various names)
   - Used by: IRR_Data_Tab_Creation.R
   - Format: "LTD Investment Transactions - {date}.xlsx"

4. **Investment Templates**
   - Used by: Parse investment template scripts
   - Format: "Investment Template_{client}.xlsx", sheet 'eFile'

5. **Form 468 Files**
   - Used by: 468_Excel_Processor.ipynb, Champlain_Capital468.ipynb
   - Format: "{date} - {fund} Form 468.xlsx"

### Data Flow Patterns:

```
SBA Form 468 → 468_Excel_Processor → Standardized output

Investment Template → Parse scripts → Deal Cloud format

GL Transactions → IRR_Data_Tab_Creation → IRR_Front_Page_Calcs → Summary reports

Quarterly Templates → Data_Review → Everside Dataset.xlsx → high_level_metrics_review

Monthly Financials → radius_monthly_financials_data_pull → Metrics output
```

---

## Column Naming Conventions Observed

**Prefixes:**
- `invested_dollars_*` - Investment amounts
- `debt_pricing_*` - Debt instrument terms
- `equity_pricing_*` - Equity instrument terms
- `at_close_*` - Metrics at investment closing
- `current_*` - Current/latest metrics
- `status_compliance_of_credit_*` - Loan status tracking
- `proceeds_*` - Cash flow components

**Suffixes:**
- `*_ttm` - Trailing twelve months
- `*_fd` - Fully diluted
- `*_fmv` - Fair market value
- `*_cost` - Cost basis
- `*_realized_value` - Realized proceeds

**Date Fields:**
- Format: Excel serial dates (origin = "1899-12-30")
- Converted using: `as.Date(value, origin = "1899-12-30")`

---

## Data Type Patterns

### Boolean Fields:
Stored as text and converted:
- Values: "Y", "Yes", "TRUE", "T" → TRUE
- Values: "N", "No", "FALSE", "F" → FALSE
- Used for: ESG metrics, compliance flags

### Categorical Fields:
**Industry:** Business Services, Consumer, Healthcare, Technology, Industrials, Materials

**Geography:** Midwest, Northeast, Southeast, Southwest, West

**Investment Type:** Debt, Equity, Partnership

**Financing Type:** Equity, Loan, Debt

**Fund Names:** Direct I, Direct II, Fund I Founders, Fund I Suncap, Fund II, Fund III, Fund IV

### Numeric Fields:
- Currency values typically in dollars (some scripts multiply by 1000 or divide by 1000000 for millions)
- Percentages stored as decimals (0.12 = 12%)
- Leverage ratios (debt/EBITDA turns)

---

## Known Data Quality Issues

1. **Hardcoded File Paths:**
   - 468_Excel_Processor: "2024.06.30 - St. Cloud Fund III Form 468.xlsx"
   - radius_monthly_financials_data_pull: $10M revolver max assumption

2. **Missing Values:**
   - IRR calculations have placeholders: commitment, unfunded_commitment, gap = NA_real_
   - proceeds_due_to_manager = NA_real_ with TODO comment

3. **Data Validation Gaps:**
   - No schema validation before processing
   - Column name typos not caught ("Directd" vs "Direct")

4. **Format Inconsistencies:**
   - Boolean values stored as strings with multiple possible values
   - Date formats mixed (Excel serial vs. string dates)
   - Numeric values sometimes stored as text

---

## Recommendations for Data Dictionary Enhancement

1. **Obtain Sample Data Files:**
   - Request actual Excel templates to document complete schemas
   - Review Form 468 structure from SBA documentation

2. **Document Business Rules:**
   - Threshold values ($2M net income, $6M tangible net worth)
   - Facility max amounts by fund (currently hardcoded)
   - Covenant breach definitions

3. **Create Data Lineage:**
   - Map source systems (GL system, Radius, Deal Cloud)
   - Document transformation logic in business terms

4. **Standardize Data Types:**
   - Create canonical boolean conversion
   - Standardize date parsing
   - Document units (dollars vs. thousands vs. millions)

5. **Deep Dive Required:**
   - Risk_Rating_Function (29K lines)
   - Actual output schemas from notebooks (inspect saved files)
   - Fund entity mapping (legal entities by fund)

---

**End of Quick Reference Data Dictionary**

*For detailed business logic and calculation definitions, see individual script source code.*
