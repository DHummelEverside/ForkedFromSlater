# External Systems - Deal Cloud Integration

**System:** Deal Cloud CRM
**Purpose:** Customer Relationship Management for portfolio companies
**Integration Type:** Excel file import/export
**Direction:** Everside → Deal Cloud (one-way)

---

## Overview

Deal Cloud is Everside's CRM system for tracking portfolio company information. Three Python scripts convert internal investment data into Deal Cloud's required format.

**Integration Pattern:**
```
Investment Templates (Everside format)
  ↓
Parser Scripts (Python)
  ↓
Deal Cloud Import Files (Excel)
  ↓
[Manual upload to Deal Cloud CRM]
```

---

## Scripts Involved

### 1. Parse investment template to deal cloud file.py
**Lines:** 81
**Type:** Production script
**Use Case:** Regular investment template conversion

**Input:**
- `Investment Template_Farragut v4.xlsx` (sheet: 'eFile', skiprows=7)
- `deal_cloud_dataset_template.xlsx` (Deal Cloud template)

**Output:**
- `output_file_farragut_v4-v7.xlsx` (Deal Cloud import format)

**Key Feature:** Hardcoded for Farragut investment

---

### 2. Parse_investment_template_to_deal_cloud_file.ipynb
**Type:** Jupyter notebook (parameterized version)
**Use Case:** Ad-hoc investment template conversion

**Input:**
- `Investment Template - Oxer.xlsx` (sheet: 'eFile', skiprows=7)
- `deal_cloud_dataset_template_for_python.xlsx` (Deal Cloud template)

**Output:**
- `output_file_Oxer-v1.xlsx` (parameterized filename)

**Key Feature:** Easy to update input/output file names in cells

---

### 3. Historic_Everside_Deal_Cloud_parsing.ipynb
**Type:** Jupyter notebook (one-time migration)
**Use Case:** Historical data migration from old Deal Cloud format to new

**Input:**
- `Fund III DealCloud Model - Q4 2023 vDRAFT (03.24.24).xlsx` (sheet: 'Fund III (Q4 2023)', skiprows=2)
- `deal_cloud_dataset_template_for_python.xlsx` (Deal Cloud template)

**Output:**
- `Fund III 2023 Q4.xlsx`

**Key Feature:** Migrates existing Deal Cloud data to updated schema

**Status:** ⚠️ Likely one-time use, not part of regular workflow

---

## Data Sources

### From Everside: Investment Templates

#### Format: Excel Workbook
**Sheet:** 'eFile'
**Skip Rows:** 7 (header rows)

**Source Columns (23 fields):**

| Column Name | Type | Description |
|------------|------|-------------|
| Company Name | String | Portfolio company name |
| Industry | String | Industry classification |
| Date of Investment Memo | Date | Investment committee memo date |
| Revenue | Numeric | Company revenue (dual: at close + current) |
| Adj. EBITDA | Numeric | Adjusted EBITDA (dual: at close + current) |
| Implied EV | Numeric | Enterprise value (dual: at close + current) |
| Description | Text | Investment thesis/description |
| Sourcing | String | Deal sourcing channel |
| Sourcing Description | Text | Sourcing details |
| Investment | String | Investment type |
| Total Investment | Numeric | Total capital invested |
| Senior Debt.1 | Numeric | Senior debt amount (dual: at close + current) |
| Sr. Subordinated debt.1 | Numeric | Subordinated debt (dual: at close + current) |
| Cash | Numeric | Cash coupon (debt pricing) |
| Dividend/PIK | Numeric | PIK/dividend rate (debt pricing) |
| Cash.1 | Numeric | Cash coupon (equity pricing) |
| Dividend/PIK.1 | Numeric | PIK/dividend rate (equity pricing) |
| Ownership.1 | Numeric | Equity ownership percentage |
| Tenor | Numeric | Debt maturity/tenor |

**Notes:**
- Some columns appear twice (e.g., Revenue, Senior Debt) representing "at close" vs. "current" values
- "Dual" columns map to two different Deal Cloud columns

---

### From Everside: Historical Deal Cloud Export

