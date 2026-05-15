# GA4 Data Transformation with Impala SQL

GA4 raw export data is often used outside the GA4 interface so teams can create more flexible and reliable reporting tables.

In this folder, the assumption is that GA4 raw data has already been copied from BigQuery into a data lake or warehouse layer where it can be queried with Impala SQL.

The goal is to transform raw GA4-style event data into flatter tables that are easier to use for dashboards, analysis, and business decision-making.

---

## Why this matters

GA4 raw data is powerful, but it is not ready for most business reporting.

It is usually:

- Event-level
- Nested or semi-structured
- Hard to join to business data
- Difficult for non-technical teams to use directly

These Impala SQL examples show how to create cleaner tables for:

- Sessions
- Users
- Events and pageviews
- Products

These tables help business teams answer questions about traffic quality, customer behaviour, funnels, product performance, and marketing effectiveness.

---

## Folder Structure

This folder has four Impala SQL examples.

### 1. Session-Level Query

Use this to answer:

- How long do sessions last?
- Which channels drive engaged visits?
- What is the landing page performance?

[Session-Level](./session-level-data-from-ga4-impala.md)

---

### 2. User-Level Query

Use this to answer:

- Where do users originally come from?
- How engaged are different user groups?
- How long do users stay active?

[User-Level](./user-level-data-from-ga4-impala.md)

---

### 3. Event and Pageview-Level Query

Use this to answer:

- What exactly did users do?
- In what order did events occur?
- Where are users dropping off?

[Event and Pageview-Level](./event-level-data-from-ga4-impala.md)

---

### 4. Product-Level Query

Use this to answer:

- Which products drive revenue?
- Which products convert poorly?
- How do promotions affect performance?

[Product-Level](./product-level-data-from-ga4-impala.md)

---

## How these queries work together

These queries create a simple analytics modelling stack:

1. Event and pageview table → full interaction history
2. Session table → visits and engagement
3. User table → behaviour and growth
4. Product table → commercial performance

---

## Important note on Impala SQL

Impala SQL is different from GoogleSQL.

Common changes include:

- No BigQuery wildcard tables such as `events_*`
- No `_TABLE_SUFFIX`
- No `UNNEST(event_params)` in the BigQuery style
- Different timestamp functions
- Different array and struct handling depending on how the data was landed

In practice, the best Impala SQL pattern depends on how GA4 raw data was flattened before being made available to Impala.

This folder assumes that GA4 parameters have either been flattened into columns or stored in a table that can be joined by `event_id`, `event_name`, `user_pseudo_id`, and `ga_session_id`.

---

## Author

**Vinay Jagannath**  
Digital Analytics and Data Transformation Consultant  
Focus areas: GA4, BigQuery, Impala SQL, Power Query, dashboards, and experimentation
