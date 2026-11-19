# KNIME Data Cleaning Guide

Rebuilds `Data_Cleaner.ipynb` in KNIME (5.x). Same input file, same rules, same output: **478,956 rows × 17 columns**.

## Workflow overview

```
CSV Reader
 → Rule-based Row Filter            (disbursed loans, rate > 0)
 → String Manipulation ×2 + Math Formula + Rule Engine   (target: Default)
 → String Manipulation + Rule Engine                     (ApprovalFQ)
 → Rule Engine ×9 + Math Formula + String Manipulation   (features)
 → Column Filter → Column Resorter
 → CSV Writer (optional)
```

Tip: put the target nodes and the feature nodes in two components ("Target", "Features") and annotate each node. Graders read the workflow.

---

## 1. CSV Reader

File: `FOIA_7a_FY2010_FY2019_asof_260930.csv`

- **Advanced settings:** untick **Limit data rows scanned**. By default KNIME guesses column types from the first 10,000 rows only, and a column can fail to parse further down.
- **Transformation tab:** set these columns to **String**:

| Column | Why String |
|---|---|
| `LocationID` | keeps the leading zeros (`0012096`) |
| `NaicsCode` | we take its first 2 characters |
| `ApprovalDate`, `ChargeOffDate` | we turn them into numbers ourselves (step 3) |

Leave everything else as detected: `GrossApproval`, `SBAGuaranteedApproval` and `InitialInterestRate` as Number (double), `ApprovalFY` as Integer.

## 2. Rule-based Row Filter: keep disbursed loans

Mode: **Include TRUE matches**.

```
$LoanStatus$ IN ("P I F", "CHGOFF", "EXEMPT") AND $InitialInterestRate$ > 0 => TRUE
```

In the raw file the status is written `"P I F"`, with spaces. This drops CANCLD and COMMIT, which were never disbursed, and the 7 loans with a 0% rate.

## 3. Target: `Default`

Default = 1 if the loan was charged off **within 7 years of approval**.

The trick: write each date as a number `yyyymmdd`. Adding `70000` gives the same day 7 years later (20100315 → 20170315), so a single number comparison does the job. It also works for 29 February: no real date falls between 20190228 and 20190229.

| # | Node | Settings |
|---|---|---|
| 3a | String Manipulation | Append column `ApprovalInt`: `toInt(replace($ApprovalDate$, "-", ""))` |
| 3b | String Manipulation | Append column `ChargeOffInt`: `toInt(replace($ChargeOffDate$, "-", ""))` |
| 3c | Math Formula | Append column `DefaultLimit`: `$ApprovalInt$ + 70000` |
| 3d | Rule Engine | Append column `Default`, rules below |

In 3b, rows without a charge-off date get a missing value, which is what we want. Leave the option *"Insert missing values as null"* unticked.

```
$LoanStatus$ = "CHGOFF" AND $ChargeOffInt$ <= $DefaultLimit$ => 1
TRUE => 0
```

## 4. `ApprovalFQ` (fiscal quarter, Q1 = Oct–Dec)

| # | Node | Settings |
|---|---|---|
| 4a | String Manipulation | Append column `ApprovalMonth`: `toInt(substr($ApprovalDate$, 5, 2))` |
| 4b | Rule Engine | Append column `ApprovalFQ`, rules below |

```
$ApprovalMonth$ >= 10 => 1
$ApprovalMonth$ <= 3 => 2
$ApprovalMonth$ <= 6 => 3
TRUE => 4
```

## 5. Features

Rule Engine stops at the **first** rule that matches, so the order of the rules matters. Copy-paste the rules from this file rather than retyping them: some values contain exact spacing, such as the double space in `"Export Import Harmonization  (EXIM)"`.

### 5a. Rule Engine: `BorrState` (Replace column)
```
$BorrState$ IN ("GU", "VI", "MP", "FM", "AS", "MH") => "Territories"
TRUE => $BorrState$
```

### 5b. Rule Engine: `LenderType` (Append)
```
NOT MISSING $BankFDICNumber$ => "Bank"
NOT MISSING $BankNCUANumber$ => "Credit union"
TRUE => "Non-bank"
```
FDIC comes first: loans with both numbers count as Bank, as in the notebook.

### 5c. Math Formula: `GuaranteedPct` (Append)
```
$SBAGuaranteedApproval$ / $GrossApproval$
```

### 5d. Rule Engine: `ChannelGroup` (Append)
```
$ProcessingMethod$ IN ("SBA Express Program", "SBA Express Bridge Loan") => "Express"
$ProcessingMethod$ IN ("Preferred Lenders Program", "Certified Lenders Program") => "Delegated"
$ProcessingMethod$ = "7a General" => "Standard"
$ProcessingMethod$ IN ("Small Loan Advantage Initiative", "Rural Loan Initiative") => "Small-loan"
$ProcessingMethod$ IN ("Community Express", "Community Advantage Initiative", "Community Advantage International Trade", "Patriot Express Loans", "Gulf Opportunity Pilot Loan Program") => "Mission"
$ProcessingMethod$ IN ("Working Capital CAPLine", "Contract CAPLine", "Seasonal CAPLine", "Builders CAPLine", "Small Asset Base Line of Credit (CAPLine)", "Export Express", "International Trade Loans", "7a with EWCP", "Preferred Lenders with EWCP", "Certified Lenders Program with EWCP", "Dealer Floor Plan Loans", "Export Import Harmonization  (EXIM)") => "Specialty"
```
There's deliberately **no `TRUE` line**. A value that isn't listed, or a typo, becomes missing and shows up in the final check (step 7).

### 5e. String Manipulation: `NaicsSector` (Append)
```
substr($NaicsCode$, 0, 2)
```