**File:** `Fund III DealCloud Model - Q4 2023 vDRAFT (03.24.24).xlsx`
**Sheet:** 'Fund III (Q4 2023)'
**Skip Rows:** 2

**Source Columns (80+ fields):**
Key fields include:
- Company Name
- Date of Investment
- Industry, Geography
- Debt (Cost), Equity (Cost), Total (Cost)
- Everside $ Exposure, Everside % Exposure
- Debt/Equity FMV values
- Cash/PIK pricing (multiple sets)
- Senior Debt, Total Debt, Liquidity
- FCCR (Covenants, Current)
- Enterprise Value, EV Multiple
- Sales, EBITDA, Capex
- Current Status, Type of Debt
- Covenants Breach, Current on Interest
- Sponsor Type

**Purpose:** Migrate older Deal Cloud data format to new standardized schema

---

## Data Destination: Deal Cloud Template

### Template Files
- `deal_cloud_dataset_template.xlsx` (used by .py script)
- `deal_cloud_dataset_template_for_python.xlsx` (used by notebooks)

**Format:** Excel workbook with pre-defined columns
**Starting Row:** 2 (row 1 contains headers)

**Column Structure:** Letter-based Excel columns (A, B, C, ..., AZ, BA, etc.)

**Total Columns:** 100+ columns (extends to at least column DF = 110)

---

## Column Mappings

### Mapping 1: Investment Template Parser

**Source:** `Parse investment template to deal cloud file.py` (lines 10-33)

| Everside Column | Deal Cloud Column(s) | Transformation | Notes |
|----------------|---------------------|----------------|-------|
| Company Name | E | Direct copy | |
| Industry | H | Direct copy | |
| Date of Investment Memo | D | Direct copy | |
| Revenue | AZ, BO | 1-to-2 mapping | At close + current |
| Adj. EBITDA | BA, BP | 1-to-2 mapping | At close + current |
| Implied EV | AX, BM | 1-to-2 mapping | At close + current |
| Description | DE | Direct copy | Column 110 |
| Sourcing | J | Direct copy | |
| Sourcing Description | K | Direct copy | |
| Investment | U | Direct copy | Investment type |
| Total Investment | R | Direct copy | |
| Senior Debt.1 | AR, BD | 1-to-2 mapping | At close + current |
| Sr. Subordinated debt.1 | AT, BF | 1-to-2 mapping | At close + current |
| Cash | AJ | Direct copy | Debt pricing |
| Dividend/PIK | AK | Direct copy | Debt pricing |
| Cash.1 | AO | Direct copy | Equity pricing |
| Dividend/PIK.1 | AP | Direct copy | Equity pricing |
| Ownership.1 | AQ | Direct copy | Equity ownership % |
| Tenor | AN | Direct copy | Debt tenor |

**Calculated Columns:**

| Calculation | Formula | Target Column |
|------------|---------|---------------|
| Total Coupon (Debt) | Cash + Dividend/PIK | AL |
| Total Debt | Senior Debt.1 + Sr. Subordinated debt.1 | P |
| Equity Investment | Total Investment - Senior Debt.1 - Sr. Subordinated debt.1 | O |

---

### Mapping 2: Notebook Parser (Oxer Example)

**Source:** `Parse_investment_template_to_deal_cloud_file.ipynb` (cell 4)

**Differences from Mapping 1:**

| Everside Column | Deal Cloud Column(s) | Change Notes |
|----------------|---------------------|--------------|
| Industry | I (not H) | Column shifted right by 1 |
| Revenue | BA, BP (not AZ, BO) | Column shifted right by 1 |
| Adj. EBITDA | BB, BQ (not BA, BP) | Column shifted right by 1 |
| Implied EV | AY, BN (not AX, BM) | Column shifted right by 1 |
| Description | DF (not DE) | Column shifted right by 1 |
| Sourcing | K (not J) | Column shifted right by 1 |
| Sourcing Description | L (not K) | Column shifted right by 1 |
| Investment | V (not U) | Column shifted right by 1 |
| Total Investment | S (not R) | Column shifted right by 1 |
| Senior Debt.1 | AS, BE (not AR, BD) | Column shifted right by 1 |
| Sr. Subordinated debt.1 | AU, BG (not AT, BF) | Column shifted right by 1 |
| Cash | AK (same) | No change |
| Dividend/PIK | AL (not AK) | Column shifted right by 1 |
| Cash.1 | AP (same) | No change |
| Dividend/PIK.1 | AQ (not AP) | Column shifted right by 1 |
| Ownership.1 | AR (not AQ) | Column shifted right by 1 |
| Tenor | AO (not AN) | Column shifted right by 1 |

