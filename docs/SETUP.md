# Repository Setup Guide

**Purpose:** Instructions to set up environment for Everside investment data processing scripts
**Last Updated:** 2025-11-12

---

## Table of Contents
1. [System Requirements](#system-requirements)
2. [Python Setup](#python-setup)
3. [R Setup](#r-setup)
4. [Directory Structure](#directory-structure)
5. [Required Input Files](#required-input-files)
6. [Verification Steps](#verification-steps)

---

## System Requirements

### Operating System
- **Linux/macOS:** Preferred (paths use ~ for home directory)
- **Windows:** Compatible but may require path adjustments

### Minimum Versions
- **Python:** 3.7 or higher
- **R:** 3.6.0 or higher
- **Excel:** Microsoft Excel or compatible (for viewing input/output files)

### Disk Space
- **Minimum:** 500 MB (for packages + data files)
- **Recommended:** 2 GB (includes space for historical data)

---

## Python Setup

### Required Packages (3 core + 2 standard library)

#### Core Packages:
```bash
pip install pandas openpyxl numpy
```

#### Detailed Requirements:

**requirements.txt:**
```
pandas>=1.0.0
openpyxl>=3.0.0
numpy>=1.18.0
```

**Standard Library (No Installation Required):**
- `datetime` - Date/time operations
- `warnings` - Warning control

---

### Installation Instructions

#### Option 1: Install with pip (Recommended)
```bash
# Create virtual environment (recommended)
python3 -m venv everside-env
source everside-env/bin/activate  # On Windows: everside-env\Scripts\activate

# Install required packages
pip install pandas openpyxl numpy

# Verify installation
python -c "import pandas, openpyxl, numpy; print('Python packages OK')"
```

#### Option 2: Install with requirements.txt
```bash
# Create requirements.txt file with content above, then:
pip install -r requirements.txt
```

---

### Python Package Details

| Package | Version | Purpose |
|---------|---------|---------|
| pandas | ≥1.0.0 | DataFrame operations, Excel I/O |
| openpyxl | ≥3.0.0 | Excel file read/write (xlsx format) |
| numpy | ≥1.18.0 | Numerical operations |

**Total Size:** ~50 MB (with dependencies)

---

## R Setup

### Required Packages (13 packages)

#### Installation Command:
```r
# Run in R console or RStudio
install.packages(c(
  "dplyr",
  "tibble",
  "openxlsx",
  "janitor",
  "data.table",
  "stats",
  "jrvFinance",
  "stringr",
  "tidyr",
  "ggplot2",
  "lubridate",
  "readxl",
  "purrr"
))
```

#### Alternative: One-by-One Installation
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

---

### R Package Details

| Package | Purpose | Used By |
|---------|---------|---------|
| dplyr | Data manipulation (filter, mutate, select) | All R scripts |
| tibble | Modern dataframes | All R scripts |
| openxlsx | Excel file read/write | All R scripts |
| janitor | Clean column names | All R scripts |
| data.table | High-performance data operations | IRR scripts, Data_Review |
| stats | Statistical functions (built-in, but explicit load) | IRR scripts |
| jrvFinance | Financial calculations (IRR, NPV) | IRR scripts |
| stringr | String manipulation | All R scripts |
| tidyr | Data reshaping (fill, pivot) | Capital viz, Data_Review |
| ggplot2 | Data visualization | Capital call charts |
| lubridate | Date operations | All R scripts |
| readxl | Excel file reading (.xls/.xlsx) | IRR scripts, Data_Review |
| purrr | Functional programming (map, reduce) | IRR scripts |

**Total Size:** ~100 MB (with dependencies)

---

### Verification

```r
# Verify all packages installed correctly
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

print("All R packages loaded successfully!")
```

---

## Directory Structure

### Required Directories

Create the following directory structure in your home directory:

```
~/
├── R Data/
│   ├── Quarterly Templates/
│   └── Monthly Financials/
└── R Output/
    └── IRR Data Output/
```

### Creation Commands

#### Linux/macOS:
```bash
mkdir -p ~/R\ Data/Quarterly\ Templates
mkdir -p ~/R\ Data/Monthly\ Financials
mkdir -p ~/R\ Output/IRR\ Data\ Output
```

#### Windows (PowerShell):
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\R Data\Quarterly Templates"
New-Item -ItemType Directory -Force -Path "$HOME\R Data\Monthly Financials"
New-Item -ItemType Directory -Force -Path "$HOME\R Output\IRR Data Output"
```

#### Windows (Command Prompt):
```cmd
mkdir "%USERPROFILE%\R Data\Quarterly Templates"
mkdir "%USERPROFILE%\R Data\Monthly Financials"
mkdir "%USERPROFILE%\R Output\IRR Data Output"
```

---

### Directory Purposes

| Directory | Purpose | Used By |
|-----------|---------|---------|
| ~/R Data/ | All input data files | All R scripts |
| ~/R Data/Quarterly Templates/ | Portfolio company quarterly submissions | Data_Review |
| ~/R Data/Monthly Financials/ | Radius Holdings monthly reports | radius_monthly_financials_data_pull |
| ~/R Output/ | General output files | Data_Review, IRR scripts |
| ~/R Output/IRR Data Output/ | IRR calculation CSVs | IRR_Data_Tab_Creation.R |

---

## Required Input Files

### Core Data Files (Must Exist in ~/R Data/)

#### 1. Everside Dataset.xlsx
**Location:** `~/R Data/Everside Dataset.xlsx`
**Used By:** Data_Review, high_level_metrics_review
**Sheets:**
- "Everside Dataset" - Portfolio company information
- "IRR Dataset" - IRR calculation data

**Purpose:** Central data warehouse for historical portfolio data

---

#### 2. Aggregated Capital Calls and Distributions.xlsx
**Location:** `~/R Data/Aggregated Capital Calls and Distributions.xlsx`
**Used By:** Capital_Call_Distribution_data.R, IRR_Front_Page_Calcs.R
**Columns Expected:**
- fund
- entity
- call_dist (type: "Call" or "Distribution")
- date
- amount
- days, years
- cumulative_called, cumulative_distribution

**Purpose:** Historical capital call and distribution activity

---

#### 3. GL Transaction Exports (Variable Names)
**Location:** `~/R Data/LTD Investment Transactions - {date}.xlsx`
**Used By:** IRR_Data_Tab_Creation.R
**Example Names:**
- "LTD Investment Transactions - 6.30.25 - Team Air update.xlsx"

**Columns Expected:**
- legal_entity
- gl_date
- position, deal_name
- trans_type
- dr_cr_amount
- batch_id

**Purpose:** General ledger transaction data for IRR calculations

---

#### 4. Fund_Alias_Lookup.xlsx
**Location:** `~/R Data/Fund_Alias_Lookup.xlsx`
**Used By:** IRR_Front_Page_Calcs.R
**Contains:** Fund-specific sheets with alias mappings

---

#### 5. IRR Model Files (Variable Names)
**Location:** `~/R Data/Everside Fund III IRR Model 3Q24.xlsx`
**Used By:** IRR_Front_Page_Calcs.R
**Sheet:** 'Data' (startRow=4)

---

### Quarterly Templates (in ~/R Data/Quarterly Templates/)

**Example:**
- `Boathouse III - Everside Quarterly Report Template (2Q24).xlsx`

**Used By:** Data_Review
**Purpose:** Quarterly portfolio company data submissions for validation

---

### Monthly Financials (in ~/R Data/Monthly Financials/)

**Example:**
- `Merion Summary Sept 2024 10-29-24_test.xlsx`

**Used By:** radius_monthly_financials_data_pull
**Sheets Expected:**
- "RH Balance Sheet"
- "RH Cash Flow"
- "9m July, Aug, Sept Q3" (or similar income statement sheet)

---

### Python Script Inputs (Current Directory)

Python scripts expect these files in the same directory as the script:

#### For Form 468 Processing:
- `2024.06.30 - St. Cloud Fund III Form 468.xlsx` (or similar)
- `Book1.xlsx` (output template)

#### For Deal Cloud Integration:
- `Investment Template_Farragut v4.xlsx` (or similar)
- `deal_cloud_dataset_template.xlsx`
- `deal_cloud_dataset_template_for_python.xlsx`

---

## Verification Steps

### Step 1: Verify Python Environment
```bash
python --version
python -c "import pandas, openpyxl, numpy; print('Python packages OK')"
```

**Expected Output:**
```
Python 3.7.x (or higher)
Python packages OK
```

---

### Step 2: Verify R Environment
```r
R.version.string
library(dplyr); library(openxlsx); library(jrvFinance); print("R packages OK")
```

**Expected Output:**
```
[1] "R version 3.6.x (or higher)"
[1] "R packages OK"
```

---

### Step 3: Verify Directory Structure
```bash
ls -la ~/R\ Data/
ls -la ~/R\ Output/
```

**Expected Output:**
```
# Should show directories: Quarterly Templates, Monthly Financials
# Should show directory: IRR Data Output
```

---

### Step 4: Verify Core Data Files
```bash
ls -lh ~/R\ Data/*.xlsx
```

**Expected Output:**
```
# Should show at least:
# - Everside Dataset.xlsx
# - Aggregated Capital Calls and Distributions.xlsx
```

---

### Step 5: Test Script Execution

#### Test Python Script:
```python
import pandas as pd
import openpyxl

# Test basic Excel read
try:
    df = pd.read_excel("test_file.xlsx")
    print("Excel read test: PASS")
except FileNotFoundError:
    print("Excel read test: SKIPPED (no test file)")
```

#### Test R Script:
```r
library(dplyr)
library(openxlsx)

# Test basic Excel read
tryCatch({
  data <- read.xlsx("test_file.xlsx")
  print("Excel read test: PASS")
}, error = function(e) {
  print("Excel read test: SKIPPED (no test file)")
})
```

---

## Common Setup Issues

### Issue 1: Package Installation Fails

**Symptom:** `pip install` or `install.packages()` fails

**Solutions:**
- **Python:** Upgrade pip: `pip install --upgrade pip`
- **R:** Install system dependencies (Linux): `sudo apt-get install libxml2-dev libcurl4-openssl-dev`
- **Both:** Check internet connection

---

### Issue 2: "Directory Not Found" Errors

**Symptom:** Script errors: "No such file or directory: ~/R Data/..."

**Solutions:**
- Create directories using commands in [Directory Structure](#directory-structure) section
- Verify paths with `ls ~/R\ Data/`
- On Windows, ensure spaces in paths are handled correctly

---

### Issue 3: "File Not Found" Errors

**Symptom:** Script errors: "File '*.xlsx' not found"

**Solutions:**
- Check file exists: `ls -lh ~/R\ Data/*.xlsx`
- Verify filename matches exactly (including spaces, capitalization)
- Place files in correct directory (~/R Data/ for R scripts, current directory for Python)

---

### Issue 4: Permission Errors

**Symptom:** "Permission denied" when creating directories or writing files

**Solutions:**
- Ensure write permissions: `chmod +w ~/R\ Data/`
- On Windows, run terminal as administrator
- Check disk space: `df -h ~`

---

### Issue 5: R Package Version Conflicts

**Symptom:** "namespace 'xxx' is already loaded" or version mismatch errors

**Solutions:**
- Restart R session: `.rs.restartR()` (RStudio) or restart R console
- Update all packages: `update.packages(ask = FALSE)`
- Remove and reinstall conflicting package:
  ```r
  remove.packages("package_name")
  install.packages("package_name")
  ```

---

## Quick Start Checklist

Use this checklist to verify setup is complete:

- [ ] Python 3.7+ installed
- [ ] R 3.6.0+ installed
- [ ] Python packages installed (pandas, openpyxl, numpy)
- [ ] R packages installed (13 packages)
- [ ] Directory structure created (~/R Data/, ~/R Output/)
- [ ] Everside Dataset.xlsx placed in ~/R Data/
- [ ] Aggregated Capital Calls and Distributions.xlsx placed in ~/R Data/
- [ ] Python verification test passed
- [ ] R verification test passed
- [ ] Can read/write to ~/R Output/

---

## Next Steps After Setup

Once setup is complete:

1. **Review Documentation:**
   - Read `docs/DEPENDENCIES.md` for script overview
   - Read `docs/DATA_FLOWS.md` for data flow understanding
   - Read individual script docs in `docs/scripts/`

2. **Run Test Scripts:**
   - Try running `Capital_Call_Distribution_data.R` (requires Aggregated Capital Calls file)
   - Try running `Data_Review` validation (requires Everside Dataset + Quarterly Template)

3. **Prepare Data Files:**
   - Obtain latest GL exports for IRR calculations
   - Obtain latest quarterly templates for validation
   - Obtain latest Form 468 files for SBA reporting

4. **Configure Parameters:**
   - Update file names in scripts to match your data files
   - Adjust date parameters to match your reporting period
   - Update fund names if processing different funds

---

## Support and Troubleshooting

### If Setup Fails:

1. **Check Logs:**
   - Python: Look for error messages in terminal output
   - R: Check warnings with `warnings()`

2. **Verify Prerequisites:**
   - Run through [Verification Steps](#verification-steps) again
   - Compare actual output with expected output

3. **Review Documentation:**
   - Check `docs/DEPENDENCIES.md` for detailed package info
   - Check `docs/reference/data_dictionary.md` for data file formats

4. **Common Solutions:**
   - Restart R/Python session
   - Clear package cache: `pip cache purge` (Python) or `.libPaths()` (R)
   - Reinstall problematic packages
   - Update all packages to latest versions

---

## Appendix: Complete Package List

### Python Packages (3)
```
pandas
openpyxl
numpy
```

### R Packages (13)
```
data.table
dplyr
ggplot2
janitor
jrvFinance
lubridate
openxlsx
purrr
readxl
stats
stringr
tibble
tidyr
```

---

## Appendix: Directory Structure Diagram

```
Home Directory (~)
│
├── R Data/                           # Input data location
│   ├── Everside Dataset.xlsx         # Core data warehouse
│   ├── Aggregated Capital Calls and Distributions.xlsx
│   ├── LTD Investment Transactions - {date}.xlsx
│   ├── Fund_Alias_Lookup.xlsx
│   ├── Everside Fund III IRR Model 3Q24.xlsx
│   │
│   ├── Quarterly Templates/          # Portfolio company submissions
│   │   ├── Boathouse III - Everside Quarterly Report Template (2Q24).xlsx
│   │   └── [Other quarterly templates]
│   │
│   └── Monthly Financials/           # Radius Holdings reports
│       ├── Merion Summary Sept 2024 10-29-24_test.xlsx
│       └── [Other monthly reports]
│
└── R Output/                         # Output location
    ├── {table}-{date}_updated_template.xlsx     # Data_Review outputs
    ├── {fund} - {table} Notes - {date}.xlsx     # Validation notes
    │
    └── IRR Data Output/              # IRR calculation outputs
        ├── Fund IV-2025-04-01-IRR_data_tab_output.csv
        └── [Other IRR outputs]
```

---

## Appendix: Minimum File Requirements

**To run at least one script successfully:**

**Minimum R Setup:**
```bash
# Create directories
mkdir -p ~/R\ Data/
mkdir -p ~/R\ Output/

# Place at least one of these files:
# - Aggregated Capital Calls and Distributions.xlsx (for capital viz)
# - Everside Dataset.xlsx (for high-level metrics)
```

**Minimum Python Setup:**
```bash
# Place in current directory:
# - Any Form 468 Excel file (for 468_Excel_Processor)
# - Book1.xlsx template
```

---

**End of Setup Guide**

**Total Setup Time:** ~30-45 minutes (including package installations)
**Disk Space Required:** ~200 MB (packages + minimal data)
**Prerequisites:** Python 3.7+, R 3.6.0+, Excel or compatible
