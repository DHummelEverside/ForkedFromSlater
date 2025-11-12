# 468 Excel Processor - Output Generation (Part 3)

**Script:** `468_Excel_Processor.ipynb`
**Focus:** Output file creation and schema (Cells 13, 16-17)
**Outputs:** Two Excel files (v1.xlsx intermediate, v3x.xlsx final)

---

## Output Files Overview

### Output Flow
```
equity_df + loan_df → Book1.xlsx template → v1.xlsx (intermediate)
                                              ↓
                                   + S12pc data → v3x.xlsx (FINAL)
```

### Files Written
1. **v1.xlsx** - Intermediate output with merged equity/debt data
2. **v3x.xlsx** - Final output with S12pc enrichment

---

## Cell 13: First Output - v1.xlsx (Merged Equity and Debt)

### Purpose
Merge processed equity and loan DataFrames into unified template structure.

### Input Files
- **Template:** `Book1.xlsx` (empty template with column structure)
- **Data Sources:** `equity_df` and `loan_df` (processed DataFrames)

### Output File
- **Filename:** `v1.xlsx`
- **Format:** Excel (.xlsx)
- **Sheets:** Single sheet (default "Sheet1")
- **Engine:** openpyxl

### Merging Logic

#### Step 1: Load Template
```python
template_file_path = "Book1.xlsx"
excel_df = pd.read_excel(template_file_path)
```
**Purpose:** Load empty template with desired output column structure.

**Note:** Book1.xlsx column structure defines final output schema. If template has columns not in equity_df or loan_df, those columns remain empty/NaN.

#### Step 2: Create Company List
```python
all_companies = set(loan_df["Portfolio Company Name"]).union(set(equity_df["Company Name"]))
excel_df["Company Name"] = list(all_companies)
```
**Logic:**
- Union of all company names from both equity and loan DataFrames
- Handles companies that appear in only one DataFrame (debt-only or equity-only)
- Uses set to ensure uniqueness
- Overwrites template's "Company Name" column

**Result:** One row per unique portfolio company

#### Step 3: Populate Data Row by Row
```python
for index, row in excel_df.iterrows():
    company_name = row['Company Name']
    loan_row = loan_df[loan_df['Portfolio Company Name'] == company_name]
    equity_row = equity_df[equity_df['Company Name'] == company_name]

    for col in excel_df.columns:
        if col == 'Company Name':
            continue  # Already populated
        elif col in loan_df.columns and not loan_row.empty:
            excel_df.loc[index, col] = loan_row[col].iloc[0]
        elif col in equity_df.columns and not equity_row.empty:
            excel_df.loc[index, col] = equity_row[col].iloc[0]
```

**Logic for Each Company:**
1. Look up company in `loan_df` by 'Portfolio Company Name'
2. Look up company in `equity_df` by 'Company Name'
3. For each output column:
   - Skip 'Company Name' (already set)
   - If column exists in loan_df AND company has debt → copy debt value
   - Else if column exists in equity_df AND company has equity → copy equity value
   - Otherwise → leave as NaN

**Priority:** Loan data takes precedence over equity data if column exists in both.

**Key Behavior:**
- Companies with only debt → equity columns remain NaN
- Companies with only equity → debt columns remain NaN
- Companies with both → debt columns from loan_df, equity columns from equity_df

### Column Mapping Strategy

The output template (Book1.xlsx) likely contains columns from both sources:

**From loan_df (29 columns):**
- Portfolio Company Name → mapped via lookup
- Employer ID
- Financing Type
- Investment Type
- Financing Description
- Initial Financing Date
- Equity Capital Investment
- Interest or Dividend Rate
- Ownership % (Fully Diluted)
- Loan/ Debt Status
- Total Cash Invested (A)
- Cost at Beginning Period
- Schedule 1C Reference Number
- Addition/\nDeduction
- Non-Cash Gain included in Cost at End of Period
- Cost at End of Period
- Unrealized Appreciation
- (Unrealized Depreciation)
- SBA Reported Value (B)
- GAAP Reported Value (C)
- Cum. Cash Proceeds (D)
- Current SBA Mult
- Current GAAP Mult
- Prior GAAP Mult
- First Investment Date
- **Debt Pricing - Maturity**
- **Status Compliance of Credit - Current Status**
- **Status Compliance of Credit - Covenants Breach**
- **Status Compliance of Credit - Current on Interest**

