# Data Review - Sample Validation Documentation

**Validation:** Industry Classification Check
**Lines:** 499-518 (within `compare_data()` function)
**Category:** Reference Data Validation
**Complexity:** ⭐ LOW - Simple whitelist comparison

---

## Purpose

Ensures that all portfolio companies have industry classifications that match the standardized taxonomy used for SBA SBIC reporting. Invalid industry values would cause filing rejections or require manual correction.

**Business Context:** SBIC reporting requires consistent industry categorization to track small business investment distribution across economic sectors.

---

## Function Context

This validation is embedded within the `compare_data()` function (lines 479-1027), which executes 18+ quality checks. It is **Validation Check #1** in the validation sequence.

### Parent Function Signature:
```r
compare_data <- function(
  quarterly_data_for_comp,    # Current quarter data (cleaned)
  Everside_Dataset,           # Historical reference data
  kfund,                      # Fund name (e.g., "Fund II")
  kinvestment_vertical,       # "Primary" or "Secondary"
  ktable,                     # Portfolio company table name
  kas_of_date,                # Current quarter end date
  kprev_as_of_date            # Previous quarter end date
)
```

### Input to This Validation:
- `comp_data` - Merged dataframe created at lines 491-497
- Columns used: `industry_current_quarter`, `industry_previous_quarter`, `company_name`

### Output from This Validation:
- `industry_provided_incorrectly` - Dataframe with companies having invalid industry codes
- Console output - Warning message with count and details

---

## Logic Breakdown

### Step 1: Filter for Invalid Industries (Lines 499-507)
```r
industry_provided_incorrectly <- comp_data %>%
  filter(
    !industry_current_quarter %in% c("Business Services", "Consumer", "Healthcare", "Technology", "Industrials", "Materials")
  ) %>%
  select(
    company_name,
    industry_current_quarter,
    industry_previous_quarter
  )
```

**What it does:**
1. **Filters** `comp_data` for rows where `industry_current_quarter` is NOT in the whitelist
2. **Negation operator:** `!` means "not in" - catches anything outside the 6 valid values
3. **Selects** only 3 columns for output: company name and both current/previous industry values

**Valid Industry Whitelist (6 values):**
- "Business Services"
- "Consumer"
- "Healthcare"
- "Technology"
- "Industrials"
- "Materials"

**What triggers a flag:**
- Any value NOT matching the whitelist exactly (case-sensitive)
- Examples that would flag:
  - "business services" (lowercase)
  - "Healthcare Services" (extra word)
  - "Tech" (abbreviation)
  - "" (blank)
  - NA (missing value)
  - "Financial Services" (not in list)

---

### Step 2: Conditional Output (Lines 509-518)
```r
if (nrow(industry_provided_incorrectly) > 0) {

  print(paste0("There are ", nrow(industry_provided_incorrectly), " companies with non-acceptable industries, please validate!"))
  print(industry_provided_incorrectly)

} else {

  print("There are NO issues due to Industry!")

}
```

**Logic Flow:**

```
IF any companies flagged (nrow > 0)
├── Print warning message with count
└── Print full table of flagged companies
ELSE
└── Print success message
```

**Console Output Examples:**

**Scenario A: Issues Found**
```
[1] "There are 3 companies with non-acceptable industries, please validate!"
     company_name industry_current_quarter industry_previous_quarter
1    Acme Corp              Tech                    Technology
2    Beta LLC               financial services      Healthcare
3    Gamma Inc              NA                      Consumer
```

**Scenario B: No Issues**
```
[1] "There are NO issues due to Industry!"
```

---

## Validation Rules

### Rule Definition
```
IF industry_current_quarter ∉ [Whitelist]
THEN flag company for manual review
```

### Whitelist Table

| Industry Code | Description | Typical Portfolio Companies |
|--------------|-------------|----------------------------|
| Business Services | B2B service providers | Staffing, consulting, marketing agencies |
| Consumer | B2C products/services | Retail, restaurants, consumer goods |
| Healthcare | Medical/health services | Medical practices, home health, dental |
| Technology | Software/IT services | SaaS, software development, IT consulting |
| Industrials | Manufacturing/distribution | Industrial equipment, logistics |
| Materials | Raw materials/processing | Chemical processing, materials suppliers |

