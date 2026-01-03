This project aims to perform a full customer segmentation and CLV analysis using SQL, RFM scoring, and weekly cohort modeling. It involves selecting one year of data, calculating recency, frequency, and monetary values, converting them into quartile-based RFM scores, and segmenting customers into key groups such as Best Customers, Loyal Customers, and Lost Customers. The project also requires building visual dashboards (Tableau/Power BI) to highlight which segments marketing should prioritize.

In the second part, the project focuses on a refined CLV approach using cohort analysis. It tracks all users—not only buyers—by their registration week, calculates weekly and cumulative ARPU over 12 weeks, and uses cohort growth percentages to predict future revenue for recent cohorts. Visual tables with conditional formatting must show weekly ARPU, cumulative ARPU, and forecasted cumulative values. The final step is to interpret trends, identify key insights, and prepare the SQL queries, dashboards, and analysis for review.

- [Queries, visualization, summary and recommendations](https://docs.google.com/spreadsheets/d/1tDAJ6TyuypeSZIeUCg0jrk5Rbib0GCXpxnwmbNLvgxE/edit?gid=1795219300#gid=1795219300)
- [Tableau Dashboard](https://public.tableau.com/app/profile/oumi.idrissi/viz/M3S3-RFM_17525857133030/RFMSegmentationDashboard?publish=yes)

**SQL Query RFM**
```sql
-- First I compute for F, M and R using a CTE

WITH 
  t1 AS (
        SELECT
                CustomerID,
                Country,
                CAST(Max (invoicedate) AS DATE) AS last_purchase_date,
                COUNT(DISTINCT InvoiceNo) AS Frequency,
                SUM (quantity*Unitprice) AS Monetary,
        
        FROM tc-da-1.turing_data_analytics.rfm

        WHERE InvoiceDate BETWEEN '2010-12-01' AND '2011-12-01'
              AND CustomerID IS NOT NULL
        GROUP BY CustomerID,
                 Country
        ),
     
  t2 AS ( 
       SELECT *,
              DATE_DIFF('2011-12-01', last_purchase_date, DAY) AS Recency

       FROM t1
        ),

-- Then I calculate percentiles

  t3 AS (
        SELECT a.*,
                --All percentiles for MONETARY
                b.percentiles[offset(25)] AS m25, 
                b.percentiles[offset(50)] AS m50,
                b.percentiles[offset(75)] AS m75, 
                b.percentiles[offset(100)] AS m100,    
                --All percentiles for FREQUENCY
                c.percentiles[offset(25)] AS f25, 
                c.percentiles[offset(50)] AS f50,
                c.percentiles[offset(75)] AS f75, 
                c.percentiles[offset(100)] AS f100,    
                --All percentiles for RECENCY
                d.percentiles[offset(25)] AS r25, 
                d.percentiles[offset(50)] AS r50,
                d.percentiles[offset(75)] AS r75, 
                d.percentiles[offset(100)] AS r100
        FROM 
                t2 a,
                (SELECT APPROX_QUANTILES(monetary, 100) percentiles 
                 FROM t2) b,

                (SELECT APPROX_QUANTILES(frequency, 100) percentiles 
                 FROM t2) c,

                (SELECT APPROX_QUANTILES(recency, 100) percentiles
                 FROM t2) d
        ),

-- I assign a score for each quantiles, recency scoring is reversed

  t4 AS (
         SELECT *, 
               CASE 
                WHEN monetary <= m25 THEN 1
                WHEN monetary <= m50 AND monetary > m25 THEN 2 
                WHEN monetary <= m75 AND monetary > m50 THEN 3 
                WHEN monetary <= m100 AND monetary > m75 THEN 4 
               END AS m_score,

               CASE 
                WHEN frequency <= f25 THEN 1
                WHEN frequency <= f50 AND frequency > f25 THEN 2 
                WHEN frequency <= f75 AND frequency > f50 THEN 3 
                WHEN frequency <= f100 AND frequency > f75 THEN 4 
               END AS f_score,

               CASE 
                WHEN recency <= r25 THEN 4
                WHEN recency <= r50 AND recency > r25 THEN 3 
                WHEN recency <= r75 AND recency > r50 THEN 2 
                WHEN recency <= r100 AND recency > r75 THEN 1 
               END AS r_score,

          FROM t3),
  
-- I compute the fm score separately

  t5 AS (
        SELECT *, 
              CAST(ROUND((f_score + m_score) / 2, 0) AS INT64) AS fm_score

        FROM t4
        ),

-- Next I segment based on scoring

  t6 AS (
        SELECT 
                CustomerID, 
                Country,
                recency,
                frequency, 
                monetary,
                r_score,
                f_score,
                m_score,
                fm_score,
        CASE 
         WHEN r_score >= 3 AND fm_score >= 3 THEN 'Loyal'
         WHEN fm_score >= 3 THEN 'Big Spender'
         WHEN r_score <= 2 AND fm_score >= 2 THEN 'Hibernating'
         WHEN r_score <= 2 AND fm_score <= 2 THEN 'Lost'
         WHEN r_score BETWEEN 2 AND 3 AND fm_score >= 2 THEN 'Promising'
         WHEN r_score = 4 AND fm_score <= 2 THEN 'New Customer'
         WHEN r_score BETWEEN 3 AND 4 AND fm_score BETWEEN 2 AND 3 THEN 'Active Regular'
         WHEN r_score BETWEEN 2 AND 3 AND fm_score >= 3 THEN 'High Potential'
         WHEN r_score <= 2 AND fm_score >= 2 THEN 'Churning'
         WHEN r_score BETWEEN 2 AND 3 AND fm_score <= 2 THEN 'Occasional Shopper'
         ELSE 'At Risk'
        END AS rfm_segment 
    
        FROM t5
        )

-- Finally, I concatenate as per the task requirement

SELECT *, 
       CONCAT(r_score, f_score, m_score) AS rfm_score

FROM t6
;
```

**SQL Query CLV**
```sql
-- First I create a CTE which: 1) change the date format 2) search for the first occured event (MIN) 3) Date trunc to the week (starting from Sunday)


WITH 
    Registration_week AS (SELECT user_pseudo_id,
                                  DATE_TRUNC(MIN(PARSE_DATE('%Y%m%d', event_date)), WEEK) AS registration_week

                           FROM tc-da-1.turing_data_analytics.raw_events

                           GROUP BY user_pseudo_id
                           ),

-- In the next CTE 1) I look for event ""purchase""  with values above 0, 2) Change the format and truncate to week, this will retrive all users who purchased and their orders value

      Revenue_week AS (SELECT user_pseudo_id, 
                              DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK) AS purchase_week, 
                              purchase_revenue_in_usd

                      FROM tc-da-1.turing_data_analytics.raw_events

                      WHERE event_name = 'purchase'
                      AND purchase_revenue_in_usd > 0
                      ),


        -- Next I calculate unique user ids, this will help us later to calculate Average Revenue per User (ARPU)
 
       user_count AS (SELECT registration_week.registration_week,
                               COUNT(DISTINCT user_pseudo_id) AS User_count,

                        FROM registration_week

                        GROUP BY registration_week
                        ),


            -- Now I calculate the sum of revenue of each week for each cohort

     Revenue AS (SELECT registration_week.registration_week,
                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = registration_week.registration_week 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_0,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 1 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_1,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 2 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_2,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 3 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_3,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 4 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_4,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 5 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_5,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 6 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_6,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 7 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_7,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 8 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_8,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 9 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_9,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 10 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_10,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 11 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_11,

                                SUM(CASE 
                                        WHEN Revenue_week.purchase_week = DATE_ADD(registration_week.registration_week, INTERVAL 12 WEEK) 
                                        THEN Revenue_week.purchase_revenue_in_usd
                                    END) AS week_12,

                                FROM Revenue_week

                                JOIN registration_week
                                ON registration_week.user_pseudo_id = Revenue_week.user_pseudo_id

                                GROUP BY registration_week

                                ORDER BY registration_week
                        )

-- Finally, I divide sum of revenue of each week from cohort by the user count of unique ids, this gives us the Average Revenue per User (ARPU)

SELECT  revenue.registration_week,
        revenue.week_0/user_count.user_count AS week_0,
        revenue.week_1/user_count.user_count AS week_1,
        revenue.week_2/user_count.user_count AS week_2,
        revenue.week_3/user_count.user_count AS week_3,
        revenue.week_4/user_count.user_count AS week_4,
        revenue.week_5/user_count.user_count AS week_5,
        revenue.week_6/user_count.user_count AS week_6,
        revenue.week_7/user_count.user_count AS week_7,
        revenue.week_8/user_count.user_count AS week_8,
        revenue.week_9/user_count.user_count AS week_9,
        revenue.week_10/user_count.user_count AS week_10,
        revenue.week_11/user_count.user_count AS week_11,
        revenue.week_12/user_count.user_count AS week_12

FROM revenue

JOIN user_count
ON user_count.registration_week = revenue.registration_week

ORDER BY revenue.registration_week
;
```
