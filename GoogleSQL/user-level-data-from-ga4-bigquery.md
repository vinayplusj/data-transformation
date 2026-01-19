# GA4 User-Level Table in BigQuery

## Overview

GA4’s raw export in Bigquery is **event-based**, which is powerful but difficult for non-technical teams to use directly. 
After creating session level table from GA4 biogquery tables, we usually transform **raw Google Analytics 4 BigQuery export data** into a **user-level analytics table** through GoogleSQL (in BigQuery).
Most business questions are **user-centric**, not event-centric.

Executives, especially marketing leaders, and product owners rarely ask “How many logins/ form starts did we have?”

They usually ask:

* How many users do we reach?
* Where do our users come from?
* Which users are engaged versus inactive?
* How does user behavior differ by device, geography, or channel?
* Do new users behave differently from returning users?

A **user-level table** solves this by:

* Create **one row per user**
* Stabilize dimensions such as source, device, and geography
* Make engagement and recency easy to analyze
* Enable joins to CRM, marketing platforms, and BI tools

This table becomes a **shared foundation** for dashboards, segmentation, and decision.

The goal of this task is to:

* Unnest the nested and repeated GA4 event data
* Define users correctly through GA4 logic
* Provide a **single, stable user view** that business teams can trust and reuse across reports, analysis, and activation use cases.

---

## Data Source

**Source:** Google Analytics 4 BigQuery Export
**Tables:** `events_*`
**Grain:** Event-level (raw)

Each row in the source data represents a single GA4 event with nested parameters.

---

## User and first traffic source Definition

The best method to identify user is through hashed user identifiers in the GA4 events. Since we dont have that for current example, a user is defined as:

* **Primary key:** `user_pseudo_id`
* Represents an anonymous GA4 user
* Stable across sessions unless cookies are reset

This task focuses on **behavioral analytics**, not identity resolution.

In GA4 BigQuery exports:

* There is no explicit first_source or first_campaign column

* Source data appears as part of traffic source traffic_source.* (user acquisition context) and event parameters such as source, medium, campaign

So, we take the earliest known event for the user as per attribution rules.

---

## Final User-Level Table

Each row represents **one user**.

### Output fields

* `user_pseudo_id`
* `first_seen_time`
* `last_seen_time`
* `user_lifetime_days`
* `total_sessions`
* `total_events`
* `total_pageviews`
* `total_engagement_time_msec`
* `is_engaged_user`
* `first_source`
* `first_medium`
* `first_campaign`
* `device_category`
* `operating_system`
* `browser`
* `country`
* `region`
* `city`

---

## SQL Query (GoogleSQL)

```sql
WITH base_events AS (

  -- Extract relevant fields from raw GA4 events
  SELECT
    user_pseudo_id,
    event_timestamp,
    event_name,

    -- Session identifier
    (SELECT value.int_value
     FROM UNNEST(event_params)
     WHERE key = 'ga_session_id') AS ga_session_id,

    -- Engagement
    (SELECT value.int_value
     FROM UNNEST(event_params)
     WHERE key = 'engagement_time_msec') AS engagement_time_msec,

    (SELECT value.string_value
     FROM UNNEST(event_params)
     WHERE key = 'session_engaged') AS session_engaged,

    -- Traffic source medium categories
    traffic_source.source AS source,
    traffic_source.medium AS medium,
    traffic_source.name AS campaign,

    -- Device categories
    device.category AS device_category,
    device.operating_system AS operating_system,
    device.web_info.browser AS browser,

    -- Geo categories
    geo.country AS country,
    geo.region AS region,
    geo.city AS city

  FROM `project_id.dataset_id.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
),

user_agg AS (

  -- Aggregate events to the user level
  SELECT
    user_pseudo_id,

    -- User age
    MIN(event_timestamp) AS first_seen_ts,
    MAX(event_timestamp) AS last_seen_ts,

    -- Common metrics
    COUNT(DISTINCT ga_session_id) AS total_sessions,
    COUNT(*) AS total_events,  -- This can be filtered to extract conversion values
    COUNTIF(event_name = 'page_view') AS total_pageviews,

    -- Engagement
    SUM(IFNULL(engagement_time_msec, 0)) AS total_engagement_time_msec,
    MAX(IF(session_engaged = '1', 1, 0)) AS is_engaged_user,

    -- First-touch attribution
    ARRAY_AGG(source IGNORE NULLS ORDER BY event_timestamp)[SAFE_OFFSET(0)] AS first_source,
    ARRAY_AGG(medium IGNORE NULLS ORDER BY event_timestamp)[SAFE_OFFSET(0)] AS first_medium,
    ARRAY_AGG(campaign IGNORE NULLS ORDER BY event_timestamp)[SAFE_OFFSET(0)] AS first_campaign,

    -- Other dimensions for deep dives
    ANY_VALUE(device_category) AS device_category,
    ANY_VALUE(operating_system) AS operating_system,
    ANY_VALUE(browser) AS browser,
    ANY_VALUE(country) AS country,
    ANY_VALUE(region) AS region,
    ANY_VALUE(city) AS city

  FROM base_events
  GROUP BY user_pseudo_id
)

-- Final user-level output
SELECT
  user_pseudo_id,

  TIMESTAMP_MICROS(first_seen_ts) AS first_seen_time,
  TIMESTAMP_MICROS(last_seen_ts) AS last_seen_time,

  DATE_DIFF(
    DATE(TIMESTAMP_MICROS(last_seen_ts)),
    DATE(TIMESTAMP_MICROS(first_seen_ts)),
    DAY
  ) AS user_lifetime_days,

  total_sessions,
  total_events,
  total_pageviews,
  total_engagement_time_msec,
  is_engaged_user,

  first_source,
  first_medium,
  first_campaign,

  device_category,
  operating_system,
  browser,

  country,
  region,
  city

FROM user_agg;
```

---

## How This Would Be Used in Practice

1. Schedule this query daily from BigQuery Scheduled Queries

2. Write results into a partitioned users table

3. Join with:

   * CRM user tables
   * Paid media cost data
   * Email or personalization platforms
4. Use it as the base for executive dashboards Tableau, Power BI, or Looker

A user-level table enables:

* **Marketing plans**
  Understand where high-value users come from and which channels attract engaged users.

* **Product and UX deep-dives**
  Identify device and platform differences in user behavior.

* **Growth reports*
  Track new versus returning users over time.

* **Executive dashboards**
  Provide simple, trusted user counts without repeated SQL logic.

* **Activation and personalization plans and activities**
  Join GA4 users to CRM or email platforms for targetted reachout.

---

## Next Improvements would be

* Conversion summaries per user
* Filter out based on consent and privacy choices and policies
* Create cohort-ready snapshots

---

## Author

**Vinay Jagannath**
Digital Analytics Consultant
GA4, BigQuery, and Analytics Engineering
