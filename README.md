# Ecommerce Web Performance & Customer Purchasing Behaviour Analysis | SQL, BigQuery
I analyzed Ecommerce dataset by using SQL in BigQuery to extract key insights about web performance, revenue trends, bounce rate, etc. to improve channel quality, cohort funnel and cross-sell

## 📑**TABLE OF CONTENTS**
I. [Project Overview](#i-project-overview) <br/>
II. [Dataset](#ii-dataset) <br/>
III. [Key Business Questions](#iii-key-business-questions) <br/>
IV. [Insights and Recommendations](#iv-insights-and-recommendations) <br/>

## I. PROJECT OVERVIEW

⚓**Business goals:** 

This project analyzes 500K+ e-commerce logs to detect conversion drop-offs, assess traffic source performance and find cross-sell opportunities, helping improve user journey and conversion rates.

❓**Business questions:**
- Which traffic sources bring the most valuable customers and revenue?
- How well do users move through the purchase funnel from product view to add-to-cart to purchase?
- How do browsing and purchasing behaviors differ between buyers and non-buyers?
- How effectively is the website converting traffic into sales and revenue?
- Which products are frequently bought together and could support cross-sell opportunities?

## II. DATASET

- **Google Analytics Sample Store** is a demo e-commerce business used to track how customers visit the website, browse products, and make purchases. It represents a typical online store where teams want to understand traffic sources, shopping behavior, and sales performance. 
- The dataset includes traffic source, device, geography, session metrics, and nested hit-level interactions such as pageviews, product views, add-to-cart actions, and purchases.

<details>
  
<summary>See data table in detailed</summary>


| Field Name | Data Type | Description |
|----------|----------|----------|
| fullVisitorId   | String   | The unique visitor ID     |
| date      | String     | The date of the session in YYYYMMDD format      |
| totals      | Record     | This section contains aggregate values across the session      |
| totals.bounces      | Integer     | Total bounces (for convenience). For a bounced session, the value is 1, otherwise it is null      |
| totals.hits      | Integer     | Total number of hits within the session      |
| totals.pageviews      | Integer     | Total number of pageviews within the session      |
| totals.visits     | Integer     | The number of sessions (for convenience). This value is 1 for sessions with interaction events. The value is null if there are no interaction events in the session      |
| totals.transactions      | Integer     | Total number of ecommerce transactions within the session      |
| trafficSource.source      | String     | The source of the traffic source. Could be the name of the search engine, the referring hostname, or a value of the utm_source URL parameter      |
| hits      | Record     | This row and nested fields are populated for any and all types of hits      |
| hits.eCommerceAction      | Record     | This section contains all of the ecommerce hits that occurred during the session. This is a repeated field and has an entry for each hit that was collected      |
| hits.eCommerceAction.action_type      | String     | The action type. Click through of product lists = 1, Product detail views = 2, Add product(s) to cart = 3, Remove product(s) from cart = 4, Check out = 5, Completed purchase = 6, Refund of purchase = 7, Checkout options = 8, Unknown = 0. Usually this action type applies to all the products in a hit, with the following exception: when hits.product.isImpression = TRUE, the corresponding product is a product impression that is seen while the product action is taking place (i.e., a "product in list view")      |
| hits.product      | Record     | This row and nested fields will be populated for each hit that contains Enhanced Ecommerce PRODUCT data      |
| hits.product.productQuantity      | Integer     | The quantity of the product purchased      |
| hits.product.productRevenue      | Integer     | The revenue of the product, expressed as the value passed to Analytics multiplied by 10^6 (e.g., 2.40 would be given as 2400000)      |
| hits.product.productSKU      | String     | Product SKU      |
| hits.product.v2ProductName      | String     | Product Name     |

</details>

## III. KEY BUSINESS QUESTIONS

This project includes 8 queries

### 🔍 Query 1. Calculate total visit, pageview, transaction for January-August 2017 (order by month).

This query is to measure total visits, page views, transactions for each month from January to August in 2017. The result identifies overall trend and growth in the site

🚀 **Query**
```sql
SELECT 
  FORMAT_DATE('%Y%m', parse_date('%Y%m%d', date)) AS month
  ,SUM(totals.visits) AS visits
  ,SUM(totals.pageviews) AS pageview
  ,SUM(totals.transactions) AS transactions
  ,ROUND(100 * SAFE_DIVIDE(SUM(totals.transactions), SUM(totals.visits)), 2) AS conversion_rate_pct

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*` 
GROUP BY month
ORDER BY month;
```

💡**Query result**

<img width="911" height="305" alt="image" src="https://github.com/user-attachments/assets/c93ba95c-696d-49ca-97ce-3aeb22f21db3" />

**Key-takeaway:**
May had the highest conversion rate (1.77%) while July brought peak volume (71.8k visits, 270k page views) but a relatively low conversion rate (1.49%).

### 🔍 Query 2. Calculate cohort map from product view to addtocart to purchase in 2017 (January-August).

This query result is to build a cohort map to track user journey from product view to purchase in order to evaluate conversion rate in each funnel stage and identify where users drop off. This is to optimize the sale process and improve customer journey experience

🚀 **Query**
```sql
WITH 
product_view AS(--count number of product_view for each month
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
    ,COUNT(product.productSKU) AS num_product_view
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) as product
  WHERE hits.eCommerceAction.action_type = '2'
  GROUP BY month
),

add_to_cart AS(--count number of add_to_cart for each month
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
    ,count(product.productSKU) as num_addtocart
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) as product
  WHERE hits.eCommerceAction.action_type = '3'
  GROUP BY month
),

purchase AS(--count number of purchase for each month
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
    ,COUNT(product.productSKU) AS num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) AS product
  WHERE hits.eCommerceAction.action_type = '6'
  AND product.productRevenue IS NOT NULL
  GROUP BY month
)

SELECT
  pv.month
  ,pv.num_product_view
  ,a.num_addtocart
  ,p.num_purchase
  --add_to_cart_rate = number_addtocart / number product_view
  ,ROUND(a.num_addtocart * 100/pv.num_product_view,2) AS add_to_cart_rate
  --purchase_rate = number product purchase / number product_view
  ,ROUND(p.num_purchase * 100/pv.num_product_view,2) AS purchase_rate
FROM product_view AS pv
LEFT JOIN add_to_cart AS a ON pv.month = a.month
LEFT JOIN purchase AS p ON pv.month = p.month
ORDER BY pv.month;
```

💡**Query result**

<img width="1031" height="307" alt="image" src="https://github.com/user-attachments/assets/d81db07a-af69-4e5f-9563-0c7f506e6e3a" />

**Key-takeaway:**
The funnel was stable across months: ~28–42% from product views to addtocart and ~8–15% of views convert to purchase.

### 🔍 Query 3. Bounce rate per traffic source in July 2017

This query is to measure user engagement per traffic source. Which channel had the most visits and low bounce (higher engagement)? Which ones had poor engagement?

🚀 **Query**

```sql
SELECT 
  trafficSource.source AS source
  ,SUM(totals.visits) AS total_visits
  ,SUM(totals.bounces) AS total_no_of_bounces
  ,ROUND(100.0*SUM(totals.bounces)/SUM(totals.visits), 3) AS bounce_rate --Bounce_rate = num_bounce/total_visit
  
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` --July 2017

GROUP BY source
ORDER BY bounce_rate DESC;
```

💡**Query result**

<img width="574" height="568" alt="image" src="https://github.com/user-attachments/assets/dd657a27-fcee-419d-b16b-86cf311b4c40" />

**Key-takeaway:**
In July, Google and (direct) had the most visits with mid-range bounce (~52% and 43%), while youtube.com brought sizable traffic but poor quality (~67% bounce). Other Email/referral sources like reddit.com (~29%), mail.google.com (~25% bounce) or blog.golang.org (~29%) showed the healthiest engagement.

### 🔍 Query 4. Revenue by traffic source in July 2017

This query result will identify which souces bring the most (and the least) revenue in July 2017

🚀 **Query**

```sql
WITH 
base AS ( --Check date string, calculate revenue by traffic source and by date
  SELECT
    PARSE_DATE('%Y%m%d', date) AS parsed
    ,trafficSource.source AS source
    ,SUM(product.productRevenue)/1000000 AS revenue --productRevenue is divided by 1000000 to shorten the result
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` --July 2017
    ,UNNEST (hits) hits
    ,UNNEST(hits.product) product
  WHERE product.productRevenue IS NOT NULL 
  GROUP BY parsed, source
)

