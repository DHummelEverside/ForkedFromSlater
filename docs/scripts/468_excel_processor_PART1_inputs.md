# 468 Excel Processor - Input Analysis (Part 1)

**Script:** `468_Excel_Processor.ipynb`
**Purpose:** Process SBA Form 468 regulatory data for St. Cloud Fund III
**Total Cells:** 20 cells
**Focus:** Input files, initial setup, and data structures only

---

## Input Files

### Primary Input File
**File:** `2024.06.30 - St. Cloud Fund III Form 468.xlsx`

**Sheets Accessed:**
1. **"S1Inv"** (Schedule 1 - Investments)
   - First sheet read
   - Contains portfolio company investment details
   - Rows skipped: 19 (header row at row 20)
   - Column headers taken from first data row

2. **"S4Del"** (Schedule 4 - Delinquencies)
   - Second sheet read
   - Contains past due information
   - Rows skipped: 9
   - Used for interest payment status validation

3. **"S12pc"** (Schedule 12 - Portfolio Company Details)
   - Third sheet read
   - Contains detailed company financials and metrics
   - No rows skipped (starts at row 1)
   - Used for enhanced company information

### Template Files (Also Inputs)
1. **"Book1.xlsx"**
   - Output template loaded
   - Used as structure for merged data output
   - Type: Empty template with column structure

2. **"v1.xlsx"**
   - Intermediate output from earlier step
   - Re-loaded as input for final processing
   - Contains merged equity and loan data

---

## Initial Setup

### Cell 0: Colab Link
**Type:** Markdown
**Content:** Google Colab badge for opening notebook in Colab
```markdown
<a href="https://colab.research.google.com/github/slatej1/Everside/Everside/blob/main/468_Excel_Processor.ipynb">
  Open In Colab
</a>
```
**Purpose:** Provides web link for cloud-based execution

---

### Cell 1: Library Imports and Initial Data Load
**Type:** Code

**Imports:**
```python
import pandas as pd           # Data manipulation
import datetime               # Date handling
import numpy as np            # Numerical operations
import warnings               # Warning suppression
from openpyxl import load_workbook  # Excel file manipulation
```

**Configuration:**
```python
warnings.filterwarnings("ignore")      # Suppress warning messages
pd.options.display.max_columns = None  # Show all columns in output
pd.options.display.max_rows = None     # Show all rows in output
```

**File Path Definition:**
```python
file_path = '2024.06.30 - St. Cloud Fund III Form 468.xlsx'
```

**Initial Data Load (S1Inv Sheet):**
```python
ironwood = pd.ExcelFile(file_path)          # Create Excel file object
start_row = 19                               # Skip first 19 rows
df = pd.read_excel(ironwood, "S1Inv", skiprows=start_row)  # Read S1Inv sheet
```

**Header Handling:**
```python
new_header = df.iloc[0]    # First row contains column names
df = df[1:]                # Remove header row from data
df.columns = new_header    # Set column names
```

**Date Parsing:**
```python
# Parse Initial Financing Date
df['Initial Financing Date'] = pd.to_datetime(
    df['Initial Financing Date'],
    format="%Y/%m/%d"
).dt.normalize()

# Parse First Investment Date
df['First Investment Date'] = pd.to_datetime(
    df['First Investment Date'],
    format="%Y/%m/%d"
).dt.normalize()
```

**Column Removal:**
```python
dropped = [
    "Critical Technology\n(if applicable)",
    "Restructured?",
    "Class 1",
    "Class 2",
    "Class 2 Date of Up Round",
    "Prior SBA Mult"
]
df = df.drop(dropped, axis=1)  # Remove unwanted columns
```

**Purpose:** Sets up environment, loads primary investment data, cleans column structure, parses dates

---

### Cell 2: Display DataFrame
**Type:** Code
```python
df
```
**Purpose:** Display the loaded and cleaned DataFrame for inspection

---

### Cell 3: Create Equity and Loan Subsets
**Type:** Code

**Classification Lists:**
```python
loan = ['Loan', 'Debt']  # Loan financing types
warrants = ["Warrants (Debt)", "Warrants (Equity)"]  # Warrant investment types
```

