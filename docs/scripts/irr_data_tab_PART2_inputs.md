# IRR Data Tab Creation - Input Processing (Part 2)

**Script:** `IRR_Data_Tab_Creation.R`
**Focus:** Lines 1-200 (Data sources, loading, entity mapping, initial filtering)
**Scope:** Stops before aggregation loop begins

---

## Overview

This section documents how the script:
1. Sets up the R environment
2. Loads general ledger transaction data
3. Maps fund names to legal entity lists
4. Filters transactions for processing
5. Extracts batch IDs for transaction matching

**Processing Stage:** Input and preparation only - no IRR calculations yet.

---

## Lines 1-36: Environment Setup

### Function: install_and_library_packages()

**Purpose:** Ensure required R packages are installed and loaded.

**Code:**
```r
install_and_library_packages <- function(install_flag = FALSE) {
  if (install_flag != FALSE) {
    # Install 9 packages if flag is TRUE
    install.packages("dplyr")
    install.packages("openxlsx")
    install.packages("janitor")
    install.packages("data.table")
    install.packages("stats")
    install.packages("stringr")
    install.packages("readxl")
    install.packages("ggplot2")
    install.packages("jrvFinance")
  }

  # Load 13 packages (some duplicated in list)
  library(dplyr)
  library(tibble)
  library(openxlsx)
  library(janitor)
  library(data.table)
  library(stats)
  library(jrvFinance)
  library(stringr)
  library(tidyr)
  library(ggplot2)
  library(lubridate)
  library(readxl)
  library(purrr)
  # Note: data.table, tidyr, stringr loaded twice (lines 31-33 duplicates)

  options(scipen = 999)  # Disable scientific notation
}
```

### Package Purposes

| Package | Primary Use in Script |
|---------|----------------------|
| **dplyr** | Data manipulation (filter, mutate, select, arrange) |
| **openxlsx** | Read Excel files (read.xlsx) |
| **janitor** | Clean column names (clean_names) |
| **data.table** | High-performance data structures |
| **stats** | Statistical functions |
| **jrvFinance** | Financial calculations (IRR, XIRR) |
| **stringr** | String manipulation (str_extract, grepl) |
| **tidyr** | Data tidying (fill, pivot) |
| **ggplot2** | Plotting (unused in this script) |
| **lubridate** | Date manipulation (year, days, etc.) |
| **readxl** | Alternative Excel reader (unused in this script) |
| **purrr** | Functional programming (map, reduce) |
| **tibble** | Modern dataframes |

**Configuration:**
```r
options(scipen = 999)
```
**Effect:** Forces R to display numbers in full decimal format, not scientific notation (e.g., 1000000 instead of 1e6). Important for financial amounts.

### Usage
**First-time setup:**
```r
install_and_library_packages(install_flag = TRUE)
```

**Normal usage:**
```r
install_and_library_packages()  # Just loads libraries, doesn't install
```

---

## Lines 39-49: Input Data Loading

### Function Signature
```r
create_irr_data_tab <- function(
  fund = "Fund IV",
  start_date = "2025-04-01",
  gl_file_input = "LTD Investment Transactions - 6.30.25 - Team Air update",
  excel_flag = TRUE
)
```

### Input File

**File Path Construction:**
```r
gl_file <- read.xlsx(path.expand(paste0("~/R Data/", gl_file_input, ".xlsx")))
```

**Path Components:**
- `~/R Data/` - Home directory + R Data folder
- `gl_file_input` - Filename without extension (parameter)
- `.xlsx` - Excel file format

**Example:** Parameter `"LTD Investment Transactions - 6.30.25 - Team Air update"` → `/home/user/R Data/LTD Investment Transactions - 6.30.25 - Team Air update.xlsx`

**File Format:** Excel 2007+ (.xlsx)

**Expected Source:** General Ledger transaction export from accounting system (QuickBooks, NetSuite, or similar).

### Column Names (Inferred from Code Usage)

Based on filters and transformations in lines 41-200, the GL file contains these columns:

