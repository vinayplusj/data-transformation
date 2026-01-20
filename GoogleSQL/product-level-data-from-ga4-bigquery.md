# GA4 Product-Level Table in BigQuery

## Overview

GA4’s raw export in Bigquery is **event-based**, which is powerful but difficult for non-technical teams to use directly. For ecommerce reports, we have transform raw Google Analytics 4 BigQuery export data into a product-level analytics table using GoogleSQL (BigQuery SQL).

Business stakeholders care about this view because it answers questions like:

* How is performance of exch product?
* Which products gained or lost momentum?
* How do promotions or campaigns affect daily conversion?

---

## Key definitions:

* **View-to-cart rate**: How often product views turn into add-to-cart actions
* **Cart-to-purchase rate**: How often carts turn into purchases
* **View-to-purchase rate**: Overall product conversion effectiveness
* **Promotion** and **coupon** fields may be null;especially for non-promoted products

These metrics allow teams to distinguish between:

* Products with strong demand but poor conversion
* Products with low visibility but high efficiency

---

## SQL Query: Product-Level

```sql

WITH base_events AS (

  -- Unnest ecommerce items and extract attribution dimensions
  SELECT
    PARSE_DATE('%Y%m%d', event_date) AS event_day,
    user_pseudo_id,

    -- Session identifier
    (SELECT value.int_value
     FROM UNNEST(event_params)
     WHERE key = 'ga_session_id') AS ga_session_id,

    event_name,

    -- Product fields
    item.item_id AS item_id,
    item.item_name AS item_name,
    item.item_category AS item_category,
    item.item_category2 AS item_category2,
    item.item_category3 AS item_category3,

    -- ecommerce metrics
    item.quantity AS quantity,
    item.revenue AS revenue,

    -- Promotion and coupon attribution
    item.promotion_name AS promotion_name,
    item.promotion_id AS promotion_id,
    item.coupon AS coupon_code,

    -- Device dimensions
    device.category AS device_category,
    device.operating_system AS operating_system,
    device.web_info.browser AS browser,

    -- Geo dimensions
    geo.country AS country,
    geo.region AS region,
    geo.city AS city

  FROM `project_id.dataset_id.events_*`,
       UNNEST(items) AS item

  WHERE _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
    AND event_name IN (
      'view_item',
      'add_to_cart',
      'purchase'
    )
),

product_daily_agg AS (

  -- Aggregate to product
  SELECT
    event_day,

    item_id,
    item_name,
    item_category,
    item_category2,
    item_category3,

    -- Promotion context
    promotion_id,
    promotion_name,
    coupon_code,

    -- Device
    device_category,
    operating_system,
    browser,

    -- Geo
    country,
    region,
    city,

    -- Funnel metrics
    COUNTIF(event_name = 'view_item') AS item_views,
    COUNTIF(event_name = 'add_to_cart') AS adds_to_cart,
    COUNTIF(event_name = 'purchase') AS purchases,

    -- Sales metrics
    SUM(IF(event_name = 'purchase', quantity, 0)) AS quantity_sold,
    SUM(IF(event_name = 'purchase', revenue, 0)) AS revenue,

    -- Reach
    COUNT(DISTINCT user_pseudo_id) AS distinct_users,
    COUNT(DISTINCT ga_session_id) AS distinct_sessions

  FROM base_events
  GROUP BY
    event_day,
    item_id,
    item_name,
    item_category,
    item_category2,
    item_category3,
    promotion_id,
    promotion_name,
    coupon_code,
    device_category,
    operating_system,
    browser,
    country,
    region,
    city
)

-- Final output with conversion rates
SELECT
  event_day,

  item_id,
  item_name,
  item_category,
  item_category2,
  item_category3,

  promotion_id,
  promotion_name,
  coupon_code,

  device_category,
  operating_system,
  browser,

  country,
  region,
  city,

  item_views,
  adds_to_cart,
  purchases,
  quantity_sold,
  revenue,

  -- Conversion rates
  SAFE_DIVIDE(adds_to_cart, item_views) AS view_to_cart_rate,
  SAFE_DIVIDE(purchases, adds_to_cart) AS cart_to_purchase_rate,
  SAFE_DIVIDE(purchases, item_views) AS view_to_purchase_rate,

  distinct_users,
  distinct_sessions

FROM product_daily_agg
ORDER BY event_day, revenue DESC;
```

---

## How Stakeholders Use This Table

* **Merchandising teams** track daily winners and losers
* **Marketing teams** measure the impact of campaigns on product conversion
* **Product teams** identify UX or price issues at the product level
* **Executives** see clear daily revenue and conversion trends

---

## Next Improvements would be

* Join with margin data for profitability analysis
---

## Author

**Vinay Jagannath**
Digital Analytics Consultant
GA4, BigQuery, and Analytics Engineering