**Data Separation:**
```python
# Equity DataFrame: Equity financing OR warrant investments
equity_df = df[(df['Financing Type'] == 'Equity') |
               (df["Investment Type"].isin(warrants))]

# Loan DataFrame: Loan/Debt financing BUT NOT warrants
loan_df = df[(df['Financing Type'].isin(loan)) &
             (~df["Investment Type"].isin(warrants))]
```

**Purpose:** Separates investments into equity and debt categories based on financing and investment types

---

## Data Structures Created

### Primary DataFrames

#### 1. `df` (Master DataFrame)
**Source:** Sheet "S1Inv" from Form 468
**Initial Size:** All rows from row 20 onward (after header)
**Structure:**
- Rows: One per investment position (multiple rows per company if multiple investments)
- Columns: All Form 468 S1Inv columns except 6 dropped columns

**Key Columns Present (from S1Inv):**
- `Portfolio Company Name` - Company identifier
- `Employer ID` - Tax ID number
- `Financing Type` - Values: 'Equity', 'Loan', 'Debt'
- `Investment Type` - Specific instrument type (includes 'Warrants (Debt)', 'Warrants (Equity)', 'Other', etc.)
- `Financing Description` - Text description of financing terms
- `Initial Financing Date` - Date (parsed to datetime)
- `First Investment Date` - Date (parsed to datetime)
- `Equity Capital Investment` - Categorical
- `Interest or Dividend Rate` - Numeric
- `Ownership % (Fully Diluted)` - Numeric percentage
- `Loan/ Debt Status` - Status indicator
- `Total Cash Invested (A)` - Dollar amount
- `Cost at Beginning Period` - Dollar amount
- `Schedule 1C Reference Number` - Reference ID
- `Addition/\nDeduction` - Adjustment amount
- `Non-Cash Gain included in Cost at End of Period` - Dollar amount
- `Cost at End of Period` - Dollar amount
- `Unrealized Appreciation` - Dollar amount
- `(Unrealized Depreciation)` - Dollar amount (negative)
- `SBA Reported Value (B)` - Dollar amount
- `GAAP Reported Value (C)` - Dollar amount
- `Cum. Cash Proceeds (D)` - Dollar amount
- `Current SBA Mult` - Multiplier
- `Current GAAP Mult` - Multiplier
- `Prior GAAP Mult` - Multiplier

**Note:** Column names are taken directly from row 20 of the Excel file. Exact spelling and spacing preserved.

#### 2. `equity_df` (Equity Subset)
**Source:** Filtered from `df`
**Filter Criteria:**
- `Financing Type` == 'Equity' OR
- `Investment Type` in ['Warrants (Debt)', 'Warrants (Equity)']
**Structure:** Same columns as `df`, subset of rows
**Purpose:** Isolate equity and equity-like (warrant) investments

#### 3. `loan_df` (Debt Subset)
**Source:** Filtered from `df`
**Filter Criteria:**
- `Financing Type` in ['Loan', 'Debt'] AND
- `Investment Type` NOT in ['Warrants (Debt)', 'Warrants (Equity)']
**Structure:** Same columns as `df`, subset of rows
**Purpose:** Isolate pure debt investments (excluding warrants)

#### 4. `data` (S4Del DataFrame) - Loaded Later
**Source:** Sheet "S4Del" from Form 468
**Rows Skipped:** 9
**Purpose:** Contains past due information for interest payment validation
**Key Columns Expected:**
- `Portfolio Company Name`
- ` Amount Past Due` (note: leading space in column name)

#### 5. `data` (S12pc DataFrame) - Loaded Later (Variable Reused)
**Source:** Sheet "S12pc" from Form 468
**Rows Skipped:** 0 (reads from row 1)
**Purpose:** Detailed portfolio company financial metrics
**Access Pattern:** Uses `data.iloc[offset + row_number, column_number]` positional indexing
**Contains:** Company financials, valuation metrics, operational data (detailed extraction in later cells)

### Supporting Variables

**File Path Variables:**
```python
file_path = '2024.06.30 - St. Cloud Fund III Form 468.xlsx'  # Main input
template_file_path = 'Book1.xlsx'                             # Output template
source_file_path = '2024.06.30 - St. Cloud Fund III Form 468.xlsx'  # Alternate reference
new_file_path = 'v1.xlsx'                                     # First output
```

**Configuration Variables:**
```python
start_row = 19          # S1Inv skip rows
start_row = 9           # S4Del skip rows (variable reused)
ironwood = pd.ExcelFile # Excel file object for S1Inv
```