### 5f. Rule Engine: `Sector` (Append)
```
$NaicsSector$ = "11" => "Agriculture"
$NaicsSector$ = "21" => "Mining"
$NaicsSector$ = "22" => "Utilities"
$NaicsSector$ = "23" => "Construction"
$NaicsSector$ IN ("31", "32", "33") => "Manufacturing"
$NaicsSector$ = "42" => "Wholesale"
$NaicsSector$ IN ("44", "45") => "Retail"
$NaicsSector$ IN ("48", "49") => "Transportation"
$NaicsSector$ = "51" => "Information"
$NaicsSector$ = "52" => "Finance & Insurance"
$NaicsSector$ = "53" => "Real Estate"
$NaicsSector$ = "54" => "Professional Services"
$NaicsSector$ = "55" => "Management of Companies"
$NaicsSector$ = "56" => "Admin & Support"
$NaicsSector$ = "61" => "Education"
$NaicsSector$ = "62" => "Health Care"
$NaicsSector$ = "71" => "Arts & Recreation"
$NaicsSector$ = "72" => "Accommodation & Food"
$NaicsSector$ = "92" => "Public Administration"
TRUE => "Other Services"
```
The `TRUE` line covers code 81 and the 2 loans with no NAICS code.

### 5g. Rule Engine: `IsFranchise` (Append)
```
MISSING $FranchiseCode$ => 0
TRUE => 1
```

### 5h. Rule Engine: `BusinessType` (Replace column)
```
MISSING $BusinessType$ => "CORPORATION"
TRUE => $BusinessType$
```

### 5i. Rule Engine: `BusinessAgeNew` (Append)
```
$BusinessAge$ = "Startup, Loan Funds will Open Business" => "Startup"
$BusinessAge$ IN ("New, Less than 1 Year old", "New Business or 2 years or less") => "New (<2y)"
$BusinessAge$ IN ("Less than 3 years old but at least 2", "Less than 4 years old but at least 3", "Less than 5 years old but at least 4", "Existing, 5 or more years", "Existing or more than 2 years old", "Change of Ownership") => "Existing (2y+)"
TRUE => "Unknown"
```

### 5j. Rule Engine: `IsRevolver` (Append)
```
$RevolverStatus$ = "Y" => 1
TRUE => 0
```

### 5k. Rule Engine: `HasCollateral` (Append)
```
$CollateralInd$ = "Y" => 1
TRUE => 0
```

## 6. Keep the final 17 columns

**Column Filter**: manual selection, include only these columns:

`LocationID, BorrState, LenderType, GrossApproval, GuaranteedPct, ApprovalFY, ApprovalFQ, ChannelGroup, InitialInterestRate, FixedorVariableInterestInd, Sector, IsFranchise, BusinessType, BusinessAgeNew, IsRevolver, HasCollateral, Default`

Everything else goes, including the helper columns `ApprovalInt`, `ChargeOffInt`, `DefaultLimit`, `ApprovalMonth` and `NaicsSector`.

**Column Resorter**: put the columns in the order above. Column Filter keeps the original order. This only matters for an exact match with the notebook; the models don't care about column order.

**CSV Writer** (optional): the rest of the KNIME workflow can continue directly from here. Write a file only if you need one, and give it a different name (e.g. `SBA_clean_knime.csv`) so it doesn't overwrite the notebook's file.

## 7. Check it matches the notebook

Attach a **Statistics** node and a **Value Counter** node (one per categorical column) to the last node. The numbers must match exactly:

| Check | Expected |
|---|---|
| Rows × columns | 478,956 × 17 |
| Missing values (any column) | 0 |
| `Default` = 1 | 27,594 (5.76%) |
| `ChannelGroup` | Express 230,237 · Delegated 155,826 · Standard 37,720 · Small-loan 30,395 · Mission 17,214 · Specialty 7,564 |
| `LenderType` | Bank 444,482 · Non-bank 18,734 · Credit union 15,740 |
| `BusinessAgeNew` | Existing (2y+) 316,592 · Startup 72,413 · New (<2y) 45,991 · Unknown 43,960 |
| `BusinessType` | CORPORATION 418,545 · INDIVIDUAL 51,380 · PARTNERSHIP 9,031 |
| `FixedorVariableInterestInd` | V 375,534 · F 103,422 |
| `ApprovalFQ` | Q1 118,259 · Q2 109,572 · Q3 126,975 · Q4 124,150 |
| `BorrState` | 53 distinct values, `Territories` = 474 |
| `LocationID` | 2,993 distinct values; first row = `0012096` |
| `IsFranchise` / `IsRevolver` / `HasCollateral` (count of 1s) | 42,555 / 143,975 / 345,952 |
| Mean `GrossApproval` | 376,761.48 |
| Mean `GuaranteedPct` | 0.649887 |
| Mean `InitialInterestRate` | 6.545226 |

`Sector` counts: Retail 65,659 · Accommodation & Food 62,651 · Construction 50,174 · Professional Services 47,321 · Health Care 44,706 · Other Services 42,854 · Manufacturing 37,961 · Transportation 25,285 · Wholesale 25,212 · Admin & Support 22,159 · Arts & Recreation 14,045 · Real Estate 10,446 · Finance & Insurance 8,161 · Agriculture 7,815 · Education 6,684 · Information 5,655 · Mining 1,373 · Utilities 402 · Management of Companies 312 · Public Administration 81.

If a number is off, the usual causes are:

| Symptom | Cause |
|---|---|
| Missing values in `ChannelGroup` | a typo in rule 5d, usually the EXIM double space |
| `LocationID` without leading zeros | not set to String in the CSV Reader |
| String Manipulation in step 3 or 4 fails | the date columns weren't set to String |
| Wrong `LenderType` counts | rules 5b in the wrong order |
