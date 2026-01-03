This project aim to do analyze how long it takes users to make their first purchase after arriving on the website on a given day, track how this duration evolves daily, and present insights through a dynamic visualization, supported by SQL-based data extraction, exploratory analysis, and critical evaluation of limitations and next analytical steps.
- [Google Slides Presentation](https://docs.google.com/presentation/d/1UBtuvUpBLOGvMtE3eFil19-mb7Mg6FUrcgtmlgtwlPI/edit?slide=id.gd9c453428_0_16#slide=id.gd9c453428_0_16)
- [Tableau Dashboard](https://public.tableau.com/app/profile/oumi.idrissi/viz/ProductAnalyticsProject_17656490703260/Dashboard1)
- [Data Source and SQL query](https://docs.google.com/spreadsheets/d/1Bpryn7mtyMJc30ACsXYLkVlaSptHRg1YFiegil6FanE/edit?usp=sharing)

**Key insights:**
- The average time customers spend on the website before purchasing is ~19 mins.
- Returning users consistently convert faster than new users, showing a structural behavior gap.
- Desktop and mobile users have the fastest and most stable conversion times, while tablet show higher friction.
- Higher user activity is associated with longer time to purchase, suggesting exploration or decision complexity.
- Conversion speed and revenue are seasonally impacted, with slower and lower performance post-holidays.

**Recommendations for Business:**
- Optimize the first-time user experience to reduce time to purchase for new users.
- Improve tablet UX to address higher friction on smaller devices.
- Leverage high-intent periods (midweek, pre-holidays) with targeted campaigns and streamlined flows.
- Use time-to-purchase as an ongoing KPI to monitor funnel efficiency and traffic quality.

**SQL query used**
```sql
# Time difference between session_start and puchase

WITH raw_data AS (
  # sort out only session_start and purchase
  SELECT
        PARSE_DATE('%Y%m%d', event_date) AS event_date,
        TIMESTAMP_MICROS(event_timestamp) AS event_time,
        event_name,
        user_pseudo_id,
        category,
        total_item_quantity AS quantity,
        purchase_revenue_in_usd AS revenue,
        transaction_id,
        campaign,
        mobile_model_name,
        mobile_brand_name,
        operating_system,
        browser,
        browser_version,
        country

  FROM tc-da-1.turing_data_analytics.raw_events

  WHERE event_name IN (""session_start"", ""purchase"")
),

previous_events AS (
  # getting the previous events and timestamps
  SELECT
        *,
        LAG(event_time) OVER 
        (PARTITION BY user_pseudo_id ORDER BY event_time) AS previous_timestamp,
        LAG(event_name) OVER 
        (PARTITION BY user_pseudo_id ORDER BY event_time) AS previous_event

  FROM raw_data
),

purchases AS (
  # getting only purchases with sessions
  SELECT
        *,
        TIMESTAMP_DIFF(event_time, previous_timestamp, MINUTE) AS start_to_purchase_time,
        ROW_NUMBER() OVER (PARTITION BY user_pseudo_id ORDER BY event_time) AS purchase_no

  FROM
    previous_events

  WHERE event_name = ""purchase"" 
  AND revenue > 0
  AND previous_event = ""session_start""
),

event_number AS (
  # count events to purchase
  SELECT
        p.user_pseudo_id,
        p.event_date,
        p.purchase_no,
        COUNT(t.event_name) AS event_count

  FROM tc-da-1.turing_data_analytics.raw_events t

  JOIN purchases p
  ON t.user_pseudo_id = p.user_pseudo_id
  AND TIMESTAMP_MICROS(t.event_timestamp) >= p.previous_timestamp
  AND TIMESTAMP_MICROS(t.event_timestamp) < p.
  
  GROUP BY p.user_pseudo_id, p.event_date, p.purchase_no
),

first_purchase AS (
SELECT 
      user_pseudo_id,
      MIN(PARSE_DATE('%Y%m%d', event_date)) AS first_purchase_date

FROM tc-da-1.turing_data_analytics.raw_events

WHERE event_name = 'purchase'
GROUP BY user_pseudo_id
)

SELECT
      p.*,
      en.event_count,
      CASE WHEN p.event_date != fp.first_purchase_date THEN 1 ELSE 0 END AS returning_customer

FROM purchases p

LEFT JOIN event_number en
ON p.user_pseudo_id = en.user_pseudo_id
AND p.purchase_no = en.purchase_no

LEFT JOIN first_purchase AS fp
ON p.user_pseudo_id = fp.user_pseudo_id

WHERE start_to_purchase_time IS NOT NULL
AND start_to_purchase_time < 300

ORDER BY event_time
;
```
