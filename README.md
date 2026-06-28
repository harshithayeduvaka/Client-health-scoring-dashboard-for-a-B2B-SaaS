# RavenStack Client Health Dashboard
**Live dashboard:** https://datastudio.google.com/reporting/d2fbc218-feff-416c-aed2-08c9a478298b
**Tools:** Google Sheets · Looker Studio
**Dataset:** RavenStack synthetic SaaS data (5 tables, ~33k rows) — credit to River @ Rivalytics
<img width="859" height="640" alt="image" src="https://github.com/user-attachments/assets/8c71b290-8f7f-410f-a88b-e372e1ca6ff6" />

---

## The brief

RavenStack is a stealth-mode B2B SaaS company in private beta with 500 paying clients across five verticals — DevTools, FinTech, Cybersecurity, HealthTech, EdTech. The Customer Success team needs to know two things every Monday morning: *which clients are at risk of churning, and of those, which ones are worth the most revenue to save.* This dashboard answers both in a single sort and gives CS leadership a defensible, repeatable way to prioritize their retention effort across a growing book of business.

---

## The numbers at a glance

| Metric | Value |
|---|---|
| Clients scored | 500 |
| Average health score (0–100) | 56.5 |
| Critical-band clients | 33 (6.6%) |
| Total active MRR tracked | ~$10.16M |
| Pillars in the score | Adoption · Reliability · Support · Revenue · Retention |
| Normalization method | Cohort percentile rank |

---

## Methodology

Every client gets a 0–100 score, blended from five equally-weighted pillars:

| Pillar | Signals | Healthy when |
|---|---|---|
| **Adoption** | Usage event volume, distinct features used | High |
| **Reliability** | Error rate per usage event | Low |
| **Support** | Ticket volume, escalation rate, CSAT | Low load, high CSAT |
| **Revenue** | Active MRR, auto-renew share | High |
| **Retention** | Trial status, downgrades, churn events | Few flags |

### The one design decision that matters

Raw metrics live on wildly different scales — one Enterprise client pays $33k/month, others pay $300. Standard min-max scaling would let that single outlier crush everyone else's revenue score toward zero. So I made a different choice:

> **Rather than min-max scaling, which lets outliers (one Enterprise client at $33k MRR) crush everyone else, I rank each client against the cohort on every metric. A median client lands at 50 on every dimension, regardless of how skewed the underlying distribution is.**

Implemented in Google Sheets via `PERCENTRANK`. For "lower-is-better" metrics (errors, escalations, ticket volume), the rank is inverted (`1 - PERCENTRANK`). The final score is the average of the five pillar sub-scores, banded into Healthy (≥70), At Risk (≥40), and Critical (<40).

---

## Findings

**1. Industry doesn't predict client health.** Across the five verticals — Cybersecurity, FinTech, EdTech, DevTools, HealthTech — average health scores fall within a single point of each other (~55–56). The operational implication: a CS team shouldn't segment retention strategies by industry. Segment by usage tier and revenue instead.

**2. Critical clients punch below their weight in revenue.** They make up 6.6% of accounts but only ~4% of total MRR. The healthiest clients pay roughly 3.5× more per month on average than the critical ones. The retention dollars are not in the Critical tier — they're in the *At Risk* tier above it. If a CS team only has bandwidth to call 50 clients a week, that's where the call list should come from.

**3. The "At Risk" band is too broad to be operational.** 82% of clients fall into it, which means the label barely functions as a triage signal — it's a bucket too wide to drive action. A production version would subdivide this using trend data (e.g., "At Risk – Stable" vs "At Risk – Declining" based on the previous 30/60/90-day score trajectory) so the CS team gets a focused list, not a near-majority dump.

---

## Architecture