--Format and round the Revenue
SELECT 
  FORMAT_DATE('%Y%m', parsed) AS month
  ,source
  ,ROUND(SUM(revenue), 4) AS revenue
FROM base
GROUP BY month, source
ORDER BY month, revenue DESC;
```
💡**Query result**

<img width="527" height="352" alt="image" src="https://github.com/user-attachments/assets/87880d26-1ebc-48b0-9d04-4c3d9645b09c" />

**Key-takeaway:**
In July, most of revenue came from (direct) and Google

### 🔍 Query 5. Average number of pageviews by purchaser type (purchasers vs non-purchasers) in July 2017.

This is to evaluate user behaviour between purchasers and non-purchasers in order to see whether purchasers ten to view more before purchasing.

🚀 **Query**

```sql
WITH 
purchaser AS (--Average number of pageviews by purchaser
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
    ,ROUND(SUM(totals.pageviews) / COUNT(DISTINCT fullVisitorId), 7) AS avg_pageviews_purchase 
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` 
    ,UNNEST (hits) hits
    ,UNNEST(hits.product) product
  WHERE totals.transactions >=1
  AND productRevenue IS NOT NULL
  GROUP BY month
)

,non_purchaser AS (--Average number of pageviews by non-purchaser
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
    ,ROUND(SUM(totals.pageviews) / COUNT(DISTINCT fullVisitorId), 7) AS avg_pageviews_non_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` 
    ,UNNEST (hits) hits
    ,UNNEST(hits.product) product
  WHERE totals.transactions IS NULL
  AND productRevenue IS NULL
  GROUP BY month
)

