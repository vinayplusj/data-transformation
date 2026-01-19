# GA4 Session-Level Analytics in BigQuery

## Overview
One of the first necessary tasks to use GA4 bigquery exports is to transform **raw Google Analytics 4 BigQuery export data** into a **clean, session-level analytics table** through **GoogleSQL (in BigQuery)**.

The goal of this task is to:

* Unnest the nested and repeated GA4 event data
* Define sessions correctly through GA4 logic
* Create analysis-ready tables for reports and dashboards

This type of session table is commonly used directly for:

* Marketing performance analysis
* Funnel and journey analysis
* Attribution modeling
* Executive dashboards in Tableau, Power BI, or Looker

---

## Data Source

**Source:** Google Analytics 4 BigQuery Export
**Tables:** `events_*`
**Grain:** Event-level (raw)

Each row in the source table represents a single GA4 event with nested parameters.

---

## Session Definition

A session is defined from GA4 best practices:

* **Session key:** `user_pseudo_id + ga_session_id`
* **Session start:** Earliest event timestamp in the session
* **Session end:** Latest event timestamp in the session
* **Session duration:** Difference between end and start timestamps
* **Engaged session:** Based on the `session_engaged` parameter

---

## Final Session-Level Table

Each row in the output table represents **one session**.

### Output fields

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
* `source`
* `medium`
* `campaign`
* `device_category`
* `operating_system`
* `browser`
* `country`
* `region`
* `city`

This table can be directly linked to BI tools or joined to cost and conversion tables in a data warehouse.

---

## SQL Query (GoogleSQL)

```sql
WITH base_events AS (

  -- Extract event-level and session-scoped fields
  SELECT
    event_date,
    user_pseudo_id,

    -- GA4 session identifier
    (SELECT value.int_value
     FROM UNNEST(event_params)
     WHERE key = 'ga_session_id') AS ga_session_id,

    event_timestamp,
    event_name,

    -- Page URL
    (SELECT value.string_value
     FROM UNNEST(event_params)
     WHERE key = 'page_location') AS page_location,

    -- Engagement
    (SELECT value.int_value
     FROM UNNEST(event_params)
     WHERE key = 'engagement_time_msec') AS engagement_time_msec,

    (SELECT value.string_value
     FROM UNNEST(event_params)
     WHERE key = 'session_engaged') AS session_engaged,

    -- Traffic source (session scoped)
    traffic_source.source AS source,
    traffic_source.medium AS medium,
    traffic_source.name AS campaign,

    -- Device dimensions
    device.category AS device_category,
    device.operating_system AS operating_system,
    device.web_info.browser AS browser,

    -- Geo dimensions
    geo.country AS country,
    geo.region AS region,
    geo.city AS city

  FROM `project_id.analytics_XXXXXX.events_*`  -- These would be custom for every Big Query project and GA4 account
  WHERE _TABLE_SUFFIX BETWEEN '20240101' AND '20241231' -- DAte filter as needed
    AND (SELECT value.int_value
         FROM UNNEST(event_params)
         WHERE key = 'ga_session_id') IS NOT NULL
),

session_agg AS (

  -- Aggregate events to the session level
  SELECT
    user_pseudo_id,
    ga_session_id,

    -- Session timing
    MIN(event_timestamp) AS session_start_ts,
    MAX(event_timestamp) AS session_end_ts,

    -- Landing page (first page_view in the session)
    ARRAY_AGG(
      page_location
      IGNORE NULLS
      ORDER BY event_timestamp
    )[SAFE_OFFSET(0)] AS landing_page,

    -- Volume metrics
    COUNT(*) AS event_count,
    COUNTIF(event_name = 'page_view') AS pageviews,

    -- Engagement metrics
    SUM(IFNULL(engagement_time_msec, 0)) AS total_engagement_time_msec,
    MAX(IF(session_engaged = '1', 1, 0)) AS engaged_session,

    -- Attribution
    ANY_VALUE(source) AS source,
    ANY_VALUE(medium) AS medium,
    ANY_VALUE(campaign) AS campaign,

    -- Device
    ANY_VALUE(device_category) AS device_category,
    ANY_VALUE(operating_system) AS operating_system,
    ANY_VALUE(browser) AS browser,

    -- Geo
    ANY_VALUE(country) AS country,
    ANY_VALUE(region) AS region,
    ANY_VALUE(city) AS city

  FROM base_events
  GROUP BY user_pseudo_id, ga_session_id
)

-- Final session-level output
SELECT
  user_pseudo_id,
  ga_session_id,

  TIMESTAMP_MICROS(session_start_ts) AS session_start_time,
  TIMESTAMP_MICROS(session_end_ts) AS session_end_time,

  TIMESTAMP_DIFF(
    TIMESTAMP_MICROS(session_end_ts),
    TIMESTAMP_MICROS(session_start_ts),
    SECOND
  ) AS session_duration_seconds,

  landing_page,

  event_count,
  pageviews,
  total_engagement_time_msec,
  engaged_session,

  source,
  medium,
  campaign,

  device_category,
  operating_system,
  browser,

  country,
  region,
  city

FROM session_agg;
```

---

## How This Would Be Used in Production

1. Schedule this query daily from BigQuery Scheduled Queries
2. Write results into a partitioned `sessions` table
3. Join with:

   * Google Ads or Meta cost data
   * CRM or lead tables
   * Conversion events
4. Visualize results in Tableau, Power BI, or Looker

---

## Next Improvements would be to create a user-level rollup table

---

## Author

**Vinay Jagannath**
Digital Analytics Consultant
GA4, BigQuery, Tableau, and Marketing Analytics