**Threshold:** Zero tolerance - ANY deviation from whitelist is flagged.

**No Exceptions:** Unlike some validations (e.g., 20% thresholds for financial changes), this is a hard rule.

---

## Data Source

### Column: `industry_current_quarter`
**Origin:** From quarterly template submission
**Data Type:** String (character)
**Source Column Name (pre-cleaning):** Likely `Industry` in raw Excel file
**Populated By:** Portfolio company or deal team during quarterly update

**Expected Values:** One of 6 whitelist entries
**Common Errors:**
- Typos: "Helathcare", "Technolgy"
- Abbreviations: "Tech", "HC"
- Extra detail: "Healthcare - Medical Devices"
- Blank/missing values
- Old taxonomy values no longer in use

---

## Output Schema

### Variable: `industry_provided_incorrectly`
**Type:** Dataframe (tibble)
**Columns:** 3

| Column Name | Type | Description |
|------------|------|-------------|
| company_name | String | Portfolio company name (primary key) |
| industry_current_quarter | String | Invalid industry value from current quarter |
| industry_previous_quarter | String | Industry value from previous quarter (for comparison) |

**Row Count:**
- 0 rows = All companies pass validation
- N rows = N companies require correction

**Use Case:**
- Deal team reviews flagged companies
- Corrects industry values in quarterly template
- Re-runs validation until row count = 0

---

## Integration with Workflow

### Position in Validation Sequence
- **Check #1 of 18** in `compare_data()` function
- Runs immediately after quarter-over-quarter merge (lines 491-497)
- Independent of other checks (no dependencies)

### Related Validations
- **Check #2:** Geography validation (lines 520-539) - uses same whitelist pattern
- No downstream checks depend on this validation passing

### Blocking Behavior
- **Non-blocking:** Script continues even if companies flagged
- Other validations run regardless of this check's results
- Notes file is generated even with invalid industries present

---

## Example Scenarios

### Scenario 1: New Company with Missing Industry
**Input:**
```
company_name: "NewCo LLC"
industry_current_quarter: NA
industry_previous_quarter: NA
```

**Result:** ✅ FLAGGED
- NA is not in whitelist
- Both quarters show NA (company newly added)

**Action Required:** Deal team must assign valid industry

---

### Scenario 2: Industry Changed to Invalid Value
**Input:**
```
company_name: "TechStartup Inc"
industry_current_quarter: "Software"
industry_previous_quarter: "Technology"
```

**Result:** ✅ FLAGGED
- "Software" is not in whitelist (should be "Technology")
- Previous quarter was correct

**Action Required:** Correct "Software" → "Technology"

---

### Scenario 3: Correct Industry Value
**Input:**
```
company_name: "MedicalCo"
industry_current_quarter: "Healthcare"
industry_previous_quarter: "Healthcare"
```

**Result:** ✅ PASSES
- "Healthcare" is in whitelist
- No change between quarters

**Action Required:** None

---

### Scenario 4: Case Sensitivity Issue
**Input:**
```
company_name: "ServiceProvider LLC"
industry_current_quarter: "business services"
industry_previous_quarter: "Business Services"
```

**Result:** ✅ FLAGGED
- "business services" (lowercase) does not match "Business Services" (title case)
- R string matching is case-sensitive

**Action Required:** Fix capitalization to match whitelist exactly

---

## Maintenance Considerations

### Adding New Industry Categories

**Current Approach:** Hardcoded in function (line 501)
```r
!industry_current_quarter %in% c("Business Services", "Consumer", ...)
```

**To Add New Category (e.g., "Energy"):**
1. Edit line 501
2. Add "Energy" to vector
3. Test with sample data
4. Update documentation

**Limitation:** Requires code change for taxonomy updates

**Better Design:**
```r
# Load from config file
valid_industries <- read.csv("~/R Data/Config/industry_taxonomy.csv")$industry_name

# Use in validation
!industry_current_quarter %in% valid_industries
```

---

### Historical Changes