**Required Columns:**
- `legal_entity` - Legal entity name (string)
- `gl_date` - Transaction date (Excel date serial number)
- `position` - Investment position name (string)
- `deal_name` - Deal/fund name (string)
- `trans_type` - Transaction type (string)
- `dr_cr_amount` - Debit/credit amount (numeric, positive or negative)
- `batch_id` - Batch identifier for grouping related transactions (string or numeric)
- `comments_batch` - Batch-level comments (string)
- `comments_transaction` - Transaction-level comments (string)

**Possible Additional Columns:**
The script uses `clean_names()` which suggests original column names may have:
- Spaces (converted to underscores)
- Special characters (removed)
- Mixed case (converted to lowercase)

**Example Original Column Names:**
- "Legal Entity" → `legal_entity`
- "GL Date" → `gl_date`
- "Trans Type" → `trans_type`
- "DR/CR Amount" → `dr_cr_amount`

### Initial Transformations (Lines 42-48)

#### Step 1: Clean Column Names
```r
clean_names()
```
**Purpose:** Standardize column names to lowercase with underscores.

**Example:**
- "Legal Entity" → `legal_entity`
- "GL Date" → `gl_date`
- "Batch ID" → `batch_id`

**Why:** Consistent naming reduces errors and makes code more readable.

#### Step 2: Parse Dates
```r
mutate(
  gl_date = as.Date(gl_date, origin = "1899-12-30")
)
```

**Date Format:** Excel date serial number
- Excel stores dates as number of days since December 30, 1899
- Example: 45000 = March 10, 2023

**Conversion:**
- Input: Numeric (e.g., 45000)
- Output: R Date object (e.g., 2023-03-10)

**Origin Date:** "1899-12-30" is Excel's epoch (day 0)

**Result:** `gl_date` column converted from numeric to Date type.

#### Step 3: Filter Out Class B Transactions
```r
filter(
  !grepl("Class B", comments_batch)
)
```

**Logic:**
- `grepl("Class B", comments_batch)` - Returns TRUE if "Class B" appears anywhere in batch comments
- `!` - Negates, so keeps rows where "Class B" does NOT appear

**Purpose:** Exclude Class B transactions (likely a specific investment class or restricted accounts).

**Impact:** Removes entire rows where batch comments contain "Class B".

### Data Structure After Loading

**Variable Name:** `gl_file`
**Type:** Dataframe (tibble)
**Rows:** All GL transactions except those with "Class B" in batch comments
**Columns:** All columns from Excel file, names cleaned, dates parsed

**Example Row:**
```
legal_entity: "Everside Fund IV, LP"
gl_date: 2024-06-30 (Date object)
position: "Acme Corp - Preferred Stock"
trans_type: "Investment - Cost"
dr_cr_amount: -1000000 (negative for investment)
batch_id: "INV-2024-001"
comments_batch: "Q2 2024 investment"
```

---

## Lines 50-120: Entity Mapping (Fund-Specific)

### Purpose
Map user-provided fund name to list of legal entities that belong to that fund. GL transactions will be filtered to only these entities.

### Why Needed?
Funds have complex structures:
- Main fund LP
- International/offshore entities
- Blocker corporations (for tax efficiency)
- PF (parallel fund) entities for certain investors

**Example:** "Fund IV" transactions could come from 6 different legal entities, all part of Fund IV structure.

### Conditional Logic Structure

```r
if (fund == "Fund I Founders") {
  entities <- c(...)
  print(paste0("Filtering for the following entities: ", entities))
} else if (fund == "Fund II") {
  entities <- c(...)
  print(paste0("Filtering for the following entities: ", entities))
} else if ...
} else {
  print("ERROR: INCORRECT FUND PROVIDED")
}
```

### Entity Mappings by Fund

#### Fund I Founders (Line 50-57)
```r
entities <- c(
  "Everside Founders Fund, LP"
)
```
**Entity Count:** 1
**Structure:** Single LP entity (simplest structure)

