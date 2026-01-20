# *WIP* GA4 Raw Data Transformation Folder (Databricks SQL) *WIP*

## Overview

This repository contains a set of **Databricks SQL** templates that transform **GA4 raw export data** (landed as Delta tables) into **business-ready analytics tables**.

Templates for session, user, and product views that are easier to use in dashboards and decisions.

---

## WIP: the four templates

---

### 1. Session Template (Behavior)

**Purpose:** One row per session with `user_pseudo_id + ga_session_id`, with landing page and engagement.

**Used for:** Marketing performance, traffic quality, landing page analysis, and executive reports.

👉 [Session Template (Databricks SQL)](./ga4-session-databricks-sql/README.md)

---

## Architecture overview

```
GA4 Raw Export (landed as Delta in Databricks)
                |
                v
+----------------------------------+
| Event and Pageview Template          |
| (flattened interactions)          |
+----------------------------------+
        |                   |
        v                   v
+------------------+   +------------------+
| Session Template    |   | Product Template    |
| (visit behavior) |   | (commerce KPIs)  |
+------------------+   +------------------+
        |
        v
+------------------+
| User Template       |
| (growth view)    |
+------------------+
        |
        v
Dashboards / BI / Activation
(Tableau, Power BI, Looker, CRM)
```

---

## Important note on GA4 schema changes

Google periodically changes and extends the GA4 export schema. In Databricks, ingestion settings can also introduce schema drift.

These projects use defensive patterns to keep outputs stable:

* Extract parameters safely from `event_params`
* Expect missing fields and tolerate nulls
* Use safe logic for arrays, sorts, and castsg
* Centralize common extraction logic to reduce duplication

This reduces risk and improves trust in reports.

---

## Databricks SQL versus BigQuery GoogleSQL

These templates assume GA4 data is queried in Databricks SQL (Spark SQL dialect). Common differences versus BigQuery GoogleSQL:

* Delta tables and partitions instead of `events_*` wildcard tables
* `explode()` and higher-order functions instead of `UNNEST()`
* Different timestamp functions and date parses
* Different approaches for “first value” logic (landing page)

Each project page calls out the key syntax differences.

---

## Author

**Vinay Jagannath**
Digital Analytics and Data Transformation 
