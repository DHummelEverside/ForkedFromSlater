# 468 Excel Processor - Processing Logic (Part 2)

**Script:** `468_Excel_Processor.ipynb`
**Focus:** Data transformation, aggregation, and enrichment (Cells 4-12)
**Scope:** Processing logic only - stops before final output writing

---

## Cell 4: Debt-Specific Transformations

### Purpose
Enrich loan DataFrame with derived columns for maturity dates, status mapping, and covenant compliance.

### Input
- **DataFrame:** `loan_df` (debt/loan investments only)
- **Columns Used:**
  - `Financing Description` - Text description containing maturity date
  - `Loan/ Debt Status` - Raw status from Form 468

### Transformations

#### 1. Extract Maturity Date from Text
```python
pattern = r'Maturity Date: (\d{2}/\d{2}/\d{4})'
loan_df["Debt Pricing - Maturity"] = loan_df["Financing Description"].str.extract(pattern)
```
**Logic:**
- Uses regex to find "Maturity Date: MM/DD/YYYY" pattern in Financing Description field
- Extracts only the date portion
- Result is string (not parsed to datetime)
- If pattern not found, returns NaN

**Example:**
- Input: "Senior Secured Note, Maturity Date: 12/31/2025, Rate: 12%"
- Output: "12/31/2025"

#### 2. Map Debt Status to Standardized Categories
```python
debt_status = {
    "Paid in Full": "Exited",
    "Performing": "Performing",
    "Delinquent/Default": "Exited",
    "Charge Off": "Exited",
    "Covenant Issues": "Under Expectation",
    "Other Concerns": "Under Expectation",
    "Foreberance": "Under Expectation"
}
loan_df["Status Compliance of Credit - Current Status"] = loan_df["Loan/ Debt Status"].map(debt_status)
```
**Status Mapping:**
| Original Status | Mapped Status |
|----------------|---------------|
| Paid in Full | Exited |
| Performing | Performing |
| Delinquent/Default | Exited |
| Charge Off | Exited |
| Covenant Issues | Under Expectation |
| Other Concerns | Under Expectation |
| Foreberance | Under Expectation |

**Note:** "Foreberance" appears to be a typo of "Forbearance" in source data.

#### 3. Flag Covenant Breaches
```python
loan_df["Status Compliance of Credit - Covenants Breach"] = loan_df["Loan/ Debt Status"].apply(
    lambda x: True if x == "Covenant Issues" else False
)
```
**Logic:**
- Boolean column
- True only if status is exactly "Covenant Issues"
- All other statuses (including "Other Concerns") = False

### Output
- **DataFrame:** `loan_df` (modified in place)
- **New Columns Added:**
  1. `Debt Pricing - Maturity` (string, format: MM/DD/YYYY)
  2. `Status Compliance of Credit - Current Status` (string: "Exited" | "Performing" | "Under Expectation")
  3. `Status Compliance of Credit - Covenants Breach` (boolean)

### Notes
- These columns are debt-specific; `equity_df` does not receive them
- Maturity date remains as string, not converted to datetime
- Covenant breach is narrow definition (only "Covenant Issues", not "Other Concerns")

---

## Cell 5: Display Equity DataFrame

### Purpose
Visual inspection of equity investments after initial filtering.

### Code
```python
equity_df
```

### Action
Displays DataFrame in notebook output for manual review.

**No transformations performed.**

---

## Cell 6: Portfolio Company Aggregation

### Purpose
Consolidate multiple investment positions into single row per portfolio company.

### Background
Form 468 S1Inv can have multiple rows per company if:
- Company received multiple rounds of financing
- Multiple investment types (e.g., preferred stock + warrants)
- Follow-on investments at different dates

This cell aggregates all positions into one row per company.

### Input
- **DataFrames:** `equity_df` and `loan_df` (separate processing)
- **Key Column:** `Portfolio Company Name` (grouping key)

### Aggregation Function