**From equity_df (24 columns):**
- Company Name → used for lookup
- Employer ID
- Financing Type
- Investment Type
- (... many overlapping columns ...)
- **Equity Pricing - Equity Ownership (FD)** (unique)
- **Invested Dollars - Equity Cost** (unique)
- **Invested Dollars - Equity FMV** (unique)

**Template-Only Columns (to be filled in Cell 17):**
- At Close Sales
- Current Sales
- At Close EBITDA
- Current EBITDA
- At Close Enterprise Value
- At Close Total Debt
- Current Total Debt
- Current Enterprise Value
- Date of Investment
- Company Address

### Commented-Out Alternative Approach
```python
'''
for col in excel_df.columns:
    if col in lc:
      excel_df[col] = loan_df[col]
    elif col in ec:
      excel_df[col] = equity_df[col]
    else:
      continue
'''
```
**Note:** This simpler approach was commented out. It would directly copy columns without company-by-company matching. Current approach is more robust for handling companies that don't appear in both DataFrames.

### Write to File
```python
with pd.ExcelWriter(new_file_path, engine='openpyxl') as writer:
    excel_df.to_excel(writer, index=False)

print("Excel file successfully populated and saved as", new_file_path)
```

**Parameters:**
- `index=False` - Don't write row numbers as first column
- `engine='openpyxl'` - Excel writer engine for .xlsx format

### v1.xlsx Output Schema

**Structure:**
- **Rows:** One per portfolio company (equity + debt union)
- **Columns:** All columns from Book1.xlsx template (exact list depends on template)
- **Data Completeness:**
  - Debt-only companies: equity-specific columns are NaN
  - Equity-only companies: debt-specific columns are NaN
  - Mixed companies: populated from both sources

**At This Stage, MISSING:**
- Detailed financial metrics (Sales, EBITDA, Enterprise Value)
- Company address
- Date of Investment (if not in S1Inv)

These are added in Cell 17 from S12pc sheet.

---

## Cell 16: S12pc Data Parser

### Purpose
Define function to extract detailed portfolio company financials from Form 468 Schedule 12.

### Input File
- **File:** `2024.06.30 - St. Cloud Fund III Form 468.xlsx`
- **Sheet:** "S12pc" (Schedule 12 - Portfolio Company Details)
- **Structure:** Repeated 48-row blocks per company

### S12pc Sheet Structure

**Layout:** Each company occupies exactly 48 rows with fixed structure:
- Rows 0-10: Company identification and high-level metrics
- Rows 11-18: Ownership and governance
- Rows 19-30+: Time-series financial data (4 periods)

**Offset Pattern:** Company N starts at row `offset = (N-1) * 48`

### Extraction Function

