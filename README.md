# Jobify — Recruitment Analytics & BI Dashboard

**Role:** Data Engineer / BI Developer (solo) <br></br>
**Context:** Built during my IT & Software Engineering internship ,NSTP (Aug–Sep 2026) <br></br>
**Stack:** MySQL/MariaDB · SQL Views · Looker Studio · Google Sheets · Data Modeling

> Jobify's live database, raw applicant records, and source queries are company-confidential, so this is a case-study write-up of the design and build rather than a public code repo. Numbers below are real, verified results from the production system.

---

## The problem

Jobify is an applicant-tracking system. Hiring activity — every application, status change, and reviewer action — was being logged, but no one could see it: there was no reporting layer, so questions like *"which jobs are stalling?"*, *"who on the hiring panel is overloaded?"*, and *"how long does hiring actually take?"* had no fast answer. My job was to design the analytics layer end to end: get the data out of MySQL, model it correctly, and turn it into a BI dashboard the hiring team could actually use.

## Dashboard Preview

The dashboard consists of 10 analytical sections, each designed to provide a different perspective on recruitment performance, operational efficiency, and hiring outcomes.

<details open>
<summary><strong>01. KPI Overview</strong></summary>

<img src="https://github.com/fizaf0012/Jobify-Recruitment-Analytics/blob/main/dashboard/Jobify_Pulse_-_360%20(1).pdf" width="100%" alt="Jobify KPI Overview Dashboard">

Executive-level overview of recruitment performance, key metrics, and overall hiring health.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%281%29.pdf)

</details>

<details>
<summary><strong>02. Recruitment Pipeline Overview</strong></summary>

<img src="dashboard/images/recruitment-pipeline.png" width="100%" alt="Recruitment Pipeline Overview">

Application pipeline distribution, stage progression, and recruitment bottlenecks.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%282%29.pdf)

</details>

<details>
<summary><strong>03. User & Action Workload</strong></summary>

<img src="dashboard/images/user-workload.png" width="100%" alt="User and Action Workload">

Reviewer activity, workload distribution, action-level performance, and time allocation.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%283%29.pdf)

</details>

<details>
<summary><strong>04. Human vs AI & Time-State Analysis</strong></summary>

<img src="dashboard/images/human-vs-ai.png" width="100%" alt="Human vs AI Analysis">

Comparison of human and AI-driven processing activity, including active and inactive time-state analysis.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%284%29.pdf)

</details>

<details>
<summary><strong>05. Jobs & User-Job Relationship</strong></summary>

<img src="dashboard/images/jobs-analysis.png" width="100%" alt="Jobs and User Job Relationship">

Job-level recruitment performance, application distribution, and reviewer-to-job relationships.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%285%29.pdf)

</details>

<details>
<summary><strong>06. Application-Level Deep Dive</strong></summary>

<img src="dashboard/images/application-deep-dive.png" width="100%" alt="Application Level Analysis">

Detailed application-level analysis, including individual processing history, rejection patterns, and time allocation.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%286%29.pdf)

</details>

<details>
<summary><strong>07. Demographics & Geography</strong></summary>

<img src="dashboard/images/demographics.png" width="100%" alt="Candidate Demographics and Geography">

Candidate distribution across demographic categories and geographical locations.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%287%29.pdf)

</details>

<details>
<summary><strong>08. Trends & Root-Cause Analysis</strong></summary>

<img src="dashboard/images/trends-root-cause.png" width="100%" alt="Trends and Root Cause Analysis">

Time-based recruitment trends and drill-down analysis to investigate the factors behind performance metrics.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%288%29.pdf)

</details>

<details>
<summary><strong>09. Conversion & Performance Benchmarking</strong></summary>

<img src="dashboard/images/conversion-benchmarking.png" width="100%" alt="Conversion and Performance Benchmarking">

Hiring conversion rates, recruitment efficiency, and comparative performance benchmarks.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%289%29.pdf)

</details>

<details>
<summary><strong>10. Operational Risk & Alerts</strong></summary>

<img src="dashboard/images/operational-risk.png" width="100%" alt="Operational Risk and Alerts">