Raw data lives in 5 CSVs (500 accounts, 5k subscriptions, 25k usage events, 2k tickets, 600 churn events). A helper column on `feature_usage` joins usage to accounts via a 2-hop VLOOKUP through subscriptions. The `Health` tab aggregates all five raw tables to one row per client with 19 columns. `PERCENTRANK` normalizes each metric 0–100 before averaging the pillars. A static copy (`Health_Static`) feeds Looker for performance.

```
accounts.csv (500)         ─┐
subscriptions.csv (5,000)  ─┼─→  Health tab          →  Health_Static  →  Looker Studio
feature_usage.csv (25,000) ─┤    (1 row per client,     (values-only       (interactive
support_tickets.csv (2,000)─┤    19 derived columns,    snapshot for        dashboard)
churn_events.csv (600)     ─┘    pillar scoring)       fast rendering)
```

The non-obvious challenge was that usage logs at the *subscription* level, not the account level — and accounts have ~10 subscriptions each. I solved the two-hop join with a single `ARRAYFORMULA(VLOOKUP(...))` helper column on the 25k-row usage table. One formula, no manual stitching.

[ *Insert screenshot of the Health tab here — it's the most credibility-building visual in the whole writeup* ]

---

## Limitations

I'd rather be honest about the gaps than oversell the model. In a production version:

- **Score is a snapshot; there's no trend over time.** A client trending downward week-on-week is more interesting than a static low score. I'd add weekly snapshots and compute *change in score* as a separate signal.
- **CSAT had 41% nulls in the source data; I substituted a fallback of 4.** In production I'd impute more carefully (e.g., last-known score per industry) or weight CSAT lower for clients with no responses, so it doesn't artificially flatter accounts that just don't engage with support.
- **Treats all five pillars equally.** In reality, with real outcome data showing which clients actually churned, I'd fit pillar weights using a logistic model so the score *predicts* churn rather than just *describing* current state — most likely weighting revenue and usage signals higher.
- **Band thresholds (70/40) are reasonable defaults, not validated.** With real churn data they should be tuned against actual churn rate per band.
- **Dataset is fully synthetic.** Findings illustrate the methodology; they're not real-world claims about any particular vertical.

---

## What's on the dashboard

- **Top-row scorecards** — Total Clients · Avg Health Score · Critical Clients · Total Active MRR
- **Bar chart** — average health score by industry, sorted descending
- **Pie chart** — share of clients in each health band
- **At-risk client table** — every client sorted ascending by health score, color-coded by band, with name, industry, plan tier, score, and MRR. This is the operational chart: the top 20 rows are the CS team's weekly call list.
- **Filter controls** — `plan_tier` and `industry` dropdowns let viewers slice the dashboard live

---

## Skills demonstrated

Data modeling across a multi-table relational schema (accounts, subscriptions, usage, tickets, churn). Advanced Google Sheets formulas including `ARRAYFORMULA`, `SUMIFS`, `COUNTIFS`, `AVERAGEIFS`, `COUNTUNIQUEIFS`, `VLOOKUP`, `PERCENTRANK`, `QUERY`, and nested `IFS`. Dashboard development in Looker Studio — scorecards, bar charts, pie charts, sortable tables, conditional formatting, calculated fields, filter controls, and data source management. KPI design for SaaS metrics (MRR, ARR, churn, CSAT, feature adoption, escalation rate). Cohort-style percentile normalization. Translating raw operational data into executive-ready visualizations and a defensible retention prioritization framework.

---

## How to read this dashboard if you're hiring me

The summary scorecards are for executives — they should be able to land on the page and read the state of the business in 5 seconds. The bar and pie charts are for product and marketing — they describe shape, not action. The bottom table is for the CS team — it's the operational chart that tells them what to do this week. Each chart serves a different reader, which is what separates a useful dashboard from a decorative one.

---

*Dataset credit: RavenStack synthetic SaaS data by River @ Rivalytics. Built as a portfolio project demonstrating end-to-end BI development from raw multi-table data to a stakeholder-ready interactive dashboard.*