```python
def collect_information(data, offset):
    dict = {
        # Company Identification
        "Name of License": data.iloc[offset + 2, 1],
        "Portfolio Company": data.iloc[offset + 3, 2],
        "Date of Investment": (data.iloc[offset + 4, 2]).strftime('%m/%d/%Y'),
        "NAICS": data.iloc[offset + 5, 2],
        "HQ City": data.iloc[offset + 6, 2],
        "HQ State": data.iloc[offset + 6, 5],
        "Company Address": data.iloc[offset + 6, 2] + ", " + data.iloc[offset + 6, 5],

        # Valuation at Investment
        "EV at 1st Closing": data.iloc[offset + 7, 2],
        "SBA Reported Valued": data.iloc[offset + 9, 2],

        # Ownership Structure
        "Fund Own%": data.iloc[offset + 4, 8],
        "Associate Own%": data.iloc[offset + 5, 8],
        "Management Own%": data.iloc[offset + 6, 8],
        "Board Representation": data.iloc[offset + 7, 8],
        "Vote%": data.iloc[offset + 9, 8],

        # Investment Performance
        "Invest. Allocation": data.iloc[offset + 4, 11],
        "Invested Capital": data.iloc[offset + 5, 11],
        "Realized Proceeds": data.iloc[offset + 6, 11],
        "GAAP Value": data.iloc[offset + 7, 11],
        "Investment Multiple": data.iloc[offset + 8, 11],
        "Gross IRR": data.iloc[offset + 9, 11],

        # Time-Series Data (4 periods each)
        "As of Date": [data.iloc[offset + 19, 2], data.iloc[offset + 19, 3],
                       data.iloc[offset + 19, 4], data.iloc[offset + 19, 5]],
        "Sales": [data.iloc[offset + 20, 2], data.iloc[offset + 20, 3],
                  data.iloc[offset + 20, 4], data.iloc[offset + 20, 5]],
        "YOY % Growth": [data.iloc[offset + 21, 2], data.iloc[offset + 21, 3],
                         data.iloc[offset + 21, 4], data.iloc[offset + 21, 5]],
        "EBITDA": [data.iloc[offset + 22, 2], data.iloc[offset + 22, 3],
                   data.iloc[offset + 22, 4], data.iloc[offset + 22, 5]],
        "EBITDA Margin": [data.iloc[offset + 24, 2], data.iloc[offset + 24, 3],
                          data.iloc[offset + 24, 4], data.iloc[offset + 24, 5]],
        "Enterprise Value": [data.iloc[offset + 20, 8], data.iloc[offset + 20, 9],
                             data.iloc[offset + 20, 10], data.iloc[offset + 20, 11]],
        "TEV Multiple": [data.iloc[offset + 21, 8], data.iloc[offset + 21, 9],
                         data.iloc[offset + 21, 10], data.iloc[offset + 21, 11]],
        "Total Debt": [data.iloc[offset + 22, 8], data.iloc[offset + 22, 9],
                       data.iloc[offset + 22, 10], data.iloc[offset + 22, 11]],
        "Total Lev Multiple": [data.iloc[offset + 23, 8], data.iloc[offset + 23, 9],
                               data.iloc[offset + 23, 10], data.iloc[offset + 23, 11]],
        "Liquidity (Cash + Available Liquidity)": [data.iloc[offset + 24, 8],
                                                     data.iloc[offset + 24, 9],
                                                     data.iloc[offset + 24, 10],
                                                     data.iloc[offset + 24, 11]],

        # Point-in-Time Metrics (At Close vs Current)
        "At Close Sales": data.iloc[offset + 20, 2],
        "Current Sales": data.iloc[offset + 20, 5],
        "At Close EBITDA": data.iloc[offset + 22, 2],
        "Current EBITDA": data.iloc[offset + 22, 5],
        "At Close Enterprise Value": data.iloc[offset + 20, 8],
        "At Close Total Debt": data.iloc[offset + 22, 8],
        "At Close Liquidity (Cash + Available Liquidity)": data.iloc[offset + 23, 8],
        "Current Total Debt": data.iloc[offset + 22, 11],
        "Current Enterprise Value": data.iloc[offset + 20, 11]
    }
    return dict
```

### Data Extracted (36 Fields)

#### Company Identification (7 fields)
- Name of License
- Portfolio Company (used for matching)
- Date of Investment (formatted as MM/DD/YYYY string)
- NAICS code
- HQ City
- HQ State
- Company Address (concatenated City + State)

#### Valuation (2 fields)
- EV at 1st Closing
- SBA Reported Valued

#### Ownership & Governance (5 fields)
- Fund Own%
- Associate Own%
- Management Own%
- Board Representation
- Vote%

#### Investment Performance (6 fields)
- Invest. Allocation
- Invested Capital
- Realized Proceeds
- GAAP Value
- Investment Multiple
- Gross IRR

#### Time-Series Arrays (10 arrays, 4 periods each)
- As of Date [period 1, period 2, period 3, period 4]
- Sales (same structure)
- YOY % Growth
- EBITDA
- EBITDA Margin
- Enterprise Value
- TEV Multiple
- Total Debt
- Total Lev Multiple
- Liquidity

**Note:** Time-series data stored as Python lists, not individual columns.

#### Point-in-Time Comparisons (9 fields)
- At Close Sales (period 1)
- Current Sales (period 4)
- At Close EBITDA (period 1)
- Current EBITDA (period 4)
- At Close Enterprise Value (period 1)
- At Close Total Debt (period 1)
- At Close Liquidity (period 1)
- Current Total Debt (period 4)
- Current Enterprise Value (period 4)

**Note:** These are extracted from the time-series arrays for convenience.

### Cell Location Guide

| Data Field | Row | Column | Description |
|------------|-----|--------|-------------|
| Name of License | offset+2 | 1 (B) | SBIC license name |
| Portfolio Company | offset+3 | 2 (C) | Company name for matching |
| Date of Investment | offset+4 | 2 (C) | Investment date (datetime) |
| NAICS | offset+5 | 2 (C) | Industry code |
| HQ City | offset+6 | 2 (C) | City |
| HQ State | offset+6 | 5 (F) | State |
| EV at 1st Closing | offset+7 | 2 (C) | Initial enterprise value |
| As of Date [0-3] | offset+19 | 2-5 (C-F) | Period dates |
| Sales [0-3] | offset+20 | 2-5 (C-F) | Revenue by period |
| EBITDA [0-3] | offset+22 | 2-5 (C-F) | EBITDA by period |
| Enterprise Value [0-3] | offset+20 | 8-11 (I-L) | EV by period |
| Total Debt [0-3] | offset+22 | 8-11 (I-L) | Debt by period |

