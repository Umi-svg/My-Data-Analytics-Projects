This project aims to build a clean and reliable sales funnel by transforming raw website event data into a structured sequence of unique user actions. It focuses on identifying one distinct event per user (based on the earliest timestamp), selecting 4–6 meaningful event types, and constructing a funnel chart that compares conversion patterns across the top three countries by event volume. The task requires writing SQL queries to deduplicate events, aggregate them by country, and prepare a funnel table including event order and drop-off percentages. The final deliverable includes a well-formatted funnel chart, supporting tables, and brief insights, along with a presentation explaining how the data was collected, validated, and translated into business-relevant findings.
- [Project Link](https://docs.google.com/spreadsheets/d/13NPAbRIV_ewkghrVAOQI95zJMiojBuoYnnuZ17KFS4E/edit?gid=409644485#gid=409644485)

**Queries used**

- Task 1. Create a query  for unique events. Copy this query into the Queries used spreadsheet.

```sql
WITH t1 AS
  (SELECT user_pseudo_id,
          event_name,
          MIN(event_timestamp) AS event_timestamp
   FROM tc-da-1.turing_data_analytics.raw_events
   GROUP BY user_pseudo_id,
            event_name
   ORDER BY user_pseudo_id,
            event_timestamp),
     t2 AS
  (SELECT *
   FROM tc-da-1.turing_data_analytics.raw_events)
SELECT t2.*
FROM t1
LEFT JOIN t2 ON t1.user_pseudo_id = t2.user_pseudo_id
AND t1.event_name = t2.event_name
AND t1.event_Timestamp = t2.event_timestamp
ORDER BY t1.user_pseudo_id,
         t1.event_timestamp;
```

- Task 2 Write a new query that aggregates your identified events per top 3 countries. Copy this query into the Queries used spreadsheet.

```sql
WITH t4 AS
  (WITH t3 AS
     (WITH t1 AS
        (SELECT user_pseudo_id,
                event_name,
                MIN(event_timestamp) AS event_timestamp
         FROM tc-da-1.turing_data_analytics.raw_events
         GROUP BY user_pseudo_id,
                  event_name
         ORDER BY user_pseudo_id,
                  event_timestamp),
           t2 AS
        (SELECT *
         FROM tc-da-1.turing_data_analytics.raw_events) SELECT t2.*
      FROM t1
      LEFT JOIN t2 ON t1.user_pseudo_id = t2.user_pseudo_id
      AND t1.event_name = t2.event_name
      AND t1.event_Timestamp = t2.event_timestamp
      WHERE t2.country = 'United States'
        AND t2.event_name IN ('first_visit',
                              'view_item',
                              'add_to_cart',
                              'begin_checkout',
                              'purchase')
      ORDER BY t1.user_pseudo_id,
               t1.event_timestamp) SELECT event_name,
                                          COUNT(user_pseudo_id) AS event_count_USA
   FROM t3
   GROUP BY event_name
   ORDER BY event_count_USA DESC)
SELECT ROW_NUMBER() OVER (
                          ORDER BY event_count_USA DESC) AS event_order,
       t4.*
FROM t4;
```

- Additional category split

```sql
WITH t4 AS
  (WITH t3 AS
     (WITH t1 AS
        (SELECT user_pseudo_id,
                event_name,
                MIN(event_timestamp) AS event_Timestamp
         FROM tc-da-1.turing_data_analytics.raw_events
         GROUP BY user_pseudo_id,
                  event_name
         ORDER BY user_pseudo_id,
                  event_Timestamp),
           t2 AS
        (SELECT *
         FROM tc-da-1.turing_data_analytics.raw_events) SELECT t2.*
      FROM t1
      LEFT JOIN t2 ON t1.user_pseudo_id = t2.user_pseudo_id
      AND t1.event_name = t2.event_name
      AND t1.event_Timestamp = t2.event_timestamp
      WHERE t2.country IN ('United States',
                           'India',
                           'Canada')
        AND t2.event_name IN ('first_visit',
                              'view_item',
                              'add_to_cart',
                              'begin_checkout',
                              'purchase')
        AND t2.category = 'desktop'
      ORDER BY t1.user_pseudo_id,
               t1.event_timestamp) SELECT event_name,
                                          COUNT(user_pseudo_id) AS event_count_Desktop
   FROM t3
   GROUP BY event_name
   ORDER BY event_count_desktop DESC)
SELECT ROW_NUMBER() OVER (
                          ORDER BY event_count_desktop DESC) AS event_order,
       t4.*
FROM t4;
```
