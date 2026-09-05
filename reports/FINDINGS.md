# Healthcare Patient Analytics — Full Findings Report

This document is the detailed companion to the project README. It walks through each dashboard page, the reasoning behind chart choices, the actual numbers found, and honest notes on what the data does and doesn't support.

---

## Data Cleaning Summary

Final cleaned dataset: **53,976 rows × 16 columns** (added `Days Stayed`).

| Issue Found | Count | How It Was Handled |
|---|---|---|
| Duplicate rows | 281 | Removed with `drop_duplicates()` |
| Negative Billing Amount | 204 | Removed — not converted with `.abs()` |
| Inconsistent text casing (e.g. "DIABETES" vs "diabetes") | — | Standardized via `.str.strip().str.title()` |
| Billing outliers (per condition, IQR method) | 434 | Removed |

**Why negative billing was removed, not flipped with `abs()`:** A negative billing value could mean a data entry error where the sign was accidentally flipped, or it could represent a refund/credit record that doesn't belong in a patient charge analysis at all. There's no way to tell which from the data alone. Using `abs()` would have silently assumed the first explanation and fabricated 204 charge amounts that may never have happened. Removing the rows was the only choice that didn't require guessing.

**Why outliers were removed per medical condition, not globally:** Cancer billing naturally runs into the tens of thousands while Asthma billing runs into the low thousands. A single global IQR threshold would have flagged almost all Cancer cases as "outliers" simply because they're expensive, not because they're erroneous. Calculating Q1/Q3/IQR separately within each condition group avoids this and only flags genuine anomalies within each diagnosis's own price range.

---

## Dashboard Revision Notes

This dashboard was originally built as 4 pages and later consolidated to 3, based on a structural review:

- **"Cost Analysis" and "Clinical & Risk" were merged into "Condition Deep-Dive."** Both pages were scoped to Medical Condition, but lived as two separate pages with two separate slicers — a viewer had to cross-reference two pages to see cost and risk for the same condition. Merging them onto one page with a single shared Medical Condition slicer removes that friction.
- **"Total Patients by Doctor" was removed entirely.** Doctor names in this dataset are randomly assigned per record, not a real recurring hospital staff roster, and the chart showed near-uniform bars (15-20 patients per doctor) with no real variation — it was taking up dashboard space without adding insight.
- **A Billing Amount vs Days Stayed scatter plot was added** to Condition Deep-Dive, plotted per patient. This directly visualizes the strongest correlation found anywhere in the data (0.92) — a finding that existed in the analysis from the start but had no dedicated chart in the original 4-page version.
- **A text callout resolving the Medicare confound was added directly to the Overview page**, next to the Insurance Provider chart, rather than requiring the viewer to discover the explanation by separately checking the Condition × Insurance matrix.
- Several low-value KPI cards from the original layout (Min/Max Billing Amount, Most Common Medication, a standalone Inconclusive Rate card) were removed or folded into existing tables, since they were floating facts disconnected from the surrounding page's story.

---

## Page 1 — Overview

**KPIs:** 39K Total Doctors · 54K Total Patients · $1.13B Total Revenue · $20.78K Avg Billing Per Patient · 7.82 days Avg Length of Stay

**Filters:** Year, Month, and Admission Type slicers apply across the whole page.

**Monthly Admission Trend (line chart):** Admissions rise from a February low, peak in August at 4,765 cases, then taper into Q4. August being the single highest calendar month (aggregated across all years in the dataset) directly drives the "Peak Month: August" KPI on the Operational page.

**Total Patients by Admission Type (donut):** Elective 40.33%, Emergency 31.56%, Urgent 28.11%. A reasonably even split — no single admission type dominates patient volume, even though they differ substantially in average cost (see below).

**Avg Billing Per Patient by Insurance Provider (bar) + callout:** Medicare sits far above the other four providers ($89,024 vs $7,611–$12,526). The callout directly beneath this chart states the resolution up front: *"Medicare's high average reflects patient mix, not insurer pricing — all insurers charge nearly the same for Cancer specifically."* This was previously something a viewer had to discover by separately checking the Condition Deep-Dive matrix — now it's stated where the misleading number first appears.

