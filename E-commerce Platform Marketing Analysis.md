This project aim to analyze user session behavior across weekdays with a focus on marketing campaign performance. The analysis explores how session duration varies by weekday and how this behavior differs between marketing-driven and non-marketing traffic.

Due to BigQuery export limitations, only a subset of the full dataset was included in the analysis and visualizations.

- [Tableau Dashboard](https://public.tableau.com/app/profile/oumi.idrissi/viz/MarketingAnalysisSpecialization/Dashboard1?publish=yes)
- [Google Slides Presentation](https://docs.google.com/presentation/d/1NzrrEf-I692ikc5oPMoVJPWjRa5lUki_smBiyTrIJhs/edit?usp=sharing)

**Keys Insights**
- Organic traffic drives most users, revenue, and purchases.
- Campaign traffic shows higher engagement but low scale.
- Weekday sessions outperform weekends for both traffic types.
- Campaign engagement varies by source, with a few consistent performers.
- Longer sessions do not directly drive higher revenue.

**Recommendations for Business**
- Prioritize campaigns on high-engagement weekdays.
- Scale top-performing campaigns and optimize weak ones.
- Improve data tracking for deleted data

**SQL Query Used**
```sql
-- Extract required fields and convert raw event timestamps into a readable format

WITH 
    process_table_1 AS (SELECT
                            user_pseudo_id AS user_id,
                            event_name AS event,
                            TIMESTAMP_MICROS(event_timestamp) AS occurred_at,
                            name,
                            campaign,
                            total_item_quantity,
                            purchase_revenue_in_usd 

                        FROM `tc-da-1.turing_data_analytics.raw_events` 
                        ),


-- Identify the previous event timestamp and event name per user to enable time difference calculation

process_table_2 AS (SELECT *,
                        LAG(occurred_at, 1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS prev_event,
                        LAG(event, 1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS prev_event_name 

                    FROM process_table_1 
                    ),


-- Calculate the time difference (in seconds) between the current and previous event

process_table_3 AS (SELECT *,
                        TIMESTAMP_DIFF(occurred_at, prev_event, second) AS time_diff,

                    FROM process_table_2 

                    WHERE NOT (prev_event_name = 'purchase' AND event = 'purchase')
                    ),


-- Flag the start of a new session when the time gap is NULL or exceeds 30 minutes

process_table_4 AS (SELECT *,
                        CASE WHEN(time_diff IS NULL or time_diff >= 1800) THEN 1 ELSE 0 END AS is_new_session 

                    FROM process_table_3 
                    ), 


-- Assign global and user-level session identifiers based on detected session boundaries

process_table_5 AS (SELECT *,
                        SUM(is_new_session) OVER (ORDER BY user_id, occurred_at) AS global_session_id,
                        SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY occurred_at) AS user_session_id 

                    FROM process_table_4 
                    ),


-- Identify the session in which a purchase event occurred for each user

process_table_6 AS (SELECT
                        user_id,
                        user_session_id,
                        COALESCE(LAG(user_session_id) OVER (PARTITION BY user_id ORDER BY user_session_id), 0) AS prev_session 
   
                    FROM process_table_5 

                    WHERE event = 'purchase' 
                    ),


-- Join session data and assign purchase session IDs; sessions without purchases are labeled as 0

process_table_7 AS (SELECT t1.*,
                            COALESCE(t2.user_session_id, 0) AS purchase_session_id 

                    FROM process_table_5 t1 

                    LEFT JOIN process_table_6 t2
                        ON t1.user_id = t2.user_id 
                        AND t1.user_session_id > t2.prev_session 
                        AND t1.user_session_id <= t2.user_session_id 

                    ORDER BY occurred_at
                    ),


-- Separate sessions based on purchase occurrences using ranking logic

process_table_8 AS (SELECT *, 
                        DENSE_RANK() OVER(PARTITION BY user_id ORDER BY purchase_session_id) AS purchase_order 

                    FROM process_table_7
                    ),


-- Extract the last known campaign associated with each global session

process_table_9 AS (SELECT 
                        DISTINCT user_id,
                        global_session_id,
                        LAST_VALUE(campaign) OVER (PARTITION BY user_id, global_session_id ORDER BY global_session_id) AS last_campaign,

                    FROM process_table_8
                    
                    WHERE campaign IS NOT NULL
                    )


-- Final aggregation at the session level with campaign and conversion indicators

SELECT 
    p1.user_id, 
    last_campaign,
    p1.global_session_id,
    p1.user_session_id,
    p1.purchase_session_id,
    CASE WHEN last_campaign IS NULL OR last_campaign IN('(direct)','(referral)','(organic)','<Other>') THEN 0 ELSE 1 END AS is_campaign,
    CASE WHEN purchase_session_id != 0 THEN 1 ELSE 0 END AS is_conversion,
    DATE(MAX(occurred_at)) AS occurred_at,
    SUM(CASE WHEN event = 'purchase' THEN total_item_quantity ELSE 0 END) AS quantity, 
    SUM(CASE WHEN event = 'purchase' THEN purchase_revenue_in_usd ELSE 0 END) AS revenue, 
    SUM(CASE WHEN event = 'purchase' THEN 1 ELSE 0 END) AS number_of_puchases, 
    SUM(CASE WHEN is_new_session = 0 THEN time_diff ELSE 0 END) AS duration_sec, 
    SUM(CASE WHEN event = 'page_view' THEN 1 ELSE 0 END) AS page_views, 

FROM process_table_8 p1

LEFT JOIN process_table_9 p2 
    ON p1.user_id = p2.user_id 
    AND p1.global_session_id = p2.global_session_id

GROUP BY 
    p1.user_id, 
    last_campaign,
    p1.global_session_id,
    p1.user_session_id,
    p1.purchase_session_id

ORDER BY p1.global_session_id
;
```