**Calculated Columns:**
| Calculation | Target Column | Change |
|------------|---------------|--------|
| Total Coupon (Debt) | AM (not AL) | Shifted right by 1 |
| Total Debt | Q (not P) | Shifted right by 1 |
| Equity Investment | P (not O) | Shifted right by 1 |

**⚠️ Issue:** Two different Deal Cloud templates exist with slightly different column layouts. This could cause import errors if wrong template used.

---

### Mapping 3: Historical Data Migration

**Source:** `Historic_Everside_Deal_Cloud_parsing.ipynb` (cell 4)

**34 column mappings** (subset shown):

| Source Column | Deal Cloud Column | Category |
|--------------|------------------|----------|
| Company Name | E | Identity |
| Geography | K | Classification |
| Industry | J | Classification |
| Date of Investment | I | Date |
| Debt (Cost) | Q | Investment |
| Equity (Cost) | R | Investment |
| Everside $ Exposure | W | Exposure |
| Everside % Exposure | X | Exposure |
| Debt (Cost).1 | AC | Cost Basis |
| Debt (FMV) | AE | Valuation |
| Equity (Cost).1 | AG | Cost Basis |
| Equity (FMV) | AI | Valuation |
| Cash (Debt Pricing) | AN | Pricing |
| PIK (Debt Pricing) | AO | Pricing |
| Total Coupon | AP | Pricing |
| Cash (Equity Pricing) | AS | Pricing |
| PIK.1 (Equity Pricing) | AT | Pricing |
| Equity Ownership (FD) | AU | Ownership |
| Senior Debt (At Close) | AV | Capital Structure |
| Total Debt (At Close) | AX | Capital Structure |
| Liquidity | AZ | Liquidity |
| FCCR (Covenants) | BA | Covenants |
| Enterprise Value (At Close) | BB | Valuation |
| Sales (At Close) | BD | Financials |
| EBITDA (At Close) | BE | Financials |
| Capex (At Close) | BF | Financials |
| Senior Debt.1 (Current) | BH | Capital Structure |
| Total Debt.1 (Current) | BJ | Capital Structure |
| Liquidity.1 (Current) | BO | Liquidity |
| FCCR (Current) | BP | Covenants |
| Enterprise Value.1 (Current) | BQ | Valuation |
| Sales.1 (Current) | BS | Financials |
| EBITDA.1 (Current) | BT | Financials |
| Capex.1 (Current) | BU | Financials |

**Pattern:** Historical format has more comprehensive fields including "at close" vs. "current" data for financial metrics.

---

## Standard Data Transformations

### 1. Pipeline Status Prefix
**Lines 46-48** (Parse investment template to deal cloud file.py)

```python
for i in range(len(data)):
    for col_num in range(1, 4):
        worksheet.cell(row=start_row + i, column=col_num, value="Pipeline")
```

**Result:** Columns A, B, C are set to "Pipeline" for all rows
**Purpose:** Status indicator in Deal Cloud (likely: Pipeline, Active, Exited)

---

### 2. Fund Identifier
**Historic script only** (lines in cell 6)

```python
for i in range(len(data)):
    for col_num in range(1, 2):
        worksheet.cell(row=start_row + i, column=col_num, value="Fund III")
```

**Result:** Column A is set to "Fund III"
**Purpose:** Fund classification for historical data

---

### 3. Combined Columns (Summation)

**Transformation:** Sum multiple source columns into single target

**Examples:**
- **Total Coupon:** Cash + Dividend/PIK → Column AL (or AM)
- **Total Debt:** Senior Debt.1 + Sr. Subordinated debt.1 → Column P (or Q)