### Positional Indexing Risk

**Critical:** Function uses positional indexing (row number, column number) with hardcoded offsets.

**Assumptions:**
- Each company occupies exactly 48 rows
- Data fields are at consistent offsets within each block
- S12pc structure never changes

**Fragility:** If SBA changes Form 468 layout, all offsets break.

---

## Cell 17: Final Output - v3x.xlsx (With S12pc Enrichment)

### Purpose
Enrich v1.xlsx with detailed financial data from S12pc, write final output.

### Input Files
- **v1.xlsx** - Intermediate file from Cell 13
- **S12pc data** - Loaded in Cell 16

### Output File
- **Filename:** `v3x.xlsx`
- **Format:** Excel (.xlsx)
- **Sheets:** Single sheet (default "Sheet1")
- **Engine:** openpyxl

### Enrichment Process

#### Step 1: Load Intermediate File
```python
template_file_path = 'v1.xlsx'
excel_df = pd.read_excel(template_file_path)
```
**Note:** v1.xlsx becomes the "template" for final output.

#### Step 2: Define Target Columns
```python
cols = ["At Close Sales", "Current Sales", "At Close EBITDA", "Current EBITDA",
        "At Close Enterprise Value", "At Close Total Debt", "Current Total Debt",
        "Current Enterprise Value", "Date of Investment", "Company Address"]
```
**Purpose:** Whitelist of columns to update from S12pc data. Only these 10 columns are enriched.

**Why Limited?** S12pc contains 36 fields, but many are time-series arrays or duplicate S1Inv data. Only add net-new fields needed for output.

#### Step 3: Parse S12pc and Match Companies
```python
offset = 0
for i in range(number_of_companies):
    # Check if company data exists (HQ State is not NaN)
    if pd.isna(data.iloc[offset + 6, 5]):
        offset += 48
        continue  # Skip empty blocks
    else:
        company = collect_information(data, offset)

    offset += 48  # Move to next company block

    # Find matching company in excel_df
    company_row = excel_df[excel_df['Company Name'] == company["Portfolio Company"]]

    # Update only whitelisted columns
    for key, value in company.items():
        if key in cols and not company_row.empty:
            excel_df.loc[company_row.index, key] = value
```

**Logic:**
1. Iterate through S12pc in 48-row blocks
2. Check if block has data (HQ State at offset+6, column 5 is not NaN)
3. If data exists, extract company info using `collect_information()`
4. Match company by name: `excel_df['Company Name'] == company["Portfolio Company"]`
5. Update only the 10 whitelisted columns
6. Move offset by 48 rows to next company

**Key Behavior:**
- Companies in S12pc but not in v1.xlsx → ignored (no new rows added)
- Companies in v1.xlsx but not in S12pc → columns remain NaN
- Only updates 10 specific columns, ignores other 26 fields from S12pc

**Variable Source:**
- `number_of_companies` - Calculated in Cell 15 as `len(set(equity_df['Employer ID'] + loan_df['Employer ID'])) + 15`
- **Note:** Adding 15 suggests template/padding for additional companies in S12pc not in S1Inv

#### Step 4: Re-derive Boolean Columns
```python
excel_df['Status Compliance of Credit - Covenants Breach'] = excel_df['Status Compliance of Credit - Current Status'].apply(
    lambda x: True if x == "Covenant Issues" else False
)

excel_df['Status Compliance of Credit - Covenants Breach'] = excel_df['Status Compliance of Credit - Covenants Breach'].astype(bool)
excel_df['Status Compliance of Credit - Current on Interest'] = excel_df['Status Compliance of Credit - Current on Interest'].astype(bool)
```

**Purpose:** Ensure boolean columns are typed correctly after merging operations.

**Logic:**
1. Re-calculate covenant breach from status column (redundant check)
2. Cast both boolean columns to bool type (may have been object type after Excel operations)

**Note:** Covenant breach already calculated in Cell 4. This is defensive coding to ensure final output has correct types.

#### Step 5: Write Final Output
```python
with pd.ExcelWriter(new_file_path, engine='openpyxl') as writer:
    excel_df.to_excel(writer, index=False)

print("Excel file successfully populated and saved as", new_file_path)
```

