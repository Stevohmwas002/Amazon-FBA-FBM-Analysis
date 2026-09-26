# Data Dictionary

Covers two layers: the **source tables** (raw synthetic data feeding Power BI) and the **calculated measures** (DAX) shown on each dashboard page.

---

## 1. Source Tables

### `orders.csv` — order-line level, one row per unit sold

| Field | Type | Description |
|---|---|---|
| `order_id` | Integer | Unique order-line identifier |
| `order_date` | Date | Date the order was placed |
| `customer_id` | Text | Foreign key to `customers.csv` |
| `is_repeat_purchase` | Boolean | `True` if this is not the customer's first order |
| `sku` | Text | Foreign key to `sku_catalog.csv` |
| `product_name` | Text | Denormalized product name |
| `category` | Text | Product category |
| `fulfillment_channel` | Text | `FBA` or `FBM` |
| `units` | Integer | Units in this order line (occasional bulk/whale orders of 15–60) |
| `revenue` | Decimal | Gross revenue for the line (price × units, with minor price noise) |
| `cogs` | Decimal | Cost of goods sold for the line |
| `fba_fee` | Decimal | Amazon fulfillment fee (FBA orders only; 0 for FBM) |
| `referral_fee` | Decimal | Amazon referral fee (15% of revenue) |
| `shipping_cost` | Decimal | Merchant-paid shipping cost (FBM orders only; 0 for FBA) |
| `ad_attributed_order` | Boolean | Whether this order is attributed to an ad click |
| `refund_status` | Text | `Complete` or `Refunded` |
| `refund_amount` | Decimal | Dollar amount refunded (0 if not refunded) |
| `session_source` | Text | Traffic source for the session that led to this order |
| `region` | Text | US region (`~1%` intentionally null, simulating messy exports) |

### `customers.csv` — one row per customer

| Field | Type | Description |
|---|---|---|
| `customer_id` | Text | Unique customer identifier |
| `acquisition_channel` | Text | `Sponsored Ads`, `Organic Search`, `External Traffic`, or `Referral` |
| `signup_region` | Text | Region at signup |
| `loyalty_score` | Decimal (0–1) | Hidden trait driving repeat-purchase likelihood |
| `signup_date` | Date | Date the customer first signed up |
| `total_orders` | Integer | Lifetime order count |
| `total_units` | Integer | Lifetime units purchased |
| `total_revenue` | Decimal | Lifetime revenue from this customer |
| `total_refunded` | Decimal | Lifetime refunded amount |
| `first_order_date` | Date | Date of first order (null if never converted) |
| `last_order_date` | Date | Date of most recent order |
| `is_repeat_customer` | Boolean | `total_orders > 1` |
| `churned_90d` | Boolean / Null | No order in the trailing 90 days of the observation window; null if never converted |

### `ads_daily.csv` — SKU × day grain

| Field | Type | Description |
|---|---|---|
| `date` | Date | Calendar date |
| `sku` | Text | Foreign key to `sku_catalog.csv` |
| `campaign_type` | Text | `Sponsored Products`, `Sponsored Brands`, or `Sponsored Display` |
| `impressions` | Integer | Ad impressions served |
| `clicks` | Integer | Ad clicks |
| `spend` | Decimal | Ad spend for the day |
| `ad_orders` | Integer | Orders attributed to ads that day |
| `ad_sales` | Decimal | Revenue attributed to ads that day |

### `search_terms.csv` — SKU × keyword × week grain

| Field | Type | Description |
|---|---|---|
| `week_start` | Date | Week start date (Monday) |
| `sku` | Text | Foreign key to `sku_catalog.csv` |
| `search_term` | Text | Customer search keyword |
| `impressions` | Integer | Impressions for that keyword |
| `clicks` | Integer | Clicks for that keyword |
| `spend` | Decimal | Spend attributed to that keyword |
| `orders` | Integer | Orders attributed to that keyword |
| `sales` | Decimal | Revenue attributed to that keyword |

### `inventory_daily.csv` — SKU × day grain

| Field | Type | Description |
|---|---|---|
| `date` | Date | Calendar date |
| `sku` | Text | Foreign key to `sku_catalog.csv` |
| `fulfillment_channel` | Text | `FBA` or `FBM` |
| `units_available` | Integer | Units in stock that day |
| `days_of_supply` | Decimal | `units_available / avg_daily_sales` |
| `stockout_flag` | Boolean | `True` if `units_available == 0` |
| `storage_age_days` | Integer | Days the current inventory batch has been in storage |
| `overstock_flag` | Boolean | `True` if `storage_age_days > 90` |
| `inbound_shipment_defects` | Integer | Damaged/lost units received that day |
| `buy_box_pct` | Decimal / Null | Buy Box win rate (FBM only; null for FBA) |

