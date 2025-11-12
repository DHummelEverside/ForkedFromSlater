# IRR Data Tab Creation - Structure Overview

**Script:** `IRR_Data_Tab_Creation.R`
**Total Lines:** 979 lines
**Purpose:** Process GL transactions into standardized IRR data format for fund performance reporting
**Type:** R data transformation script with complex entity-specific logic

---

## Quick Summary

**What it does:** Transforms general ledger (GL) transaction exports into clean IRR calculation format, separating cash flows by investment position and categorizing investment types.

**Primary use:** Input for IRR front page calculations and performance reporting.

**Complexity level:** ⚠️ **HIGH** - Contains fund-specific entity mappings, batch ID matching, and dual processing paths (Direct funds vs. Partnership funds).

---

## File Structure Map

### Lines 1-38: Package Management
```
Lines 1:      Blank
Lines 2-36:   install_and_library_packages() function
              - Conditional package installation
              - 13 package imports (dplyr, openxlsx, jrvFinance, etc.)
Lines 37-38:  Blank separator
```

### Lines 39-979: Main Function - create_irr_data_tab()
**Function signature:**
```r
create_irr_data_tab(
  fund = "Fund IV",
  start_date = "2025-04-01",
  gl_file_input = "LTD Investment Transactions - 6.30.25 - Team Air update",
  excel_flag = TRUE
)
```

#### Section Breakdown:

**Lines 39-49: GL File Loading**
- Load Excel file from ~/R Data/
- Clean column names
- Parse GL dates
- Filter out Class B transactions

**Lines 50-120: Entity Mapping (Fund-Specific)**
- **6 conditional blocks** for different funds:
  - Line 50: Fund I Founders → 4 entities
  - Line 58: Fund II → 8 entities
  - Line 70: Fund III → 8 entities
  - Line 83: Fund IV → 9 entities
  - Line 96: Direct I → 3 entities
  - Line 107: Direct II → 2 entities
- Maps fund name to legal entity list
- **CRITICAL:** Hardcoded entity names per fund

**Lines 121-499: DIRECT FUNDS PROCESSING BLOCK**
_Marker: `# DIRECTS #` (line 121) to `# END DIRECTS #` (line 499)_

Structure:
- **Lines 123-149:** Filter GL data for fund entities, extract batch IDs
- **Lines 150-175:** Direct investment cash out processing
  - Create position list
  - Initialize empty aggregation dataframe
- **Lines 176-280:** Cash OUT loop (for each investment position)
  - Filter by position and transaction types
  - Extract investment amounts and expenses
  - Aggregate by position
- **Lines 281-465:** Cash IN loop (for each investment position)
  - Filter return of capital, interest, dividends, realized gains
  - Aggregate proceeds by type
- **Lines 466-498:** Combine cash out/in, create output schema
  - Classify investment type (Debt/Equity/Partnership) using regex
  - Create 18-column output format
  - Sort by position, date, investment type

**Lines 500-941: PARTNERSHIP FUNDS PROCESSING BLOCK**
_Marker: Comment at line 500 indicates start of alternate path_

Structure:
- **Lines 500-650:** Similar to Direct funds but uses `deal_name` field
- **Lines 651-806:** Cash OUT loop for partnerships
- **Lines 807-906:** Cash IN loop for partnerships
- **Lines 907-940:** Combine and format output
  - Uses `deal_name` instead of `position`
  - Same 18-column output schema
  - Investment type classification

**Lines 941-978: Final Assembly and Output**
- **Line 943:** Combine Direct and Partnership outputs (`rbind`)
- **Lines 946-960:** Recalculate realized_proceeds (handle NAs)
- **Lines 962-972:** Conditional Excel write (if `excel_flag == TRUE`)
  - Outputs CSV to ~/R Output/IRR Data Output/
  - Filename: `{fund}-{start_date}-IRR_data_tab_output.csv`
- **Lines 974-978:** Return dataframe

---

## Functions Defined