#### Fund II (Line 58-69)
```r
entities <- c(
  "Everside Fund II F1, LP",
  "Everside Fund II, LP",
  "Everside International Fund II, LP",
  "Everside F1 Blocker, LP",
  "Everside Offshore Fund II Blocker, LP"
)
```
**Entity Count:** 5
**Structure:**
- 2 main fund entities (F1 and standard)
- 1 international entity
- 2 blocker entities

**Note:** F1 may be a separate vehicle for certain investor types.

#### Fund III (Line 70-82)
```r
entities <- c(
  "Everside Fund III PF, LP",
  "Everside Fund III, LP",
  "Everside International Fund III, LP",
  "Everside Offshore Fund III Blocker LP",
  "Everside PF Blocker, LP",
  "Everside Treeline Blocker III, LP"
)
```
**Entity Count:** 6
**Structure:**
- 2 main fund entities (PF parallel fund and standard)
- 1 international entity
- 3 blocker entities (offshore, PF, Treeline)

**Note:** "Treeline" may be investor-specific blocker.

#### Fund IV (Line 83-95)
```r
entities <- c(
  "Everside Fund IV PF, LP",
  "Everside Fund IV, LP",
  "Everside International Fund IV, LP",
  "Everside Offshore Fund IV Blocker, LP",
  "Everside PF IV Blocker, LP",
  "Everside Fund IV Blocker, LP"
)
```
**Entity Count:** 6
**Structure:**
- 2 main fund entities (PF parallel and standard)
- 1 international entity
- 3 blocker entities

#### Direct I (Line 96-106)
```r
entities <- c(
  "Everside Overflow Direct Fund, LP",
  "Everside Overflow Direct Fund, L.P.",  # Note: period after "L.P."
  "Everside Offshore Overflow Direct Fund, LP",
  "Everside Direct I Blocker, LLC"
)
```
**Entity Count:** 4
**Structure:**
- 2 overflow fund entities (duplicate names with different punctuation!)
- 1 offshore entity
- 1 blocker LLC (not LP)

**⚠️ Data Quality Issue:** Lines 99-100 have same entity name with different punctuation:
- "Everside Overflow Direct Fund, LP"
- "Everside Overflow Direct Fund, L.P." (with periods)

**Implication:** Accounting system may use inconsistent naming. Including both ensures all transactions captured.

#### Direct II (Line 107-115)
```r
entities <- c(
  "Everside SBIC I, LP",
  "Everside SBIC I, Offshore"
)
```
**Entity Count:** 2
**Structure:**
- 1 SBIC (Small Business Investment Company) main entity
- 1 offshore entity

**Note:** SBIC entities have SBA licensing and different reporting requirements.

#### Invalid Fund (Line 116-120)
```r
} else {
  print("ERROR: INCORRECT FUND PROVIDED")
}
```

**Issue:** Function prints error but **continues execution** with `entities` undefined from previous iteration or not set.

**Better Implementation:**
```r
} else {
  stop("ERROR: INCORRECT FUND PROVIDED. Valid values: Fund I Founders, Fund II, Fund III, Fund IV, Direct I, Direct II")
}
```

### Entity Mapping Summary Table

| Fund | Entities | PF | International | Blocker(s) | Special |
|------|----------|----|--------------|-----------:|---------|
| Fund I Founders | 1 | No | No | 0 | Simple structure |
| Fund II | 5 | F1 | Yes | 2 | F1 parallel |
| Fund III | 6 | Yes | Yes | 3 | Treeline blocker |
| Fund IV | 6 | Yes | Yes | 3 | Standard structure |
| Direct I | 4 | No | Offshore | 1 (LLC) | Punctuation variants |
| Direct II | 2 | No | Offshore | 0 | SBIC entities |
| **Total** | **28 unique** | - | - | - | - |

**Note:** Some entity names may overlap or be shared across funds (e.g., shared blockers). This table counts as listed in code.

### Output of This Section

**Variable:** `entities`
**Type:** Character vector
**Content:** List of legal entity names to filter GL transactions