**Code Pattern:**
```python
'CombinedColumn': {'sources': ['Cash', 'Dividend/PIK'], 'target': 'AL'}
```

---

### 4. Subtracted Columns

**Transformation:** Subtract sum of multiple columns from base value

**Example:**
- **Equity Investment:** Total Investment - (Senior Debt.1 + Sr. Subordinated debt.1) → Column O (or P)

**Code Pattern:**
```python
'SubtractedColumn': {'sources': ['Total Investment', 'Senior Debt.1', 'Sr. Subordinated debt.1'], 'target': 'O'}
```

**Logic:**
```python
value = row[sources[0]] - sum(row[source] for source in sources[1:])
```

---

## Column Letter Conversion

**Function:** `col_letter_to_num(letter)` (lines 39-43 in .py, similar in notebooks)

```python
def col_letter_to_num(letter):
    num = 0
    for char in letter:
        num = num * 26 + (ord(char.upper()) - ord('A') + 1)
    return num
```

**Purpose:** Convert Excel column letters (A, B, ..., AA, AB, ..., AZ, BA, etc.) to numeric indices for openpyxl

**Examples:**
- 'A' → 1
- 'Z' → 26
- 'AA' → 27
- 'AZ' → 52
- 'BA' → 53
- 'DE' → 109
- 'DF' → 110

---

## Data Flow Diagram

```
┌─────────────────────────────────────┐
│ Everside Investment Templates       │
│ - Investment Template_Farragut.xlsx │
│ - Investment Template - Oxer.xlsx   │
│ - Sheet: 'eFile', skiprows=7        │
│ - 23 source columns                 │
└──────────────┬──────────────────────┘
               ↓
┌──────────────────────────────────────┐
│ Parser Scripts (Python)              │
│ - Parse investment template...py     │
│ - Parse_investment_template...ipynb  │
│                                      │
│ Transformations:                     │
│ 1. Column remapping (letter-based)  │
│ 2. 1-to-2 mappings (at close + now) │
│ 3. Calculated columns (sums/diffs)  │
│ 4. Add "Pipeline" prefix             │
└──────────────┬───────────────────────┘
               ↓
┌──────────────────────────────────────┐
│ Deal Cloud Template Files            │
│ - deal_cloud_dataset_template.xlsx   │
│ - 100+ columns (A through DF+)       │
│ - Populated from row 2 onwards       │
└──────────────┬───────────────────────┘
               ↓
┌──────────────────────────────────────┐
│ Output Files (Deal Cloud Import)     │
│ - output_file_farragut_v4-v7.xlsx    │
│ - output_file_Oxer-v1.xlsx           │
│ - Fund III 2023 Q4.xlsx              │
└──────────────┬───────────────────────┘
               ↓
        [Manual Upload]
               ↓
┌──────────────────────────────────────┐
│ Deal Cloud CRM System                │
│ (External - web-based)               │
└──────────────────────────────────────┘
```

---

## Known Issues and Limitations

### 1. Two Different Template Versions
**Issue:** `deal_cloud_dataset_template.xlsx` vs. `deal_cloud_dataset_template_for_python.xlsx`
**Impact:** Column letters differ by 1 position (e.g., Industry is 'H' vs. 'I')
**Risk:** Using wrong template causes data to appear in wrong columns in Deal Cloud

**Recommendation:** Standardize to single template version

---

### 2. Hardcoded File Names
**Issue:** Input/output file names are hardcoded in scripts
**Impact:** Must edit code for each new investment
**Example:** `source_file_path = 'Investment Template_Farragut v4.xlsx'`

**Workaround:** Notebook versions allow easy cell editing

---

### 3. Manual Upload Required
**Issue:** No API integration - outputs must be manually uploaded to Deal Cloud
**Impact:** Extra step in workflow, potential for upload errors
**Recommendation:** Investigate Deal Cloud API for automated uploads

---

### 4. No Data Validation
**Issue:** Scripts do not validate:
- Required fields are populated
- Data types are correct
- Values are within expected ranges

**Risk:** Invalid data may be uploaded to Deal Cloud, causing:
- Import failures
- Data quality issues
- Manual cleanup required

---

