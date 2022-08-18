# RFM analysis


✨ ✨ Solution for RFM analysis task for Turing College✨ ✨ 

<a href = 'https://datastudio.google.com/s/hg38ZuongVQ
'> The Data Studio report with visualisations.</a> It explains the main insights of the analysis and provides a solution for user segmentation.

:rocket: SQL (BigQuery), Data Studio, RFM analysis

### The task:

- You have a follow up task from your marketing manager to identify overall trends of all marketing campaigns on your ecommerce site. She is particularly interested in finding out if users tend to spend more time on your website on certain weekdays and how that behavior differs across campaigns.
- Create a presentation centered around the dynamic weekday duration, focusing on differences between marketing campaigns.
- See whether you can apply 1-2 techniques learned in this or other modules throughout the course material to enhance your presentation on this subject.
- Explore the data. See whether there are interesting data points that can give more insights to your presentation.
- Provide analytical insights, what are the drawbacks of this analysis, what further analysis could you recommend?

You should use the turing_college.raw_events table to answer this question. Write a SQL query that would extract data from BigQuery, make a visualization using Google Sheets and briefly comment your findings. As we do not have session identifiers in the dataset, you will have to come up with your own logic for how you will model sessions. Have in mind that a single user can come to your website on multiple days and if you were to calculate time on the website this may have an impact on this metric.

### Main SQL Query:

``` SQL 
WITH t1 AS
 (
    SELECT  
    CustomerID,
    MAX(InvoiceDate) AS last_purchase_date,
    COUNT(DISTINCT InvoiceNo) AS frequency,
    SUM(UnitPrice*Quantity) AS monetary 
    FROM `tc-da-1.turing_data_analytics.rfm`
    WHERE InvoiceDate BETWEEN '2010-12-01' AND '2011-12-01'
    AND CustomerID IS NOT NULL
    GROUP BY 1
),

t2 AS 
(
    SELECT MAX(last_purchase_date) AS reference_date
    FROM t1
),

RFM AS 
(
  SELECT t1.CustomerID,
    DATE_DIFF(reference_date, t1.last_purchase_date, DAY) + 1 AS recency,
    t1.frequency AS frequency,
    t1.monetary AS monetary
    FROM t1, t2
),


percentiles AS 
(
  SELECT 
    a.*,
    r.percentiles[offset(25)] AS recency_25, 
    r.percentiles[offset(50)] AS recency_50,
    r.percentiles[offset(75)] AS recency_75,   
    m.percentiles[offset(25)] AS monetary_25, 
    m.percentiles[offset(50)] AS monetary_50,
    m.percentiles[offset(75)] AS monetary_75,     
    f.percentiles[offset(25)] AS frequency_25, 
    f.percentiles[offset(50)] AS frequency_50,
    f.percentiles[offset(75)] AS frequency_75, 
FROM RFM AS a,
    (SELECT APPROX_QUANTILES(recency, 100) percentiles 
    FROM RFM) AS r,
    (SELECT APPROX_QUANTILES(monetary, 100) percentiles 
    FROM RFM) AS m,
    (SELECT APPROX_QUANTILES(frequency, 100) percentiles 
    FROM RFM) AS f
),

RFM_scores AS 
(
    SELECT *
FROM (
    SELECT *,
        CASE
            WHEN recency <= recency_25 THEN 4
            WHEN recency <= recency_50 AND recency > recency_25 THEN 3
            WHEN recency <= recency_75 AND recency > recency_50 THEN 2
            WHEN recency > recency_75 THEN 1
        END AS r_score,
        CASE
            WHEN frequency <= frequency_25 THEN 1
            WHEN frequency <= frequency_50 AND frequency > frequency_25 THEN 2
            WHEN frequency <= frequency_75 AND frequency > frequency_50 THEN 3
            WHEN frequency > frequency_75 THEN 4
        END AS f_score,
                CASE
            WHEN monetary <= monetary_25 THEN 1
            WHEN monetary <= monetary_50 AND monetary > monetary_25 THEN 2
            WHEN monetary <= monetary_75 AND monetary > monetary_50 THEN 3
            WHEN monetary > monetary_75 THEN 4
        END AS m_score
    FROM percentiles
)
)

SELECT CustomerID,
    recency,
    frequency,
    monetary,
    r_score,
    f_score,
    m_score,
    CONCAT(r_score, f_score, m_score) AS RFM_score,
    CASE
    WHEN CONCAT(r_score, f_score, m_score) = '444'
        THEN 'Champions' 
    WHEN CONCAT(r_score, f_score, m_score) IN ('343', '344', '244', '243')
        THEN 'Loyal Customers'  
    WHEN CONCAT(r_score, f_score, m_score) LIKE '4%3' 
        OR CONCAT(r_score, f_score, m_score) LIKE '43%' 
        OR CONCAT(r_score, f_score, m_score) LIKE '4%2' 
        OR CONCAT(r_score, f_score, m_score) LIKE '42%'
        THEN 'Potential Loyalists'
    WHEN CONCAT(r_score, f_score, m_score) LIKE '41%'
        THEN 'Recent Customers'
    WHEN CONCAT(r_score, f_score, m_score) LIKE '31%'
        OR CONCAT(r_score, f_score, m_score) LIKE '3%1'
        THEN 'Promising'
    WHEN CONCAT(r_score, f_score, m_score) IN ('222','223','232','233','333','323','332','322')
        THEN 'Customers Needing Attention'
    WHEN CONCAT(r_score, f_score, m_score) LIKE '24%'
        OR CONCAT(r_score, f_score, m_score) LIKE '2%4'
        THEN 'At Risk'
    WHEN CONCAT(r_score, f_score, m_score) LIKE '14%'
        OR CONCAT(r_score, f_score, m_score) LIKE '1%4'
        THEN 'Cant Lose Them'
    WHEN CONCAT(r_score, f_score, m_score) LIKE '21%'
        THEN 'About to Sleep'
    WHEN CONCAT(r_score, f_score, m_score) IN ('123','132', '133', '122', '112','113','121','131')
        THEN 'Hibernating' 
    WHEN CONCAT(r_score, f_score, m_score) = '111'
      THEN 'Lost' 
  END AS rfm_segment
FROM RFM_scores
ORDER BY 9, 8
```
