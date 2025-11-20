# Ecommerce Web Performance & Customer Purchasing Behaviour Analysis | SQL, BigQuery
I analyzed Ecommerce dataset by using SQL in BigQuery to extract key insights about web performance, revenue trends, bounce rate, etc. to improve channel quality, cohort funnel and cross-sell

## I. PROJECT OVERVIEW
This project contains an eCommerce dataset that I will explore using SQL on Google BigQuery. 
**Business goals:** 
This project analyzes Google Analytics 360 e-commerce sessions to find practical levers for **growing monthly revenue**. I compare baseline performance and seasonality (visits, pageviews, transactions), with focus on July, and evaluate **channel quality** and **onsite funnel efficiency** using bounce rate, revenue per session, and the view → add-to-cart → purchase path. The analysis identifies which traffic sources to scale or fix, where users drop off, the expected AOV/upsell potential, and **cross-sell pairs** for “frequently bought together”, turning raw GA data into concrete actions for acquisition, conversion, and merchandising.

## II. DATASET
**Dataset access:**

This eCommerce dataset is stored in a public Google BigQuery dataset. To access the dataset, follow these steps:
- Log in to your Google Cloud Platform account
- Get data through:
  - this link https://console.cloud.google.com/bigquery?ws=!1m5!1m4!4m3!1sbigquery-public-data!2sgoogle_analytics_sample!3sga_sessions_20170801
  - Or create a new project in BigQuery -> select "Add Data" -> "Search a project" -> Enter the project ID "bigquery-public-data.google_analytics_sample.ga_sessions" -> "Enter" -> Click on the "ga_sessions_" table to open it

**Dataset description:** 

