# Pre--Sales-Intelligence-System-for-B2B-Businesses

## The Problem This Project Solves

Every B2B sales manager has said it: **"Push harder. More calls. More emails."**

But what if the reason meetings aren't being booked has nothing to do with effort?

In most B2B sales environments, SDRs are evaluated on activity — calls made, emails sent, sequences completed. What rarely gets examined is the quality of the data they are working from. This project was built to examine exactly that.

**The core question this system answers:**

> Are we failing to book meetings because of poor effort — or because the data we are working with is broken before outreach even starts?

This is a **pre-sales analytics pipeline** that evaluates lead data quality, pipeline volume, and early-stage funnel performance — before meetings are ever booked, before revenue is ever discussed.

---

## What This Project Actually Does

Most sales dashboards start at revenue. This one starts at inputs.

The system works in three stages:

1. **Data extraction and structuring** — A Python script takes a raw CRM export (Salesloft-style CSV), strips out irrelevant columns, and retains only the fields needed for analysis. Critically, null values are **preserved intentionally** — because missing data is not a problem to hide, it is a signal to measure.

2. **Quality and funnel analysis** — The structured dataset is analysed across two dimensions: data quality (completeness, duplication, validity) and funnel performance (how leads move from created → attempted → connected → qualified → meeting held).

3. **Power BI visualisation** — Two dashboard pages translate the analysis into decisions. Page one answers "is our data good enough?" Page two answers "where is the pipeline breaking and why?"

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| Python (pandas) | Data extraction, column selection, completeness scoring |
| CSV / leads_cleaned.csv | Structured analytical dataset |
| Power BI Desktop | Interactive dashboard and DAX measures |
| DAX | Calculated columns and measures for all KPIs |

---

## Project Structure

```
pre-sales-intelligence/
│
├── data/
│   └── leads_cleaned.csv          # Structured dataset (nulls preserved)
│
├── python/
│   ├── cell1_extraction.py        # Load raw CSV, select columns, score completeness
│   └── cell2_analysis.py          # Core metric calculations and diagnostic output
│
├── powerbi/
│   └── dashboard_screenshots/     # Page 1 and Page 2 screenshots
│
├── README.md
```

---

## Python Pipeline

### Cell 1 — Data Extraction and Structuring

```python
import pandas as pd
import numpy as np
from google.colab import files

# Upload raw CRM export
uploaded = files.upload()
df = pd.read_csv("leads.csv")
print(f"✓ Raw data loaded: {df.shape}")

# Select only required columns — strip CRM noise
required_columns = [
    "lead_id", "email", "phone", "company", "job_title", "source", "created_date",
    "email_valid_flag", "phone_valid_flag", "duplicate_flag",
    "total_calls_attempted", "total_emails_sent", "total_linkedin_touches",
    "replied_flag", "positive_reply_flag",
    "attempted_flag", "connected_flag", "qualified_flag",
    "meeting_booked_flag", "meeting_held_flag",
    "primary_email_subject", "primary_template_id", "primary_call_script",
    "contact_time_bucket", "contact_day"
]

leads_clean = df[required_columns].copy()
print(f"✓ Clean dataset created: {leads_clean.shape}")

# IMPORTANT: Null values are NOT filled or removed
# They are the primary signal for data quality analysis
print("Null values preserved for analysis ✔")

# Add completeness scoring
leads_clean['completeness_score'] = (
    leads_clean['email'].notna().astype(int) +
    leads_clean['phone'].notna().astype(int) +
    leads_clean['company'].notna().astype(int) +
    leads_clean['job_title'].notna().astype(int)
)

leads_clean['quality_tier'] = leads_clean['completeness_score'].map(
    {4: 'PASS', 3: 'WARN', 2: 'WARN', 1: 'FAIL', 0: 'FAIL'}
)

leads_clean['completeness_pct'] = (leads_clean['completeness_score'] / 4 * 100).round(0)

leads_clean.to_csv("leads_cleaned.csv", index=False)
print("✓ Saved as leads_cleaned.csv")
```

### Cell 2 — Analysis