**Parameters:**
- `index=False` - No row numbers
- `engine='openpyxl'` - Excel format

### v3x.xlsx Final Output Schema

**Structure:**
- **Rows:** One per portfolio company (equity + debt union from v1.xlsx)
- **Columns:** All columns from Book1.xlsx template + 10 enriched from S12pc
- **Source Breakdown:**
  - From S1Inv (via equity_df/loan_df): Investment positions, valuations, returns
  - From S4Del: Interest payment status
  - From S12pc: Financial metrics, company address, detailed performance

### Final Column List (Estimated)

**Note:** Exact columns depend on Book1.xlsx template. Based on code analysis:

#### Company Identification
- Company Name (key)
- Employer ID
- Date of Investment (from S12pc)
- Company Address (from S12pc)
- NAICS (in S12pc dict, not in cols whitelist → not in output)

#### Investment Classification
- Financing Type
- Investment Type
- Financing Description

#### Investment Amounts
- Total Cash Invested (A)
- Cost at Beginning Period
- Cost at End of Period (or Invested Dollars - Equity Cost for equity)
- Invested Dollars - Equity Cost (equity only)
- Invested Dollars - Equity FMV (equity only)
- SBA Reported Value (B)
- GAAP Reported Value (C) (or Invested Dollars - Equity FMV for equity)
- Cum. Cash Proceeds (D)

#### Debt-Specific
- Debt Pricing - Maturity
- Interest or Dividend Rate
- Status Compliance of Credit - Current Status
- Status Compliance of Credit - Covenants Breach (boolean)
- Status Compliance of Credit - Current on Interest (boolean)
- Loan/ Debt Status

#### Equity-Specific
- Equity Pricing - Equity Ownership (FD)
- Ownership % (Fully Diluted) (debt only)

#### Financial Metrics (from S12pc)
- **At Close Sales** ← enriched
- **Current Sales** ← enriched
- **At Close EBITDA** ← enriched
- **Current EBITDA** ← enriched
- **At Close Enterprise Value** ← enriched
- **At Close Total Debt** ← enriched
- **Current Total Debt** ← enriched
- **Current Enterprise Value** ← enriched

#### Dates
- Initial Financing Date
- First Investment Date
- Date of Investment (from S12pc) ← enriched

#### Other
- Schedule 1C Reference Number
- Addition/\nDeduction
- Non-Cash Gain included in Cost at End of Period
- Unrealized Appreciation
- (Unrealized Depreciation)
- Current SBA Mult
- Current GAAP Mult
- Prior GAAP Mult
- Equity Capital Investment

### Columns NOT Included from S12pc

Despite being extracted by `collect_information()`, these are NOT added to final output (not in `cols` whitelist):
- Name of License
- NAICS
- HQ City
- HQ State
- EV at 1st Closing
- SBA Reported Valued
- Fund Own%, Associate Own%, Management Own%
- Board Representation
- Vote%
- Invest. Allocation
- Invested Capital
- Realized Proceeds
- GAAP Value
- Investment Multiple
- Gross IRR
- All time-series arrays (Sales[4], EBITDA[4], etc.)
- At Close Liquidity
- TEV Multiple, Total Lev Multiple, etc.

**Rationale:** These may be:
1. Already present from S1Inv
2. Not needed for intended use case
3. Too detailed/granular for summary output
4. Time-series data difficult to fit in flat structure

---

## Output Comparison

### v1.xlsx (Intermediate)
**Contains:**
- All equity and debt investment data from S1Inv
- Debt status mappings
- Covenant and interest payment flags
- Aggregated by portfolio company

**Missing:**
- Detailed financial metrics (Sales, EBITDA, Enterprise Value, Debt)
- Company address
- Date of Investment (if not in S1Inv)

**Use Case:** Checkpoint file for debugging, partial output

### v3x.xlsx (Final)
**Contains:**
- Everything from v1.xlsx
- Plus 10 enriched columns from S12pc
- Properly typed boolean columns

**Missing:**
- Time-series data (only point-in-time snapshots)
- 26 other S12pc fields not whitelisted

**Use Case:** Complete regulatory/internal reporting file

---

## Data Flow Summary