SELECT p.month, p.avg_pageviews_purchase, n.avg_pageviews_non_purchase
FROM purchaser p
FULL JOIN non_purchaser n
ON p.month = n.month
ORDER BY p.month;
Non-purchasers view far more pages than purchasers (~334 vs ~124/pageviews per user), hinting at friction or dead-ends before checkout
```

💡**Query result**

<img width="593" height="72" alt="image" src="https://github.com/user-attachments/assets/6f94631b-4144-4bf2-8b81-b273e95b3c0f" />

**Key-takeaway:**
Non-purchasers' pageviews were far more than purchasers' (~334 vs ~124 pageviews per user), hinting at friction or dead-ends before checkout.

### 🔍 Query 6. Average transactions per purchasing user in July 2017 

This step aims to measure buyers' purchase frequency and their loyalty. Company marketing teams can use the result for site performance and marketing strategies.

🚀 **Query**

```sql
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
  --avg transaction per user = total transaction / total visitor
  ,ROUND(SUM(totals.transactions) / COUNT(DISTINCT fullVisitorId), 7) AS Avg_total_transactions_per_user 

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` --July 2017
  ,UNNEST (hits) hits
  ,UNNEST(hits.product) product
WHERE totals.transactions >=1
AND productRevenue IS NOT NULL
GROUP BY month;
```

💡**Query result**

<img width="387" height="69" alt="image" src="https://github.com/user-attachments/assets/301504ed-899a-451c-b814-d4464878d2a2" />

### 🔍 Query 7. Average amount of money spent per session in July 2017

This query result shows how much each customer is worth on average to help company optimize pricing, marketing spend or have strategies to increase revenue per customer

🚀 **Query**

```sql
SELECT  
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
  -- avg_revenue_per_session = total revenue/ total visit
  ,ROUND((SUM(productRevenue) / SUM(totals.visits)) / 1000000, 2) AS avg_revenue_by_user_per_visit
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` --July 2017
    ,UNNEST (hits) hits
    ,UNNEST(hits.product) product
WHERE totals.transactions >=1
AND productRevenue IS NOT NULL
GROUP BY month;
```

💡**Query result**

<img width="373" height="71" alt="image" src="https://github.com/user-attachments/assets/9ea99366-7730-4898-aa06-3c70d3346805" />

### 🔍 Query 8. Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered.

This query reveals which products are frequently bought together with product "YouTube Men's Vintage Henley" in July 2017, helping optimizing cross-sell offers, bundles or personalized recommendations on the site

🚀 **Query**

```sql
WITH buyer_list AS(
  SELECT
    DISTINCT fullVisitorId  
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) AS product
  WHERE product.v2ProductName = "YouTube Men's Vintage Henley"
  AND totals.transactions>=1
  AND product.productRevenue IS NOT NULL 
)

SELECT
  product.v2ProductName AS other_purchased_products,
  SUM(product.productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
, UNNEST(hits) AS hits
, UNNEST(hits.product) AS product
JOIN buyer_list USING(fullVisitorId)
WHERE product.v2ProductName != "YouTube Men's Vintage Henley"
 AND product.productRevenue IS NOT NULL 
 AND totals.transactions>=1
GROUP BY other_purchased_products
ORDER BY quantity DESC;
```

💡**Query result**

<img width="427" height="568" alt="image" src="https://github.com/user-attachments/assets/685078a1-db4d-4a07-a70d-d63640f2b310" />

**Key-takeaway:**
Buyers of *YouTube Men’s Vintage Henley* frequently also purchased *Google Sunglasses* (20), *Women’s Vintage Hero Tee Black* (7), and *SPF-15 & Lip Balm* (6) etc.


## IV. INSIGHTS AND RECOMMENDATIONS

1.  **Direct and Google brought the strongest revenue contribution**, while some traffic sources generated visits but lower business value. <br/>
--> **Recommendation:** focus more on high-value channels and review lower-quality traffic sources before increasing budget.

2. **The largest drop happened early in the funnel, from product view to add-to-cart.**
--> **Recommendation:** improve product pages with clearer product information, stronger calls to action, and a smoother shopping experience. <br/>

3. **Non-purchasers browsed more than purchasers**, which suggests friction in product discovery or decision-making. <br/>
--> **Recommendation:** improve site search, filters, and category navigation to help users find products faster.

4. **Higher traffic did not always lead to better conversion.** Some months had more visits, but weaker conversion efficiency. <br/>
--> **Recommendation:** optimize for conversion quality, not only traffic volume.

5. **Some products were often bought together, showing cross-sell potential.**  <br/>
--> **Recommendation:** use product bundles and “frequently bought together” suggestions to increase average order value.


