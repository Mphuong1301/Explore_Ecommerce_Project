# [SQL] Explore_Ecommerce_Project
## Table of Contents:
### I. Introduction 
The eCommerce dataset is hosted in a public Google BigQuery dataset and contains information about user sessions on a website, collected through Google Analytics in 2017.

Based on the eCommerce dataset, the author conducts queries to analyze website activity during 2017. This includes calculating the bounce rate, identifying the days with the highest revenue, examining user behavior on different pages, and performing various other analyses. The project's goal is to gain insights into the business's performance, evaluate the effectiveness of marketing activities, and analyze product-related data.

In this project, I focused on data exploration and calculation of several metrics in the e-commerce sector.

### II. Import raw data
The eCommerce dataset is stored in a public Google BigQuery dataset. To access the dataset, follow these steps:

- Log in to your Google Cloud Platform account and create a new project.
- Navigate to the BigQuery console and select your newly created project.
- Select "Add Data" in the navigation panel and then "Search a project".
- Enter the project ID "bigquery-public-data.google_analytics_sample.ga_sessions" and click "Enter".
- Click on the "ga_sessions_" table to open it.

### III. Read and explain dataset
Data is stored in tables, each table stores data for one day of the year. The entire dataset is a system of tables for each day from January 8, 2016 to August 1, 2017.

The tables all have the same format as follows:

 
|       Field Name      |    Data Type     | Description     |
| :------------:|:-------------:|:-----:|
|    fullVisitorId   |   STRING   |  The unique visitor ID.   |
|    date     |   STRING  |   The date of the session in YYYYMMDD format.   |
|     totals    | RECORD  |    This section contains aggregate values across the session.  |
|    totals.bounces    |  INTEGER	 | Total bounces (for convenience). For a bounced session, the value is 1, otherwise it is null.   |
|   totals.hits     |  INTEGER	 |   Total number of hits within the session.   |
|     totals.pageviews    | INTEGER	  |    Total number of pageviews within the session.  |
|    totals.visits   |  INTEGER	 |  The number of sessions (for convenience). This value is 1 for sessions with interaction events. The value is null if there are no interaction events in the session.  |
|    trafficSource.source   |  STRING	  |  The source of the traffic source. Could be the name of the search engine, the referring hostname, or a value of the utm_source URL parameter.  |
|   hits   | RECORD   |  This row and nested fields are populated for any and all types of hits. |
|   hits.eCommerceAction |   RECORD  | This section contains all of the ecommerce hits that occurred during the session. This is a repeated field and has an entry for each hit that was collected.  |
|   hits.eCommerceAction.action_type   |   STRING  |  The action type. Click through of product lists = 1, Product detail views = 2, Add product(s) to cart = 3, Remove product(s) from cart = 4, Check out = 5, Completed purchase = 6, Refund of purchase = 7, Checkout options = 8, Unknown = 0.Usually this action type applies to all the products in a hit, with the following exception: when hits.product.isImpression = TRUE, the corresponding product is a product impression that is seen while the product action is taking place (i.e., a product in list view).  |
|    hits.product   | RECORD  |   This row and nested fields will be populated for each hit that contains Enhanced Ecommerce PRODUCT data.  |
|   hits.product.productQuantity   |  INTEGER	 | The quantity of the product purchased.   |
|   hits.product.productRevenue  |  INTEGER	 |   The revenue of the product, expressed as the value passed to Analytics multiplied by 10^6 (e.g., 2.40 would be given as 2400000).  |
|    hits.product.productSKU   | STRING  |   Product SKU. |
|    hits.product.v2ProductName	  |  STRING	 |  Product Name.  |
|   fullVisitorId  |  STRING	  | The unique visitor ID. |

### IV. Exploring the Dataset
08 queries in Bigquery based on the Google Analytics dataset I wrote
#### Query 1. Calculate total visit, pageview, transaction for Jan, Feb and March 2017 (order by month)
- Code
```sh
SELECT

   FORMAT_DATE("%Y%m",PARSE_DATE("%Y%m%d",date)) month

   ,SUM(totals.visits) visits

   ,SUM(totals.pageviews) pageviews

   ,SUM(totals.transactions) transactions

   ,ROUND(SUM(totals.totalTransactionRevenue)/POW(10,6),2) revenue

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`

WHERE _table_suffix BETWEEN '0101' AND '0331'

GROUP BY 1

ORDER BY 1;
```

- Result table:
!([https://imgur.com/a/ljjjdlsadaoiwjdoqd-45jkhCC](https://github.com/user-attachments/assets/9f857d70-e406-4b38-983b-eb3cb54d59eb))

Query 2. Bounce rate per traffic source in July 2017 (Bounce_rate = num_bounce/total_visit) (order by total_visit DESC)
SELECT 
  trafficSource.source AS source,
  COUNT (totals.bounces) AS total_no_of_bounces,
  COUNT (totals.visits) AS total_visits,
  ROUND((COUNT (totals.bounces) / COUNT (totals.visits) * 100),7) AS bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` 
GROUP BY trafficSource.source
ORDER BY source;
Result table:


c2

