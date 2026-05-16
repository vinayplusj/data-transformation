# GA4 User-Level Table Using Impala SQL

## Overview

GA4 raw exports are event-based, which is powerful but difficult for non-technical teams to use directly.

After creating a [session-level](session-level-data-from-ga4-impala.md) table from GA4 raw export data, we usually transform the data into a **user-level analytics table**.

In this version, the assumption is that raw Google Analytics 4 BigQuery export data has been copied to another data lake, such as **Azure Data Lake Storage**, and is queried using **Impala SQL**.

Most business questions are **user-centric**, not event-centric.

Executives, marketing leaders, and product owners rarely ask:

> How many events did we have?

They usually ask:

- How many users do we reach?
- Where do our users come from?
- Which users are engaged versus inactive?
- How does user behaviour differ by device, geography, or channel?
- Do new users behave differently from returning users?

A **user-level table** solves this by:

- Creating **one row per user**
- Stabilizing dimensions such as source, device, and geography
- Making engagement and recency easier to analyze
- Enabling joins to CRM, marketing platforms, and BI tools

This table becomes a **shared foundation** for dashboards, segmentation, and decision-making.

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

## User and First Traffic Source Definition

The best method to identify a user is through a durable hashed user identifier in the GA4 events.

Since this example does not assume that a hashed identifier is available, a user is defined as:

- **Primary key:** `user_pseudo_id`
- Represents an anonymous GA4 user
- Stable across sessions unless cookies are reset

This task focuses on **behavioural analytics**, not identity resolution.

In GA4 exports:

- There is no simple, explicit `first_source` or `first_campaign` column for this model.
- Source data appears in `traffic_source.*` and can also appear in event parameters such as source, medium, and campaign.
- For this model, first-touch attribution is based on the earliest known event for the user.

---

## Final User-Level Table

Each row represents **one user**.

### Output fields

- `user_pseudo_id`
- `first_seen_time`
- `last_seen_time`
- `user_lifetime_days`
- `total_sessions`
- `total_events`
- `total_pageviews`
- `total_engagement_time_msec`
- `is_engaged_user`
- `first_source`
- `first_medium`
- `first_campaign`
- `device_category`
- `operating_system`
- `browser`
- `country`
- `region`
- `city`

---

## SQL Query (Impala SQL)

```sql
-- GA4 User-Level Table using Impala SQL
-- Source: nested GA4 BigQuery export copied to Azure Data Lake
-- Grain: one row per user
-- User key: user_pseudo_id

WITH exploded_params AS (

  -- Expand event_params so selected GA4 parameters can be extracted
  SELECT
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
      'engagement_time_msec',
      'session_engaged'
    )
),

pivoted_events AS (

  -- Convert selected GA4 parameters into event-level columns
  SELECT
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
    CAST(MAX(CASE WHEN param_key = 'engagement_time_msec' THEN param_value END) AS BIGINT) AS engagement_time_msec,
    MAX(CASE WHEN param_key = 'session_engaged' THEN param_value END) AS session_engaged

  FROM exploded_params
  GROUP BY
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

first_touch AS (

  -- Select the first non-null traffic source values for each user
  SELECT
    user_pseudo_id,
    source AS first_source,
    medium AS first_medium,
    campaign AS first_campaign
  FROM (
    SELECT
      user_pseudo_id,
      source,
      medium,
      campaign,
      ROW_NUMBER() OVER (
        PARTITION BY user_pseudo_id
        ORDER BY event_timestamp
      ) AS source_rank
    FROM pivoted_events
    WHERE source IS NOT NULL
       OR medium IS NOT NULL
       OR campaign IS NOT NULL
  ) ranked_sources
  WHERE source_rank = 1
),

user_agg AS (

  -- Aggregate events to the user level
  SELECT
    user_pseudo_id,

    -- User lifetime
    MIN(event_timestamp) AS first_seen_ts,
    MAX(event_timestamp) AS last_seen_ts,

    -- Common metrics
    COUNT(DISTINCT ga_session_id) AS total_sessions,
    COUNT(*) AS total_events,

    -- This can be filtered further to extract specific conversion values
    SUM(
      CASE
        WHEN event_name = 'page_view' THEN 1
        ELSE 0
      END
    ) AS total_pageviews,

    -- Engagement
    SUM(COALESCE(engagement_time_msec, 0)) AS total_engagement_time_msec,

    MAX(
      CASE
        WHEN session_engaged = '1' THEN 1
        ELSE 0
      END
    ) AS is_engaged_user,

    -- Other dimensions for deep dives.
    -- These use MAX as a simple rollup rule.
    -- In production, you may prefer first, latest, or most frequent value.
    MAX(device_category) AS device_category,
    MAX(operating_system) AS operating_system,
    MAX(browser) AS browser,

    MAX(country) AS country,
    MAX(region) AS region,
    MAX(city) AS city

  FROM pivoted_events
  GROUP BY
    user_pseudo_id
)

-- Final user-level output
SELECT
  u.user_pseudo_id,

  FROM_UNIXTIME(CAST(u.first_seen_ts / 1000000 AS BIGINT)) AS first_seen_time,
  FROM_UNIXTIME(CAST(u.last_seen_ts / 1000000 AS BIGINT)) AS last_seen_time,

  DATEDIFF(
    CAST(FROM_UNIXTIME(CAST(u.last_seen_ts / 1000000 AS BIGINT)) AS TIMESTAMP),
    CAST(FROM_UNIXTIME(CAST(u.first_seen_ts / 1000000 AS BIGINT)) AS TIMESTAMP)
  ) AS user_lifetime_days,

  u.total_sessions,
  u.total_events,
  u.total_pageviews,
  u.total_engagement_time_msec,
  u.is_engaged_user,

  ft.first_source,
  ft.first_medium,
  ft.first_campaign,

  u.device_category,
  u.operating_system,
  u.browser,

  u.country,
  u.region,
  u.city

FROM user_agg u
LEFT JOIN first_touch ft
  ON u.user_pseudo_id = ft.user_pseudo_id
;