```python
def aggregate_values(group):
    # Start with first row as template
    result = group.iloc[0].copy()

    # Sum all dollar amounts
    result['Total Cash Invested (A)'] = group['Total Cash Invested (A)'].sum()
    result['Cost at Beginning Period'] = group['Cost at Beginning Period'].sum()
    result['Non-Cash Gain included in Cost at End of Period'] = group['Non-Cash Gain included in Cost at End of Period'].sum()
    result['Unrealized Appreciation'] = group['Unrealized Appreciation'].sum()
    result['Cost at End of Period'] = group['Cost at End of Period'].sum()
    result['(Unrealized Depreciation)'] = group['(Unrealized Depreciation)'].sum()
    result['SBA Reported Value (B)'] = group['SBA Reported Value (B)'].sum()
    result['GAAP Reported Value (C)'] = group['GAAP Reported Value (C)'].sum()
    result['Cum. Cash Proceeds (D)'] = group['Cum. Cash Proceeds (D)'].sum()
    result['Addition/\nDeduction'] = group['Addition/\nDeduction'].sum()

    # Take earliest dates
    result['Initial Financing Date'] = group['Initial Financing Date'].min()
    result['First Investment Date'] = group['First Investment Date'].min()

    # Special handling for ownership percentage
    ownership = group['Ownership % (Fully Diluted)']
    if ownership.isna().all():
        result['Ownership % (Fully Diluted)'] = np.nan
    else:
        result['Ownership % (Fully Diluted)'] = ownership.fillna(0).sum()

    # Select first non-"Other" investment type
    investment_types = group['Investment Type']
    non_other_types = investment_types[investment_types != 'Other'].dropna().unique()
    if len(non_other_types) > 0:
        result['Investment Type'] = non_other_types[0]
    else:
        result['Investment Type'] = 'Other'

    return result
```

### Aggregation Rules by Column Type

#### Numeric Columns - Sum
All dollar amounts are summed across positions:
- Total Cash Invested (A)
- Cost at Beginning Period
- Non-Cash Gain included in Cost at End of Period
- Unrealized Appreciation
- Cost at End of Period
- (Unrealized Depreciation)
- SBA Reported Value (B)
- GAAP Reported Value (C)
- Cum. Cash Proceeds (D)
- Addition/\nDeduction

#### Date Columns - Minimum (Earliest)
Dates take the earliest value:
- Initial Financing Date → `min()`
- First Investment Date → `min()`

**Rationale:** Represents when company first received investment.

#### Ownership Percentage - Conditional Sum
```python
if ownership.isna().all():
    result = np.nan
else:
    result = ownership.fillna(0).sum()
```
**Logic:**
- If all positions have NaN ownership → result is NaN
- If any position has ownership percentage → fill NaNs with 0, then sum
- Assumes ownership percentages are additive across positions

**Example:**
- Position 1: 15% ownership
- Position 2: NaN
- Position 3: 5% ownership
- **Result:** 15% + 0% + 5% = 20%

#### Investment Type - First Non-"Other"
```python
non_other_types = investment_types[investment_types != 'Other'].dropna().unique()
if len(non_other_types) > 0:
    result = non_other_types[0]
else:
    result = 'Other'
```
**Logic:**
- Filters out "Other" and NaN values
- Takes first remaining investment type
- If only "Other" types exist, result is "Other"

**Priority:** Specific types > "Other"

**Example:**
- Position 1: "Preferred Stock"
- Position 2: "Other"
- Position 3: "Warrants (Equity)"
- **Result:** "Preferred Stock" (first non-Other)

#### All Other Columns - First Row Value
```python
result = group.iloc[0].copy()
```
Non-aggregated columns inherit value from first row:
- Portfolio Company Name
- Employer ID
- Financing Type
- Financing Description
- Interest or Dividend Rate
- Loan/ Debt Status
- Schedule 1C Reference Number
- Current SBA Mult
- Current GAAP Mult
- Prior GAAP Mult
- Plus any debt-specific columns added in Cell 4

### Application

```python
equity_df = equity_df.groupby('Portfolio Company Name').apply(aggregate_values).reset_index(drop=True)
loan_df = loan_df.groupby('Portfolio Company Name').apply(aggregate_values).reset_index(drop=True)
```

