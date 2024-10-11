# Marketing-Analytics

# E-commerce Analytics with Google Analytics 4 (GA4) and BigQuery

This repository contains SQL queries and Python code for analyzing and visualizing e-commerce data using Google Analytics 4 (GA4) data in BigQuery.

## Table of Contents
1. [E-commerce Funnel Analysis](#e-commerce-funnel-analysis)
2. [Funnel Visualization](#funnel-visualization)
3. [Additional E-commerce Metrics](#additional-e-commerce-metrics)
   - [Conversion Rate](#conversion-rate)
   - [Average Order Value (AOV)](#average-order-value-aov)
   - [Customer Lifetime Value (CLV)](#customer-lifetime-value-clv)
   - [Cart Abandonment Rate](#cart-abandonment-rate)
   - [Return on Ad Spend (ROAS)](#return-on-ad-spend-roas)

## E-commerce Funnel Analysis

The following SQL query creates an e-commerce funnel using GA4 data in BigQuery:

```sql
-- E-commerce Funnel Analysis using GA4 data in BigQuery
WITH funnel_stages AS (
  SELECT
    user_pseudo_id,
    DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
    MAX(CASE WHEN event_name = 'page_view' AND page_location LIKE '%/product%' THEN 1 ELSE 0 END) AS product_view,
    MAX(CASE WHEN event_name = 'add_to_cart' THEN 1 ELSE 0 END) AS add_to_cart,
    MAX(CASE WHEN event_name = 'begin_checkout' THEN 1 ELSE 0 END) AS begin_checkout,
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS purchase
  FROM
    `your_project_id.your_dataset_id.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN '20240101' AND '20240331'
  GROUP BY
    user_pseudo_id,
    event_date
)

SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS total_users,
  SUM(product_view) AS product_views,
  SUM(add_to_cart) AS add_to_carts,
  SUM(begin_checkout) AS begin_checkouts,
  SUM(purchase) AS purchases
FROM
  funnel_stages
GROUP BY
  event_date
ORDER BY
  event_date
```

This query analyzes the main stages of the e-commerce funnel: product view, add to cart, begin checkout, and purchase. It provides daily aggregates for each stage.

## Funnel Visualization

Visualize the e-commerce funnel:

```python
import plotly.graph_objects as go
import pandas as pd

# Sample data (replace this with your actual data from BigQuery)
data = {
    'Stage': ['Product Views', 'Add to Cart', 'Begin Checkout', 'Purchase'],
    'Users': [10000, 3000, 1500, 1000]
}

df = pd.DataFrame(data)

fig = go.Figure(go.Funnel(
    y = df['Stage'],
    x = df['Users'],
    textinfo = "value+percent initial"
))

fig.update_layout(
    title = "E-commerce Funnel Visualization",
    width = 800,
    height = 500
)

fig.show()
```

To use this script:
1. Install the required libraries: `pip install plotly pandas`
2. Replace the sample data with your actual data from BigQuery
3. Run the script to generate an interactive funnel chart

## E-commerce Metrics

### Conversion Rate

```sql
SELECT
  DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
  COUNT(DISTINCT CASE WHEN event_name = 'purchase' THEN user_pseudo_id END) / 
    COUNT(DISTINCT user_pseudo_id) AS conversion_rate
FROM
  `your_project_id.your_dataset_id.events_*`
WHERE
  _TABLE_SUFFIX BETWEEN '20240101' AND '20240331'
GROUP BY
  event_date
ORDER BY
  event_date
```

### Average Order Value (AOV)

```sql
SELECT
  DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
  SUM(ecommerce.purchase_revenue) / COUNT(DISTINCT ecommerce.transaction_id) AS aov
FROM
  `your_project_id.your_dataset_id.events_*`
WHERE
  _TABLE_SUFFIX BETWEEN '20240101' AND '20240331'
  AND event_name = 'purchase'
GROUP BY
  event_date
ORDER BY
  event_date
```

### Customer Lifetime Value (CLV)

```sql
WITH customer_purchases AS (
  SELECT
    user_pseudo_id,
    SUM(ecommerce.purchase_revenue) AS total_revenue,
    MIN(DATE(TIMESTAMP_MICROS(event_timestamp))) AS first_purchase_date,
    MAX(DATE(TIMESTAMP_MICROS(event_timestamp))) AS last_purchase_date
  FROM
    `your_project_id.your_dataset_id.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN '20240101' AND '20240331'
    AND event_name = 'purchase'
  GROUP BY
    user_pseudo_id
)

SELECT
  AVG(total_revenue) AS avg_clv,
  AVG(DATE_DIFF(last_purchase_date, first_purchase_date, DAY)) AS avg_customer_lifespan
FROM
  customer_purchases
```

### Cart Abandonment Rate

```sql
WITH cart_actions AS (
  SELECT
    user_pseudo_id,
    DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
    MAX(CASE WHEN event_name = 'add_to_cart' THEN 1 ELSE 0 END) AS added_to_cart,
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS purchased
  FROM
    `your_project_id.your_dataset_id.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN '20240101' AND '20240331'
  GROUP BY
    user_pseudo_id,
    event_date
)

SELECT
  event_date,
  1 - (SUM(purchased) / SUM(added_to_cart)) AS cart_abandonment_rate
FROM
  cart_actions
WHERE
  added_to_cart = 1
GROUP BY
  event_date
ORDER BY
  event_date
```

### Return on Ad Spend (ROAS)

```sql
WITH ad_costs AS (
  SELECT
    DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
    SUM(CAST(event_params.value.string_value AS FLOAT64)) AS total_cost
  FROM
    `your_project_id.your_dataset_id.events_*`,
    UNNEST(event_params) AS event_params
  WHERE
    _TABLE_SUFFIX BETWEEN '20240101' AND '20240331'
    AND event_name = 'ad_impression'
    AND event_params.key = 'ad_cost'
  GROUP BY
    event_date
),

ad_revenue AS (
  SELECT
    DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
    SUM(ecommerce.purchase_revenue) AS total_revenue
  FROM
    `your_project_id.your_dataset_id.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN '20240101' AND '20240331'
    AND event_name = 'purchase'
  GROUP BY
    event_date
)

SELECT
  ad_costs.event_date,
  ad_revenue.total_revenue / ad_costs.total_cost AS roas
FROM
  ad_costs
JOIN
  ad_revenue
ON
  ad_costs.event_date = ad_revenue.event_date
ORDER BY
  ad_costs.event_date
```

Remember to replace `your_project_id` and `your_dataset_id` with your actual Google Cloud project ID and BigQuery dataset ID.
