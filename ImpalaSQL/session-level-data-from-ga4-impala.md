# GA4 Session-Level Table Using Impala SQL

## Overview

One of the first necessary tasks when using GA4 raw export data outside BigQuery is to transform **raw Google Analytics 4 BigQuery export data** into a **clean, session-level analytics table**.

In this version, the assumption is that GA4 BigQuery export data has been copied to another data lake, such as **Azure Data Lake Storage**, and is queried using **Impala SQL**.

The goal of this task is to:

- Work with nested and repeated GA4 event data after it has been moved out of BigQuery
- Define sessions correctly using GA4 logic
- Create analysis-ready tables for reports and dashboards
- Make GA4 data easier to use in an enterprise data lake environment

This type of session table is commonly used directly for:

- Marketing performance analysis
- Funnel and journey analysis
- Attribution modelling
- Landing page analysis
- Executive dashboards in Tableau, Power BI, or Looker

---

## Data Source

**Original source:** Google Analytics 4 BigQuery Export  
**Storage location:** Azure Data Lake or another data lake  
**Query engine:** Impala SQL  
**Example external table:** `analytics.ga4_events_nested`  
**Grain:** Event-level raw data  

Each row in the source table represents a single GA4 event with nested fields.

The source is assumed to retain the GA4 export structure, including:

- `event_params` as an array of key-value structs
- `device` as a struct
- `geo` as a struct
- `traffic_source` as a struct

---

## Session Definition

A session is defined from GA4 best practices:

- **Session key:** `user_pseudo_id + ga_session_id`
- **Session start:** Earliest event timestamp in the session
- **Session end:** Latest event timestamp in the session
- **Session duration:** Difference between end and start timestamps
- **Landing page:** First non-null `page_location` in the session
- **Engaged session:** Based on the `session_engaged` parameter

The `ga_session_id` value is extracted from `event_params`.

---

## Final Session-Level Table

Each row in the output table represents **one session**.

### Output fields

- `user_pseudo_id`
- `ga_session_id`
- `session_start_time`
- `session_end_time`
- `session_duration_seconds`
- `landing_page`
- `event_count`
- `pageviews`
- `total_engagement_time_msec`
- `engaged_session`
- `source`
- `medium`
- `campaign`
- `device_category`
- `operating_system`
- `browser`
- `country`
- `region`
- `city`

This table can be linked to BI tools or joined to cost, campaign, CRM, or conversion tables in a warehouse or lakehouse environment.

---

## SQL Query (Impala SQL)

```sql
-- GA4 Session-Level Table using Impala SQL
-- Source: nested GA4 BigQuery export copied to Azure Data Lake
-- Grain: one row per session
-- Session key: user_pseudo_id + ga_session_id

WITH exploded_params AS (

  -- Expand event_params so selected GA4 parameters can be extracted
  SELECT
    e.event_date,
    e.user_pseudo_id,
    e.event_timestamp,
    e.event_name,

    -- Struct fields from the GA4 export
    e.device.category AS device_category,
    e.device.operating_system AS operating_system,
    e.device.web_info.browser AS browser,

    e.geo.country AS country,
    e.geo.region AS region,
    e.geo.city AS city,

    e.traffic_source.source AS source,
    e.traffic_source.medium AS medium,
    e.traffic_source.name AS campaign,

    -- event_params is an array of structs.
    -- In Impala, array elements are commonly accessed through the item field.
    ep.item.key AS param_key,

    COALESCE(
      ep.item.value.string_value,
      CAST(ep.item.value.int_value AS STRING),
      CAST(ep.item.value.double_value AS STRING),
      CAST(ep.item.value.float_value AS STRING)
    ) AS param_value

  FROM analytics.ga4_events_nested e,
       e.event_params ep

  WHERE e.event_date BETWEEN '20240101' AND '20241231'
    AND ep.item.key IN (
      'ga_session_id',
      'page_location',
      'engagement_time_msec',
      'session_engaged'
    )
),

pivoted_events AS (

  -- Convert selected GA4 parameters into event-level columns
  SELECT
    event_date,
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

    CAST(MAX(CASE WHEN param_key = 'ga_session_id' THEN param_value END) AS BIGINT) AS ga_session_id,
    MAX(CASE WHEN param_key = 'page_location' THEN param_value END) AS page_location,
    CAST(MAX(CASE WHEN param_key = 'engagement_time_msec' THEN param_value END) AS BIGINT) AS engagement_time_msec,
    MAX(CASE WHEN param_key = 'session_engaged' THEN param_value END) AS session_engaged

  FROM exploded_params
  GROUP BY
    event_date,
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
    campaign
),

landing_pages AS (

  -- Identify the first non-null page URL in each session
  SELECT
    user_pseudo_id,
    ga_session_id,
    page_location AS landing_page
  FROM (
    SELECT
      user_pseudo_id,
      ga_session_id,
      page_location,
      ROW_NUMBER() OVER (
        PARTITION BY user_pseudo_id, ga_session_id
        ORDER BY event_timestamp
      ) AS page_rank
    FROM pivoted_events
    WHERE ga_session_id IS NOT NULL
      AND page_location IS NOT NULL
  ) ranked_pages
  WHERE page_rank = 1
),

session_agg AS (

  -- Aggregate event rows to session level
  SELECT
    user_pseudo_id,
    ga_session_id,

    MIN(event_timestamp) AS session_start_ts,
    MAX(event_timestamp) AS session_end_ts,

    COUNT(*) AS event_count,

    SUM(
      CASE
        WHEN event_name = 'page_view' THEN 1
        ELSE 0
      END
    ) AS pageviews,

    SUM(COALESCE(engagement_time_msec, 0)) AS total_engagement_time_msec,

    MAX(
      CASE
        WHEN session_engaged = '1' THEN 1
        ELSE 0
      END
    ) AS engaged_session,

    -- Dimensions assumed stable enough at session grain
    MAX(source) AS source,
    MAX(medium) AS medium,
    MAX(campaign) AS campaign,

    MAX(device_category) AS device_category,
    MAX(operating_system) AS operating_system,
    MAX(browser) AS browser,

    MAX(country) AS country,
    MAX(region) AS region,
    MAX(city) AS city

  FROM pivoted_events
  WHERE ga_session_id IS NOT NULL
  GROUP BY
    user_pseudo_id,
    ga_session_id
)

-- Final session-level output
SELECT
  s.user_pseudo_id,
  s.ga_session_id,

  FROM_UNIXTIME(CAST(s.session_start_ts / 1000000 AS BIGINT)) AS session_start_time,
  FROM_UNIXTIME(CAST(s.session_end_ts / 1000000 AS BIGINT)) AS session_end_time,

  CAST((s.session_end_ts - s.session_start_ts) / 1000000 AS BIGINT) AS session_duration_seconds,

  lp.landing_page,

  s.event_count,
  s.pageviews,
  s.total_engagement_time_msec,
  s.engaged_session,

  s.source,
  s.medium,
  s.campaign,

  s.device_category,
  s.operating_system,
  s.browser,

  s.country,
  s.region,
  s.city

FROM session_agg s
LEFT JOIN landing_pages lp
  ON s.user_pseudo_id = lp.user_pseudo_id
 AND s.ga_session_id = lp.ga_session_id
;