**Process:**
1. Group by `Portfolio Company Name`
2. Apply `aggregate_values` function to each group
3. Reset index to create clean sequential row numbers

### Output
- **DataFrames:** `equity_df` and `loan_df` (modified in place)
- **Structure:** One row per portfolio company
- **Row Count:** Reduced from ~N positions to ~M companies (where M < N)

### Example Aggregation

**Before (Multiple Positions):**
| Portfolio Company | Investment Type | Total Cash Invested | Ownership % |
|-------------------|----------------|---------------------|-------------|
| Acme Corp | Preferred Stock | $1,000,000 | 10% |
| Acme Corp | Warrants | $50,000 | 2% |
| Acme Corp | Common Stock | $500,000 | 5% |

**After (Aggregated):**
| Portfolio Company | Investment Type | Total Cash Invested | Ownership % |
|-------------------|----------------|---------------------|-------------|
| Acme Corp | Preferred Stock | $1,550,000 | 17% |

### Notes
- Aggregation logic assumes positions are additive (true for dollar amounts and ownership)
- First row's non-numeric values become canonical (potential data loss if values differ)
- Investment Type selection is arbitrary (first non-Other) - may not represent largest position

---

## Cell 7: Display Loan DataFrame

### Purpose
Visual inspection of aggregated loan investments.

### Code
```python
loan_df
```

### Action
Displays aggregated `loan_df` for manual review.

**No transformations performed.**

---

## Cell 8: Display Loan Columns

### Purpose
Inspect column names after aggregation to verify structure.

### Code
```python
loan_df.columns
```

### Output
Returns Index object with current column names.

**No transformations performed.**

---

## Cell 9: Standardize Column Names

### Purpose
Rename columns to align with Everside's internal data model, differentiating equity and debt fields.

### Input
- **DataFrames:** `loan_df` and `equity_df`
- **Current Columns:** Original Form 468 column names + derived columns from Cells 4 and 6

### Transformations

#### Loan DataFrame Column Renaming
```python
loan_df.columns = [
    'Portfolio Company Name',
    'Employer ID',
    'Financing Type',
    'Investment Type',
    'Financing Description',
    'Initial Financing Date',
    'Equity Capital Investment',
    'Interest or Dividend Rate',
    'Ownership % (Fully Diluted)',
    'Loan/ Debt Status',
    'Total Cash Invested (A)',
    'Cost at Beginning Period',
    'Schedule 1C Reference Number',
    'Addition/\nDeduction',
    'Non-Cash Gain included in Cost at End of Period',
    'Cost at End of Period',
    'Unrealized Appreciation',
    '(Unrealized Depreciation)',
    'SBA Reported Value (B)',
    'GAAP Reported Value (C)',
    'Cum. Cash Proceeds (D)',
    'Current SBA Mult',
    'Current GAAP Mult',
    'Prior GAAP Mult',
    'First Investment Date',
    'Debt Pricing - Maturity',                        # Added in Cell 4
    'Status Compliance of Credit - Current Status',   # Added in Cell 4
    'Status Compliance of Credit - Covenants Breach'  # Added in Cell 4
]
```

**Note:** Most columns retain original names. Key additions are debt-specific fields from Cell 4.

#### Equity DataFrame Column Renaming
```python
equity_df.columns = [
    'Company Name',                            # Changed from 'Portfolio Company Name'
    'Employer ID',
    'Financing Type',
    'Investment Type',
    'Financing Description',
    'Initial Financing Date',
    'Equity Capital Investment',
    'Interest or Dividend Rate',
    'Equity Pricing - Equity Ownership (FD)',  # Changed from 'Ownership % (Fully Diluted)'
    'Loan/ Debt Status',
    'Total Cash Invested (A)',
    'Cost at Beginning Period',
    'Schedule 1C Reference Number',
    'Addition/\nDeduction',
    'Non-Cash Gain included in Cost at End of Period',
    'Invested Dollars - Equity Cost',          # Changed from 'Cost at End of Period'
    'Unrealized Appreciation',
    '(Unrealized Depreciation)',
    'SBA Reported Value (B)',
    'Invested Dollars - Equity FMV',           # Changed from 'GAAP Reported Value (C)'
    'Cum. Cash Proceeds (D)',
    'Current SBA Mult',
    'Current GAAP Mult',
    'Prior GAAP Mult',
    'First Investment Date'
]
```