### Function 1: install_and_library_packages()
**Line:** 2-36
**Parameters:**
- `install_flag` - Boolean, default FALSE
**Purpose:** Optionally install and load required R packages
**Usage:** Call once at script start if packages missing
**Packages:** dplyr, openxlsx, janitor, data.table, stats, stringr, readxl, ggplot2, jrvFinance, tibble, tidyr, lubridate, purrr

### Function 2: create_irr_data_tab()
**Line:** 39-979 (941 lines!)
**Parameters:**
- `fund` - Fund name (string, e.g., "Fund IV")
- `start_date` - Cutoff date for transactions (string, "YYYY-MM-DD")
- `gl_file_input` - GL file name without .xlsx extension
- `excel_flag` - Boolean, whether to write CSV output
**Returns:** Dataframe with 18 columns (IRR data format)
**Purpose:** Main processing function - converts GL to IRR format

---

## Key Processing Patterns

### 1. Dual Processing Paths
Script has TWO nearly identical processing blocks:
- **Direct Funds** (lines 121-499): Uses `position` field
- **Partnership Funds** (lines 500-941): Uses `deal_name` field

**Why?** Different accounting structure for direct investments vs. fund-of-funds.

### 2. Batch ID Matching
**Lines 133-148:** Critical logic
```r
# Find all batch IDs that contain "Cash Out" transactions
fund_iv_batch_ids <- fund_iv_data_all %>%
  filter(trans_type %in% c("Cash Out - Investment", ...)) %>%
  select(batch_id) %>%
  pull()

# Then filter for transactions in those batches (excluding the Cash Out itself)
fund_iv_data <- fund_iv_data_all %>%
  filter(batch_id %in% fund_iv_batch_ids, ...)
```

**Purpose:** Match related transactions (investment + proceeds) by batch ID.

### 3. Position-Level Aggregation
Two loops (cash out, cash in) for each unique position/deal:
```r
for (i in 1:length(position_list)) {
  # Filter for current position
  # Aggregate amounts by transaction type
  # Append to master aggregation dataframe
}
```

**Result:** One row per position per transaction date.

### 4. Investment Type Classification
**Lines 474-478 and 916-920:**
```r
investment_type = case_when(
  grepl("Promissory Note|Debt|Note|Loan", position) ~ "Debt",
  grepl("Preferred|Common|Stock|Equity", position) ~ "Equity",
  grepl("(Primary)|(Secondary)", position) ~ "Partnership",
  TRUE ~ NA
)
```

**Method:** Regex pattern matching on position name.

---

## Output Schema (18 Columns)

| Column | Type | Source/Calculation |
|--------|------|-------------------|
| vintage_year | Numeric | year(gl_date) |
| transaction_date | Date | gl_date |
| position_name | String | position or deal_name |
| investment_type | Categorical | Regex classification (Debt/Equity/Partnership) |
| commitment | Numeric | NA_real_ (placeholder) |
| unfunded_commitment | Numeric | NA_real_ (placeholder) |
| gap | Numeric | NA_real_ (placeholder) |
| investment_amount | Numeric | Sum of "Investment - Cost" transactions |
| investment_expense | Numeric | Sum of "Interest Expense" transactions |
| proceeds_return_of_capital | Numeric | Sum of "Cash In - Investment" |
| proceeds_interest_income | Numeric | Sum of "Interest Income" |
| proceeds_dividend_income | Numeric | Sum of "Dividend Income" |
| proceeds_realized_gain | Numeric | Sum of "Realized Gain" |
| proceeds_mfee_rebate | Numeric | Sum of management fee rebates |
| proceeds_closing_fee | Numeric | Sum of closing fees |
| proceeds_due_to_manager | Numeric | NA_real_ (TODO: follow up with Hao) |
| realized_proceeds | Numeric | Sum of all proceeds columns |
| fund_name | String | legal_entity |

**Note:** 3 columns are always NA (commitment, unfunded_commitment, gap) - placeholders for future enhancement.

---

## Complexity Assessment

### 🔴 HIGH COMPLEXITY SECTIONS

