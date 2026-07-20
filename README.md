# CMS Dialysis Facility Analysis

**Author:** Emmanuel Osei Kuffuor  
**Data Source:** CMS Medicare Dialysis Facility Reports (FY2024, Public Dataset)  
**Tools:** Python, pandas, matplotlib, seaborn, Google Colab  
**Last Updated:** 06/17/2026

---

## Project Overview

This project analyzes clinical quality metrics across 8,247 Medicare-certified
dialysis facilities in the United States using publicly available CMS Dialysis
Facility Report data. The goal is to identify patterns in patient outcomes,
facility performance, and regional disparities across all 18 ESRD Networks.

This analysis is informed by six years of experience working with DaVita
Kidney Care's enterprise clinical data warehouse, including ETL validation
against the IBM Unified Data Model for Healthcare (UDMH) and clinical feed
quality assurance across Informatica Power Center and IBM Netezza pipelines.

---

## Key Findings

**Finding 1: Septicemia Hospitalization Rates Vary by ESRD Network**

Average septicemia hospitalization rates across 18 ESRD Networks ranged from
10.99% (Network 6 - South Atlantic) to 13.36% (Network 2 - New York) in 2022.

- **Highest rates:** New York, Southern California, Texas
- **Lowest rates:** South Atlantic, Northwest, Mid-Atlantic/Caribbean

The 2.37 percentage point spread suggests relative national consistency in
dialysis care quality, likely reflecting CMS oversight effectiveness. However,
urban-dense networks trend higher — driven by patient complexity, vascular
access type, and social determinants of health rather than facility quality alone.

---

**Finding 2: Risk-Adjusted Mortality Varies Significantly by ESRD Network**

Standardized Mortality Ratios (SMR) across 18 ESRD Networks ranged from
0.90 (Network 4 - Mid-Atlantic) to 1.09 (Network 16 - Northwest) in 2022.

- **Below expectation (SMR < 1.0 — better than expected):** Mid-Atlantic,
  Illinois, New York, Upper Midwest
- **Above expectation (SMR > 1.0 — worse than expected):** Northwest, Texas,
  Deep South

**The New York Paradox:** Network 2 (New York) had the highest septicemia rate
in Finding 1 but one of the lowest SMRs (0.91). SMR accounts for patient
complexity — New York facilities serve higher-acuity populations, so CMS
expects more deaths. An SMR of 0.91 means they're delivering better outcomes
than predicted given how sick their patients are. Raw infection rates and
risk-adjusted mortality tell different stories; both are needed for complete
quality assessment.

---

**Finding 3: Large Chain Scale Advantage in Septicemia Control**

Septicemia rates among the five largest dialysis chains ranged from
12.04% (Fresenius Medical Care) to 13.12% (American Renal Associates) in 2022.
DaVita ranked second lowest at 12.19%.

- **Lowest rates:** Fresenius Medical Care (12.04%), DaVita (12.19%)
- **Highest rate:** American Renal Associates (13.12%)

The two largest chains — DaVita and Fresenius — perform best, suggesting
scale advantages in standardized infection-control protocols, dedicated
quality-improvement teams, real-time data monitoring, and staff training
infrastructure. Smaller chains may have fewer centralized resources, leading
to greater variation in infection-prevention practices across facilities.

**Important caveat:** Chain size alone should not be interpreted as the direct
cause. Patient complexity, catheter prevalence, staffing ratios, socioeconomic
conditions, and reporting practices may also influence observed rates.

---

## Clinical Context

Septicemia (bloodstream infection) was selected as the primary outcome measure
because it is the most facility-controllable hospitalization cause in dialysis
care. Unlike cardiac conditions (often pre-existing comorbidities), septicemia
is directly linked to vascular access management — specifically catheter care
and fistula maintenance protocols.

Higher septicemia rates in urban networks like New York are clinically plausible:
- Dense urban populations more commonly use central venous catheters (CVCs)
  rather than arteriovenous fistulas (AVFs) — CVCs carry significantly higher
  infection risk
- Urban patients face greater socioeconomic barriers: transportation challenges,
  housing instability, and missed treatments all increase infection risk over time
- Emergency dialysis initiation is more common in urban safety-net settings,
  leading to higher CVC prevalence at treatment start

**Implication for quality improvement:** Interventions targeting CVC reduction
and care continuity in high-rate networks would likely have greater impact than
facility-level quality programs alone.

---

## Data

**Source:** CMS Medicare Dialysis Facility Reports (FY2024)  
**URL:** https://catalog.data.gov/dataset/medicare-dialysis-facilities-79b67  
**Data Dictionary:** https://data.cms.gov/sites/default/files/2026-01/FY2026%20DFR%20PUF%20Data%20Dictonary.xlsx  
**License:** Public domain (U.S. Government Works)