### Key Renamings

| Original Column | Equity Rename | Loan Rename |
|-----------------|---------------|-------------|
| Portfolio Company Name | **Company Name** | Portfolio Company Name |
| Ownership % (Fully Diluted) | **Equity Pricing - Equity Ownership (FD)** | Ownership % (Fully Diluted) |
| Cost at End of Period | **Invested Dollars - Equity Cost** | Cost at End of Period |
| GAAP Reported Value (C) | **Invested Dollars - Equity FMV** | GAAP Reported Value (C) |

### Additional Variable Assignment
```python
ec = ['Company Name', 'Equity Pricing - Equity Ownership (FD)', 'Invested Dollars - Equity Cost', 'Invested Dollars - Equity FMV']
lc = ['Company Name', 'Debt Pricing - Total Coupon', 'Invested Dollars - Debt Cost', 'Invested Dollars - Debt FMV']
```

**Purpose:** Define column lists for later merging (though `lc` references columns not yet created).

**Note:** `lc` references "Debt Pricing - Total Coupon", "Invested Dollars - Debt Cost", "Invested Dollars - Debt FMV" which **do not exist** in `loan_df` at this point. These may be expected from output template or are errors.

### Output
- **DataFrames:** `equity_df` and `loan_df` (columns renamed in place)
- **Variables:** `ec` and `lc` lists created

### Notes
- Equity DataFrame gets more semantic column names aligned with internal standards
- Loan DataFrame retains mostly original Form 468 names
- Renaming is positional (by index), not by matching names - **fragile if column order changes**
- Variable `lc` contains non-existent column names (likely template placeholders)

---

## Cell 10: Display Loan DataFrame

### Purpose
Visual inspection after column renaming.

### Code
```python
loan_df
```

### Action
Displays renamed `loan_df` for manual review.

**No transformations performed.**

---

## Cell 11: Add Interest Payment Status from S4Del Sheet

### Purpose
Enrich loan DataFrame with whether companies are current on interest payments.

### Background
SBA Form 468 Schedule 4 (S4Del) tracks delinquent/past due amounts. This cell:
1. Loads S4Del data
2. Checks for past due amounts
3. Adds boolean flag to `loan_df`

### Input Files
- **File:** `2024.06.30 - St. Cloud Fund III Form 468.xlsx`
- **Sheet:** "S4Del" (Schedule 4 - Delinquencies)
- **Rows Skipped:** 9

### S4Del Data Load
```python
source_file_path = "2024.06.30 - St. Cloud Fund III Form 468.xlsx"
start_row = 9
data = pd.read_excel(source_file_path, sheet_name="S4Del", skiprows=start_row)
```

### Transformation Logic

#### Step 1: Define "Zero" Values
```python
zero = [0, 0.0, None, np.nan, pd.NA]
```
**Purpose:** List of values considered "no past due amount."

#### Step 2: Check for Past Due Amounts
```python
loan_df["Status Compliance of Credit - Current on Interest"] = data.apply(
    lambda x: (
        (x["Portfolio Company Name"] not in zero and x[' Amount Past Due'] not in zero)
        if pd.notna(x["Portfolio Company Name"]) and pd.notna(x[' Amount Past Due'])
        else False
    ),
    axis=1
)
```

**Logic:**
- For each row in S4Del `data`:
  - Check if `Portfolio Company Name` is not null AND not a "zero" value
  - Check if ` Amount Past Due` (note leading space) is not null AND not a "zero" value
  - If both conditions true → **True** (has past due amount)
  - Otherwise → **False**
- Result: Boolean series from S4Del

**Important:** This creates a column with same length as S4Del, not matched to `loan_df` companies yet.

#### Step 3: Invert Logic (Current = NOT Past Due)
```python
loan_df["Status Compliance of Credit - Current on Interest"] = loan_df["Status Compliance of Credit - Current on Interest"].apply(
    lambda x: False if x else True
)
```

