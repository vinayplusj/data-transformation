# GA4 Session-Level Table in Databricks

## Overview

One of the first necessary tasks to use GA4 bigquery exports is to transform **raw Google Analytics 4 BigQuery export data** into a **clean, session-level** table. 
GA4 raw data is event-level and nested. A session table turns that complexity into a format that:

* Matches how stakeholders talk about traffic
* Makes engagement and visit quality easy to measure
* Reduces duplicated SQL across reports
* Improves trust with consistent session definitions

The output creates **one row per session** (per user visit) and is designed for marketing performance reports, engagement analysis, and executive dashboards. 
This answers common stakeholder questions like:

* How many visits did we get last week?
* Which channels drive high-quality sessions?
* Did users stayg longer after a site change?
* Which landing pages lead to engaged visits?
* Did engagement improve over time?

The biggest challenge to adopt BigQuery for data transformation is that the enterprise has to be on Google Cloud. Databricks and Snowflake seem to eb gaining popularity for databases. 
With its Spark backbone, Databricks excels in machine learning workflows. Databricks supports Python, R, Scala, and SQL; so the teams get flexibility. 
Below template is the Databricks SQL version to create a [GA4 session-level table](GoogleSQL/session-level-data-from-ga4-bigquery.md) from raw data that was landed from **BigQuery** into **Databricks Delta tables**.

---

## What changes when we move from BigQuery GoogleSQL to Databricks SQL

### 1. Wildcard tables vs Delta tables

* **BigQuery** queries `events_*` and filters dates through `_TABLE_SUFFIX`.
* **Databricks** typically queries a Delta table (or view) and filters through a column such as `event_date` or a derived `event_day`.

### 2. Parameter extraction syntax

