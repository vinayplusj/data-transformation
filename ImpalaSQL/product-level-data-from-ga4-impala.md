# GA4 Product-Level Table Using Impala SQL

## Overview

GA4 raw exports are event-based, which is powerful but difficult for non-technical teams to use directly.

For ecommerce reporting, the raw Google Analytics 4 BigQuery export data can be transformed into a **product-level analytics table** using **Impala SQL**.

In this version, the assumption is that GA4 BigQuery export data has been copied to another data lake, such as **Azure Data Lake Storage**, and is queried through Impala.

The table has a daily product grain, with optional device, geo, promotion, and coupon splits.

---

## Why business stakeholders care

Business stakeholders care about this view because it answers questions like:

- How is each product performing?
- Which products gained or lost momentum?
- Which products receive interest but do not convert?
- How do promotions or campaigns affect daily conversion?
- Are there device or regional differences in product performance?

This table helps teams move from raw ecommerce events to a clear business view of product performance.

---

## Key definitions

- **View-to-cart rate**: How often product views turn into add-to-cart actions
- **Cart-to-purchase rate**: How often carts turn into purchases
- **View-to-purchase rate**: Overall product conversion effectiveness
- **Promotion and coupon fields**: May be null, especially for non-promoted products

These metrics allow teams to distinguish between:

- Products with strong demand but poor conversion
- Products with low visibility but high efficiency
- Products where promotion drives interest but not purchase
- Products where conversion differs by device or region

---

## Data Source

**Original source:** Google Analytics 4 BigQuery Export  
**Storage location:** Azure Data Lake or another data lake  
**Query engine:** Impala SQL  
**Example external table:** `analytics.ga4_events_nested`  
**Grain:** Event-level raw data with nested ecommerce `items`

Each row in the source table represents a single GA4 event.

The source is assumed to retain the GA4 export structure, including:

- `event_params` as an array of key-value structs
- `items` as an array of ecommerce item structs
- `device` as a struct
- `geo` as a struct

---

## Product-level grain

Each row in the output table represents:

```text
event_day + product + promotion/coupon + device + geo