**Dataset characteristics:**
- 11,679,242 rows × 14 columns (long format — one row per facility per measure per time period)
- 8,247 unique Medicare-certified dialysis facilities
- 2,632 unique quality measures across 54 categories
- ~54% fill rate (dataset is ~46% sparse — not all facilities report all measures)

---

## Data Engineering Notes

Several data engineering decisions were made during this analysis that reflect
real-world ETL practice:

**1. Encoding — Latin-1 vs UTF-8**  
The CMS dataset uses Latin-1 encoding (ISO-8859-1), common in older government
systems. Loading with default UTF-8 produces a `UnicodeDecodeError` at byte
`0x96` — a special dash character. Solution: `pd.read_csv(url, encoding='latin-1')`.
In production ETL, encoding mismatches cause data corruption or failed pipeline
loads — a common defect category in clinical data warehousing.

**2. Mixed Data Types — low_memory=False**  
Pandas chunk-by-chunk type inference produced `DtypeWarning` for columns 6 and 9.
Solution: `low_memory=False` forces full-file type inference before assignment.
In warehouse terms this mirrors a schema validation step — ensuring consistent
data types before transformation.

**3. Data Quality Defects Identified**  
Four source data defects were identified during exploratory analysis:
- `nan` — failed regex extraction on non-standard measure name formats
- `(Pediatric) Prevalent Waitlist` — reversed naming convention vs all other
  pediatric categories
- `SEDR`, `SRR`, `STR`, `STrR` — unexpanded acronyms with possible duplicates
  due to inconsistent capitalization (`STR` vs `STrR`)
- `Phosphoous (Adult)` — typo creating a duplicate of `Phosphorus (Adult)`

**4. Scope Decision — Measure Prefix Filtering**  
Three non-clinical measure prefixes were identified and excluded:

| Prefix | Description | Decision |
|--------|-------------|----------|
| `F (AFS):` | Annual Facility Survey — patient counts, transfer rates | Excluded |
| `F(QIES):` | Quality Improvement & Evaluation — compliance, citations | Excluded |
| `CWFacDir:` | Facility directory — hemodialysis station counts | Excluded |

Filtering to `F:` prefixed measures retained all 8,247 facilities while
reducing rows from 11,679,242 to 9,156,455. This mirrors a source filter
applied at ETL ingestion — consciously excluding out-of-scope records with
documented justification.

**5. Sparsity Check**  
Potential rows (8,247 facilities × 2,632 measures) = 21,706,104.  
Actual rows = 11,679,242 (~54% fill rate).  
Any aggregate analysis accounts for this sparsity — unfiltered averages
across sparse columns produce misleading results.

**6. Sample Size Validation**  
Ownership type comparison was initially planned (For-Profit vs Non-Profit)
but abandoned after sample size check revealed only 1 Non-Profit facility
reported the chosen measure. Analysis pivoted to ESRD Network grouping (n=18
networks, ~383 facilities per network average). This reflects a core principle:
always validate group sizes before aggregating and visualizing.

**7. SettingWithCopyWarning — View vs Copy**  
Resolved by adding `.copy()` when creating filtered dataframes. Without this,
pandas may silently fail to persist new columns added to a view — a subtle
bug that causes downstream data corruption in production pipelines.

**8. ESRD Network Mapping**  
Network numbers (1-18) were mapped to geographic region names for visualization
clarity. Mapping was verified against official CMS sources before inclusion:
- KCER Coalition ESRD Network directory: https://www.kcercoalition.com/en/esrd-networks/
- CMS ESRD Network Organizations: https://www.cms.gov/ESRDNetworkOrganizations

Three corrections were identified during verification (Network 8 corrected
from "Midwest" to "Deep South"; Network 10 corrected from "Great Lakes" to
"Illinois"; Network 13 corrected from "Southwest" to "West South Central"),
demonstrating the importance of source verification over assumed knowledge.

---

## Repository Structure

```
cms-dialysis-analysis/
│
├── cms_dialysis_analysis.ipynb   # Main analysis notebook
└── README.md                     # This file
```

---

## How to Run

1. Open `cms_dialysis_analysis.ipynb` in Google Colab or Jupyter
2. Run all cells — data loads directly from CMS via URL (no download needed)
3. Runtime: approximately 3-5 minutes for full dataset load (11.6M rows)

**Requirements:**
```
pandas
matplotlib
seaborn
```
All available by default in Google Colab.

---

## Background

This project is part of a healthcare data and AI portfolio built to demonstrate
applied clinical data analysis skills. The author has six years of experience
as a BI QA Analyst at DaVita Kidney Care, specializing in enterprise data
warehouse validation, ETL testing, and clinical feed quality assurance against
the IBM Unified Data Model for Healthcare (UDMH).

**Connect:** [linkedin.com/in/eokuffuor](https://linkedin.com/in/eokuffuor) |
[ekuffuor.github.io](https://ekuffuor.github.io)