```python
import pandas as pd

leads_clean = pd.read_csv("leads_cleaned.csv")
total = len(leads_clean)

print("=" * 60)
print("CORE METRICS: LEAD QUALITY + VOLUME")
print("=" * 60)

# Completeness
print("\n🔹 COMPLETENESS:")
print(f"Email completeness:     {(leads_clean['email'].notna().sum()/total)*100:.2f}%")
print(f"Phone completeness:     {(leads_clean['phone'].notna().sum()/total)*100:.2f}%")
print(f"Job title completeness: {(leads_clean['job_title'].notna().sum()/total)*100:.2f}%")
print(f"Company completeness:   {(leads_clean['company'].notna().sum()/total)*100:.2f}%")

# Duplication
print(f"\n🔹 Duplicate Leads: {leads_clean['duplicate_flag'].sum()}")

# Volume
print("\n🔹 LEAD VOLUME:")
leads_clean['created_date'] = pd.to_datetime(leads_clean['created_date'])
print(leads_clean.groupby(leads_clean['created_date'].dt.date).size())

# Source breakdown
print("\n🔹 SOURCE BREAKDOWN:")
print(leads_clean['source'].value_counts(normalize=True) * 100)

print("\n" + "=" * 60)
print("FUNNEL ANALYSIS")
print("=" * 60)

attempted = leads_clean['attempted_flag'].sum()
connected = leads_clean['connected_flag'].sum()
qualified = leads_clean['qualified_flag'].sum()
booked    = leads_clean['meeting_booked_flag'].sum()
held      = leads_clean['meeting_held_flag'].sum()

print(f"Total Leads:     {total}")
print(f"Attempted:       {attempted} ({attempted/total*100:.2f}%)")
print(f"Connected:       {connected} ({connected/attempted*100:.2f}%)")
print(f"Qualified:       {qualified} ({qualified/connected*100:.2f}%)")
print(f"Meeting Booked:  {booked} ({booked/qualified*100:.2f}%)")
print(f"Meeting Held:    {held} ({held/booked*100:.2f}%)")

print("\n" + "=" * 60)
print("DIAGNOSTIC METRICS")
print("=" * 60)

print("\n🔹 Engagement:")
print(f"Reply Rate:          {(leads_clean['replied_flag'].sum()/total)*100:.2f}%")
print(f"Positive Reply Rate: {(leads_clean['positive_reply_flag'].sum()/total)*100:.2f}%")

print("\n🔹 Activity Intensity:")
print(f"Avg Calls per Lead:  {leads_clean['total_calls_attempted'].mean():.2f}")
print(f"Avg Emails per Lead: {leads_clean['total_emails_sent'].mean():.2f}")

print("\n🔹 Quality Tier Breakdown:")
print(leads_clean['quality_tier'].value_counts())

print("\n🔹 Held Rate by Quality Tier:")
print(leads_clean.groupby('quality_tier')['meeting_held_flag'].mean() * 100)

print("\n🔹 Subject Line Performance (Positive Reply Rate):")
print(leads_clean.groupby('primary_email_subject')['positive_reply_flag'].mean().sort_values(ascending=False))

print("\n🔹 Best Contact Time:")
print(leads_clean.groupby('contact_time_bucket')['connected_flag'].mean())

print("\n🔹 Best Contact Day:")
print(leads_clean.groupby('contact_day')['connected_flag'].mean().sort_values(ascending=False))

print("\n" + "=" * 60)
print("✓ ANALYSIS COMPLETE")
print("=" * 60)
```

---

## Metrics Framework

### Core Metrics — Lead Quality and Volume

These are the primary KPIs. They answer: *"Is our data good enough and do we have enough of it to generate meetings?"*

| Metric | What it measures |
|--------|----------------|
| Email Completeness % | % of leads with a valid email address present |
| Phone Completeness % | % of leads with a phone number present |
| Company Completeness % | % of leads with company name present |
| Job Title Completeness % | % of leads with job title present |
| Overall Completeness % | Average across all four fields |
| Total Leads (Weekly) | Volume of leads entering the pipeline |
| Lead Source Distribution | Inbound / outbound / referral split |
| Duplicate Rate % | % of leads flagged as duplicates |

### Diagnostic Metrics — Why Things Happen

| Metric | What it measures |
|--------|----------------|
| Connect Rate % | % of attempted leads that result in a real conversation |
| Reply Rate % | % of leads that replied to any email |
| Positive Reply Rate % | % of leads that replied with genuine interest |
| No-Show Rate % | % of booked meetings that were not held |
| Subject Line Performance | Positive reply rate by email subject variant |
| Connect Rate by Day | Which day of the week produces most connections |
| Connect Rate by Time | Morning vs afternoon connection performance |

### Funnel Metrics — Where the Pipeline Breaks

| Stage | Description |
|-------|------------|
| Lead → Attempted | How many leads had at least one contact attempt |
| Attempted → Connected | How many actual conversations happened |
| Connected → Qualified | How many conversations showed real intent |
| Qualified → Meeting Booked | Booking conversion from qualified leads |
| Meeting Booked → Meeting Held | No-show rate — final outcome |

---

## Power BI — DAX Measures