**Classification Lists:**
```python
loan = ['Loan', 'Debt']
warrants = ["Warrants (Debt)", "Warrants (Equity)"]
dropped = [list of 6 column names to remove]
```

---

## Column Headers Source

**Important:** Column names are **NOT hardcoded** in Cell 1. They are extracted from the Excel file:

```python
new_header = df.iloc[0]  # Row 20 of Excel (after skipping 19) becomes column names
df.columns = new_header  # Apply these names to DataFrame
```

This means:
- Column names match exactly what appears in row 20 of the S1Inv sheet
- Any typos, extra spaces, or special characters in Excel are preserved
- The specific columns listed above are inferred from later code usage, not from Cell 1

To see actual column names, one would need to:
1. Open the Form 468 Excel file
2. Navigate to S1Inv sheet
3. Look at row 20

---

## Data Loading Strategy

### Multi-Pass Approach:
1. **Pass 1 (Cell 1):** Load S1Inv sheet → `df` → split into `equity_df` and `loan_df`
2. **Pass 2 (Later cell):** Load S4Del sheet → `data` → validate interest payment status
3. **Pass 3 (Later cell):** Load S12pc sheet → `data` → extract detailed financials
4. **Pass 4 (Later cell):** Load template Book1.xlsx → merge processed data
5. **Pass 5 (Later cell):** Reload v1.xlsx → add S12pc data → write v3x.xlsx

### Why Multiple Passes?
- Each Form 468 schedule contains different information
- S1Inv: Core investment positions
- S4Del: Delinquency/payment status
- S12pc: Detailed portfolio company metrics
- Different sheets have different structures requiring separate parsing logic

---

## Initial Data Quality Notes

### Date Handling:
- **Format:** "%Y/%m/%d" expected (e.g., "2024/06/30")
- **Normalization:** `.dt.normalize()` removes time component, keeps date only
- **Columns Parsed:** 'Initial Financing Date', 'First Investment Date'
- **Other dates:** Likely exist but not parsed in Cell 1

### Dropped Columns Rationale:
The 6 dropped columns are likely:
- Not needed for downstream processing
- Contain incomplete data
- Are SBA-specific fields not used in Everside's internal system
- Include "Critical Technology", "Restructured", "Class 1/2", "Prior SBA Mult"

### Missing Data Handling:
Not addressed in initial load. Later cells may handle:
- NaN values in ownership percentages
- Empty investment amounts
- Missing dates

---

## Hardcoded Values Identified

### File Names:
```python
'2024.06.30 - St. Cloud Fund III Form 468.xlsx'  # Specific quarter and fund
'Book1.xlsx'                                      # Generic template name
```
**Implication:** Script must be manually updated for:
- Different quarters (change date)
- Different funds (change fund name)
- Different template files

### Row Skip Counts:
```python
start_row = 19  # S1Inv
start_row = 9   # S4Del
```
**Implication:** Form 468 structure must remain consistent. If SBA changes form layout, these values break.

### Column Names to Drop:
```python
["Critical Technology\n(if applicable)", "Restructured?", ...]
```
**Implication:** Exact column name matches required, including newline characters and punctuation.

---

## Input Dependencies Summary

**Required Files:**
1. Form 468 Excel file (specific quarter/fund)
2. Book1.xlsx template (empty structure)

**External Dependencies:**
- Python packages: pandas, numpy, datetime, warnings, openpyxl
- Excel file must conform to SBA Form 468 structure
- Row 20 of S1Inv must contain column headers

**Assumptions:**
- S1Inv sheet starts data at row 20 (after 19 header/metadata rows)
- S4Del sheet starts data at row 10 (after 9 header rows)
- S12pc sheet starts data at row 1
- Column names in Excel are stable across quarters
- Date format is YYYY/MM/DD

---

## Next Steps (Not Covered in Part 1)

Part 2 will document:
- Data transformation logic (aggregation function)
- Merging of equity and loan DataFrames
- S4Del integration for interest status
- S12pc parsing for detailed metrics
- Output file generation (v1.xlsx, v3x.xlsx)

---

**End of Part 1: Input Analysis**

**Document Status:** Complete - covers Cells 0-3 (setup and initial data load only)
**Last Updated:** 2025-11-12