Query 3. Revenue by traffic source by week, by month in June 2017
WITH sub1 AS (
  SELECT *,
  PARSE_DATE('%Y%m%d', date) AS date1
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`
),
sub2 AS (
  SELECT  
    'Month' as time_type, 
    FORMAT_DATE ('%Y%m',date1) AS time,
    trafficSource.source AS source,
    ROUND ((SUM (productRevenue) / 1000000 ),4) AS revenue
  FROM sub1,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
  WHERE productRevenue IS NOT NULL
  GROUP BY trafficSource.source, time
),
sub3 AS (
  SELECT  
    'Month' as time_type, 
    FORMAT_DATE ('%Y%W',date1) AS time,
    trafficSource.source AS source,
    ROUND ((SUM (productRevenue) / 1000000 ),4) AS revenue
  FROM sub1,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
  WHERE productRevenue IS NOT NULL
  GROUP BY trafficSource.source, time
)
SELECT *
FROM sub2
UNION ALL
SELECT *
FROM sub3
ORDER BY time;
Result table:

c3

Query 4. Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.
SELECT
  FORMAT_DATE ("%Y%m", PARSE_DATE ("%Y%m%d", date)) AS month, 
  ROUND(SUM (CASE WHEN totals.transactions >= 1 AND product.productRevenue IS NOT NULL THEN totals.pageviews END) / COUNT (DISTINCT (CASE
            WHEN totals.transactions >= 1 AND product.productRevenue IS NOT NULL THEN fullVisitorId
        END
          )),7) AS avg_pageviews_purchase,
  ROUND(SUM (CASE WHEN totals.transactions IS NULL AND product.productRevenue IS NULL THEN totals.pageviews END) / COUNT (DISTINCT (CASE
                WHEN totals.transactions IS NULL AND product.productRevenue IS NULL THEN fullVisitorId
            END
              )),7) AS avg_pageviews_non_purchase
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`, UNNEST (hits) hits, UNNEST (hits.product) product
WHERE
  _table_suffix BETWEEN '0601'AND '0731'
GROUP BY
  month
ORDER BY month;
Result table:

c4

Query 5. Average number of transactions per user that made a purchase in July 2017
SELECT 
  FORMAT_DATE ("%Y%m", PARSE_DATE ("%Y%m%d", date)) AS month,
  SUM (totals.transactions)/ COUNT (DISTINCT fullVisitorId) AS Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
UNNEST (hits.product) product
WHERE totals.transactions >=1 AND productRevenue IS NOT NULL AND (eCommerceAction.action_type) LIKE '6'
GROUP BY FORMAT_DATE ("%Y%m", PARSE_DATE ("%Y%m%d", date));
Result table:

c5

Query 6. Average amount of money spent per session. Only include purchaser data in July 2017
SELECT 

  FORMAT_DATE ("%Y%m", PARSE_DATE ("%Y%m%d", date)) AS month,

  ROUND(SUM (productRevenue) / 1000000 / COUNT(visitId), 2) AS avg_revenue_by_user_per_visit

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,

UNNEST (hits) hits,

UNNEST (hits.product) product

WHERE totals.transactions IS NOT NULL AND product.productRevenue is not null

GROUP BY FORMAT_DATE ("%Y%m", PARSE_DATE ("%Y%m%d", date));
Result table:

c6

Query 7. Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered.
WITH sub AS (
  SELECT DISTINCT fullVisitorId
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
UNNEST (hits.product) product
WHERE v2ProductName LIKE "YouTube Men's Vintage Henley" 
AND totals.transactions >=1 
AND productRevenue IS NOT NULL
),
sub2 AS (
SELECT  *
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
UNNEST (hits.product) product
WHERE totals.transactions >=1 
AND productRevenue IS NOT NULL
)
SELECT 
  v2ProductName AS other_purchased_products,
  SUM(productQuantity) AS quantity
from sub LEFT join sub2 ON sub.fullVisitorId=sub2.fullVisitorId
WHERE totals.transactions >=1 
  AND productRevenue IS NOT NULL
  AND v2ProductName NOT LIKE "YouTube Men's Vintage Henley"
GROUP BY v2ProductName;
Result table:

c7

Query 8. Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017. (For example, 100% product view then 40% add_to_cart and 10% purchase. Add_to_cart_rate = number product add to cart/number product view. Purchase_rate = number product purchase/number product view). The output should be calculated in product level.
SELECT

  FORMAT_DATE ("%Y%m", PARSE_DATE ("%Y%m%d", date)) AS month,

  COUNT (CASE WHEN hits.eCommerceAction.action_type LIKE '2' 

    THEN v2ProductName END) AS num_product_view,

  COUNT (CASE WHEN hits.eCommerceAction.action_type LIKE '3' 

    THEN v2ProductName END) AS num_addtocart,

  COUNT (CASE WHEN hits.eCommerceAction.action_type LIKE '6' AND totals.transactions >= 1 AND productRevenue IS NOT NULL 
  
    THEN v2ProductName END) AS num_purchase,

  ROUND ((COUNT (CASE WHEN hits.eCommerceAction.action_type LIKE '3' THEN v2ProductName END)/COUNT (CASE WHEN hits.eCommerceAction.action_type LIKE '2' THEN v2ProductName END) * 100.0),2) AS add_to_cart_rate,

  ROUND ((COUNT (CASE WHEN hits.eCommerceAction.action_type LIKE '6' 
    AND totals.transactions >= 1 AND productRevenue IS NOT NULL
    THEN v2ProductName END)/COUNT (CASE WHEN hits.eCommerceAction.action_type LIKE '2' THEN v2ProductName END)* 100.0), 2) AS purchase_rate

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,

UNNEST (hits) hits,

UNNEST (hits.product) product

WHERE _table_suffix BETWEEN '0101'AND '0331'

GROUP BY 1 

ORDER BY 1;
Result table:

c8
