# External Systems - Other Integrations

**Systems Documented:** SBA/SBIC, Radius Holdings, General Ledger (GL) System
**Integration Types:** File-based imports (Excel/CSV)
**Last Updated:** 2025-11-12

---

## Table of Contents
1. [SBA/SBIC Regulatory System](#sbasbic-regulatory-system)
2. [Radius Holdings Financial Reporting](#radius-holdings-financial-reporting)
3. [General Ledger (GL) System](#general-ledger-gl-system)

---

## SBA/SBIC Regulatory System

### Overview

**System:** Small Business Administration (SBA) - Small Business Investment Company (SBIC) Licensing
**Purpose:** Regulatory reporting and compliance for SBIC license holders
**Integration Type:** Excel file export → Manual submission
**Direction:** Everside → SBA (one-way)
**Frequency:** Quarterly

---

### What is SBIC?

**SBIC (Small Business Investment Company):** Federal program providing financing to small businesses through licensed investment funds. Everside operates as an SBIC, requiring quarterly regulatory filings.

**Regulatory Requirement:** Form 468 - Portfolio Financing Report

---

### Scripts Involved

#### 468_Excel_Processor.ipynb
**Lines:** 19 cells (Python notebook)
**Type:** Quarterly regulatory filing processor

**Input:**
- `2024.06.30 - St. Cloud Fund III Form 468.xlsx` (3 sheets)
  - S1Inv (skiprows=19) - Investment positions
  - S4Del (skiprows=9) - Delinquency data
  - S12pc (no skip) - Portfolio company details

**Output:**
- `v1.xlsx` - Intermediate aggregated data
- `v3x.xlsx` - Final processed Form 468

**Purpose:** Process raw portfolio data into SBA Form 468 format for quarterly submission

---

#### Champlain_Capital468.ipynb
**Type:** Client-specific Form 468 processor
**Purpose:** Similar to 468_Excel_Processor but for different client/fund
**Output:**
- Excel workbook
- CSV export

---

### Form 468 Structure

#### Schedule S1Inv - Investment Positions
**Purpose:** List all portfolio company investments

**Key Fields (25+ columns):**
- Portfolio Company Name
- NAICS Code (6-digit industry classification)
- Total Cash Invested (A) - Cost basis
- SBA Reported Value (B) - Fair market value
- GAAP Reported Value (C) - Book value
- Cumulative Cash Proceeds (D) - Realized proceeds
- Current SBA Mult - Valuation multiple
- Initial Financing Date, Maturity Date
- Investment Type (Senior, Mezzanine, Equity, Warrant)
- Ownership Percentage
- Status (Performing, Under Expectation, Exited)

**Aggregation Rule:** If company has multiple positions, combine into single row
- Sum: Total Cash Invested, SBA Reported Value, GAAP Reported Value
- Min: Initial Financing Date
- Conditional: Ownership % (only sum for Equity/Warrant)

---

#### Schedule S4Del - Delinquency Data
**Purpose:** Report problem investments

**Key Fields:**
- Portfolio Company Name
- Written Off (Yes/No)
- Write Off Date
- Loss Amount
- Restructured (Yes/No)

**Row Alignment Risk:** ⚠️ Uses positional matching (assumes companies in same order as S1Inv)

---

#### Schedule S12pc - Portfolio Company Details
**Purpose:** Detailed company information (48-row blocks per company)

**Key Fields (36 extracted):**
- At Close Sales, Current Sales
- At Close EBITDA, Current EBITDA
- At Close Enterprise Value, Current Enterprise Value
- At Close Total Debt, Current Total Debt
- At Close Employees, Current Employees
- ESG Metrics (12 fields):
  - Net Income 2M, Tangible Net Worth 6M
  - Low/Mod Income Area
  - HUB Zone, Opportunity Zone, Rural
  - Minority Owned, Woman Owned, Veteran Owned
  - Woman/Minority/Veteran Management

**Extraction Pattern:** 48 rows per company with fixed cell positions (e.g., company name at offset+3)

---

### SBA Reporting Requirements

#### Quarterly Filing Schedule
- Q1: Due 45 days after March 31
- Q2: Due 45 days after June 30
- Q3: Due 45 days after September 30
- Q4: Due 45 days after December 31

**Submission Method:** Upload Form 468 Excel file to SBA SBIC portal

---

### Data Transformations

#### 1. Investment Aggregation
**Purpose:** Combine multiple positions for same company

**Logic:**
```python
def aggregate_values(group):
    result = group.iloc[0].copy()
    result['Total Cash Invested (A)'] = group['Total Cash Invested (A)'].sum()
    result['SBA Reported Value (B)'] = group['SBA Reported Value (B)'].sum()
    result['Initial Financing Date'] = group['Initial Financing Date'].min()
    # Ownership % only sum if Equity/Warrant
    if result['Financing Type'] in ['Equity', 'Warrant']:
        result['Ownership %'] = group['Ownership %'].sum()
    return result
```

---

#### 2. Status Mapping
**Source Values (7):** Performing, Marginally Performing, Exited, Partially Exited, Exited/Held for Sale, Under Expectation, Not Available

**Target Values (3):**
- "Performing" → "Performing"
- "Marginally Performing" → "Performing"
- "Exited" / "Partially Exited" / "Exited/Held for Sale" → "Exited"
- "Under Expectation" → "Under Expectation"
- "Not Available" → (excluded)

---

#### 3. Investment Type Classification
**Regex-based categorization:**
- **Debt:** Promissory Note, Debt, Note, Loan
- **Equity:** Preferred, Common, Stock, Equity
- **Partnership:** Primary, Secondary

---

### Data Quality Issues

#### Known Issues (5):
1. **Dropped SBA-specific columns** - Critical Technology, Restructured, Class 1/2, Prior SBA Mult removed
2. **Hardcoded skiprows** - Form structure changes break script
3. **Row alignment assumption** - S4Del relies on S1Inv row order
4. **No validation** - Missing NAICS codes, invalid dates not caught
5. **Whitelisted S12pc fields** - Only 10 of 36 fields retained (others dropped)

---

### SBA System Outputs

**From Everside:**
- Form 468 Excel file (v3x.xlsx)

**To SBA:**
- Upload to SBIC Licensing Portal
- Manual review by SBA examiner

**SBA Actions:**
- Quarterly review of portfolio performance
- Annual on-site examination
- License compliance monitoring

---

## Radius Holdings Financial Reporting

### Overview

**System:** Radius Holdings LLC
**Type:** Portfolio Company (Everside investment)
**Purpose:** Monthly financial reporting from portfolio company to Everside
**Integration Type:** Excel file import
**Direction:** Radius → Everside (one-way)
**Frequency:** Monthly

---

### Scripts Involved

#### radius_monthly_financials_data_pull
**Lines:** 157 (R function)
**Type:** Data extraction and transformation

**Input:**
- `~/R Data/Monthly Financials/{file_name}.xlsx`
- Example: "Merion Summary Sept 2024 10-29-24_test.xlsx"
- 3 sheets: Balance Sheet, Cash Flow, Income Statement

**Output:**
- Dataframe (14 rows × 2 columns: Field, Amount)
- Console print of summary metrics

**Function Signature:**
```r
radius_monthly_financials_data_pull(
  as_of_date = "2024-09-30",
  file_name = "Merion Summary Sept 2024 10-29-24_test",
  balance_sheet_tab = "RH Balance Sheet",
  cash_flow_tab = "RH Cash Flow",
  income_statement_tab = "9m July, Aug, Sept  Q3",
  current_month_column = "september"
)
```

---

### Input Data Structure

#### Sheet 1: RH Balance Sheet
**Format:** Hierarchical asset/liability categories with nested indentation

**Key Fields Extracted:**
- WSFS Loan Payable (Senior Debt)
- Everside Loan Payable (Subordinated Debt)
- Merion Loan Payable (Subordinated Debt)
- Revolver balance

**Extraction Logic:** Uses column naming (x2, x3, x4, x5, x6, x7) with case_when to identify categories

---

#### Sheet 2: RH Cash Flow
**Format:** Cash flow statement

**Key Field Extracted:**
- Cash and cash equivalents - end of year

**Column Mapping:**
- Field name: `radius_holdings_llc` column
- Amount: `x2` column

---

#### Sheet 3: Income Statement (9m July, Aug, Sept Q3)
**Format:** Monthly columns with YTD totals

**Key Fields Extracted:**
- Revenue
- Payroll and Related's (payroll expense)
- Adjusted EBITDA

**Extraction Logic:**
- Uses parameterized month column (e.g., "september")
- Nested x1, x2, x3, x4, x5 columns for category hierarchy

---

### Data Transformations

#### Calculated Metrics (8 fields)

| Metric | Calculation | Units |
|--------|------------|-------|
| Revenue | Direct from income statement | Thousands ($000) |
| Gross Profit | Revenue - Payroll Expense | Thousands |
| GP Margin % | Gross Profit / Revenue | Percentage |
| Adj. EBITDA | Direct from income statement | Thousands |
| EBITDA Margin % | Adj. EBITDA / Revenue | Percentage |
| Senior Debt | Sum(WSFS Loan) | Thousands |
| Sub Debt | Sum(Everside Loan, Merion Loan) | Thousands |
| Total Debt | Senior Debt + Sub Debt | Thousands |
| Cash | Cash and cash equivalents | Thousands |
| Revolver Availability | $10M - Revolver Balance | Thousands |
| Liquidity | Cash + Revolver Availability | Thousands |

---

#### Hardcoded Assumptions

**Line 113:**
```r
revolver_remaining <- 10000000 - revolver_amount
```

**⚠️ Issue:** $10M revolver capacity is hardcoded
- **Risk:** If revolver terms change, calculation incorrect
- **Recommendation:** Pass as parameter or extract from file

---

### Output Schema

**Format:** data.table with 14 rows × 2 columns

| Row | Field | Amount (Thousands) |
|-----|-------|-------------------|
| 1 | Revenue | Numeric |
| 2 | Gross Profit | Numeric |
| 3 | GP Margin % | Percentage (0-1) |
| 4 | Adj. EBITDA | Numeric |
| 5 | EBITDA Margin % | Percentage (0-1) |
| 6 | Rolling LTM Revenue | "" (placeholder) |
| 7 | Rolling LTM Adj. EBITDA | "" (placeholder) |
| 8 | skip_row | "" (blank separator) |
| 9 | Senior Debt | Numeric |
| 10 | Sub Debt | Numeric |
| 11 | Total Debt | Numeric |
| 12 | Cash | Numeric |
| 13 | Revolver Availability | Numeric |
| 14 | Liquidity | Numeric |

**Note:** Rows 6-7 (Rolling LTM metrics) are placeholders - not calculated

---

### Use Cases

1. **Monthly Portfolio Monitoring:** Track Radius Holdings performance vs. plan
2. **Covenant Compliance:** Monitor debt levels, liquidity, EBITDA
3. **Investment Committee Updates:** Provide monthly updates on key investment
4. **Risk Management:** Early warning if metrics deteriorate

---

### Data Quality Considerations

#### Issues (4):
1. **Hardcoded revolver capacity** ($10M) - no validation
2. **No LTM calculations** - Rolling 12-month metrics not implemented (lines 79-80)
3. **Column name assumptions** - x2, x3, etc. assumed to exist
4. **No error handling** - Missing sheets/columns cause crash

#### Fragility:
- Sheet names must match parameters exactly
- Column structure must be consistent month-to-month
- Field names must match regex patterns (e.g., "WSFS Loan Payable")

---

## General Ledger (GL) System

### Overview

**System:** Accounting System (QuickBooks, NetSuite, or similar)
**Purpose:** Record all fund accounting transactions
**Integration Type:** Excel file export
**Direction:** GL System → Everside R Scripts (one-way)
**Frequency:** As-needed for IRR calculations

---

### Scripts Involved

#### IRR_Data_Tab_Creation.R
**Lines:** 979 (R function library)
**Type:** GL transaction processor

**Input:**
- `~/R Data/LTD Investment Transactions - {date}.xlsx`
- Example: "LTD Investment Transactions - 6.30.25 - Team Air update.xlsx"
- Single sheet with GL transaction data

**Output:**
- `~/R Output/IRR Data Output/{fund}-{start_date}-IRR_data_tab_output.csv`
- Example: "Fund IV-2025-04-01-IRR_data_tab_output.csv"

---

### GL Export Format

#### Expected Columns (10+ fields):

| Column | Type | Description |
|--------|------|-------------|
| legal_entity | String | Legal entity name (fund/feeder entity) |
| gl_date | Excel Date Serial | Transaction date |
| position | String | Investment position name (for Direct funds) |
| deal_name | String | Deal name (for Partnership funds) |
| trans_type | String | Transaction type code |
| dr_cr_amount | Numeric | Debit/credit amount (signed) |
| batch_id | String/Numeric | Batch identifier for related transactions |
| comments_batch | String | Batch-level comments |
| comments_transaction | String | Transaction-level comments |

---

### Transaction Types

**Cash Out (Investment):**
- "Cash Out - Investment"
- "Investment - Cost"
- "Interest Expense"

**Cash In (Returns):**
- "Cash In - Investment" (return of capital)
- "Interest Income"
- "Dividend Income"
- "Realized Gain"
- Management fee rebates
- Closing fees

---

### Entity Mapping

**Purpose:** Map fund names to legal entity list for GL filtering

**Fund Structure (6 funds, 28 entities):**

| Fund | Entity Count | Examples |
|------|--------------|----------|
| Fund I Founders | 1 | Everside Founders Fund, LP |
| Fund II | 5 | Everside Fund II, LP; Everside Fund II F1, LP; etc. |
| Fund III | 7 | Everside Fund III, LP; Everside Fund III [Entity], LP; etc. |
| Fund IV | 9 | Everside Fund IV, LP; Everside Fund IV Offshore Fund, Ltd; etc. |
| Direct I | 3 | Everside Direct I, LP; Everside Direct I F1, LP; etc. |
| Direct II | 3 | Everside Direct II, LP; Everside Direct II Offshore, LP; etc. |

**Total:** 28 unique legal entities across 6 funds

**Hardcoded:** Entity lists are hardcoded in IRR_Data_Tab_Creation.R (lines 50-120)

---

### GL Data Processing

#### 1. Batch ID Matching
**Purpose:** Link related transactions (investment + proceeds) by batch

**Logic (lines 133-142):**
```r
# Find all batch IDs with "Cash Out" transactions
fund_iv_batch_ids <- fund_iv_data_all %>%
  filter(trans_type %in% c("Cash Out - Investment", "Investment - Cost")) %>%
  select(batch_id) %>%
  pull()

# Filter for transactions in those batches
fund_iv_data <- fund_iv_data_all %>%
  filter(batch_id %in% fund_iv_batch_ids)
```

**Purpose:** Exclude standalone transactions not part of investment batches

---

#### 2. Debit/Credit Validation
**Purpose:** Ensure batch totals net to zero (accounting integrity)

**Logic (lines 145-158):**
```r
debit_credit_check <- fund_iv_data %>%
  group_by(batch_id) %>%
  summarize(total = sum(dr_cr_amount))

# Should sum to 0 for each batch (debits = credits)
```

**Note:** Validation check only prints warning, does not halt processing

---

#### 3. Position-Level Aggregation
**Purpose:** Create one row per position per date

**Logic:** Two loops (cash out, cash in) for each unique position
- Filter GL transactions for position
- Aggregate by transaction type
- Append to master dataframe

**Result:** 18-column IRR data format

---

### Output Schema (18 columns)

| Column | Type | Source |
|--------|------|--------|
| vintage_year | Numeric | year(gl_date) |
| transaction_date | Date | gl_date |
| position_name | String | position or deal_name |
| investment_type | Categorical | Regex classification (Debt/Equity/Partnership) |
| commitment | Numeric | NA_real_ (placeholder) |
| unfunded_commitment | Numeric | NA_real_ (placeholder) |
| gap | Numeric | NA_real_ (placeholder) |
| investment_amount | Numeric | Sum("Investment - Cost") |
| investment_expense | Numeric | Sum("Interest Expense") |
| proceeds_return_of_capital | Numeric | Sum("Cash In - Investment") |
| proceeds_interest_income | Numeric | Sum("Interest Income") |
| proceeds_dividend_income | Numeric | Sum("Dividend Income") |
| proceeds_realized_gain | Numeric | Sum("Realized Gain") |
| proceeds_mfee_rebate | Numeric | Sum(management fee rebates) |
| proceeds_closing_fee | Numeric | Sum(closing fees) |
| proceeds_due_to_manager | Numeric | NA_real_ (TODO: follow up with Hao) |
| realized_proceeds | Numeric | Sum of all proceeds columns |
| fund_name | String | legal_entity |

---

### GL System Characteristics

**Assumed System:** QuickBooks or NetSuite (based on export format)

**Export Process:**
1. Accounting team runs GL transaction report
2. Filter for investment-related accounts
3. Export to Excel
4. Save to ~/R Data/ directory
5. Name file with date identifier

**Manual Steps:**
- Filter GL accounts to investment transactions only
- Exclude Class B transactions (if present)
- Exclude certain comment patterns (e.g., "Transfer")

---

### Data Flow

```
Accounting System (QuickBooks/NetSuite)
  ↓ (manual export)
GL Transaction Excel
  ↓
IRR_Data_Tab_Creation.R
  ↓
IRR Data CSV (18 columns)
  ↓
IRR_Front_Page_Calcs.R
  ↓
Investor Report Metrics
```

---

### Known Issues

#### 1. No Direct GL API Integration
**Issue:** Manual export required from accounting system
**Impact:**
- Extra manual step
- Potential for human error (wrong filters, wrong accounts)
- Version control issues (multiple exports with different dates in filename)

---

#### 2. Entity Mapping Maintenance
**Issue:** 28 legal entities hardcoded in script (lines 50-120)
**Impact:** Adding new entity requires code change
**Recommendation:** Store in configuration table or lookup file

---

#### 3. No Fund Validation
**Issue:** If fund parameter doesn't match known funds, script processes with empty entity list
**Result:** Silent failure - no output generated

---

#### 4. Batch ID Dependency
**Issue:** Relies on batch_id integrity in source data
**Risk:** If accounting team doesn't use batch IDs consistently:
- Related transactions not linked
- Incomplete cash flows
- Incorrect IRR calculations

---

#### 5. Missing Calculations
**Issue:** 3 columns always NA:
- commitment
- unfunded_commitment
- proceeds_due_to_manager

**Status:** Either not implemented or awaiting specification

---

## System Integration Summary

| System | Type | Direction | Frequency | Integration Method | Automation Level |
|--------|------|-----------|-----------|-------------------|-----------------|
| SBA/SBIC | Regulatory | Everside → SBA | Quarterly | Excel upload | Semi-automated |
| Radius Holdings | Portfolio Co | Radius → Everside | Monthly | Excel import | Manual |
| GL System | Accounting | GL → Everside | As-needed | Excel export | Manual |
| Deal Cloud | CRM | Everside → Deal Cloud | Ad-hoc | Excel upload | Semi-automated |

---

## Common Patterns Across Systems

### 1. File-Based Integration
**All systems** use Excel files as integration layer
- No API integrations implemented
- Manual file movement required
- Version control via filename dates

---

### 2. Hardcoded Business Logic
**All scripts** contain hardcoded:
- File paths (~/R Data/, ~/R Output/)
- Entity lists (fund to legal entity mappings)
- Column structures (skiprows, column names)
- Business rules (revolver capacity, status mappings)

---

### 3. Limited Error Handling
**All scripts** lack:
- File existence validation
- Column existence validation
- Data type validation
- Try-catch error wrappers

---

### 4. No Data Lineage Tracking
**Missing across all systems:**
- Source file tracking (which GL export used for IRR calc?)
- Transformation history
- Version control of outputs
- Audit trails

---

## Recommendations for External System Improvements

### High Priority
1. **Add API integrations** where available (Deal Cloud, accounting system)
2. **Implement error handling** - graceful failures with clear messages
3. **Add input validation** - check required fields, data types, value ranges
4. **Configuration management** - move hardcoded values to config files

### Medium Priority
5. **Data lineage tracking** - log which source files produced which outputs
6. **Automated testing** - unit tests for data transformations
7. **Version control** - standardize output file naming and versioning
8. **Documentation** - document expected file formats and business rules

### Low Priority
9. **Workflow orchestration** - automate file movement and script execution
10. **Monitoring/alerting** - notify if expected files missing or processing fails

---

**End of External Systems Documentation (Part 2)**

**Systems Documented:** 3 (SBA/SBIC, Radius Holdings, GL System)
**Total Scripts Involved:** 5 (2 Form 468 processors + 1 Radius + 1 IRR + 1 IRR Front Page)
**Integration Type:** File-based (Excel) for all systems
**Automation Level:** Manual to semi-automated