**Unknown:** When current 6-category taxonomy was established
**Risk:** Legacy data may use old category names
**Recommendation:** Document taxonomy version and effective date

---

## Error Handling

### Missing Column
**If `industry_current_quarter` column doesn't exist in `comp_data`:**
- Script will crash with error: "object 'industry_current_quarter' not found"
- No tryCatch wrapper to handle gracefully

**Recommendation:** Add column existence check:
```r
if (!"industry_current_quarter" %in% colnames(comp_data)) {
  stop("Required column 'industry_current_quarter' not found in data")
}
```

---

### Entirely NULL Column
**If ALL companies have NA for industry:**
- All rows would be flagged
- Console output would show large table
- No special handling for "all missing" scenario

**Potential Enhancement:** Detect if >50% missing and warn about systemic data issue

---

## Performance

**Computational Complexity:** O(n) where n = number of portfolio companies
**Typical Dataset Size:** 20-100 companies per fund/table
**Execution Time:** < 1 second

**Bottlenecks:** None - simple filter operation

---

## Comparison with Geography Validation

This validation uses the **exact same pattern** as Geography validation (Check #2, lines 520-539):

| Aspect | Industry | Geography |
|--------|----------|-----------|
| Lines | 499-518 | 520-539 |
| Whitelist Size | 6 values | 5 values |
| Column Name | `industry_current_quarter` | `geography_current_quarter` |
| Message | "non-acceptable industries" | "non-acceptable geography values" |
| Logic | Identical | Identical |

**Implication:** Documentation for Geography validation would be nearly identical - just swap column names and whitelist values.

---

## Testing Recommendations

### Test Cases

| Test Case | Input | Expected Result |
|-----------|-------|-----------------|
| Valid value | "Healthcare" | PASS (no flag) |
| Invalid value | "Medical" | FLAGGED |
| Blank value | "" | FLAGGED |
| NULL value | NA | FLAGGED |
| Case mismatch | "healthcare" | FLAGGED |
| Extra spaces | "Healthcare " | FLAGGED (trailing space) |
| All companies valid | 10 companies, all valid | "NO issues" message |
| Mixed valid/invalid | 8 valid, 2 invalid | "There are 2 companies..." |

---

## Business Impact

### Why This Validation Matters

1. **Regulatory Compliance:** SBA requires standardized industry reporting
2. **Portfolio Analytics:** Consistent classification enables industry-level performance analysis
3. **Risk Management:** Track industry concentration risk
4. **Investor Reporting:** LP reports show portfolio diversification by industry

### Cost of Invalid Industries

- **Filing Rejection:** SBA may reject quarterly report
- **Manual Correction Time:** 5-10 minutes per company to research and fix
- **Delayed Reporting:** Corrections delay report submission
- **Audit Issues:** Inconsistent classification complicates audits

---

## Documentation Template

This validation follows a **simple whitelist pattern** used for categorical field validation. The pattern is:

```r
# PATTERN: Categorical Field Validation
flagged_records <- data %>%
  filter(!column_name %in% c("Valid1", "Valid2", "Valid3")) %>%
  select(identifier, column_name, column_name_previous)

if (nrow(flagged_records) > 0) {
  print(paste0("There are ", nrow(flagged_records), " records with invalid values!"))
  print(flagged_records)
} else {
  print("There are NO issues!")
}
```

**Other validations using this pattern:**
- Geography validation (lines 520-539)

**Other validations using different patterns:**
- Percentage change detection (8 validations)
- Status change detection (2 validations)
- Completeness checks (2 validations)

---

## Summary

**Validation Type:** Whitelist Comparison
**Complexity:** Simple (1 filter operation)
**Execution:** Always runs (no conditions)
**Output:** Console print + dataframe
**Blocking:** No (non-fatal)

**Key Takeaway:** This is the simplest validation in the script - a straightforward categorical check. Other validations follow similar patterns but with more complex logic (thresholds, percentage calculations, multi-condition filters).

---

**End of Sample Validation Documentation**

**Created:** 2025-11-12
**Purpose:** Template for documenting remaining 17 validations
**Time to complete:** ~12 minutes
**Pattern identified:** Categorical whitelist validation
