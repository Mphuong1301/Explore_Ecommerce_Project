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

#### Query 1: Calculate total visit, pageview, transaction for Jan, Feb and March 2017 (order by month)

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

<img width="631" alt="Image" src="https://github.com/user-attachments/assets/9a724cfb-fb93-4547-85b6-357f6603227e" />

#### Query 2: Bounce rate per traffic source in July 2017 (Bounce_rate = num_bounce/total_visit) (order by total_visit DESC)

- Code
```sh
SELECT trafficSource.source

   ,COUNT(visitNumber) total_visits

   ,SUM(totals.bounces) total_no_of_bounces

   ,ROUND((SUM(totals.bounces)/COUNT(visitNumber))*100,2) bounce_rate

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`

GROUP BY trafficSource.source

ORDER BY 2 DESC;
```

- Result table:

<img width="534" alt="Image" src="https://github.com/user-attachments/assets/e94a9856-ccd2-4b4d-9b42-685f49156c93" />

#### Query 3: Revenue by traffic source by week, by month in June 2017

- Code
```sh
WITH month_data AS (

   SELECT

       DISTINCT CASE WHEN 1=1 THEN "Month" END time_type

       ,FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS time

       ,trafficSource.source AS source

       ROUND(SUM(totals.totalTransactionRevenue/1000000) OVER(PARTITION BY trafficSource.source),2) revenue

   FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`

),

week_data AS (

   SELECT

       CASE WHEN 1=1 THEN "WEEK" END time_type

       ,FORMAT_DATE("%Y%W", PARSE_DATE("%Y%m%d", date)) AS time

       ,trafficSource.source AS source

       ,SUM(totals.totalTransactionRevenue)/1000000 revenue

   FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`

   WHERE _table_suffix BETWEEN '0601' AND '0630'

   GROUP BY 1,2,3

)

SELECT * FROM month_data

UNION ALL

SELECT * FROM week_data

ORDER BY revenue DESC;
```

- Result table:

<img width="604" alt="Image" src="https://github.com/user-attachments/assets/d79c5da3-f007-47d7-9499-a5d92355cf31" />

#### Query 4. Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.

- Code
```sh
WITH purchaser_data as (

   SELECT

       FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) as month

       ,SUM(totals.pageviews) / COUNT(distinct fullvisitorid) as avg_views_purchase

   FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`

       ,UNNEST(hits) hits

       ,UNNEST(hits.product) product

   WHERE _table_suffix BETWEEN '0601' AND '0731'

       AND totals.transactions >= 1

       AND productRevenue IS NOT NULL

   GROUP BY 1

),

non_purchaser_data as (

   SELECT
       FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) month

       ,SUM(totals.pageviews) / COUNT(distinct fullvisitorid) avg_views_non_purchase

   FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`

       ,UNNEST(hits) hits

       ,UNNEST(hits.product) product

   WHERE _table_suffix BETWEEN '0601' AND '0731'

       AND totals.transactions IS NULL

       AND productRevenue IS NULL

   GROUP BY 1

)

SELECT

   month

   ,avg_views_purchase

   ,avg_views_non_purchase

FROM purchaser_data

FULL JOIN non_purchaser_data

USING (month)

ORDER BY 1;
```

- Result table:

<img width="392" alt="Image" src="https://github.com/user-attachments/assets/0e126e86-2ddb-482c-964c-3dd7e1e2e4d0" />

#### Query 5. Average number of transactions per user that made a purchase in July 2017

- Code
```sh
SELECT

   FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) month

   ,ROUND(SUM(totals.transactions)/COUNT(DISTINCT fullVisitorId), 9) as avg_total_transactions

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`

   ,UNNEST (hits) as hits

   ,UNNEST (hits.product) as product

WHERE totals.transactions >= 1

AND product.productRevenue IS NOT NULL

GROUP BY 1;
```

- Result table:

<img width="349" alt="Image" src="https://github.com/user-attachments/assets/349be46f-8251-415c-94df-d9956dd1b349" />