**Example:**
```r
# For fund = "Fund IV"
entities
# [1] "Everside Fund IV PF, LP"
# [2] "Everside Fund IV, LP"
# [3] "Everside International Fund IV, LP"
# [4] "Everside Offshore Fund IV Blocker, LP"
# [5] "Everside PF IV Blocker, LP"
# [6] "Everside Fund IV Blocker, LP"
```

**Console Output:**
```
[1] "Filtering for the following entities: Everside Fund IV PF, LPEverside Fund IV, LP..."
```

---

## Lines 123-161: Initial Data Filtering

### Purpose
Filter loaded GL transactions to relevant subset for processing.

### Step 1: Main Data Filter (Lines 123-131)
```r
fund_iv_data_all <- gl_file %>%
  filter(
    legal_entity %in% entities,
    gl_date >= start_date,
    !grepl("RSM Payment|Transfer", comments_batch),
    !grepl("Transfer", comments_transaction),
    (position == "RF Ramsoft Investment, LP (Primary)" | !grepl("Primary|Secondary", position)),
    !grepl("Everside ", position)
  )
```

**Variable Name:** `fund_iv_data_all` (name from Fund IV but used for all funds)

#### Filter Conditions:

**1. Entity Filter**
```r
legal_entity %in% entities
```
**Logic:** Keep only transactions from entities defined in previous section.
**Example:** For Fund IV, keeps 6 entities; excludes all other Everside entities.

**2. Date Filter**
```r
gl_date >= start_date
```
**Logic:** Keep transactions on or after start_date parameter.
**Example:** `start_date = "2025-04-01"` → only Q2 2025 onward.

**Purpose:** Allows incremental processing (e.g., "only new transactions since last report").

**3. Batch Comment Filter**
```r
!grepl("RSM Payment|Transfer", comments_batch)
```
**Logic:** Exclude batches with "RSM Payment" OR "Transfer" in comments.

**RSM:** Likely RSM accounting firm (auditor/tax preparer). Payments to them are operational expenses, not investment transactions.

**Transfer:** Internal transfers between entities (not actual investments/proceeds).

**4. Transaction Comment Filter**
```r
!grepl("Transfer", comments_transaction)
```
**Logic:** Also exclude individual transactions flagged as "Transfer".

**Redundant?** Yes, partially. But catches transfers not flagged at batch level.

**5. Position Filter (Complex)**
```r
(position == "RF Ramsoft Investment, LP (Primary)" | !grepl("Primary|Secondary", position))
```

**Logic Breakdown:**
- `position == "RF Ramsoft Investment, LP (Primary)"` → Keep this specific position
- OR `!grepl("Primary|Secondary", position)` → Keep positions without "Primary" or "Secondary"

**Translation:**
- Include: RF Ramsoft Primary (special case)
- Include: All direct investments (no "Primary"/"Secondary" in name)
- Exclude: All other Primary/Secondary positions (fund-of-funds investments)

**Why Special Case?** RF Ramsoft may be a co-investment alongside a fund-of-funds, needs to be included despite "(Primary)" in name.

**6. Everside Filter**
```r
!grepl("Everside ", position)
```
**Logic:** Exclude positions with "Everside " (note trailing space) in name.

**Purpose:** Filter out investments in other Everside funds (internal cross-holdings). We only want external portfolio company investments.

**Example Excluded:** "Everside Fund III, LP" (if Fund IV invested in Fund III).

### Step 2: Extract Batch IDs (Lines 133-138)
```r
fund_iv_batch_ids <- fund_iv_data_all %>%
  filter(trans_type %in% c("Cash Out - Investment", "Cash Out - Main Account", "Cash Out - Consolidation")) %>%
  select(batch_id) %>%
  pull()
```

**Purpose:** Identify batches that contain investment (cash out) transactions.

**Logic:**
1. Filter for transaction types that represent money leaving the fund
2. Extract unique batch IDs
3. Convert to vector using `pull()`

**Transaction Types:**
- **"Cash Out - Investment"** - Investment in portfolio company
- **"Cash Out - Main Account"** - Payment from main fund account
- **"Cash Out - Consolidation"** - Consolidation entry (accounting)