* **BigQuery:** extracts parameters with `UNNEST(event_params)` through a scalar subquery per key.
* **Databricks:** commonly extracts parameters by exploded arrays (for example,`LATERAL VIEW explode(event_params)` on arrays.

### 3. Landing page logic

* **BigQuery** often uses `ARRAY_AGG(... ORDER BY ...)[SAFE_OFFSET(0)]` to select the first value.
* **Databricks** typically collects and orders values (for example, `collect_list` + sort), then selects the first element.
* 
### 4. Timestamp and duration functions

* **BigQuery:** uses `TIMESTAMP_MICROS()` and `TIMESTAMP_DIFF()`.
* **Databricks:** commonly uses `timestamp_micros()` and computes duration through arithmetic on microsecond integers.
* 
---

## Data source and assumptions

* GA4 export data is copied from BigQuery and stored in Databricks as **Delta**.
* The table keeps nested arrays such as `event_params`.
* A session is defined from GA4 best practice:

  * **Session key:** `user_pseudo_id + ga_session_id`
  * **Start time:** earliest event timestamp in the session
  * **End time:** latest event timestamp in the session

---

## Output table

One row per session, with these fields:

* `session_day`
* `user_pseudo_id`
* `ga_session_id`
* `session_start_time`
* `session_end_time`
* `session_duration_seconds`
* `landing_page`
* `event_count`
* `pageviews`
* `total_engagement_time_msec`
* `engaged_session`
* `device_category`, `operating_system`, `browser`
* `country`, `region`, `city`
* `source`, `medium`, `campaign`

---

## Databricks SQL query

> Replace `catalog.schema.ga4_events` with your Delta table or view.

```sql
-- GA4 Session-Level Table (Databricks SQL)
-- Uses PIVOT to extract key event_params
-- Grain: one row per session (user_pseudo_id + ga_session_id)

WITH exploded_params AS (

  -- Explode event_params into (param_key, param_value) rows
  SELECT
    event_date,
    to_date(event_date, 'yyyyMMdd') AS event_day,
    user_pseudo_id,
    event_timestamp,
    event_name,

    -- Dimensions (struct columns in GA4 export)
    device.category AS device_category,
    device.operating_system AS operating_system,
    device.web_info.browser AS browser,

    geo.country AS country,
    geo.region AS region,
    geo.city AS city,

    traffic_source.source AS source,
    traffic_source.medium AS medium,
    traffic_source.name AS campaign,

    ep.key AS param_key,

    -- Normalize GA4 param values into a single string for pivot
    coalesce(
      ep.value.string_value,
      cast(ep.value.int_value AS string),
      cast(ep.value.double_value AS string),
      cast(ep.value.float_value AS string)
    ) AS param_value

  FROM catalog.schema.ga4_events
  LATERAL VIEW explode(event_params) AS ep

  WHERE event_date BETWEEN '20240101' AND '20241231'
    AND ep.key IN ('ga_session_id', 'page_location', 'engagement_time_msec', 'session_engaged')
),

pivoted_events AS (

  -- Pivot selected params into columns per event
  SELECT
    event_date,
    event_day,
    user_pseudo_id,
    event_timestamp,
    event_name,

    device_category,
    operating_system,
    browser,

    country,
    region,
    city,

    source,
    medium,
    campaign,

    -- Pivoted params become columns (as strings), then cast as needed
    cast(ga_session_id AS bigint) AS ga_session_id,
    page_location AS page_location,
    cast(engagement_time_msec AS bigint) AS engagement_time_msec,
    session_engaged AS session_engaged

  FROM exploded_params
  PIVOT (
    max(param_value) FOR param_key IN (
      'ga_session_id',
      'page_location',
      'engagement_time_msec',
      'session_engaged'
    )
  )
),

session_agg AS (

  -- Aggregate events to the session level
  SELECT
    user_pseudo_id,
    ga_session_id,

    min(event_day) AS session_day,

    min(event_timestamp) AS session_start_ts,
    max(event_timestamp) AS session_end_ts,

    -- Landing page: first non-null page_location by event time
    element_at(
      array_sort(
        collect_list(
          named_struct(
            'ts', event_timestamp,
            'url', page_location
          )
        ),
        (l, r) -> case
          when l.ts < r.ts then -1
          when l.ts > r.ts then 1
          else 0
        end
      ),
      1
    ).url AS landing_page,

    count(1) AS event_count,
    sum(case when event_name = 'page_view' then 1 else 0 end) AS pageviews,

    sum(coalesce(engagement_time_msec, 0)) AS total_engagement_time_msec,
    max(case when session_engaged = '1' then 1 else 0 end) AS engaged_session,

    -- Dimensions assumed stable within a session
    any_value(device_category) AS device_category,
    any_value(operating_system) AS operating_system,
    any_value(browser) AS browser,

    any_value(country) AS country,
    any_value(region) AS region,
    any_value(city) AS city,

    any_value(source) AS source,
    any_value(medium) AS medium,
    any_value(campaign) AS campaign

  FROM pivoted_events
  WHERE ga_session_id IS NOT NULL
  GROUP BY user_pseudo_id, ga_session_id
)

SELECT
  session_day,
  user_pseudo_id,
  ga_session_id,

  timestamp_micros(session_start_ts) AS session_start_time,
  timestamp_micros(session_end_ts) AS session_end_time,

  cast((session_end_ts - session_start_ts) / 1000000 as bigint) AS session_duration_seconds,

  landing_page,

  event_count,
  pageviews,
  total_engagement_time_msec,
  engaged_session,

  device_category,
  operating_system,
  browser,

  country,
  region,
  city,

  source,
  medium,
  campaign

FROM session_agg;

```



---

## Notes on schema changes and reliability

Google periodically changes and extends the GA4 export schema. In Databricks, ingestion settings can also affect schema drift.

Recommended practices:

* Centralize parameter extraction in a base view.
* Expect nulls for optional parameters.
* Partition session tables by `session_day`.
* Validate key metrics after GA4 or pipeline updates.

---

## Author

**Vinay Jagannath**
Digital Analytics and Data Transformation Consultant
