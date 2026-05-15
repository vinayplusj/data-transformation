# Session-Level Data from GA4 Using Impala SQL

## What this page shows

This page shows how to create a **session-level table** from GA4-style raw event data using **Impala SQL**.

The assumption is that GA4 raw export data has already been moved out of BigQuery and into a data lake or warehouse layer that can be queried with Impala.

The output table has **one row per session**, using:

```text
user_pseudo_id + ga_session_id
```

as the session key.

---

## Why this matters to business stakeholders

Most business teams do not want to work directly with raw GA4 events. They usually want to understand visits, engagement, traffic quality, and landing page performance.

A session-level table helps answer questions such as:

* How many visits did the site or app receive?
* Which campaigns or channels drove better-quality visits?
* Which landing pages started the most sessions?
* How long did users stay?
* Which sessions were engaged?

This table turns detailed event data into a format that is easier to use in dashboards, scorecards, and regular business reviews.

---

## Source data assumption

Impala SQL does not handle GA4 BigQuery nested data in the same way as GoogleSQL.

For this example, the GA4 data is assumed to have already been prepared into a flatter event table with one row per event.

Example source table:

```text
analytics.ga4_events_flat
```

Expected fields:

| Field                  | Description                     |
| ---------------------- | ------------------------------- |
| `event_date`           | Event date in `yyyyMMdd` format |
| `event_timestamp`      | Event timestamp in microseconds |
| `user_pseudo_id`       | GA4 anonymous user identifier   |
| `ga_session_id`        | GA4 session identifier          |
| `event_name`           | GA4 event name                  |
| `page_location`        | Page URL                        |
| `engagement_time_msec` | Engagement time in milliseconds |
| `session_engaged`      | GA4 engaged session flag        |
| `source`               | Traffic source                  |
| `medium`               | Traffic medium                  |
| `campaign`             | Campaign name                   |
| `device_category`      | Device category                 |
| `operating_system`     | Operating system                |
| `browser`              | Browser                         |
| `country`              | Country                         |
| `region`               | Region                          |
| `city`                 | City                            |

---

## Output table

Each row represents **one session**.

Output fields:

* `session_date`
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

---

## Impala SQL query

```sql
-- GA4 Session-Level Table using Impala SQL
-- Grain: one row per session
-- Session key: user_pseudo_id + ga_session_id

WITH base_events AS (

  SELECT
    event_date,
    user_pseudo_id,
    ga_session_id,
    event_timestamp,
    event_name,
    page_location,
    engagement_time_msec,
    session_engaged,
    source,
    medium,
    campaign,
    device_category,
    operating_system,
    browser,
    country,
    region,
    city

  FROM analytics.ga4_events_flat
  WHERE event_date BETWEEN '20240101' AND '20241231'
    AND ga_session_id IS NOT NULL
),

landing_pages AS (

  -- Identify the first page URL in each session
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
    FROM base_events
    WHERE page_location IS NOT NULL
  ) ranked_pages
  WHERE page_rank = 1
),

session_agg AS (

  SELECT
    user_pseudo_id,
    ga_session_id,

    MIN(event_date) AS session_date,
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

    -- Dimensions assumed to be stable enough at session grain
    MAX(source) AS source,
    MAX(medium) AS medium,
    MAX(campaign) AS campaign,
    MAX(device_category) AS device_category,
    MAX(operating_system) AS operating_system,
    MAX(browser) AS browser,
    MAX(country) AS country,
    MAX(region) AS region,
    MAX(city) AS city

  FROM base_events
  GROUP BY
    user_pseudo_id,
    ga_session_id
)

SELECT
  s.session_date,
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
```

---

## Notes on the SQL logic

### Session key

The session key is:

```text
user_pseudo_id + ga_session_id
```

This is important because `ga_session_id` alone is not globally unique.

### Landing page

The landing page is selected as the first non-null `page_location` within the session, ordered by `event_timestamp`.

This is done with `ROW_NUMBER()` because Impala SQL does not use BigQuery’s `ARRAY_AGG(... ORDER BY ...)[SAFE_OFFSET(0)]` pattern.

### Session duration

GA4 event timestamps are usually stored in microseconds.

The query calculates duration as:

```text
(session_end_ts - session_start_ts) / 1,000,000
```

### Session dimensions

Fields such as source, medium, campaign, device, and geography are rolled up using `MAX()`.

In a production model, you may choose a more precise rule, such as:

* first value in the session
* last value in the session
* most frequent value in the session

The right choice depends on how the business defines attribution and reporting.

---

## What changes from GoogleSQL to Impala SQL

### 1. No wildcard tables

BigQuery often uses:

```sql
FROM `project.dataset.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
```

Impala usually queries a table directly:

```sql
FROM analytics.ga4_events_flat
WHERE event_date BETWEEN '20240101' AND '20241231'
```

### 2. Nested GA4 parameters usually need preparation

GoogleSQL can directly query nested GA4 fields with `UNNEST(event_params)`.

In Impala, the cleanest pattern is often to prepare a flatter event table first, then build analytics models on top of it.

### 3. First-value logic uses window functions

GoogleSQL often uses ordered arrays.

Impala can use:

```sql
ROW_NUMBER() OVER (
  PARTITION BY user_pseudo_id, ga_session_id
  ORDER BY event_timestamp
)
```

This works well for landing page extraction and event sequencing.

### 4. Timestamp conversion is different

GoogleSQL uses `TIMESTAMP_MICROS()`.

Impala commonly uses:

```sql
FROM_UNIXTIME(CAST(event_timestamp / 1000000 AS BIGINT))
```

---

## Business use cases

This session-level table can support:

* Marketing performance dashboards
* Landing page analysis
* Channel quality reporting
* Funnel and journey diagnostics
* Executive traffic summaries

---

## Production considerations

For production use:

* Partition the output table by `session_date`
* Validate session counts against GA4 or downstream benchmarks
* Store this as a managed table or scheduled transformation
* Centralize the flattened GA4 event preparation step
* Document attribution assumptions clearly

---

## Author

**Vinay Jagannath**