#### Lines 50-120: Entity Mapping
**Why complex:**
- 6 conditional blocks with hardcoded entity lists
- 34 unique entity names across 6 funds
- Must be manually updated if entities change
- No validation that entity names in GL match expected names

**Maintenance burden:** Adding new fund or entity requires code changes.

#### Lines 133-148: Batch ID Matching Logic
**Why complex:**
- Multi-step filtering to link related transactions
- Relies on batch_id integrity in source data
- If batch IDs malformed → silently excludes transactions
- Debit check (line 145) validates total but doesn't halt on error

**Risk:** Wrong batch ID matching → incomplete cash flows → incorrect IRR.

#### Lines 176-465 and 651-906: Aggregation Loops
**Why complex:**
- Nested filtering within loops (filter GL for each position)
- Multiple transaction type mappings
- Separate loops for cash out vs. cash in
- 280+ lines of repetitive position-level logic

**Performance:** O(n × m) where n = positions, m = GL rows. Could be slow for large datasets.

### 🟡 MODERATE COMPLEXITY SECTIONS

#### Lines 470-498 and 912-940: Output Schema Transformation
**Why moderate:**
- Regex-based investment type classification
- Multiple transmute operations
- NA handling for proceeds calculation
- Investment type priority logic (Debt > Equity > Partnership)

**Issue:** Regex patterns may not cover all position name variations.

#### Lines 962-972: Conditional CSV Output
**Why moderate:**
- File path construction with parameters
- Directory must exist or write fails (no validation)
- Filename includes fund name and date (must be valid filesystem characters)

### 🟢 STRAIGHTFORWARD SECTIONS

#### Lines 2-36: Package Management
**Why straightforward:**
- Standard package loading
- Clear conditional install logic
- No dependencies on external state

#### Lines 39-49: GL File Loading
**Why straightforward:**
- Standard read.xlsx workflow
- Simple date parsing
- One filter condition

#### Lines 943-960: Final DataFrame Assembly
**Why straightforward:**
- Simple rbind of two dataframes
- Straightforward NA handling with ifelse
- No complex logic

---

## Data Dependencies

### Input File
**Path:** `~/R Data/{gl_file_input}.xlsx`
**Expected columns:**
- legal_entity
- gl_date (Excel date serial)
- position (for Directs) or deal_name (for Funds)
- trans_type (categorical: "Investment - Cost", "Cash In - Investment", etc.)
- dr_cr_amount (numeric, positive/negative)
- batch_id (string or numeric)
- comments_batch (string, used to filter)
- comments_transaction (string, used to filter)

**Source:** Likely exported from accounting system (QuickBooks, NetSuite, or similar).

### Output File
**Path:** `~/R Output/IRR Data Output/{fund}-{start_date}-IRR_data_tab_output.csv`
**Format:** CSV with 18 columns (no headers written explicitly - uses dataframe column names)
**Used by:** IRR_Front_Page_Calcs.R (sources this file for front page metrics)

---

## Known Issues and TODOs

### 1. NA Placeholder Columns
**Lines 480-482, 922-924:**
```r
commitment = NA_real_,
unfunded_commitment = NA_real_,
gap = NA_real_,
```
**Issue:** Three columns always NA. Either remove or implement.

### 2. Missing Manager Proceeds
**Lines 491, 933:**
```r
proceeds_due_to_manager = NA_real_, # FOLLOW UP WITH HAO
```
**Issue:** Calculation not implemented. Affects realized_proceeds total.

### 3. Hardcoded Entity Lists
**Lines 50-120:** 34 entity names hardcoded.
**Better approach:** Store in configuration table/file.

### 4. No Fund Validation
**Line 116:** Prints error if fund not recognized but function continues.
**Risk:** Processes with empty entity list → no output → silent failure.

### 5. Repetitive Code
**Direct vs. Partnership blocks** are 90% identical.
**Opportunity:** Extract common logic into helper function.

### 6. No Error Handling
- File not found → crash
- Missing required columns → crash
- Invalid date formats → crash
- Directory doesn't exist → crash
- No try-catch blocks anywhere

---

## Execution Flow Diagram

