# GA4 Data transformation through BigQuery + GoogleSQL

Since October 2020, when Google released Google Analytics 4, under the acronym of GA4, Google has [promoted the Google Analytics BigQuery integration](https://support.google.com/analytics/answer/9358801?hl=en&ref_topic=9359001&sjid=1763140874384498038-NC). Unlike the previous version of Universal Analytics, GA4's UI has more limits (sampling, row limits, fixed schemas, 14 month retention in reports), while [BigQuery](https://en.wikipedia.org/wiki/BigQuery) stores the full raw event stream that users can query however you want with SQL. This allows users to work with the raw, unsampled event data outside the limits of the GA4 interface. Also, companies can now join GA4 events to CRM, ad platforms, or backend data (e.g., orders, refunds) for full funnel and quality revenue analyses.

The downside is that GA4 data in BigQuery is raw and [denormalized](https://en.wikipedia.org/wiki/Denormalization) with nested rows, so one does not get clean flat tables ready for deep dives. BigQuery uses GoogleSQL as the query language. So, value generation from GA4 data in BigQuery usually requires SQL knowledge or work with someone who does. The first task then is to create one or more of session, user, event and product level tables. 

---

## Folder Structure

The folder has **four GoogleSQL queries**, each serve a purpose.


### 1. Session-Level Query 

* How long do sessions last?
* Which channels drive engaged visits?
* What is the landing page performance?

👉 [**Session-Level**](session-level-data-from-ga4-bigquery.md)

---

### 2. User-Level Query 

* Where do new users originally come from?
* How engaged are different user groups?
* How long do users stay active?

👉 [**User-Level**](user-level-data-from-ga4-bigquery.md)

---

### 3. Event & Pageview-Level Query 
* What exactly did users do?
* In what order did events occur?
* Where are users dropping off?

👉 [**Event & Pageview-Level**](event-level-data-from-ga4-bigquery.md)

---
### 4. Product-Level Query 

* Which products drive revenue?
* Which products convert poorly?
* How do promotions affect performance?

👉 [**Product-Level**](product-level-data-from-ga4-bigquery.md)

---

## How These Queries Work Together

These queries stack on each other:

1. **Session** →  visits and engagement
2. **User** → behavior and growth
3. **Event & Pageview** → interaction history
4. **Product** → commercial performance and optimization

---

## Important Note on GA4 BigQuery Schema Changes

Google periodically changes and extends the GA4 BigQuery export schema. These changes can include:

* New event parameters or fields

* Renamed or deprecated fields

* Changes to nested structures

* Differences between web and app exports

Small schema changes can silently break reports if SQL is not written defensively. Also, Google changes historical data stored in big query; even weeks after the day in question has passed. So it is important to have safe and resilient SQL patterns, like:
  
* Extract parameters through UNNEST(event_params) instead of hardcoded column assumptions

* Use SAFE_OFFSET with arrays

* Use SAFE_DIVIDE to avoid calculation failures

* Allow for NULL values where GA4 does not guarantee population

In practice, analytics teams should:

* Remember that daily data tables in BgQuery will change. So run transformation queries on raw data for past full week or longer periods 
  
* Monitor GA4 release notes and BigQuery export updates
  
* Validate key tables after schema changes
  
* Prefer reusable base models