### `reviews_weekly.csv` — SKU × week grain

| Field | Type | Description |
|---|---|---|
| `week_start` | Date | Week start date |
| `sku` | Text | Foreign key to `sku_catalog.csv` |
| `new_reviews` | Integer | New reviews that week |
| `cumulative_reviews` | Integer | Running total of reviews |
| `avg_rating` | Decimal | Average star rating (drifts weekly) |

### `sku_catalog.csv` — reference table

| Field | Type | Description |
|---|---|---|
| `sku` | Text | Unique SKU identifier |
| `name` | Text | Product name |
| `category` | Text | Product category |
| `channel` | Text | Primary fulfillment channel (`FBA` or `FBM`) |
| `price` | Decimal | List price |
| `cogs` | Decimal | Cost of goods sold per unit |

---

## 2. Calculated Measures (by dashboard page)

### Page 1 — Profit & Sales

| Measure | Formula | Notes |
|---|---|---|
| Gross Revenue | `SUM(orders[revenue])` | Before any fees |
| Net Profit | `Gross Revenue − COGS − FBA Fees − Shipping − Ad Spend − Refunds` | |
| Profit Margin % | `Net Profit / Gross Revenue` | |
| Return On Investment | `Net Profit / (COGS + FBA Fees + Shipping + Ad Spend)` | |
| TACOS (Total Advertising Cost of Sales) | `Total Ad Spend / Gross Revenue` | Uses *total* sales, not just ad sales |
| Refund Rate % | `SUM(orders[refund_amount]) / Gross Revenue` | |

### Page 2 — Advertising / PPC

| Measure | Formula | Notes |
|---|---|---|
| Total Spent on Ads | `SUM(ads_daily[spend])` | |
| Ad-Driven Revenue % | `SUM(ads_daily[ad_sales]) / Gross Revenue` | |
| ACOS | `Ad Spend / Ad Sales` | Lower is better |
| Return of Advertising Spent (ROAS-style) | `Ad Sales / Ad Spend` | Shown here as a % return, not a multiple — confirm this convention before comparing to industry-standard ROAS multiples |
| CTR | `Clicks / Impressions` | |
| CVR (as used here) | `Ad Orders / Clicks` | |

### Page 3 — Inventory & Operations

| Measure | Formula | Notes |
|---|---|---|
| Inventory Turnover | `Annual COGS / Average Inventory Value` | Average, not a single-day snapshot |
| Days of Supply | `Units Available / Average Daily Sales` | Alert threshold: <30 days |
| Stockout Rate % | `COUNTROWS(stockout days) / COUNTROWS(total days)` | Per SKU or store-wide |
| Overstock % | `COUNTROWS(storage_age_days > 90) / Total SKU-days` | |
| Total Inventory Value | `SUMX(FILTER(inventory_daily, date = MAX(date)), units_available × cogs)` | Must filter to a single snapshot date — see Known Issues in README |
| FBA Fee % by Category | `SUM(orders[fba_fee]) / SUM(orders[revenue])`, by category | |

### Page 4 — Customers

| Measure | Formula | Notes |
|---|---|---|
| Customer Acquisition Cost (CAC) | `Total Ad Spend / New Customers Acquired (same period)` | |
| Customer Lifetime Value (CLV) | `AVERAGEX(customers, customers[total_revenue])` | Must be filtered per-customer, not summed across the full table — see Known Issues in README |
| Repurchase Rate | `COUNTROWS(FILTER(customers, total_orders > 1)) / COUNTROWS(customers)` | |
| Churn Rate (30d) | `COUNTROWS(FILTER(customers, churned in last 30 days)) / Total Customers` | Distinct from the `churned_90d` field in the raw table, which uses a 90-day window |
| Total Churn | `COUNTROWS(FILTER(customers, churned_flag = TRUE))` | |

---

## Glossary of Abbreviations

| Term | Meaning |
|---|---|
| FBA | Fulfillment by Amazon |
| FBM | Fulfillment by Merchant |
| COGS | Cost of Goods Sold |
| TACOS | Total Advertising Cost of Sales |
| ACOS | Advertising Cost of Sale |
| ROAS | Return on Ad Spend |
| CTR | Click-Through Rate |
| CVR | Conversion Rate |
| CAC | Customer Acquisition Cost |
| CLV / LTV | Customer Lifetime Value |
| OMTM | One Metric That Matters |
| MAD | Mean Absolute Deviation |
| IQR | Interquartile Range |
| AIC | Akaike Information Criterion |
| ROC / AUC | Receiver Operating Characteristic / Area Under the Curve |