#### Query 6. Average amount of money spent per session. Only include purchaser data in July 2017

- Code
```sh
SELECT

   FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month

   ,ROUND((SUM(product.productRevenue) / SUM(totals.visits))/1000000, 2) AS avg_revenue_per_visit

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`

   ,UNNEST(hits) AS hits

   ,UNNEST(hits.product) AS product

WHERE

   _TABLE_SUFFIX BETWEEN '0701' AND '0731'

   AND product.productRevenue IS NOT NULL

   AND totals.transactions IS NOT NULL

GROUP BY 1;
```

- Result table:

<img width="317" alt="Image" src="https://github.com/user-attachments/assets/bd80f3b5-7564-4acc-830a-8fbe2bf254b2" />

#### Query 7. Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered.

- Code
```sh
WITH CUS_ID AS (

SELECT

   DISTINCT fullVisitorId as Henley_customer_id

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`

   ,UNNEST(hits) AS hits

   ,UNNEST(hits.product) as product

WHERE product.v2ProductName = "YouTube Men's Vintage Henley"

AND product.productRevenue IS NOT NULL

)

SELECT

   product.v2ProductName AS other_purchased_products

   ,SUM(product.productQuantity) AS quantity

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` ga_session

RIGHT JOIN CUS_ID

ON CUS_ID.Henley_customer_id=ga_session.fullVisitorId

   ,UNNEST(hits) AS hits

   ,UNNEST(hits.product) as product

WHERE ga_session.fullVisitorId IN (

   SELECT *

   FROM CUS_ID

)

AND product.v2ProductName <> "YouTube Men's Vintage Henley"

AND product.productRevenue IS NOT NULL

GROUP BY product.v2ProductName

ORDER BY 2 DESC;
```

- Result table:

<img width="315" alt="Image" src="https://github.com/user-attachments/assets/6b5a186c-a4f5-4fb8-b102-3a5150fc6113" />

#### Query 8. Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017. (For example, 100% product view then 40% add_to_cart and 10% purchase. Add_to_cart_rate = number product add to cart/number product view. Purchase_rate = number product purchase/number product view). The output should be calculated in product level.

- Code
```sh
WITH product_data AS (

SELECT

   FORMAT_DATE("%Y%m",PARSE_DATE("%Y%m%d",date)) AS month

   ,COUNT(CASE WHEN eCommerceAction.action_type = '2' THEN product.v2ProductName END) AS num_product_view

   ,COUNT(CASE WHEN eCommerceAction.action_type = '3' THEN product.v2ProductName END) AS num_add_to_cart

   ,COUNT(CASE WHEN eCommerceAction.action_type = '6' AND product.productRevenue IS NOT NULL THEN product.v2ProductName END) AS num_purchase

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`

   ,UNNEST (hits) AS hits

   ,UNNEST (hits.product) as product

WHERE _table_suffix BETWEEN '0101' AND '0331'

AND eCommerceAction.action_type IN ('2', '3', '6')

GROUP BY 1

ORDER BY 1

)

SELECT *

   ,ROUND(num_add_to_cart/num_product_view * 100, 2) add_to_cart_rate

   ,ROUND(num_purchase/num_product_view * 100, 2) purchase_rate

from product_data;
```

- Result table:

<img width="606" alt="Image" src="https://github.com/user-attachments/assets/72154214-a1d6-4dd8-8b88-ca5e86983e26" />

### V. Conclusions

- By analyzing the e-commerce dataset using BigQuery, we understands customer behavior through the bounce rate, transaction, revenue, visit, and purchase.

- We gained insights into which marketing channels drive traffic and sales by examining referral sources. Investing resources in effective channels and optimizing underperforming ones can improve marketing ROI.

- In conclusion, exploring the e-commerce dataset on BigQuery unearthed a wealth of insights critical for strategic decision-making to help the business can optimize operations, enhance customer experiences, and drive revenue.
