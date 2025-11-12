# Repository Dependencies and File Structure

**Repository:** ForkedFromSlater (Everside Investment Data Processing)
**Total Lines of Code:** ~58,034 lines
**Last Analyzed:** 2025-11-12

---

## Table of Contents
1. [File Inventory](#file-inventory)
2. [External Package Dependencies](#external-package-dependencies)
3. [Data File Dependencies](#data-file-dependencies)
4. [Script Dependency Graph](#script-dependency-graph)
5. [Entry Point Scripts](#entry-point-scripts)
6. [Utility/Module Scripts](#utilitymodule-scripts)
7. [Execution Prerequisites](#execution-prerequisites)

---

## File Inventory

### Python Scripts (.py)

#### 1. **Parse investment template to deal cloud file.py**
- **Lines:** 81
- **Language:** Python 3.x
- **Purpose:** Maps investment template data to Deal Cloud format
- **Package Imports:**
  - `pandas` (as pd)
  - `openpyxl` (load_workbook)
- **Type:** Standalone utility script

#### 2. **Parsing investment file into everside dataset template.py**
- **Lines:** 164 (notebook format)
- **Language:** Python 3.x (Jupyter notebook)
- **Purpose:** Same as above, notebook version
- **Package Imports:**
  - `pandas` (as pd)
  - `openpyxl`
- **Type:** Standalone utility script

---

### R Scripts (.R)

#### 3. **Capital_Call_Distribution_data.R**
- **Lines:** 815
- **Language:** R
- **Purpose:** Visualizes capital call and distribution rates across Everside funds
- **Package Imports:**
  - `dplyr`
  - `openxlsx`
  - `janitor`
  - `ggplot2`
  - `tidyr`
  - `lubridate`
- **Type:** Standalone analysis script

#### 4. **Capital_For_Filtered_Funds.R**
- **Lines:** Similar to Capital_Call_Distribution_data.R
- **Language:** R
- **Purpose:** Fund-specific capital analysis with filtering capability
- **Package Imports:**
  - `dplyr`
  - `openxlsx`
  - `janitor`
  - `ggplot2`
  - `tidyr`
- **Type:** Standalone analysis script

#### 5. **IRR_Data_Tab_Creation.R**
- **Lines:** 979
- **Language:** R
- **Purpose:** Core IRR calculation engine from GL transaction data
- **Package Imports:**
  - `dplyr`
  - `tibble`
  - `openxlsx`
  - `janitor`
  - `data.table`
  - `stats`
  - `jrvFinance`
  - `stringr`
  - `tidyr`
  - `ggplot2`
  - `lubridate`
  - `readxl`
  - `purrr`
- **Type:** Function library (sourced by other scripts)

#### 6. **IRR_Front_Page_Calcs.R**
- **Lines:** 199
- **Language:** R
- **Purpose:** Generates summary IRR metrics for investor report front pages
- **Package Imports:** Inherits from IRR_Data_Tab_Creation.R
- **External Script Dependencies:**
  - **SOURCES:** `~/R Data/IRR_Data_Tab_Creation.R` (line 53)
- **Type:** Analysis script (depends on IRR_Data_Tab_Creation.R)

#### 7. **Data_Review**
- **Lines:** 1,066
- **Language:** R
- **Purpose:** Automated data quality control and validation for quarterly reports
- **Package Imports:**
  - `dplyr`
  - `tibble`
  - `openxlsx`
  - `janitor`
  - `data.table`
  - `stats`
  - `jrvFinance`
  - `stringr`
  - `tidyr`
  - `ggplot2`
  - `lubridate`
  - `readxl`
  - `purrr`
- **Type:** Standalone validation script

#### 8. **high_level_metrics_review**
- **Lines:** 140
- **Language:** R
- **Purpose:** Portfolio-level metric aggregation and reporting
- **Package Imports:** Same as Data_Review (libraries must be pre-loaded)
- **Type:** Analysis script (requires library setup)

#### 9. **radius_monthly_financials_data_pull**
- **Lines:** 157
- **Language:** R
- **Purpose:** Processes monthly financial data from Radius Holdings
- **Package Imports:**
  - `openxlsx`
  - `janitor`
  - `data.table`
- **Type:** Standalone data extraction function

#### 10. **Risk_Rating_Function**
- **Lines:** ~29,084 (large file, not fully analyzed)
- **Language:** R
- **Purpose:** Portfolio company risk assessment algorithms
- **Type:** Function library

---

### Jupyter Notebooks (.ipynb)

#### 11. **468_Excel_Processor.ipynb**
- **Lines:** ~19 cells
- **Language:** Python 3.x
- **Purpose:** Processes SBA Form 468 regulatory data
- **Package Imports:**
  - `pandas` (as pd)
  - `datetime`
  - `numpy` (as np)
  - `warnings`
  - `openpyxl` (load_workbook)
- **Type:** Standalone processing script

#### 12. **Champlain_Capital468.ipynb**
- **Lines:** Similar to 468_Excel_Processor.ipynb
- **Language:** Python 3.x
- **Purpose:** Client-specific (Champlain Capital) Form 468 processing
- **Package Imports:**
  - `pandas` (as pd)
  - `openpyxl`
- **Type:** Standalone processing script

#### 13. **Historic_Everside_Deal_Cloud_parsing.ipynb**
- **Lines:** ~8 cells
- **Language:** Python 3.x
- **Purpose:** Historical data migration to Deal Cloud format
- **Package Imports:**
  - `pandas` (as pd)
  - `openpyxl`
- **Type:** One-time migration script (possibly deprecated)

#### 14. **Parse_investment_template_to_deal_cloud_file.ipynb**
- **Lines:** ~6 cells
- **Language:** Python 3.x
- **Purpose:** Investment template to Deal Cloud conversion (notebook version)
- **Package Imports:**
  - `pandas` (as pd)
  - `openpyxl` (load_workbook)
- **Type:** Standalone utility script

---

## External Package Dependencies

### Python Environment Requirements

```
pandas>=1.0.0
openpyxl>=3.0.0
numpy>=1.18.0
datetime (standard library)
warnings (standard library)
```

**Installation:**
```bash
pip install pandas openpyxl numpy
```

### R Environment Requirements

```r
install.packages("dplyr")
install.packages("tibble")
install.packages("openxlsx")
install.packages("janitor")
install.packages("data.table")
install.packages("stats")
install.packages("jrvFinance")
install.packages("stringr")
install.packages("tidyr")
install.packages("ggplot2")
install.packages("lubridate")
install.packages("readxl")
install.packages("purrr")
```

**Minimum R Version:** R >= 3.6.0 (inferred from package usage)

---

## Data File Dependencies

### Input Files (Must Exist Before Execution)

#### Python Scripts

| Script | Input File(s) | Location | Sheet/Format |
|--------|--------------|----------|--------------|
| Parse investment template to deal cloud file.py | Investment Template_Farragut v4.xlsx | Current directory | Sheet: 'eFile', skiprows=7 |
| Parse investment template to deal cloud file.py | deal_cloud_dataset_template.xlsx | Current directory | Active worksheet |
| Parse_investment_template_to_deal_cloud_file.ipynb | Investment Template - Oxer.xlsx | Current directory | Sheet: 'eFile', skiprows=7 |
| Parse_investment_template_to_deal_cloud_file.ipynb | deal_cloud_dataset_template_for_python.xlsx | Current directory | Active worksheet |
| 468_Excel_Processor.ipynb | 2024.06.30 - St. Cloud Fund III Form 468.xlsx | Current directory | Sheets: 'S1Inv', 'S4Del', 'S12pc' |
| 468_Excel_Processor.ipynb | Book1.xlsx (template) | Current directory | Active worksheet |

#### R Scripts

| Script | Input File(s) | Location | Sheet/Format |
|--------|--------------|----------|--------------|
| Capital_Call_Distribution_data.R | Aggregated Capital Calls and Distributions.xlsx | ~/R Data/ | Default sheet |
| Capital_For_Filtered_Funds.R | Aggregated Capital Calls and Distributions.xlsx | ~/R Data/ | Default sheet |
| IRR_Data_Tab_Creation.R | LTD Investment Transactions - {date}.xlsx | ~/R Data/ | GL transaction export |
| IRR_Front_Page_Calcs.R | Everside Fund III IRR Model 3Q24.xlsx | ~/R Data/ | Sheet: 'Data', startRow=4 |
| IRR_Front_Page_Calcs.R | Aggregated Capital Calls and Distributions.xlsx | ~/R Data/ | Default sheet |
| IRR_Front_Page_Calcs.R | Fund_Alias_Lookup.xlsx | ~/R Data/ | Fund-specific sheets |
| Data_Review | Everside Dataset.xlsx | ~/R Data/ | Sheet: 'Everside Dataset' |
| Data_Review | {quarterly_template}.xlsx | ~/R Data/Quarterly Templates/ | Parameterized sheet name |
| high_level_metrics_review | Everside Dataset.xlsx | ~/R Data/ | Sheets: 'Everside Dataset', 'IRR Dataset' |
| radius_monthly_financials_data_pull | {file_name}.xlsx | ~/R Data/Monthly Financials/ | Multiple sheets (parameterized) |

### Output Files (Generated by Scripts)

| Script | Output File(s) | Location | Format |
|--------|---------------|----------|--------|
| Parse investment template to deal cloud file.py | output_file_farragut_v4-v7.xlsx | Current directory | Excel workbook |
| Parse_investment_template_to_deal_cloud_file.ipynb | {output_file_path}.xlsx | Current directory | Excel workbook |
| 468_Excel_Processor.ipynb | v1.xlsx | Current directory | Intermediate Excel |
| 468_Excel_Processor.ipynb | v3x.xlsx | Current directory | Final processed Excel |
| Champlain_Capital468.ipynb | {updated_file_path}.xlsx | Current directory | Excel workbook |
| Champlain_Capital468.ipynb | {csv_file_path}.csv | Current directory | CSV export |
| Historic_Everside_Deal_Cloud_parsing.ipynb | {output_file_path}.xlsx | Current directory | Excel workbook |
| IRR_Data_Tab_Creation.R | {fund}-{start_date}-IRR_data_tab_output.csv | ~/R Output/IRR Data Output/ | CSV export |
| Data_Review | {table_name}-{as_of_date}_updated_template.xlsx | ~/R Output/ | Validated data |
| Data_Review | {fund} - {table} Notes - {as_of_date}.xlsx | ~/R Output/ | Validation notes |

---

## Script Dependency Graph

### Independent Scripts (No Dependencies on Other Scripts)

These scripts can be executed standalone:

```
Level 1: Standalone Scripts
├── Parse investment template to deal cloud file.py
├── Parse_investment_template_to_deal_cloud_file.ipynb
├── Parsing investment file into everside dataset template.py
├── 468_Excel_Processor.ipynb
├── Champlain_Capital468.ipynb
├── Historic_Everside_Deal_Cloud_parsing.ipynb
├── Capital_Call_Distribution_data.R
├── Capital_For_Filtered_Funds.R
├── IRR_Data_Tab_Creation.R
├── Data_Review
├── high_level_metrics_review
├── radius_monthly_financials_data_pull
└── Risk_Rating_Function
```

### Dependent Scripts (Require Other Scripts)

```
Level 2: Scripts with Dependencies
└── IRR_Front_Page_Calcs.R
    ├── REQUIRES: IRR_Data_Tab_Creation.R (sourced at line 53)
    └── INPUT: Calls create_irr_data_tab() function from IRR_Data_Tab_Creation.R
```

### Data Flow Dependencies

```
Sequential Processing Chains (by Business Process):

1. SBA Form 468 Processing:
   Input: Raw Form 468 Excel → 468_Excel_Processor.ipynb → Output: Standardized data

2. Deal Cloud Integration:
   Input: Investment Template → Parse investment template... → Output: Deal Cloud format

3. IRR Calculation & Reporting:
   GL Data → IRR_Data_Tab_Creation.R (creates IRR data) → IRR_Front_Page_Calcs.R → Summary metrics

4. Quarterly Reporting:
   Quarterly Template → Data_Review (validates) → Cleansed data + Validation notes

5. High-Level Metrics:
   Everside Dataset.xlsx → high_level_metrics_review → Portfolio summaries
```

---

## Entry Point Scripts

**Entry points** are scripts designed to be executed directly by users:

### For SBA Regulatory Reporting:
- **468_Excel_Processor.ipynb** - Process Form 468 filings
- **Champlain_Capital468.ipynb** - Client-specific Form 468 processing

### For Deal Management:
- **Parse investment template to deal cloud file.py** - Convert templates to Deal Cloud format
- **Parse_investment_template_to_deal_cloud_file.ipynb** - Notebook version of above
- **Historic_Everside_Deal_Cloud_parsing.ipynb** - Migrate historical data (one-time use)

### For Financial Analysis:
- **Capital_Call_Distribution_data.R** - Analyze capital deployment
- **IRR_Front_Page_Calcs.R** - Generate IRR summary reports
- **Data_Review** - Validate quarterly data submissions
- **high_level_metrics_review** - Portfolio-wide metrics
- **radius_monthly_financials_data_pull** - Process monthly financials

---

## Utility/Module Scripts

**Utility scripts** provide functions called by other scripts:

### R Function Libraries:
- **IRR_Data_Tab_Creation.R**
  - **Exported Function:** `create_irr_data_tab(fund, start_date, gl_file_input, excel_flag)`
  - **Used By:** IRR_Front_Page_Calcs.R (line 53: `source("~/R Data/IRR_Data_Tab_Creation.R")`)
  - **Purpose:** Provides core IRR calculation engine

- **Risk_Rating_Function**
  - **Purpose:** Risk assessment algorithms (likely sourced by other scripts)
  - **Status:** Not directly referenced in analyzed scripts (may be sourced elsewhere)

### Shared Data Resources:
- **Everside Dataset.xlsx** - Central data warehouse used by:
  - Data_Review
  - high_level_metrics_review

- **Aggregated Capital Calls and Distributions.xlsx** - Used by:
  - Capital_Call_Distribution_data.R
  - Capital_For_Filtered_Funds.R
  - IRR_Front_Page_Calcs.R

---

## Execution Prerequisites

### File System Requirements

#### Required Directory Structure:
```
/home/user/
├── R Data/
│   ├── Quarterly Templates/
│   ├── Monthly Financials/
│   └── IRR Data Output/ (created if doesn't exist)
└── R Output/
```

#### Current Working Directory Files:
Python scripts expect these in the execution directory:
- Investment template Excel files
- Form 468 Excel files
- Deal Cloud template files

### Execution Order (When Running Multiple Scripts)

#### For Quarterly Reporting Workflow:
```
1. Receive quarterly data from SBIC partners
2. Run: Data_Review (validates data quality)
3. Review validation notes
4. Correct data issues
5. Run: high_level_metrics_review (generate summaries)
```

#### For IRR Reporting Workflow:
```
1. Export GL transactions to Excel
2. Place file in ~/R Data/ with correct naming
3. Run: IRR_Data_Tab_Creation.R (or called by IRR_Front_Page_Calcs.R)
4. Run: IRR_Front_Page_Calcs.R (generates investor-ready summaries)
```

#### For SBA Compliance Workflow:
```
1. Receive Form 468 from fund administrator
2. Run: 468_Excel_Processor.ipynb
3. Review output files (v1.xlsx, v3x.xlsx)
4. Submit to SBA as required
```

### Script-Specific Prerequisites

#### IRR_Front_Page_Calcs.R
**Must run AFTER:**
- IRR_Data_Tab_Creation.R function is available (sourced)
- GL transaction data is exported and placed in ~/R Data/
- Fund alias lookup file exists

**Parameters Required:**
- `irr_data_file`: Name of existing IRR model Excel file
- `fund_filter`: Fund name (e.g., "Fund III")
- `date_filter`: As-of date (YYYY-MM-DD format)

#### Data_Review
**Must run AFTER:**
- Quarterly template received from SBIC
- Everside Dataset.xlsx is current
- Previous quarter data exists in Everside Dataset.xlsx

**Parameters Required:**
- `date_input`: Current quarter end date
- `fund_input`: Fund name
- `investment_vertical_input`: Investment type
- `table_input`: SBIC table name
- `quarterly_template_input`: Template file name
- `quarterly_template_Sheet`: Sheet name in template

#### 468_Excel_Processor.ipynb
**Must run AFTER:**
- Form 468 Excel file received
- Template files (Book1.xlsx) are in current directory

**Hard-coded Dependencies:**
- File: `2024.06.30 - St. Cloud Fund III Form 468.xlsx`
- Template: `Book1.xlsx`

---

## Orphaned/Experimental Files

### Potentially Deprecated:
- **Historic_Everside_Deal_Cloud_parsing.ipynb**
  - **Reason:** Appears to be one-time historical data migration
  - **Status:** May no longer be needed if migration is complete
  - **Action:** Verify with team before deletion

### Duplicate Functionality:
- **Parse investment template to deal cloud file.py** vs. **Parse_investment_template_to_deal_cloud_file.ipynb**
  - Same purpose, different format (.py vs .ipynb)
  - Notebook version may be for interactive exploration
  - Both maintained for user preference

- **Parsing investment file into everside dataset template.py**
  - Similar to Parse scripts but different output template
  - May serve different business process

---

## Known Issues & Gaps

### Hard-Coded File Paths:
Many scripts contain hard-coded file names that must be manually updated:
- **468_Excel_Processor.ipynb**: Hard-coded to specific date (2024.06.30)
- **IRR_Front_Page_Calcs.R**: Hard-coded IRR model file name
- **Data_Review**: Requires manual parameter configuration

**Recommendation:** Parameterize file names or create configuration files

### Missing Version Information:
- No Python version specified (assumed 3.x)
- No R version specified (assumed >= 3.6)
- Package versions not pinned (risk of compatibility issues)

**Recommendation:** Create requirements.txt (Python) and renv.lock (R)

### Path Assumptions:
- R scripts assume `~/R Data/` and `~/R Output/` exist
- No validation that directories are present
- No error handling for missing files

**Recommendation:** Add directory creation and file existence checks

### Documentation Gaps:
- No README explaining execution order
- No parameter documentation for R functions
- No data dictionary for Excel file formats

**Recommendation:** Create user guides for each major workflow

---

## Summary Statistics

| Category | Count |
|----------|-------|
| Total Files | 14 |
| Python Scripts | 2 |
| Jupyter Notebooks | 5 |
| R Scripts | 7 |
| Entry Point Scripts | 11 |
| Utility/Module Scripts | 2 |
| Dependent Scripts | 1 |
| Independent Scripts | 13 |
| External Python Packages | 3 core (pandas, openpyxl, numpy) |
| External R Packages | 12 unique |
| Input Data Files Referenced | ~15 unique files |
| Output Files Generated | ~10+ types |

---

**Document Version:** 1.0
**Last Updated:** 2025-11-12
**Maintainer:** Claude Code Agent
