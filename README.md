# Clinical Trial Data-Quality Dashboard
**Clinical Data Track — Project 1**

## Problem
Poor data quality is one of the most persistent risks in clinical trials. Missing lab values, out-of-range results, duplicate records, and incomplete participant follow-up can all distort a trial's findings if they go undetected — leading to unreliable safety and efficacy conclusions, delayed regulatory submissions, and, in the worst cases, decisions made on flawed data. Catching these issues early and consistently is a core part of clinical data quality management.

Rule-based checks (fixed thresholds and range validations) catch the obvious errors, but they miss subtler patterns — combinations of borderline values that don't individually break a threshold but are still statistically unusual. This project addresses that gap by building a working pipeline that combines rule-based SQL checks with AI-assisted anomaly detection, so both types of data-quality issues can be identified and presented in a single, reviewable dashboard. A synthetic dataset built to resemble a real clinical trial export was used, since real trial data is not accessible outside a sponsoring organization.

## Approach
1. **Synthetic dataset generation** — A Python/Faker script produced 1,224 visit records across 300 subjects and 5 sites, with realistic lab values (hemoglobin, WBC count, ALT, systolic BP). Data-quality issues were deliberately injected to give the downstream checks real problems to find: missing values (~4% per lab field), out-of-range values (~3% of rows), duplicate visit-label entries (~1.5%), and a small number of implausible ages.
2. **SQL analysis** — Seven queries were run against a SQLite database, covering: missing values by site, out-of-range lab results, duplicate subject/visit records, adverse event rate by site, visit completeness, visit date sequencing, and sex distribution by site.
3. **AI-assisted anomaly detection** — An unsupervised Isolation Forest model (scikit-learn) was trained on each visit's lab values and age, flagging statistically unusual records independent of any fixed rule.
4. **Comparison** — SQL-flagged and model-flagged records were compared directly, at both the individual-visit and subject level, to distinguish confirmed rule-breaks from records that require human review.
5. **Power BI dashboard** — An interactive dashboard presents the full analysis, with site and visit slicers, click-to-filter drill-down, and a clear visual separation between confirmed issues and records flagged for review.

## Results
- **36 records** were flagged by SQL for out-of-range lab values, checked against standard adult reference ranges (Hemoglobin 12–17 g/dL, WBC 4–11 x10^9/L, ALT 7–56 U/L, Systolic BP 90–140 mmHg).
- The Isolation Forest model flagged **62 records**. **100% of the SQL-flagged records were also caught by the model**, confirming the model does not miss the obvious, rule-breaking errors.
- **26 records** were flagged by the model *only* — visits that pass every individual range check but combine multiple borderline values at once (for example, hemoglobin and ALT both sitting near opposite edges of their normal ranges in the same visit). The model also independently identified a data-entry error in the age field (age = 0), with no age-specific rule provided to it.
- **36 duplicate records** and **5 subjects** flagged at more than one separate visit were identified — the strongest candidates for prioritized follow-up.
- Each subject was scheduled for 5 visits — **Screening, Baseline, Week 4, Week 8, and Week 12**. **66.67% of subjects did not complete all 5 scheduled visits.**
- A chronological sequencing check (verifying no visit was dated before an earlier one) returned zero errors — a legitimate "checked, nothing found" result.
- Sex distribution was balanced (48–52%) at every site, ruling out gender imbalance as a confound in site-level comparisons.
- Adverse event rate increased with age (Young Adult 7.0% → Adult 9.9% → Older Adult 10.6%), consistent with real-world clinical patterns.

## Limitations
- **Synthetic data only.** No real patient data was used, in line with data-privacy requirements for a public portfolio project.
- **Simplified reference ranges.** The lab thresholds used are general adult ranges, not adjusted for sex, age, or specific assay — a real trial would use more precise, population-specific ranges.
- **The cause of the incomplete-visit rate is unknown.** Each subject was randomly assigned 3, 4, or 5 completed visits to simulate real-world variation, but in an actual trial an incomplete visit record can result from several different causes — a genuine dropout, a rescheduled appointment, a data-entry delay, or a record simply not yet submitted by the site. Before treating any incomplete case as a dropout, the underlying cause would need to be investigated, since the correct follow-up action differs by cause.
- **Mean-imputation of missing values was a modeling workaround, not a clinical data-handling practice.** The Isolation Forest cannot process missing values, so they were filled with the column mean solely for that model's internal calculation. The source dataset and every other part of the analysis (SQL queries, dashboard tables) retain missing values as missing. In real clinical data work, a coordinator would never fill in a missing lab value on a source record, as doing so would compromise the "Original" data principle under ALCOA+ (Attributable, Legible, Contemporaneous, Original, Accurate, Complete, Consistent, Enduring, Available) . Rows with missing critical fields should instead be excluded from automated scoring or flagged separately for manual review.
- **The 26 model-only anomalies are not confirmed errors.** They represent statistically unusual but medically plausible combinations of near-extreme values, and require review by clinical or data-coordination staff to determine significance. The model's role is to prioritize records for review, not to make a final determination.
- **Duplicate detection assumes duplication, not legitimate re-testing.** Identical duplicate visit-label entries were treated as data-entry duplication. In a real dataset, a duplicate visit label with *differing* lab values could represent a legitimate re-test (e.g., a sample re-drawn after an initial error), which would need to be handled differently from a true duplicate.
- **Known duplicate rows were excluded from visit-completeness and repeat-offender counts**, to prevent a single duplicated entry from artificially inflating a subject's flagged-visit count.

## Tools
Python (pandas, scikit-learn, Faker), SQLite/SQL, Power BI Desktop, DAX.
