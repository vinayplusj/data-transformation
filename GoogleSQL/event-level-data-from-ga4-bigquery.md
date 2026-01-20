# GA4 Event- and Pageview-Level Table in BigQuery

### Overview
GA4 raw exports are event-based, but the schema is complex and nested. After we create [session](session-level-data-from-ga4-bigquery.md) and [user level](user-level-data-from-ga4-bigquery.md) tables, the next task typically is to use GA4 bigquery exports is to transform **raw Google Analytics 4 BigQuery export data** into a **clean, session-level analytics table** through **GoogleSQL (in BigQuery)**.

The goal of this task is to:

* Create a consistent row structure for every interaction with a **flattened event- and pageview-level table** 
* Make pageviews and custom events easy to analyze together for all downstream analytics
* Enable detailed journey and funnel analysis from below use cases

---

## Business Use Cases

While executives usually consume aggregated dashboards, many critical business questions; especially at the operational level, require **granular visibility**.

Business stakeholders often need to understand:

* What exactly users do on key pages
* Where users drop off or get confused
* Whether measurement and tags work correctly
* How specific interactions lead to conversions
* Why reported numbers change over time
* Do we measure accurately?
* Can journey funnels be recreated consistently?
* Is there enough information to investigate sudden changes in metrics?

This table enables:

* **Journey analysis**
  Understand how users move page by page before conversion.

* **Funnel diagnostics**
  Identify exactly where users drop out of key flows.

* **Tag validation**
  Confirm that events fire correctly across devices and regions.

* **Marketing and UX investigations**
  Tie specific interactions to campaign or design changes.

* **Advanced analytics**
  Feed clean interaction data into session, user, and product models.

---

## Data Source

**Source:** Google Analytics 4 BigQuery Export
**Tables:** `events_*`
**Grain:** Event-level (raw)

Each row in the source data represents a single GA4 event with nested parameters.

---

## Event and Pageview Definition

In GA4, certain events captured by default. Businesses can send any additional events they want to track along with custom parameters; which provide additional context, to GA4. 
At the same time, many organizations have chosen to use the "category action, label, value structure from Universal Analytics so that users are not confused where their previous metrics went. This model includes:

* All GA4 events along with their *category*, *action*, *label*, *value* classifications
* Explicit pageview events (`page_view`)
* One row per event occurrence* 

Key identifiers:

* `event_timestamp`
* `user_pseudo_id`
* `ga_session_id`

---

## Final Event- and Pageview-Level Table

Each row represents **one user interaction**.

### Output fields

* `event_date`
* `event_timestamp`
* `event_name`
* `user_pseudo_id`
* `ga_session_id`
* `page_location`
* `page_title`
* `event_category`
* `event_label`
* `event_value`
* `device_category`
* `operating_system`
* `browser`
* `country`
* `region`
* `city`
* `source`
* `medium`
* `campaign`

---

## SQL Query (GoogleSQL)

```sql

SELECT
event_date,


-- Event sequence within session
ROW_NUMBER() OVER (
PARTITION BY user_pseudo_id,
(SELECT value.int_value
FROM UNNEST(event_params)
WHERE key = 'ga_session_id')
ORDER BY event_timestamp
) AS event_step_in_session,


TIMESTAMP_MICROS(event_timestamp) AS event_time,
event_name,


-- User and session identifiers
user_pseudo_id,
(SELECT value.int_value
FROM UNNEST(event_params)
WHERE key = 'ga_session_id') AS ga_session_id,


-- Page information
(SELECT value.string_value
FROM UNNEST(event_params)
WHERE key = 'page_location') AS page_location,


(SELECT value.string_value
FROM UNNEST(event_params)
WHERE key = 'page_title') AS page_title,


-- Event parameters (continuing the UA structure)
(SELECT value.string_value
FROM UNNEST(event_params)
WHERE key = 'event_category') AS event_category,


(SELECT value.string_value
FROM UNNEST(event_params)
WHERE key = 'event_action') AS event_action,


(SELECT value.string_value
FROM UNNEST(event_params)
WHERE key = 'event_label') AS event_label,


(SELECT value.int_value
FROM UNNEST(event_params)
WHERE key = 'value') AS event_value,


-- Device dimensions
device.category AS device_category,
device.operating_system AS operating_system,
device.web_info.browser AS browser,


-- Geo dimensions
geo.country AS country,
geo.region AS region,
geo.city AS city,


-- Traffic source (session scoped)
traffic_source.source AS source,
traffic_source.medium AS medium,
traffic_source.name AS campaign


FROM `project_id.dataset_id.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
;
```

---

## How This Would Be Used in Production

1. Schedule this query daily from BigQuery Scheduled Queries
2. Write results into a partitioned `events` table
3. Join with offline data on interactions

---

## Next Improvements would be to

* Extract custom event parameters
* Consent and privacy flags
* Parse and normalize page path
* Calculate time-between-steps

---

## Author

**Vinay Jagannath**
Digital Analytics Consultant
GA4, BigQuery, Tableau, and Marketing Analytics
