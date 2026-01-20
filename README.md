# Data Transformation Folder

## Purpose

This repository has data transformation templates from most common tasks faced in projects to build reliable datasets that business teams can use for dashboards, experimentation, marketing performance, and product analysis.

**Clean inputs and consistent logic** decides whether data can be trusted.

---

## Folder Structure

### GoogleSQL (BigQuery)

Templates for transformations on large datasets, especially GA4 exports and analytics models.

👉 [GoogleSQL folder](./GoogleSQL)

### Power Query (M)

Templates for transformations inside Power BI or Excel, including file and API ingestion patterns.

👉 [Power Query (M) folder](./Power%20Query%20%28M%29)

## Design principles for current and future templates

* Defensive logic for schema changes and missing fields
* Clear granularity and metric definitions
* Reusable base layers that reduce duplicated code
* Notes written in plain language
