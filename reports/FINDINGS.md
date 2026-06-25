# Healthcare Patient Analytics — Full Findings Report

This document is the detailed companion to the project README. It walks through each dashboard page, the reasoning behind chart choices, and the actual numbers found.

**On the data:** as noted in the README, the base dataset is the synthetic [Healthcare Dataset on Kaggle](https://www.kaggle.com/datasets/prasad22/healthcare-dataset). In its original form it had no real relationships between fields — billing, condition, and insurance were all assigned independently of each other, so there was nothing to analyze. I edited the dataset to introduce realistic structure (cost scaling with condition severity, length-of-stay correlating with cost, etc.) before running the cleaning and EDA pipeline on it. Every finding below is the result of analyzing that edited data with the same rigor I'd apply to a real dataset — the value of this project is in demonstrating the analysis process and choices, not in claiming to have discovered facts about real-world hospital costs.

---

## Data Cleaning Summary

Final cleaned dataset: **54,151 rows × 16 columns** (added `Days Stayed`)

| Issue Found | How It Was Handled |
|---|---|
| Discharge date before admission date | Removed — no way to know which date was actually wrong |
| Negative Billing Amount | Removed |
| Negative or unrealistic Age (>110) | Removed |
| Duplicate rows | Removed with `drop_duplicates()` |
| Inconsistent text casing (e.g. "DIABETES" vs "diabetes") | Standardized via `.str.strip().str.title()` |
| Billing outliers (per condition, IQR method) | Removed |

**Why negative billing was removed, not flipped with `abs()`:** A negative billing value could mean a data entry error where the sign was accidentally flipped, or it could represent a refund/credit record that doesn't belong in a patient charge analysis at all. There's no way to tell which from the data alone. Using `abs()` would have silently assumed the first explanation and fabricated charge amounts that may never have happened. Removing the rows was the only choice that didn't require guessing.

**Why outliers were removed per medical condition, not globally:** Cancer billing runs into the tens of thousands while Asthma billing runs into the low thousands. A single global IQR threshold would have flagged almost all Cancer cases as "outliers" simply because they're expensive, not because they're erroneous. Calculating Q1/Q3/IQR separately within each condition group avoids this and only flags genuine anomalies within each diagnosis's own price range.

---

## Page 1 — Executive Summary

**KPIs:** 54K Total Patients · $1.13B Total Revenue · $20.78K Avg Billing Per Patient · 7.82 days Avg Length of Stay · 35.11% Abnormal Test Rate · 39K Total Doctors

**Monthly Admission Trend (line chart):** Admissions rise from a February low, peak in August at 4,765 cases, then taper into Q4. August being the single highest calendar month (aggregated across all years in the dataset) directly drives the "Peak Month: August" KPI on the Operational page.

**Total Patients by Admission Type (donut):** Elective 40.33%, Emergency 31.56%, Urgent 28.11%. A reasonably even split — no single admission type dominates patient volume, even though (as shown on the Cost Analysis page) they differ substantially in average cost.

**Avg Billing Per Patient by Insurance Provider (bar):** Medicare sits far above the other four providers ($89,024 vs $7,611–$12,526). This looked like a strong insurance-driven cost story until cross-checking on the Cost Analysis page revealed the real cause (see below) — Medicare's high average is almost entirely a byproduct of which condition its patients have, not the insurer itself.

---

## Page 2 — Clinical & Risk

**KPIs:** 35.11% Abnormal Test Rate · 26.85% Inconclusive Rate · 9.73 days Avg Stay for Abnormal Cases · 39K Total Doctors

**Medical Condition × Test Results (heatmap table):**

| Condition | Abnormal | Inconclusive | Normal |
|---|---|---|---|
| Cancer | 5,260 | 2,171 | 1,296 |
| Obesity | 3,239 | 3,306 | 4,957 |
| Diabetes | 2,545 | 1,984 | 2,394 |
| Arthritis | 3,102 | 1,755 | 1,609 |
| Hypertension | 1,829 | 715 | 461 |
| Injury | 1,154 | 1,751 | 3,717 |
| Asthma | 1,882 | 2,859 | 6,165 |

Cancer has by far the highest concentration of Abnormal results relative to its total case volume — most Cancer patients test Abnormal rather than Normal. Asthma is the opposite: the large majority of Asthma cases come back Normal. This is the clearest condition-level risk signal in the data, and it lines up with Cancer also being the highest-cost and longest-stay condition (see Cost Analysis page) — risk, cost, and length of stay all point the same direction for this one condition, which is a direct result of how the data was structured rather than an independently surprising discovery.

**Abnormal Test Rate by Age (bins):** Rate climbs through the middle age bins and tapers at the oldest bin — though the oldest bin also has the smallest patient count, so that tail should be read with some caution since smaller samples make percentages less stable.

**Average Days Stayed by Medical Condition (bar):** Cancer patients stay noticeably longer than every other condition — consistent with Cancer also being the highest-cost, highest-risk condition across every other page of this dashboard.

**Total Patients by Doctor:** Each doctor in this dataset has a relatively small, fairly even patient count. See the caveat at the bottom of this document — doctor assignment in this dataset doesn't represent a real recurring hospital staff roster.

---

## Page 3 — Cost Analysis

**KPIs:** Cancer — Highest Condition Cost · Medicare — Highest Cost Insurance · Lipitor — Most Common Medication · $658.85 Min Billing · $106.20K Max Billing

**Average Billing by Medical Condition:**

| Condition | Avg Billing |
|---|---|
| Cancer | $89,030.05 |
| Diabetes | $13,185.50 |
| Arthritis | $8,511.10 |
| Obesity | $7,747.11 |
| Injury | $7,125.01 |
| Hypertension | $5,899.85 |
| Asthma | $4,420.30 |

This is the single most important number in the whole project. Cancer costs roughly **6.8x more than the next-highest condition (Diabetes)** and up to **20x more than the cheapest (Asthma)**. Almost every other cost-related pattern on this dashboard is, to some extent, explained by whether or not Cancer is involved.

**Medical Condition × Insurance Provider (matrix table):** Looking at the Cancer row specifically — Blue Cross ($88,916), Cigna ($89,196), Medicare ($89,024) — all insurers charge almost the same amount for Cancer patients. The "Medicare = highest cost insurer" KPI on Page 1 is therefore **not actually about Medicare's pricing behavior** — Medicare's patient population in this dataset is concentrated in Cancer cases (it has blank/missing cells for several other conditions in the matrix), so its overall average is pulled up by condition mix, not insurer-specific cost differences. The real cost driver is the diagnosis, not the insurer. This is a useful example of the kind of confounding a real analyst needs to check for, regardless of whether the underlying data is synthetic or real.

**Average Billing by Admission Type:** Urgent ($28,677), Elective ($23,017), Emergency ($10,900). Unlike Insurance Provider, this is a direct, uncomplicated difference — Urgent admissions cost roughly 2.6x more than Emergency admissions on average, independent of condition mix.

---

## Page 4 — Operational Deep-Dive

**KPIs:** 5K Peak Month Count · August Peak Month · 49.48% / 50.52% Gender split · 87.12% Weekday Admission Rate · 12.88% Weekend Admission Rate

**Total Patients by Admission Day:** Monday (~18K) and Tuesday (~13K) are dramatically higher than every other day, including the rest of the weekdays. Weekend days sit at roughly a third of Monday's volume. This is the strongest single operational signal in the data — in a real hospital setting this kind of pattern would be the basis for weighting staffing toward early week.

**Total Patients by Year and Admission Type (stacked area):** Volume rises from 2019 through 2021–2023 and drops sharply by 2024 — likely an incomplete final year in the source data rather than a genuine decline, and worth stating as a caveat if asked.

**Total Patients by Age (bins) and Gender:** Roughly even Male/Female split (49.48% / 50.52%) across all age bins, with the 40–60 age range carrying the highest patient volume overall.

---

## Correlation Analysis

Billing Amount, Days Stayed, and Age correlate strongly with each other (Billing↔Days Stayed: 0.92, Age↔Days Stayed: 0.71, Age↔Billing: 0.58). The strongest of these — Billing and Days Stayed at 0.92 — indicates that **length of stay is the dominant driver of cost** in this dataset: condition determines how long a patient stays, and stay length in turn determines the bill. This is a direct consequence of how the dataset was edited (cost was built to scale with stay duration, which itself scales with condition severity), so it's reported here as a check that the editing produced internally consistent data, not as an independent real-world discovery.

---

## Limitations (say these proactively in interviews)

- **The dataset is synthetic and was edited by me to add realistic relationships between fields** — see the note at the top of this document. Every finding here demonstrates the analysis process applied correctly, not a claim about real hospital economics.
- **Doctor names don't represent a real recurring hospital staff roster** — they're assigned per record in this dataset, not tied to a fixed set of actual doctors. The "Total Patients by Doctor" chart demonstrates the *technique* (Top-N ranking, case-load analysis) rather than a genuine staffing insight.
- **2024 admission volume drop is likely a partial-year artifact** in the source data, not a real downward trend — flagged rather than presented as a finding.
- **Insurance Provider's apparent cost variation is confounded by Medical Condition** — addressed directly above. This is presented as a finding about avoiding a wrong conclusion, not hidden or glossed over.
- All Cancer-related findings (cost, length of stay, abnormal rate) reinforce each other by design, since the dataset was edited with condition severity as the underlying driver of all three.

---

## What I'd add with more time

A classification model (Decision Tree or Logistic Regression, matching current skill level) predicting Abnormal test results from Age, Medical Condition, and Admission Type — turning this from descriptive analytics into a predictive screening tool that flags at-risk patients on admission. On real hospital data, this would also be the point to validate whether the relationships found here actually hold.