**Result:** Vector of batch IDs that have at least one cash-out transaction.

**Example:**
```r
fund_iv_batch_ids
# [1] "INV-2024-001" "INV-2024-002" "INV-2024-005" ...
```

### Step 3: Filter to Matched Batches (Lines 140-142)
```r
fund_iv_data <- fund_iv_data_all %>%
  filter(
    batch_id %in% fund_iv_batch_ids,
    !trans_type %in% c("Cash Out - Investment", "Cash Out - Main Account", "Cash Out - Consolidation")
  )
```

**Purpose:** Keep only transactions in batches with investments, but exclude the cash-out entries themselves.

**Why?**
- Investment batches typically have:
  - 1 cash out entry (the investment itself)
  - Multiple credit entries (investment amounts by position, expenses, fees)
- We want the credit entries (detailed breakdown), not the cash-out summary

**Logic:**
1. Filter to batches identified in Step 2
2. But exclude the cash-out transaction types

**Result:** Dataframe with investment details (what was invested in what), excluding the summary cash-out line.

### Step 4: Debit/Credit Validation (Lines 145-158)

#### Debit Check (Lines 145-149)
```r
debit_check <- fund_iv_data_all %>%
  filter(
    batch_id %in% fund_iv_batch_ids,
    trans_type %in% c("Cash Out - Investment", "Cash Out - Main Account", "Cash Out - Consolidation")
  ) %>%
  summarize(sum(dr_cr_amount, na.rm = TRUE)) %>%
  pull()
```
**Purpose:** Sum all cash-out amounts in matched batches.
**Expected:** Negative number (money leaving fund).

#### Credit Check (Lines 151-155)
```r
credit_check <- fund_iv_data_all %>%
  filter(
    batch_id %in% fund_iv_batch_ids,
    !trans_type %in% c("Cash Out - Investment", "Cash Out - Main Account", "Cash Out - Consolidation")
  ) %>%
  summarize(sum(dr_cr_amount, na.rm = TRUE)) %>%
  pull()
```
**Purpose:** Sum all credit entries in matched batches.
**Expected:** Positive number (investment allocations).

#### Balance Check (Lines 157-158)
```r
check <- debit_check + credit_check
print(paste0("Difference between debits and credits: ", debit_check + credit_check))
```

**Accounting Rule:** Debits and credits in a batch should net to zero.

**Expected Output:**
```
[1] "Difference between debits and credits: 0"
```

**If Not Zero:**
- Data entry error in accounting system
- Missing transactions in batch
- Batch not fully exported

**⚠️ Issue:** Script prints warning but continues processing. Does not halt on imbalance.

**Better Implementation:**
```r
if (abs(check) > 0.01) {  # Allow for rounding
  warning("Debits and credits don't balance: ", check)
  # Or: stop() to halt execution
}
```

### Step 5: Extract Position List (Lines 160-161)
```r
fund_iv_data_accounts <- fund_iv_data %>%
  distinct(position)
```

**Purpose:** Get unique list of investment positions (portfolio companies) to process.

**Output:** Dataframe with one column (`position`) and one row per unique position.

**Example:**
```
position
---------
"Acme Corp - Preferred Stock"
"Beta Inc - Senior Secured Note"
"Gamma LLC - Common Units"
```

**Row Count:** Typically 20-50 portfolio companies per fund.

### Step 6: Initialize Aggregation (Lines 163-165)
```r
aggregated_cash_out_investments <- data.table()

i <- 1
```

**Purpose:** Create empty dataframe to store aggregated results and initialize loop counter.

**`data.table()`:** High-performance dataframe from data.table package.

**Loop Counter `i`:** Will iterate through position list in next section (lines 167+).

---

## Data Flow Summary (Lines 39-165)