**Logic:**
- If past due flag is True (has past due) → set to False (NOT current)
- If past due flag is False (no past due) → set to True (IS current)

**Result:** True = current on interest, False = delinquent on interest

### Output
- **DataFrame:** `loan_df` (modified in place)
- **New Column:** `Status Compliance of Credit - Current on Interest` (boolean)
  - **True:** Company is current on interest payments (no past due amount in S4Del)
  - **False:** Company has past due interest (amount listed in S4Del)

### Data Quality Issue

**Critical Problem:** The code creates a boolean series from S4Del and assigns it to `loan_df` **without matching company names**.

**Assumptions:**
1. S4Del rows are in same order as loan_df rows
2. Both DataFrames have same length
3. Row N in S4Del corresponds to row N in loan_df

**Risk:** If row orders don't align, wrong companies get flagged as delinquent.

**Better Approach (Not Used):**
```python
# Create lookup from S4Del
past_due_companies = data[data[' Amount Past Due'] > 0]['Portfolio Company Name'].tolist()

# Match by company name
loan_df["Status Compliance of Credit - Current on Interest"] = ~loan_df["Portfolio Company Name"].isin(past_due_companies)
```

### Notes
- Column name has leading space: `' Amount Past Due'` (typo in Excel)
- Logic double-inverts (check for past due, then flip to current)
- No validation that S4Del and loan_df have matching companies/order
- Equity DataFrame does not receive this column (interest payment status is debt-only concept)

---

## Cell 12: Display Loan DataFrame

### Purpose
Visual inspection after adding interest payment status.

### Code
```python
loan_df
```

### Action
Displays `loan_df` with new "Current on Interest" column for manual review.

**No transformations performed.**

---

## Processing Summary

### Data Flow Through Cells 4-12

```
Input: equity_df, loan_df (from Cell 3 split)
  ↓
Cell 4: Add debt-specific columns to loan_df
  - Debt Pricing - Maturity (extracted from text)
  - Status Compliance of Credit - Current Status (mapped from status)
  - Status Compliance of Credit - Covenants Breach (boolean flag)
  ↓
Cell 6: Aggregate by Portfolio Company Name
  - Sum dollar amounts
  - Min dates
  - Sum ownership %
  - Select first non-Other investment type
  ↓
Cell 9: Rename columns
  - Equity: semantic names (Company Name, Equity Pricing, Invested Dollars)
  - Loan: mostly original names
  ↓
Cell 11: Add interest payment status to loan_df
  - Load S4Del sheet
  - Check for past due amounts
  - Add "Current on Interest" boolean
  ↓
Output: Processed equity_df and loan_df ready for merging
```

### Transformations by DataFrame

#### Equity DataFrame (`equity_df`)
1. **Filtered** from `df` (Equity financing OR Warrants) - Cell 3
2. **Aggregated** by Portfolio Company Name - Cell 6
3. **Renamed** columns to internal standards - Cell 9
4. **Final Columns (24):**
   - Company Name
   - Employer ID
   - Financing Type
   - Investment Type
   - Financing Description
   - Initial Financing Date
   - Equity Capital Investment
   - Interest or Dividend Rate
   - Equity Pricing - Equity Ownership (FD)
   - Loan/ Debt Status
   - Total Cash Invested (A)
   - Cost at Beginning Period
   - Schedule 1C Reference Number
   - Addition/\nDeduction
   - Non-Cash Gain included in Cost at End of Period
   - Invested Dollars - Equity Cost
   - Unrealized Appreciation
   - (Unrealized Depreciation)
   - SBA Reported Value (B)
   - Invested Dollars - Equity FMV
   - Cum. Cash Proceeds (D)
   - Current SBA Mult
   - Current GAAP Mult
   - Prior GAAP Mult
   - First Investment Date