```
Form 468 S1Inv Sheet
  ↓ (Cell 1-3: Load, clean, split)
equity_df (24 cols) + loan_df (29 cols)
  ↓ (Cell 4-12: Transform, aggregate, enrich from S4Del)
Processed DataFrames
  ↓ (Cell 13: Merge into Book1.xlsx template)
v1.xlsx (Intermediate, ~50+ columns)
  ↓ (Cell 16: Parse S12pc function defined)
  ↓ (Cell 17: Enrich with 10 S12pc columns)
v3x.xlsx (FINAL OUTPUT, ~60+ columns)
```

---

## Output File Characteristics

### File Format
- **Extension:** .xlsx (Excel 2007+)
- **Engine:** openpyxl (pure Python, no Excel required)
- **Compression:** Default ZIP compression
- **Compatibility:** Opens in Excel, Google Sheets, LibreOffice

### Data Types in Output
- **Strings:** Company names, descriptions, status categories
- **Numeric:** Dollar amounts, percentages, multipliers
- **Dates:** Stored as Excel datetime (may display as numbers if not formatted)
- **Booleans:** Covenant breach, interest current (typed as bool in v3x.xlsx)
- **NaN/None:** Empty cells for missing data

### Row Count
- One row per unique portfolio company
- Count = `len(set(equity companies) ∪ set(debt companies))`
- Typically 20-50 companies for a fund

### Column Count
- v1.xlsx: ~50-60 columns (depends on Book1.xlsx template)
- v3x.xlsx: Same as v1.xlsx (enriches existing columns, doesn't add new ones)

---

## Potential Output Issues

### 1. Company Name Mismatches
**Problem:** S12pc uses "Portfolio Company" field to match against v1.xlsx "Company Name".
- If names differ (typos, abbreviations, legal entity differences) → no match → missing S12pc data
- No fuzzy matching or validation

**Impact:** Companies in v1.xlsx remain without financial metrics.

### 2. Missing S12pc Data
**Problem:** `if pd.isna(data.iloc[offset + 6, 5]):` skips company if HQ State is NaN.
- Incomplete S12pc entries silently skipped
- offset still increments → next company may be misaligned

**Impact:** Some companies missing enrichment data.

### 3. 48-Row Block Assumption
**Problem:** Hardcoded 48-row offset for each company.
- If S12pc has different block size → data extraction fails
- If companies have variable row counts → all subsequent companies misaligned

**Impact:** Incorrect data mapping, potential crashes.

### 4. Boolean Type Inconsistency
**Problem:** Cell 17 re-casts boolean columns after merge.
- Suggests boolean type may be lost during Excel operations
- May display as TRUE/FALSE text instead of checkbox in some Excel versions

**Impact:** Downstream tools may not recognize boolean values correctly.

### 5. Time-Series Data Loss
**Problem:** S12pc contains 4-period time-series but only at-close and current values exported.
- Historical trend data lost
- Period 2 and 3 ignored

**Impact:** Cannot analyze growth trajectories or interim performance.

---

## Best Practices for Output

### Validating Output Files

**After running script, verify:**
1. Row count matches expected company count
2. No entirely empty rows
3. Key columns populated (Company Name, Total Cash Invested, SBA Reported Value)
4. Boolean columns display as checkboxes (not TRUE/FALSE text)
5. Dates formatted as dates (not 5-digit Excel serials)
6. Spot-check: pick 2-3 companies and verify data against source Form 468

### Common Fixes

**If output has issues:**
- **Missing S12pc data:** Check company name spelling in S1Inv vs. S12pc
- **Wrong data in columns:** Verify Book1.xlsx column order matches expected
- **Boolean columns as text:** Manually format as boolean in Excel, or fix openpyxl write options
- **Dates as numbers:** Apply date format to date columns after opening

---

## Output Usage

### v3x.xlsx Typical Use Cases
1. **SBA Reporting:** Submit to SBA as part of Form 468 package
2. **Internal Dashboards:** Import into BI tools (Tableau, Power BI)
3. **LP Reporting:** Extract portfolio company summaries for investor reports
4. **Risk Analysis:** Feed into risk rating models (Risk_Rating_Function)
5. **Data Review:** Input to Data_Review script for validation

### Downstream Scripts
Based on repository analysis, v3x.xlsx or similar files may be:
- Loaded into "Everside Dataset.xlsx" (central data warehouse)
- Processed by Data_Review script for quality checks
- Used by high_level_metrics_review for fund-level aggregations

---

**End of Part 3: Output Generation**

**Document Status:** Complete - covers Cells 13, 16-17 (all output operations)
**Files Documented:** v1.xlsx (intermediate), v3x.xlsx (final)
**Last Updated:** 2025-11-12