All measures are created in a dedicated `_Measures` table. Calculated columns (`quality_tier`, `completeness_score`, `completeness_pct`) are added directly to the `leads_cleaned` table.

### Calculated Columns (in leads_cleaned table)

```dax
-- Scores each lead 0–4 based on how many contact fields are present
completeness_score =
    (IF(leads_cleaned[email] <> BLANK(), 1, 0)) +
    (IF(leads_cleaned[phone] <> BLANK(), 1, 0)) +
    (IF(leads_cleaned[company] <> BLANK(), 1, 0)) +
    (IF(leads_cleaned[job_title] <> BLANK(), 1, 0))

-- Labels each lead PASS / WARN / FAIL based on completeness
quality_tier =
VAR score =
    (IF(leads_cleaned[email] <> BLANK(), 1, 0)) +
    (IF(leads_cleaned[phone] <> BLANK(), 1, 0)) +
    (IF(leads_cleaned[company] <> BLANK(), 1, 0)) +
    (IF(leads_cleaned[job_title] <> BLANK(), 1, 0))
RETURN
    IF(score = 4, "PASS",
    IF(score >= 2, "WARN",
    "FAIL"))

-- Completeness as a percentage per lead (0, 25, 50, 75, or 100)
completeness_pct =
    DIVIDE(
        (IF(leads_cleaned[email] <> BLANK(), 1, 0)) +
        (IF(leads_cleaned[phone] <> BLANK(), 1, 0)) +
        (IF(leads_cleaned[company] <> BLANK(), 1, 0)) +
        (IF(leads_cleaned[job_title] <> BLANK(), 1, 0)),
        4
    ) * 100
```

### Measures (in _Measures table)

```dax
Total Leads = COUNTROWS(leads_cleaned)

Email Completeness % =
DIVIDE(
    CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[email] <> BLANK()),
    COUNTROWS(leads_cleaned)
) * 100

Phone Completeness % =
DIVIDE(
    CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[phone] <> BLANK()),
    COUNTROWS(leads_cleaned)
) * 100

Company Completeness % =
DIVIDE(
    CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[company] <> BLANK()),
    COUNTROWS(leads_cleaned)
) * 100

Job Title Completeness % =
DIVIDE(
    CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[job_title] <> BLANK()),
    COUNTROWS(leads_cleaned)
) * 100

Overall Completeness % =
DIVIDE(
    [Email Completeness %] +
    [Phone Completeness %] +
    [Company Completeness %] +
    [Job Title Completeness %],
    4
)

Connect Rate % =
DIVIDE(
    SUM(leads_cleaned[connected_flag]),
    [Total Leads]
) * 100

Meeting Held Rate % =
DIVIDE(
    SUM(leads_cleaned[meeting_held_flag]),
    [Total Leads]
) * 100

Positive Reply Rate % =
DIVIDE(
    SUM(leads_cleaned[positive_reply_flag]),
    [Total Leads]
) * 100

No-Show Rate % =
DIVIDE(
    CALCULATE(
        COUNTROWS(leads_cleaned),
        leads_cleaned[meeting_booked_flag] = 1,
        leads_cleaned[meeting_held_flag] = 0
    ),
    CALCULATE(
        COUNTROWS(leads_cleaned),
        leads_cleaned[meeting_booked_flag] = 1
    )
) * 100

Duplicate Count =
CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[duplicate_flag] = 1)

Duplicate Rate % =
DIVIDE(
    CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[duplicate_flag] = 1),
    [Total Leads]
) * 100

Missing Email Count =
CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[email] = BLANK())

Missing Phone Count =
CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[phone] = BLANK())

Missing Company Count =
CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[company] = BLANK())

Missing Title Count =
CALCULATE(COUNTROWS(leads_cleaned), leads_cleaned[job_title] = BLANK())
```

---

## Key Findings From the Analysis

### Finding 1 — Data quality is blocking outreach before effort is even applied

- Phone completeness sits at **72%** — meaning 28% of leads have no phone number
- Email completeness is **78.5%** — 43 leads cannot be reached by email
- **60% of the pipeline falls in the WARN tier** (2–3 of 4 fields missing)
- Only **35.5% of leads are fully complete (PASS)** with all four contact fields present

> The implication: a significant portion of leads are being worked on by SDRs who have no realistic path to contact. Effort is high; reachability is structurally limited.

### Finding 2 — Activity is not the problem

- **99.5% of leads were attempted** — coverage is near-perfect
- Yet only **47% of leads result in a connection**
- And only **8.5% end in a meeting held**

> This gap confirms that the issue is not how hard the team is working. The issue is the quality of the data they are working from.