**Average of Billing Amount by Admission Type (bar):** Urgent ($28,677), Elective ($23,017), Emergency ($10,900) — a genuine, direct cost difference independent of condition mix.

---

## Page 2 — Condition Deep-Dive

**KPIs:** 39K Total Doctors · Cancer (Highest Condition Cost) · Medicare (Highest Cost Insurance) · 35.11% Abnormal Test Rate · 9.73 days Avg Stay for Abnormal Cases

**Filters:** Medical Condition and Admission Type slicers, shared across every visual on this page.

**Medical Condition × Test Results (heatmap table):**

| Condition | Abnormal | Inconclusive | Normal |
|---|---|---|---|
| Arthritis | 3,102 | 1,755 | 1,609 |
| Asthma | 1,882 | 2,859 | 6,165 |
| Cancer | 5,260 | 2,171 | 1,296 |
| Diabetes | 2,545 | 1,984 | 2,394 |
| Hypertension | 1,829 | 715 | 461 |

Cancer has by far the highest concentration of Abnormal results relative to its total case volume. Asthma is the opposite — the large majority of Asthma cases come back Normal.

**Medical Condition × Insurance Provider (matrix table):** Looking at the Cancer row specifically — Blue Cross ($88,916), Cigna ($89,196), Medicare ($89,024) — all insurers charge almost the same amount for Cancer patients. This is the underlying data behind the Page 1 callout; the matrix here is where a viewer can verify the claim directly, row by row.

**Abnormal Test Rate by Age (bins):** Rate climbs through the middle age bins and tapers at the oldest bin — though the oldest bin also has the smallest patient count, so that tail should be read with some caution since smaller samples make percentages less stable.

**Billing Amount vs Days Stayed (scatter, per patient) — new addition:** Plots every patient's total billing against their length of stay. The near-linear upward pattern visible in the chart is the direct visual evidence behind the 0.92 correlation figure — cost rises consistently with length of stay across almost the entire patient population, not just on average.

---

## Page 3 — Operational & Demographics

**KPIs:** 5K Peak Month Count · August Peak Month · 87.12% Weekday Admission Rate · 12.88% Weekend Admission Rate

**Total Patients by Admission Day:** Monday (~18K) and Tuesday (~13K) are dramatically higher than every other day, including the rest of the weekdays. Weekend days sit at roughly a third of Monday's volume — the strongest single operational signal in the dataset, relevant to staffing decisions.

**Total Patients by Year and Admission Type (stacked area):** Volume rises from 2019 through 2021–2023 and drops sharply by 2024 — likely an incomplete final year in the source data rather than a genuine decline.

**Total Patients by Age (bins) and Gender (stacked bar):** Roughly even Male/Female split (49.48% / 50.52%) across all age bins, with the 40–60 age range carrying the highest patient volume overall. A separate gender-only donut chart was removed from this page since it duplicated the same split already visible here in more detail.

---

## Honest Limitations (say these proactively in interviews)

- **Doctor names don't represent a real recurring hospital staff roster** — they're assigned per record in this dataset, not tied to a fixed set of actual doctors. This is also why the earlier "Total Patients by Doctor" chart was removed from the dashboard entirely rather than kept as a weak, near-uniform chart.
- **2024 admission volume drop is likely a partial-year artifact** in the source data, not a real downward trend — flagged rather than presented as a finding.
- **Insurance Provider's apparent cost variation is confounded by Medical Condition** — now addressed directly on the dashboard itself via the Page 1 callout, not just in this document.
- All Cancer-related findings (cost, length of stay, abnormal rate) reinforce each other — this is expected and clinically plausible, but worth stating explicitly rather than treating each chart as an independent discovery.

---

## What I'd Add With More Time

A classification model (Decision Tree or Logistic Regression, matching current skill level) predicting Abnormal test results from Age, Medical Condition, and Admission Type — turning this from descriptive analytics into a predictive screening tool that flags at-risk patients on admission.