```
START
  ↓
Load GL file (~/R Data/)
  ↓
Determine fund → Map to entity list (6 options)
  ↓
Filter GL for entities + date + exclude transfers
  ↓
Extract batch IDs (cash out transactions)
  ↓
┌─────────────────────────┬─────────────────────────┐
│  DIRECT FUNDS PATH      │  PARTNERSHIP FUNDS PATH │
│  (position field)       │  (deal_name field)      │
├─────────────────────────┼─────────────────────────┤
│  Get unique positions   │  Get unique deals       │
│  Loop: Cash OUT         │  Loop: Cash OUT         │
│  Loop: Cash IN          │  Loop: Cash IN          │
│  Combine + format       │  Combine + format       │
└─────────────────────────┴─────────────────────────┘
  ↓
Merge Direct + Partnership outputs
  ↓
Recalculate realized_proceeds (handle NAs)
  ↓
Classify investment_type (regex)
  ↓
If excel_flag: Write CSV
  ↓
Return dataframe
  ↓
END
```

---

## Usage Examples

```r
# Load packages (first time only)
install_and_library_packages(install_flag = TRUE)

# Generate IRR data for Fund IV
irr_data <- create_irr_data_tab(
  fund = "Fund IV",
  start_date = "2024-01-01",
  gl_file_input = "LTD Investment Transactions - 3.31.24",
  excel_flag = TRUE
)

# Generate without saving file
irr_data_memory_only <- create_irr_data_tab(
  fund = "Fund III",
  start_date = "2023-01-01",
  gl_file_input = "LTD Investment Transactions - 12.31.23",
  excel_flag = FALSE
)
```

---

## Maintenance Recommendations

### High Priority
1. **Extract entity mapping** to configuration file
2. **Add fund validation** - halt if fund not recognized
3. **Add error handling** - try-catch around file operations
4. **Implement or remove** NA placeholder columns

### Medium Priority
5. **Refactor dual paths** - extract common logic to helper function
6. **Add input validation** - check required GL columns exist
7. **Add logging** - track processing progress and errors
8. **Calculate proceeds_due_to_manager** or remove column

### Low Priority
9. **Performance optimization** - vectorize instead of loops
10. **Add progress indicators** - show loop progress for large datasets
11. **Create unit tests** - test entity mapping, batch matching, aggregation

---

## Related Scripts

**Downstream consumers:**
- `IRR_Front_Page_Calcs.R` - Reads CSV output, calculates summary metrics
- `high_level_metrics_review` - May use IRR data for portfolio analytics

**Data flow:**
```
Accounting System
  ↓ (export)
GL Transaction Excel
  ↓ (this script)
IRR Data Tab CSV
  ↓ (IRR_Front_Page_Calcs.R)
Front Page Metrics
  ↓
Investor Reports
```

---

## Quick Reference - Entity Counts by Fund

| Fund | Entity Count | Lines |
|------|--------------|-------|
| Fund I Founders | 4 entities | 50-56 |
| Fund II | 8 entities | 58-68 |
| Fund III | 8 entities | 70-81 |
| Fund IV | 9 entities | 83-94 |
| Direct I | 3 entities | 96-105 |
| Direct II | 2 entities | 107-115 |
| **Total** | **34 unique entities** | - |

---

## Documentation Status

✅ **Structure mapped** - All major sections identified
✅ **Functions documented** - 2 functions with signatures
✅ **Complexity assessed** - High/Medium/Low ratings assigned
✅ **Data flow diagrammed** - Input → Processing → Output
⏳ **Detailed line-by-line** - Deferred to Part 2 (if needed)

**Next steps for deeper documentation:**
- Part 2: Entity mapping details (lines 50-120)
- Part 3: Batch matching logic (lines 133-148)
- Part 4: Aggregation loop internals (lines 176-465)
- Part 5: Output schema and investment type classification

---

**End of Part 1: Structure Overview**

**Created:** 2025-11-12
**Purpose:** Roadmap for future detailed documentation
**Estimated time for full documentation:** 2-3 hours (if all 5 parts completed)

