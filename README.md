# Search Ranking A/B Test: Improving Conversion Without Increasing Booking Time

**A product analytics and experimentation case study using Python**

This project investigates whether an updated search-ranking experience improves booking conversion on an online travel platform while keeping the booking process from becoming slower.

The analysis combines session-level activity with user-level experiment assignments. It checks the integrity of the experiment split, measures the impact of the variant on conversion, evaluates booking time as a guardrail metric, and applies explicit criteria to produce a rollout recommendation.

> **Project type:** A/B testing · Product analytics · Statistical hypothesis testing · Data preparation · Experiment evaluation  
> **Tools:** Python, pandas, NumPy, SciPy, statsmodels, Jupyter Notebook

---

## Table of contents

- [1. Project context](#1-project-context)
- [2. Objectives and analytical questions](#2-objectives-and-analytical-questions)
- [3. Repository structure](#3-repository-structure)
- [4. Dataset overview](#4-dataset-overview)
- [5. Data preparation](#5-data-preparation)
- [6. Experiment design and metrics](#6-experiment-design-and-metrics)
- [7. Statistical methodology](#7-statistical-methodology)
- [8. Rollout decision framework](#8-rollout-decision-framework)
- [9. User-level sensitivity analysis](#9-user-level-sensitivity-analysis)
- [10. Interpreting the outputs](#11-interpreting-the-outputs)



---

## 1. Project context

Search ranking is an important part of the online travel booking journey. The order in which hotels, flights, or other travel options appear can influence what users explore and whether they complete a booking.

A revised ranking algorithm may generate more bookings, but conversion alone does not describe the entire user experience. A change that increases bookings while making the booking process materially slower could introduce a different product problem.

This case study evaluates two experiment groups:

- **Control:** users assigned to the existing search-ranking experience.
- **Variant:** users assigned to the updated search-ranking experience.

The analysis focuses on two outcomes:

1. Whether the variant increases the proportion of sessions that result in a booking.
2. Whether the variant increases the time taken to complete a booking.

The goal is to translate the experiment data into an evidence-based recommendation about whether the updated experience meets the defined rollout criteria.

## 2. Objectives and analytical questions

The project is organised around the following questions:

| Area | Analytical question | Why it matters |
|---|---|---|
| Experiment integrity | Is the observed user allocation consistent with the expected 50/50 split? | A sample ratio mismatch may indicate an assignment, logging, or data-quality issue. |
| Primary outcome | Does the variant produce a higher session conversion rate than control? | Conversion is the main measure of the experiment's intended benefit. |
| Guardrail outcome | Does the variant increase mean time to booking among sessions that convert? | A conversion improvement should be considered alongside the booking experience. |
| Decision | Do the primary and guardrail results satisfy the predefined rollout rules? | Makes the recommendation consistent and transparent. |

### Expected analytical deliverables

Running the notebook produces:

- A merged analysis dataset named `sessions_x_users`.
- A session-level `conversion` indicator.
- A sample ratio mismatch (SRM) chi-square statistic and p-value.
- A two-proportion z-test for the primary conversion metric.
- A Welch two-sample t-test for the booking-time guardrail.
- Relative effect calculations for both metrics.
- A Boolean evaluation of the primary and guardrail criteria.
- A final `decision_full_on` value (`yes` or `no`).
- A user-level sensitivity analysis to examine the influence of repeated sessions.

## 3. Repository structure

Keep the notebook and both datasets in the same project directory:

```text
search-ranking-ab-test/
│
├── README.md
├── search_ranking_ab_test_analysis.ipynb
├── sessions_data.csv
└── users_data.csv
```

The notebook loads the CSV files using relative paths. It does not depend on a machine-specific directory such as a personal Windows or macOS file location.

## 4. Dataset overview

The project uses two CSV files. They represent different levels of the same experiment: session activity and user-level experiment assignment.

### 4.1 `sessions_data.csv`

Contains session-level activity. The supplied file contains **16,981 rows and 5 columns**.

| Column | Description | Role in the analysis |
|---|---|---|
| `session_id` | Identifier for a session. | Used to validate session uniqueness and count sessions. |
| `user_id` | Identifier for the user associated with a session. It may be missing. | Join key linking sessions to experiment assignment. |
| `session_start_timestamp` | Timestamp indicating when the session started. | Session context; not used directly in the hypothesis tests. |
| `booking_timestamp` | Timestamp of a booking, when a booking occurred. Missing when no booking was recorded. | Used to derive the primary conversion indicator. |
| `time_to_booking` | Time from session start to booking, measured in minutes; missing when no booking occurred. | Used for the guardrail analysis. |

### 4.2 `users_data.csv`

Contains user-level experiment assignment. The supplied file contains **10,000 rows and 2 columns**.

| Column | Description | Role in the analysis |
|---|---|---|
| `user_id` | Unique user identifier. | Join key linking assignments to sessions. |
| `experiment_group` | Experiment group assigned to the user: `control` or `variant`. | Defines the comparison groups. |

The supplied assignment file contains 4,991 control users and 5,009 variant users. The session data contains records with missing `user_id`; these sessions cannot be linked to a user-level assignment and are excluded from the control-versus-variant comparison.

### 4.3 Data relationship

The assignment is stored at the **user level**, whereas activity and outcomes are stored at the **session level**. A user can therefore be associated with more than one session.

The notebook uses a left join from sessions to users. This preserves the session records while attaching the experiment assignment where a matching `user_id` is available.

## 5. Data preparation

The notebook performs the following preparation steps before testing the experiment:

1. **Load the source files.** Read both CSV files into pandas DataFrames.
2. **Validate identifiers.** Check that `session_id` is unique in the session table and `user_id` is unique in the assignment table.
3. **Validate group labels.** Confirm that the assignment column contains only the expected `control` and `variant` values.
4. **Join the data.** Merge session activity with user assignment using `user_id` and a left join. The merge uses `validate="many_to_one"` to check that a session can match no more than one user-assignment record.
5. **Create the primary outcome.** Derive `conversion` as 1 when `booking_timestamp` is present and 0 when it is missing.
6. **Define the analysis population.** Retain sessions assigned to either experiment group for the group comparisons. Sessions without an assignment remain identifiable in the merged data but are not included in the experiment comparison.

The main merged DataFrame is named `sessions_x_users`. The assigned-session analysis subset is named `analysis_data`.

## 6. Experiment design and metrics

### 6.1 Significance threshold

The notebook uses a predefined confidence level of **90%**, corresponding to a significance level of:

\[
\alpha = 1 - 0.90 = 0.10
\]

The decision rule therefore compares relevant p-values with 0.10. This threshold is intentionally retained from the experiment specification used for this case study.

### 6.2 Primary metric: session conversion rate

Conversion is defined at the session level:

\[
\text{Conversion} =
\begin{cases}
1 & \text{if } booking\_timestamp \text{ is present} \\
0 & \text{otherwise}
\end{cases}
\]

For each experiment group:

\[
\text{Conversion rate} = \frac{\text{Number of converted sessions}}{\text{Number of assigned sessions}}
\]

A higher conversion rate in the variant is favourable.

**Important:** this is a session conversion rate, not the percentage of unique users who booked at least once. Users with multiple sessions contribute multiple session observations to the primary metric.

### 6.3 Guardrail metric: time to booking

The guardrail is the mean `time_to_booking`, measured in minutes, among sessions that resulted in a booking.

Only booked sessions have an observed booking time, so missing values are excluded from the group means and the guardrail test.

A lower mean time to booking in the variant is favourable. The metric is conditional on a booking having occurred; it does not measure time to booking for every session, including sessions that never converted.

## 7. Statistical methodology

### 7.1 Sample ratio mismatch (SRM) check

**Purpose:** check whether the user-level assignment counts are consistent with the expected 50/50 allocation.

**Method:** chi-square goodness-of-fit test.

- Observed counts: number of users assigned to control and variant.
- Expected counts: half of all assigned users in each group.
- Null hypothesis: the observed assignment proportions are consistent with a 50/50 split.
- Interpretation: a p-value below `alpha` indicates evidence that the observed split differs from the expected allocation and should be investigated.

The SRM check uses `users_data.csv`, because randomization is recorded at the user level. It does not use session counts, which may differ because users can generate multiple sessions.

### 7.2 Primary metric test: two-proportion z-test

**Purpose:** test whether the session conversion proportions differ between control and variant.

**Method:** two-sided two-proportion z-test.

- Null hypothesis: control and variant have the same session conversion proportion.
- Alternative hypothesis: the session conversion proportions are different.
- Decision threshold: `alpha = 0.10`.

The statistical test is two-sided. The direction of the observed difference is assessed separately: a positive relative effect is required for the primary metric to pass the rollout criteria.

### 7.3 Guardrail test: Welch's two-sample t-test

**Purpose:** test whether mean time to booking among converted sessions differs between control and variant.

**Method:** two-sided Welch independent-samples t-test (`equal_var=False`).

- Null hypothesis: the two groups have the same mean time to booking.
- Alternative hypothesis: the group means are different.
- Decision threshold: `alpha = 0.10`.

Welch's t-test is used because it does not require the two groups to have equal variances. Only non-missing booking times are included.

### 7.4 Relative effect calculations

For both metrics, the notebook calculates the relative difference between variant and control:

\[
\text{Relative effect} =
\frac{\text{Variant mean} - \text{Control mean}}
{\text{Control mean}}
\]

For conversion, the group mean of the binary conversion variable is the conversion rate. A positive relative effect indicates a higher conversion rate in the variant.

For booking time, a negative relative effect indicates a lower mean time to booking in the variant.

These values are relative percentage changes, not absolute percentage-point differences and not standardized effect sizes such as Cohen's *d*.

## 8. Rollout decision framework

The notebook applies the following explicit decision logic.

### Primary metric passes when both conditions are true

- The primary p-value is below `alpha`.
- The variant's relative conversion effect is positive.

In code:

```python
primary_pass = (pval_primary < alpha) and (effect_size_primary > 0)
```

### Guardrail passes when either condition is true

- The guardrail p-value is greater than or equal to `alpha`, so the test does not find a statistically significant difference; **or**
- The variant's relative booking-time effect is negative, indicating a lower observed mean time to booking.

In code:

```python
guardrail_pass = (pval_guardrail >= alpha) or (effect_size_guardrail < 0)
```

### Final decision

A full-rollout recommendation is made only when both criteria pass:

```python
decision_full_on = "yes" if (primary_pass and guardrail_pass) else "no"
```

| Primary criterion | Guardrail criterion | Decision |
|---|---|---|
| Pass | Pass | `yes` — full rollout under the specified rules |
| Pass | Fail | `no` — do not recommend full rollout |
| Fail | Pass | `no` — do not recommend full rollout |
| Fail | Fail | `no` — do not recommend full rollout |

This is a decision rule for this case study, not a universal standard for experimentation. In particular, a non-significant guardrail result is not proof that booking time is unchanged or that a longer booking time has been ruled out. See [Assumptions and limitations](#12-assumptions-and-limitations).

## 9. User-level sensitivity analysis

The main conversion analysis uses sessions as observations, but experiment assignment takes place at the user level. A user may contribute multiple sessions, so session outcomes from the same user may not be statistically independent.

To examine whether repeated sessions materially affect the interpretation, the notebook performs an additional user-level sensitivity analysis:

1. Group assigned sessions by `experiment_group` and `user_id`.
2. Calculate each user's average session conversion (their session conversion rate).
3. Calculate each user's mean observed booking time, using their booked sessions.
4. Compare the user-level conversion-rate observations between groups with Welch's two-sample t-test.
5. Compare user-level mean booking times among users with at least one booked session using Welch's two-sample t-test.

This gives each user one observation in the sensitivity comparisons rather than allowing users with more sessions to contribute more observations.

The sensitivity analysis is a diagnostic, not a direct replacement for the primary session-level analysis. The user-level average session conversion rate gives equal weight to each user, whereas the pooled session conversion rate gives equal weight to each session. These are different estimands and may produce different estimates.

## 10. Interpreting the outputs

The notebook prints intermediate tables and a final summary. Focus on the following values:

| Output | Meaning | How to read it |
|---|---|---|
| `srm_chi2_pval` | P-value from the user-level assignment split check. | A value below `alpha` flags a possible sample ratio mismatch. |
| `pval_primary` | P-value from the two-proportion conversion test. | A value below `alpha` indicates evidence of a difference in conversion rates; check that the variant effect is positive. |
| `pval_guardrail` | P-value from Welch's test of mean booking time among booked sessions. | A value below `alpha` indicates evidence of a difference in mean booking time; inspect the direction of the effect. |
| `effect_size_primary` | Relative conversion change from control to variant. | Positive is favourable for conversion. |
| `effect_size_guardrail` | Relative change in mean booking time from control to variant. | Negative indicates a shorter mean booking time in variant. |
| `decision_full_on` | Final decision generated by the specified rules. | `yes` means both criteria pass; `no` means at least one criterion fails. |

A p-value should not be interpreted as the probability that the treatment works or as the probability that the result occurred by chance. It is calculated under the null hypothesis and the assumptions of the selected test.

When presenting the findings, report the group-level rates or means, the direction and size of the effect, the p-value, and the limitations of the comparison. Avoid reporting a p-value without the corresponding business or product interpretation.

