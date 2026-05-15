# GA4 Event- and Pageview-Level Table Using Impala SQL

## Overview

GA4 raw exports are event-based, but the schema is complex and nested.

In this version, the assumption is that **raw Google Analytics 4 BigQuery export data** has been copied to another data lake, such as **Azure Data Lake Storage**, and is queried using **Impala SQL**.

The goal of this task is to:

- Create a consistent row structure for every interaction with a flattened event- and pageview-level table
- Make pageviews and custom events easy to analyze together
- Enable detailed journey and funnel analysis
- Support downstream session, user, and product models
- Make GA4 raw data easier to use outside BigQuery

---

## Business Use Cases

While executives usually consume aggregated dashboards, many critical business questions require granular visibility.

Business stakeholders often need to understand:

- What exactly users do on key pages
- Where users drop off or get confused
- Whether measurement and tags work correctly
- How specific interactions lead to conversions
- Why reported numbers change over time
- Whether journeys and funnels can be recreated consistently
- Whether there is enough detail to investigate sudden metric changes

This table enables:

- **Journey analysis**  
  Understand how users move page by page before conversion.

- **Funnel diagnostics**  
  Identify exactly where users drop out of key flows.

- **Tag validation**  
  Confirm that events fire correctly across devices and regions.

- **Marketing and UX investigations**  
  Tie specific interactions to campaign or design changes.

- **Advanced analytics**  
  Feed clean interaction data into session, user, and product models.

---

## Data Source

**Original source:** Google Analytics 4 BigQuery Export  
**Storage location:** Azure Data Lake or another data lake  
**Query engine:** Impala SQL  
**Example external table:** `analytics.ga4_events_nested`  
**Grain:** Event-level raw data  

Each row in the source table represents a single GA4 event with nested parameters.

The source is assumed to retain the GA4 export structure, including:

- `event_params` as an array of key-value structs
- `device` as a struct
- `geo` as a struct
- `traffic_source` as a struct

---

## Event and Pageview Definition

In GA4, some events are collected automatically. Businesses can also send additional custom events and custom parameters to describe important interactions.

Many organizations also keep a Universal Analytics-style structure so teams can continue to recognize familiar event fields:

- `event_category`
- `event_action`
- `event_label`
- `event_value`

This model includes:

- All GA4 events with selected custom parameters
- Explicit pageview events (`page_view`)
- One row per event occurrence

Key identifiers:

- `event_timestamp`
- `user_pseudo_id`
- `ga_session_id`

---

## Final Event- and Pageview-Level Table

Each row represents **one user interaction**.

### Output fields

- `event_date`
- `event_time`
- `event_step_in_session`
- `event_name`
- `user_pseudo_id`
- `ga_session_id`
- `page_location`
- `page_title`
- `event_category`
- `event_action`
- `event_label`
- `event_value`
- `device_category`
- `operating_system`
- `browser`
- `country`
- `region`
- `city`
- `source`
- `medium`
- `campaign`

---

## SQL Query (Impala SQL)

```sql
-- GA4 Event- and Pageview-Level Table using Impala SQL
-- Source: nested GA4 BigQuery export copied to Azure Data Lake
-- Grain: one row per event

WITH exploded_params AS (

  -- Expand event_params so selected GA4 parameters can be extracted
  SELECT
    e.event_date,
    e.user_pseudo_id,
    e.event_timestamp,
    e.event_name,

    -- Device dimensions from GA4 struct fields
    e.device.category AS device_category,
    e.device.operating_system AS operating_system,
    e.device.web_info.browser AS browser,

    -- Geo dimensions from GA4 struct fields
    e.geo.country AS country,
    e.geo.region AS region,
    e.geo.city AS city,

    -- Traffic source fields
    e.traffic_source.source AS source,
    e.traffic_source.medium AS medium,
    e.traffic_source.name AS campaign,

    -- event_params is an array of structs.
    -- In Impala, array elements are commonly accessed through the item field.
    ep.item.key AS param_key,

    -- Normalize GA4 parameter values into one string column.
    -- This allows selected parameters to be pivoted with conditional aggregation.
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
      'page_title',
      'event_category',
      'event_action',
      'event_label',
      'value'
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
    MAX(CASE WHEN param_key = 'page_title' THEN param_value END) AS page_title,

    MAX(CASE WHEN param_key = 'event_category' THEN param_value END) AS event_category,
    MAX(CASE WHEN param_key = 'event_action' THEN param_value END) AS event_action,
    MAX(CASE WHEN param_key = 'event_label' THEN param_value END) AS event_label,
    CAST(MAX(CASE WHEN param_key = 'value' THEN param_value END) AS BIGINT) AS event_value

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

sequenced_events AS (

  -- Assign event order inside each session
  SELECT
    event_date,

    FROM_UNIXTIME(CAST(event_timestamp / 1000000 AS BIGINT)) AS event_time,

    ROW_NUMBER() OVER (
      PARTITION BY user_pseudo_id, ga_session_id
      ORDER BY event_timestamp
    ) AS event_step_in_session,

    event_name,

    user_pseudo_id,
    ga_session_id,

    page_location,
    page_title,

    event_category,
    event_action,
    event_label,
    event_value,

    device_category,
    operating_system,
    browser,

    country,
    region,
    city,

    source,
    medium,
    campaign

  FROM pivoted_events
  WHERE ga_session_id IS NOT NULL
)

SELECT
  event_date,
  event_time,
  event_step_in_session,
  event_name,

  user_pseudo_id,
  ga_session_id,

  page_location,
  page_title,

  event_category,
  event_action,
  event_label,
  event_value,

  device_category,
  operating_system,
  browser,

  country,
  region,
  city,

  source,
  medium,
  campaign

FROM sequenced_events
;