### Finding 3 — The biggest funnel drop is at the connect stage

The sharpest drop in the funnel is between attempted and connected — **52.8% of leads are lost here**. This stage is most directly affected by:

- Missing or invalid phone numbers
- Incorrect email addresses
- Wrong contact data at entry point

> Fixing data quality at the source would directly improve connect rate without any change to activity levels.

### Finding 4 — Timing and messaging are secondary levers, not primary ones

- **Wednesday morning** produces the highest connect rate (57.6%)
- **Thursday** is the weakest day (35.7%)
- **Subject B** outperforms Subject A by 6 percentage points on positive reply rate

> These optimisations matter — but only after data quality reaches a baseline threshold. Sending a perfect email to an invalid address still produces zero results.

---

## What This Means for a Business

This project reframes how sales underperformance should be diagnosed.

**Instead of asking:** *"Why are conversions low?"*
**The system enables:** *"Is the pipeline even built on usable data?"*

For a business leader, the practical implications are:

- **Make contact fields mandatory in the CRM at the point of lead entry.** Phone and email should not be optional.
- **Score data quality before assigning leads to SDRs.** A FAIL-tier lead should be enriched or discarded before any outreach begins — not discovered as unreachable after three failed sequences.
- **Audit lead sources by data quality, not just volume.** A channel that delivers 70% of volume but 79% completeness is less efficient than it appears.
- **Use Wednesday morning as the default contact window.** The data shows a 22 percentage point gap between best and worst days.

---

## Power BI Dashboard

The dashboard is split across two pages.

**Page 1 — Data Quality**
Answers: *Is our data good enough to work from?*
Visuals: completeness by field, quality tier donut, stacked bar by channel, matrix heatmap, missing fields bar, duplicate count, weekly intake line chart

**Page 2 — Funnel and Performance**
Answers: *Where is the pipeline breaking and why?*
Visuals: funnel chart, meeting held rate by quality tier, connect rate by day and time, subject line A/B performance, connect rate by tier

---

## How to Run This Project

```bash
# 1. Open Google Colab
# 2. Upload your raw CRM CSV (rename to leads.csv)
# 3. Run Cell 1 — extracts and structures the data
# 4. Download leads_cleaned.csv from Colab:
from google.colab import files
files.download("leads_cleaned.csv")

# 5. Open Power BI Desktop
# 6. Home → Get Data → Text/CSV → select leads_cleaned.csv
# 7. Create DateTable and relationship (see DAX section)
# 8. Create all measures and calculated columns
# 9. Build dashboard pages
```

---

## Automation — What Is and Is Not Automated

| Step | Automated? | How |
|------|-----------|-----|
| Column extraction from raw CSV | ✅ Yes | Python Cell 1 |
| Completeness scoring per lead | ✅ Yes | Python Cell 1 |
| Quality tier assignment | ✅ Yes | Python Cell 1 + DAX calculated column |
| Core metric calculation | ✅ Yes | Python Cell 2 + DAX measures |
| Dashboard refresh | Manual | Re-load CSV in Power BI |
| CRM to CSV to Power BI | Manual | Direct API connection is a future enhancement |

### Connecting Python directly to Power BI (Future Enhancement)

Power BI Desktop supports Python scripts as a data source. This means the entire extraction pipeline can be run inside Power BI itself — eliminating the manual CSV step.

**How it would work:**
Home → Get Data → Python Script → paste Cell 1 code → Power BI runs it and imports the output directly as a table. When you refresh the dataset, the Python script re-runs automatically.

This is not implemented in the current version but is a straightforward next step.

---

## Future Enhancements

- **Direct CRM API connection** — pull from Salesloft or HubSpot automatically on a schedule
- **Claude API integration** — pass weekly metric summaries to Claude programmatically and receive a natural language digest of anomalies and recommendations
- **Anomaly detection** — flag when connect rate drops more than 20% week-on-week
- **Predictive lead scoring** — rank leads by likelihood to convert based on completeness and source
- **Power BI scheduled refresh** — automate weekly dataset updates without manual CSV re-upload

---

## A Note on AI in This Project

Claude (Anthropic) was used in this project as a thinking and building tool — for schema design, DAX formula writing, Python debugging, and analysis interpretation.

But there is an important distinction worth making clearly:

**AI can help you build a dashboard. It cannot decide what the dashboard should measure.**

The decisions that made this project meaningful — measuring data quality instead of revenue, preserving null values instead of cleaning them, focusing on pre-sales instead of post-sales — were analytical decisions that required understanding the business problem first.

The skill is not knowing how to use AI. The skill is knowing what problem to point it at.

---