#### Loan DataFrame (`loan_df`)
1. **Filtered** from `df` (Loan/Debt AND NOT Warrants) - Cell 3
2. **Added** debt-specific columns - Cell 4
3. **Aggregated** by Portfolio Company Name - Cell 6
4. **Renamed** columns (mostly original names) - Cell 9
5. **Added** interest payment status from S4Del - Cell 11
6. **Final Columns (29):**
   - Portfolio Company Name
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
   - **Debt Pricing - Maturity** (added Cell 4)
   - **Status Compliance of Credit - Current Status** (added Cell 4)
   - **Status Compliance of Credit - Covenants Breach** (added Cell 4)
   - **Status Compliance of Credit - Current on Interest** (added Cell 11)

### Key Differences Between Equity and Loan DataFrames

| Aspect | Equity DataFrame | Loan DataFrame |
|--------|------------------|----------------|
| **Row Filter** | Financing Type = 'Equity' OR Investment Type = Warrants | Financing Type = 'Loan'/'Debt' AND Investment Type ≠ Warrants |
| **Company Name Column** | 'Company Name' | 'Portfolio Company Name' |
| **Ownership Column** | 'Equity Pricing - Equity Ownership (FD)' | 'Ownership % (Fully Diluted)' |
| **Cost Column** | 'Invested Dollars - Equity Cost' | 'Cost at End of Period' |
| **Value Column** | 'Invested Dollars - Equity FMV' | 'GAAP Reported Value (C)' |
| **Debt-Specific Columns** | None | Maturity, Current Status, Covenants Breach, Current on Interest |
| **Total Columns** | 24 | 29 |

---

## Key Processing Patterns

### 1. Separation Then Parallel Processing
- Split data early (Cell 3)
- Process equity and debt separately
- Apply same operations (aggregation) but different outputs (column names, additional fields)

### 2. Positional Column Renaming
```python
loan_df.columns = [long list]
```
- Fragile: Depends on column order staying constant
- No validation that old names match
- Order matters - one extra/missing column breaks everything

### 3. Text Parsing for Structured Data
```python
pattern = r'Maturity Date: (\d{2}/\d{2}/\d{4})'
loan_df["Debt Pricing - Maturity"] = loan_df["Financing Description"].str.extract(pattern)
```
- Unstructured text field contains structured data
- Regex extraction assumes consistent format
- No error handling if pattern not found (returns NaN)

### 4. Aggregation with Mixed Rules
- Dollar amounts: Sum
- Dates: Minimum
- Ownership: Conditional sum
- Investment Type: First non-"Other"
- Everything else: First row

**Implication:** Some data loss during aggregation (e.g., if multiple positions have different descriptions)

### 5. Cross-Sheet Data Enrichment
- Load additional sheets (S4Del) to add derived columns
- **Risk:** Assumes row alignment between sheets (no key-based join)

---

## Potential Issues

### 1. S4Del Row Alignment
Cell 11 assumes S4Del rows align with loan_df rows. If companies are in different orders or counts differ, wrong companies get flagged as delinquent.

### 2. Positional Column Renaming
Cell 9 renames by position. If Form 468 changes column order or adds/removes columns, renaming breaks silently (wrong column gets wrong name).

### 3. Investment Type Selection
Aggregation selects "first non-Other" investment type. For companies with multiple investment types (e.g., Preferred + Common), choice is arbitrary and may not represent largest/most important position.

### 4. Ownership Percentage Summing
Assumes ownership percentages are additive. Could overstate ownership if Form 468 records:
- Overlapping positions
- Already-aggregated values
- Fully-diluted percentages that shouldn't be summed

### 5. Unmatched Columns in `lc` Variable
```python
lc = ['Company Name', 'Debt Pricing - Total Coupon', 'Invested Dollars - Debt Cost', 'Invested Dollars - Debt FMV']
```
These columns don't exist in `loan_df`. Variable appears unused or meant for output template that has these column names.

---

## Next Steps (Part 3)

Remaining cells to document:
- **Cell 13:** Merge equity and loan data into template, write v1.xlsx (OUTPUT)
- **Cell 16:** Load S12pc sheet, parse detailed company financials
- **Cell 17:** Merge S12pc data into v1.xlsx, write v3x.xlsx (FINAL OUTPUT)

---

**End of Part 2: Processing Logic**

**Document Status:** Complete - covers Cells 4-12 (all processing logic before output writing)
**Last Updated:** 2025-11-12