### 5. Duplicate Column Mapping Keys
**Issue:** `column_mapping` dict has duplicate 'CombinedColumn' keys (lines 30-32)
**Example:**
```python
'CombinedColumn': {'sources': ['Cash', 'Dividend/PIK'], 'target': 'AL'},
'CombinedColumn': {'sources': ['Senior Debt.1', 'Sr. Subordinated debt.1'], 'target': 'P'},
```
**Impact:** Python dicts can't have duplicate keys - second definition overwrites first
**Result:** May not calculate both combined columns correctly

**Fix Required:** Use unique keys (e.g., 'CombinedColumn_Coupon', 'CombinedColumn_Debt')

---

### 6. No Error Handling
**Missing:**
- File not found handling
- Missing column handling
- Null/NA value handling
- Excel write errors

**Result:** Script crashes on any error with cryptic stack trace

---

## Deal Cloud Schema Insights

### Column Categories (Inferred)

| Column Range | Category | Fields |
|-------------|----------|--------|
| A-D | Status/Metadata | Pipeline status, dates |
| E-L | Company Identity | Name, industry, geography, sourcing |
| M-U | Investment Type | Investment structure, type |
| V-AB | Investment Amounts | Debt cost, equity cost, total |
| AC-AR | Cost Basis & Pricing | Debt/equity cost, cash/PIK rates |
| AS-BB | At Close Metrics | Senior debt, total debt, FCCR, EV, sales, EBITDA |
| BC-BU | Current Metrics | Current debt, liquidity, FCCR, EV, sales, EBITDA |
| BV+ | Unknown | Additional fields not mapped in analyzed scripts |

**Total Columns:** At least 110 (column DF documented)

---

## Usage Pattern

### Typical Workflow:

1. **Prepare Investment Template**
   - Deal team completes investment template Excel file
   - Sheet 'eFile' contains all required data

2. **Run Parser Script**
   - For new investments: Use notebook version (easy to change file names)
   - For standardized process: Use .py script (if file naming consistent)

3. **Review Output**
   - Open generated Excel file
   - Verify data mapped correctly
   - Check calculated columns

4. **Upload to Deal Cloud**
   - Log into Deal Cloud CRM
   - Import Excel file
   - Validate import succeeded

5. **Verify in Deal Cloud**
   - Check company appears in system
   - Verify all fields populated correctly

---

## Related Systems

### Upstream (Data Sources)
- **Investment Committee Process:** Creates investment templates
- **Deal Team:** Populates investment memo data

### Downstream (Data Consumers)
- **Deal Cloud CRM:** Primary consumer of parsed data
- **LP Reporting:** May pull from Deal Cloud for reports
- **Portfolio Monitoring:** Deal Cloud tracks company status

---

## Maintenance Notes

### To Add New Field to Mapping:

1. Identify source column name in investment template
2. Identify target column letter in Deal Cloud template
3. Add to `column_mapping` dict:
   ```python
   'Source Column Name': 'Target Letter'
   ```
4. For calculated fields, use dict format:
   ```python
   'UniqueKey': {'sources': ['Col1', 'Col2'], 'target': 'Letter'}
   ```

### To Change Deal Cloud Template:

1. Update template file (deal_cloud_dataset_template.xlsx)
2. Review all column letters in mapping - may need adjustment
3. Test with sample data
4. Update documentation

---

## Summary

**Integration Type:** File-based (Excel)
**Direction:** One-way (Everside → Deal Cloud)
**Frequency:** Ad-hoc (per new investment)
**Automation Level:** Semi-automated (manual upload required)

**Scripts:** 3 Python scripts (1 .py + 2 .ipynb)
**Source Columns:** 23-80 fields (depends on source)
**Target Columns:** 100+ columns (A-DF minimum)
**Transformations:** 3 types (direct copy, 1-to-2, calculated)

**Key Limitation:** No API integration - requires manual upload to Deal Cloud after file generation.

---

**End of Deal Cloud Integration Documentation**

**Created:** 2025-11-12
**Scripts Analyzed:** 3 (Parse to Deal Cloud + Historic migration)
**Column Mappings Documented:** 3 variations
**Issues Identified:** 6 (template versions, hardcoded names, no validation, etc.)