The dataset is based on the Google Analytics public dataset and contains data from an eCommerce website. (`bigquery-public-data.google_analytics_sample.ga_sessions_20170801` )
The sample dataset contains obfuscated Google Analytics 360 data from the [Google Merchandise Store](https://www.googlemerchandisestore.com/shop.axd/Home?utm_source=Partners&utm_medium=affiliate&utm_campaign=Data%20Share%20Promo), a real ecommerce store. The Google Merchandise Store sells Google branded merchandise. The data is typical of what you would see for an ecommerce website. 

**Data description table:**

Below are the fields, data type, description of the fields I used in this SQL project

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

## III. EXPLORE DATASET
This project includes 8 queries

### Query 1. Calculate total visit, pageview, transaction for January-August 2017 (order by month).
**SQL code**
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

**Query result**

<img width="911" height="305" alt="image" src="https://github.com/user-attachments/assets/c93ba95c-696d-49ca-97ce-3aeb22f21db3" />

**Key-takeaway:**
May had the highest conversion rate (1.77%) while July brought peak volume (71.8k visits, 270k page views) but a relatively low conversion rate (1.49%).

### Query 2. Calculate cohort map from product view to addtocart to purchase in 2017 (January-August).
**SQL code**
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

**Query result**

<img width="1031" height="307" alt="image" src="https://github.com/user-attachments/assets/d81db07a-af69-4e5f-9563-0c7f506e6e3a" />

**Key-takeaway:**
The funnel was stable across months: ~28–42% from product views to addtocart and ~8–15% of views convert to purchase.

### Query 3. Bounce rate per traffic source in July 2017
**SQL code**
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

**Query result**

<img width="574" height="568" alt="image" src="https://github.com/user-attachments/assets/dd657a27-fcee-419d-b16b-86cf311b4c40" />

**Key-takeaway:**
In July, Google and (direct) had the most visits with mid-range bounce (~52% and 43%), while youtube.com brought sizable traffic but poor quality (~67% bounce). Other Email/referral sources like reddit.com (~29%), mail.google.com (~25% bounce) or blog.golang.org (~29%) showed the healthiest engagement.

### Query 4. Revenue by traffic source in July 2017
**SQL code**
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
**Query result**

<img width="527" height="352" alt="image" src="https://github.com/user-attachments/assets/87880d26-1ebc-48b0-9d04-4c3d9645b09c" />

**Key-takeaway:**
In July, most of revenue came from (direct) and Google

### Query 5. Average number of pageviews by purchaser type (purchasers vs non-purchasers) in July 2017.
**SQL code**
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

**Query result**

<img width="593" height="72" alt="image" src="https://github.com/user-attachments/assets/6f94631b-4144-4bf2-8b81-b273e95b3c0f" />

**Key-takeaway:**
Non-purchasers' pageviews were far more than purchasers' (~334 vs ~124 pageviews per user), hinting at friction or dead-ends before checkout.

### Query 6. Average transactions per purchasing user in July 2017 
**SQL code**
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

**Query result**

<img width="387" height="69" alt="image" src="https://github.com/user-attachments/assets/301504ed-899a-451c-b814-d4464878d2a2" />

### Query 7. Average amount of money spent per session in July 2017
**SQL code**
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

**Query result**

<img width="373" height="71" alt="image" src="https://github.com/user-attachments/assets/9ea99366-7730-4898-aa06-3c70d3346805" />

### Query 8. Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered.
**SQL code**
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

**Query result**

<img width="427" height="568" alt="image" src="https://github.com/user-attachments/assets/685078a1-db4d-4a07-a70d-d63640f2b310" />

**Key-takeaway:**
Buyers of *YouTube Men’s Vintage Henley* frequently also purchased *Google Sunglasses* (20), *Women’s Vintage Hero Tee Black* (7), and *SPF-15 & Lip Balm* (6) etc.


## IV. INSIGHTS
**Business goal:** Grow monthly revenue by improving channel quality, onsite funnel, and cross-sell.

**Baseline health check (January-August 2017)**

- May had highest conversion rate (1,160 txns, 1.77% conversion rate), while July had highest visits and pageviews (71.8k visits, 270k pageviews, 1.49% conversion rate). Recommend to **set a goal to lift July’s conversion rate to May’s level**.
- Cohort map funnel was consistent through the months, ~38–42% of product views add to cart and ~8–13% convert to purchase. Thus, the biggest loss was **View → Add to Cart**. Recommend to **improve product detail page**, for example, stronger CTA, size/fit guide, faster images, better navigation etc.

**July in focus**

- Revenue mainly came from **direct (~79%)** and **Google search (~20%)**
  - Recommendations:
        - Add standard query parameters to your links so analytics can attribute traffic and revenue to the right campaigns, instead of tagging them all into “(direct)”.
        - Scale search/owned demand, and either fix or downweight YouTube/m.facebook (high bounce, low revenue) with intent-matched landing pages and Revenue per Session targets.
- Non-purchasers browsed more than purchasers (~334 vs ~124 pageviews per user), signaling friction. Recommend to improve filters, breadcrumbs, and zero-results search fixes to shorten paths to cart.
- In July, a “purchasing user” bought **~4.16 times** (across the month), and each **purchasing session** averages **~$43.86** in revenue. That meant customers already showed **repeat-buy behavior** and a **basket value** just under $45.
    - Recommend to set **free shipping at ~$49–$55** (≈10–20% above $43.86) to nudge carts slightly higher without scaring buyers. Also, **launch bundles** because repeat buyers are primed to add one more complementary item, which increases attach rate (orders with a bundle/add-on) and average order value (AOV). We then can track and monitor AOV, attach rate and checkout conversion. If conversion drops, adjust the threshold or bundle pricing.
- Launch data-backed cross-sell on the *YouTube Men’s Vintage Henley*  Product Detail Page. The data shows Henley buyers often also purchase products like *Google Sunglasses*, *Women’s Vintage Hero Tee Black*, and *SPF-15 Slim & Slender Lip Balm*. Thus, surface these as *“Frequently bought together”*, reinforce in the cart, and follow up in post-purchase/email. The goal is to increase attach rate (the % of orders with an add-on) and Average order value (AOV).
- **Add checkout accelerators**, for example, guest checkout, popular payment methods, address autofill etc. **Target a +2–3 percentage points lift** in **ATC→Purchase** to compound the July gains. July already had big traffic, so turning a few more carts into orders multiplies revenue without needing more visitors.