Identification of operational bottlenecks, recruitment risks, and areas requiring management attention.

[View Full Dashboard (PDF)](dashboard/Jobify_Pulse_-_360%20%2810%29.pdf)

</details>

## Data pipeline

MySQL (Laravel app DB) → SQL Views (phpMyAdmin) → CSV export → Google Sheets → Looker Studio


With no direct warehouse or BI connector available, Google Sheets became the bridge layer — CSV exports of the SQL views, refreshed and re-pointed into Looker Studio as the data model evolved.

## Getting the data model right

The source schema spans five relational tables (applications, application history, time-tracking logs, users, jobs). The first and most important decision was **grain**: a naive join produces one row per *reviewer action*, not one row per *application* — so a direct `AVG(days_to_hire)` on the raw joined data silently double- and triple-counts every multi-step application.

I built a layered view structure instead of one flat table:

- **`vw_jobify_fact_activity`** — the action/session-grain fact view (every reviewer action, time-logged), filtered to active time only
- **`vw_job_application_summary`** — one row per application (fixes the grain bug above; powers time-to-hire, status, and rejection analysis)
- **`vw_candidate_summary`** — one row per unique candidate (by email, not name — see data-quality note below)
- **`vw_stage_transitions`** — one row per stage-to-stage move, built with a `LAG()` window function, to measure how long applications sit at each pipeline stage

This separation meant every chart in the dashboard could be built on the view with the *correct* grain for its question, instead of guessing at `COUNT` vs `COUNT(DISTINCT ...)` each time.

## Data quality issues found and fixed

Real production data is never clean. Some of what I diagnosed and corrected:

- **Inconsistent geography text** — `"Islamabad"` and `"Islamabad, Pakistan"` were being treated as different cities, fragmenting the location breakdown; normalized with a cleanup pass.
- **Candidate name collisions** — grouping candidates by name alone merged different real people who happened to share a name, undercounting unique candidates by ~40%. Fixed by keying on email instead.
- **Blank-identity applications** — a handful of applications had no name/email and were collapsing into phantom "repeat candidates" when grouped; excluded from candidate-level views.
- **Uninformative status/reason fields** — some categorical fields had near-zero variance, which would have made several planned charts meaningless; worked with the team to decide on realistic, signal-driven population logic rather than leaving the fields flat.

## The dashboard

10 sections, 13 verified KPIs, ~45 charts and tables, and roughly 100 dynamic "storytelling" text elements that update live with the applied filters:

| Area | What it answers |
|---|---|
| Overview | Top-line hiring health at a glance |
| Pipeline | Where applications are getting stuck, funnel conversion |
| People / Workload | Who on the hiring panel is doing the work, and how fast |
| Automation | How much of the pipeline is AI- vs human-processed |
| Jobs | Which roles are hardest to fill, and why |
| Applications | Rejection reasons, source quality, complexity vs. time-to-hire |
| Demographics | Candidate pool composition |
| Trends | Time-to-hire and volume over time |
| Root-cause drill-down | User → Job → Action → Session level detail behind any KPI |

Every KPI was cross-checked against the source data independently (direct SQL aggregates and pandas recomputation on the exported CSVs) before being treated as "final" — including catching a chart that had silently reverted from median to sum aggregation, and a filter-scope bug that was double-counting rows that shared a timestamp.

## A sample insight

Looking at application complexity (number of pipeline actions) against time-to-hire surfaced a moderate positive correlation (Pearson r ≈ 0.55) — more review steps tend to mean a longer process, as expected, but it's a threshold effect rather than a clean linear relationship, and the sample size at the high-complexity end is small enough to flag as a caveat rather than a firm conclusion.

## Outcome

A live, filterable Looker Studio dashboard the hiring team uses to track pipeline health and reviewer workload, backed by a documented, grain-correct SQL layer — plus a full project report documenting every calculated field, chart, and data source for handover.

## What I'd do differently

The schema has since evolved (the custom time-tracking table was replaced by AI-driven resume-parsing fields), which meant going back and re-auditing which charts the new schema could still support — a good reminder to design the view layer to be resilient to upstream schema changes where possible, not just correct for the schema on day one.