```
Excel File (~/R Data/)
  ↓ (read.xlsx, clean_names)
GL Transactions (all)
  ↓ (parse dates, filter Class B)
gl_file
  ↓ (filter by fund entities + date + exclude transfers)
fund_iv_data_all
  ↓ (extract batch IDs with cash-out)
fund_iv_batch_ids (vector)
  ↓ (filter to those batches, exclude cash-out)
fund_iv_data
  ↓ (validate debits = credits)
Balance Check → Console Print
  ↓ (get unique positions)
fund_iv_data_accounts
  ↓
Ready for Aggregation Loop (Lines 167+)
```

---

## Key Variables After Input Processing

| Variable | Type | Content | Example |
|----------|------|---------|---------|
| `gl_file` | Dataframe | All GL transactions (cleaned, filtered for Class B) | ~10,000 rows |
| `entities` | Character vector | Legal entities for selected fund | 6 entities |
| `fund_iv_data_all` | Dataframe | GL transactions for fund, date filtered | ~1,000 rows |
| `fund_iv_batch_ids` | Vector | Batch IDs with investments | ~50 batches |
| `fund_iv_data` | Dataframe | Investment details (matched batches, no cash-out) | ~500 rows |
| `debit_check` | Numeric | Sum of cash-out amounts | -10,000,000 |
| `credit_check` | Numeric | Sum of credit entries | 10,000,000 |
| `check` | Numeric | Difference (should be 0) | 0 |
| `fund_iv_data_accounts` | Dataframe | Unique positions | 30 positions |
| `aggregated_cash_out_investments` | Data.table | Empty (to be filled) | 0 rows |
| `i` | Integer | Loop counter | 1 |

---

## Data Transformations Applied

### 1. Column Name Cleaning
**Function:** `clean_names()` (janitor package)
**Before:** "Legal Entity", "GL Date", "Trans Type"
**After:** `legal_entity`, `gl_date`, `trans_type`

### 2. Date Parsing
**Function:** `as.Date(gl_date, origin = "1899-12-30")`
**Before:** 45000 (numeric)
**After:** 2023-03-10 (Date object)

### 3. String Filtering (Multiple)
**Functions:** `grepl()`, `!grepl()`
**Patterns:**
- "Class B" - Exclude
- "RSM Payment|Transfer" - Exclude
- "Primary|Secondary" - Exclude (with exception)
- "Everside " - Exclude

### 4. Value Negation
**Code:** `dr_cr_amount * -1` (in subsequent lines)
**Purpose:** Flip sign for investment amounts (DR/CR convention)

### 5. NA Handling
**Function:** `sum(..., na.rm = TRUE)`
**Effect:** Ignore NA values in sums (treat as 0)

---

## Null/Missing Data Handling

### Implicit Handling
**NA values in filters:** Treated as FALSE (excluded from results)
- Example: `legal_entity %in% entities` where legal_entity is NA → row excluded

**NA in sums:** Removed with `na.rm = TRUE`
- Example: `sum(dr_cr_amount, na.rm = TRUE)` → treats NA as 0

### No Explicit Validation
**Missing:**
- No check for required columns existing
- No validation that date parsing succeeded
- No handling for empty entity list
- No check for zero transactions after filtering

**Risk:** Silent failures if data structure unexpected.

---

## Input Processing Checklist

At line 165, the script has:
- ✅ Loaded Excel file
- ✅ Cleaned column names
- ✅ Parsed dates
- ✅ Mapped fund to entities
- ✅ Filtered to relevant transactions
- ✅ Extracted investment batches
- ✅ Validated debit/credit balance
- ✅ Identified unique positions
- ✅ Initialized aggregation structure

**Ready For:** Position-level aggregation loop (starts line 167)

---

## Next Section Preview (Not Documented Here)

**Lines 167-280:** Cash OUT aggregation loop
- For each position in `fund_iv_data_accounts`
- Filter transactions by position
- Extract investment amounts and expenses
- Aggregate and append to `aggregated_cash_out_investments`

**Covered in:** Part 3 (Aggregation Logic) - if needed

---

**End of Part 2: Input Processing**

**Document Status:** Complete - covers lines 1-165
**Next Stage:** Aggregation loops (lines 167-465)
**Last Updated:** 2025-11-12
