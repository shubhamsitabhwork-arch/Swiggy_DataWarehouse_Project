# Swiggy Data Warehouse Design

## 📖 Project Overview
This repository contains the design, architecture, and SQL implementations for a Swiggy-style on-demand food delivery Data Warehouse. The business context captures the full consumer and hyperlocal delivery lifecycle: customers browse restaurants, view menus, add items to cart, place orders, and make payments. On the operational side, restaurants accept and prepare orders, delivery partners (riders) are assigned, pick up orders, and fulfill deliveries.

Revenue is driven primarily by the platform take rate on item totals plus platform and delivery fees, minus customer refunds and adjustments. The analytical model focuses on key operational and financial levers including on-time delivery performance, cancellations, order preparation SLAs, rider utilization, and customer retention.

## 🗂️ Repository Structure
The repository is structured to separate documentation, ER diagrams, and project assets cleanly:
* **`01_Docs/`**: Contains core documentation, data warehouse specifications, metric definitions, and SQL queries (`Swiggy_DWH.pdf`).
* **`02_Architecture/`**: Houses high-resolution entity-relationship (ER) diagrams and dimensional models (`SwiggyDWH.jpg`).
* **`03_Recordings/`**: Dedicated folder for pipeline walk-throughs, architecture explanations, or demo videos.
* **`04_Assets/`**: Stores static assets, icons, and report mockups.

---

## 🏗️ Data Architecture (Star Schema)

### Dimension Tables (Context & Attributes)
* **`dim_date`**: Calendar dimension containing 1 row per day, storing date attributes like year, quarter, month, week of year, day of week, and period milestone flags (`is_weekend`, `is_month_start`, `is_month_end`, etc.).
* **`dim_time`**: Granular time dimension covering 24-hour intervals down to the minute/second with day-part classifications (`am_pm`, `hour_24`, `day_part`).
* **`dim_geo`**: Geographical attributes mapped down to the city and market zone, storing coordinates (`latitude`, `longitude`), administrative boundaries, and timezone (`tz_name`)[cite: 4].
* **`dim_customer`**: Slowly Changing Dimension (SCD Type 2) managing customer identity, personal hashes, loyalty tier, marketing opt-ins, user segment, and risk status[cite: 4].
* **`dim_device`**: Captures client technical environments including device type, operating system, OS version, application vs. web modality, and browser builds[cite: 4].
* **`dim_channel`**: Tracks acquisition and marketing attribution parameters (`source`, `medium`, `campaign`, `ad_group`, `ad_id`, UTM tracking, partner identifiers)[cite: 4].
* **`dim_restaurant`**: SCD Type 2 dimension storing restaurant operational profiles, brand names, primary cuisine, customer ratings, commission bands, chain status, opening hours, and location foreign keys (`geo_sk`)[cite: 4].
* **`dim_menu_item`**: SCD Type 2 dimension cataloging dish items, linked restaurant foreign keys, dish categories, food preference flags (`veg_flag`), spice levels, calories, allergens, and base pricing[cite: 4].
* **`dim_delivery_partner`**: SCD Type 2 dimension profiling delivery partners (riders), containing masked contact details, vehicle specifications, licensing, joining dates, and home markets (`geo_sk`)[cite: 4].
* **`dim_payment_method`**: Stores payment channel profiles, card networks, digital wallet providers, issuing country, and 3D Secure capability[cite: 4].
* **`dim_promo`**: Manages promotional campaigns, discount structures, promotional valid date ranges, and eligibility targeting rules[cite: 4].
* **`dim_fee_policy`**: Manages dynamic fee evaluation rules, platform charging schemes, calculation logic, and parameter configurations stored in JSON[cite: 4].
* **`dim_currency_rate`**: Currency conversion table tracking foreign exchange conversions against the reporting currency as of a specific timestamp (`fx_to_report`, `as_of_ts`)[cite: 4].

### Fact Tables (Events & Transactions)
* **`fact_app_session`**: Tracks individual browsing sessions, aggregating session counts, screen impressions, add-to-cart milestones, and checkout actions[cite: 4].
* **`fact_menu_impression`**: Granular logging of restaurant dish listings displayed to users, tracking price points presented and attribution sources[cite: 4].
* **`fact_add_to_cart`**: Item-level basket creation events storing added quantities and line item prices at time of addition (`item_price_atc`)[cite: 4].
* **`fact_order_header`**: Central transactional fact recording placed orders, gross merchandise value (`gmv_amount`), merchant/platform discounts, fees collected, fulfillment timestamps, take-rate percentages, and final status[cite: 4].
* **`fact_order_item`**: Order line-item breakdown storing quantities, line pricing, applied discounts, add-on costs, dish categories, and dietary flags[cite: 4].
* **`fact_assignment`**: Tracks fulfillment offers sent to restaurant partners, recording offer dispatch and acceptance timestamps (`accepted_flag`, `accept_ts`)[cite: 4].
* **`fact_delivery_trip`**: Detailed breakdown of the delivery journey, capturing dispatch, arrival at merchant, order pickup, dropoff dispatch, completion, mileage, trip durations, and surge multipliers[cite: 4].
* **`fact_cancellation`**: Itemizes canceled orders, capturing the initiating cancellation party (merchant, customer, or rider), reason taxonomy, and assessed penalties[cite: 4].
* **`fact_payment`**: Logs financial processing events including transaction authorizations, captures, processing types, and total paid values[cite: 4].
* **`fact_refund`**: Financial settlement ledger tracking customer adjustments, refund amounts, reason codes, and dispute/chargeback markers[cite: 4].
* **`fact_restaurant_availability`**: Aggregates daily merchant operational availability, total online minutes, outage durations, and uptime percentages[cite: 4].
* **`fact_rider_shift`**: Evaluates rider shift productivity, tracking total scheduled shift length, active on-trip seconds, and idle idle-waiting duration[cite: 4].

---

## 📊 SQL Implementations for Business Metrics

### 1) App-to-Order Conversion Rate
**Definition:** The percentage of unique application sessions that result in at least one completed order within the reporting period[cite: 4].
```sql
WITH s AS (
    SELECT COUNT(DISTINCT session_id) AS sessions
    FROM fact_app_session fs
    JOIN dim_date d ON fs.date_key = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
o AS (
    SELECT COUNT(DISTINCT oh.order_id_nk) AS orders
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
      AND oh.status IN ('DELIVERED', 'PLACED', 'ACCEPTED')
)
SELECT
    s.sessions,
    o.orders,
    ROUND(o.orders * 100.0 / NULLIF(s.sessions, 0), 2) AS app_to_order_conversion_percent
FROM s CROSS JOIN o;
```

### 2) Average Order Value (AOV)
**Definition:** The average pre-discount basket size for completed deliveries across the period[cite: 4].
```SQL
SELECT
    COUNT(*) AS orders,
    SUM(oh.gmv_amount) AS total_gmv,
    ROUND(SUM(oh.gmv_amount) / NULLIF(COUNT(*), 0), 2) AS average_order_value
FROM fact_order_header oh
JOIN dim_date d ON oh.date_key_ordered = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
  AND oh.status = 'DELIVERED';
```

### 3) Platform Take Rate %
**Definition:** The share of net commercial revenue captured by the platform relative to total Gross Merchandise Value (GMV)[cite: 4].
```SQL
WITH base AS (
    SELECT
        SUM(oh.gmv_amount) AS gmv,
        SUM(oh.gmv_amount * oh.take_rate_pct + oh.platform_fee + oh.delivery_fee) 
        - COALESCE(SUM(rf.refund_amount), 0) AS net_revenue
    FROM fact_order_header oh
    LEFT JOIN fact_refund rf ON rf.order_id_nk = oh.order_id_nk
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
      AND oh.status IN ('DELIVERED')
)
SELECT
    gmv,
    net_revenue,
    ROUND(net_revenue * 100.0 / NULLIF(gmv, 0), 2) AS platform_take_rate_percent
FROM base;
```

### 4) On-Time Delivery Rate (OTD)
**Definition:** The proportion of delivered orders where the actual dropoff timestamp was on or before the promised delivery window[cite: 4].
```SQL
SELECT
    COUNT(*) AS delivered_orders,
    SUM(CASE WHEN oh.actual_dropoff_ts <= oh.promised_dropoff_ts THEN 1 ELSE 0 END) AS on_time_orders,
    ROUND(
        SUM(CASE WHEN oh.actual_dropoff_ts <= oh.promised_dropoff_ts THEN 1 ELSE 0 END) * 100.0 
        / NULLIF(COUNT(*), 0), 2
    ) AS on_time_delivery_percent
FROM fact_order_header oh
JOIN dim_date d ON oh.date_key_ordered = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
  AND oh.status = 'DELIVERED';
```

### 5) Order Cancellation Rate
**Definition:** The percentage of placed orders that result in cancellation by any party (restaurant, rider, or customer)[cite: 4].
```SQL
WITH placed AS (
    SELECT COUNT(*) AS placed_orders
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
cancelled AS (
    SELECT COUNT(DISTINCT c.order_id_nk) AS cancelled_orders
    FROM fact_cancellation c
    JOIN dim_date d ON c.date_key_cancel = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    placed_orders,
    cancelled_orders,
    ROUND(cancelled_orders * 100.0 / NULLIF(placed_orders, 0), 2) AS cancellation_rate_percent
FROM placed CROSS JOIN cancelled;
```
### 6) Refund Rate
**Definition:** The proportion of placed orders that had any refund (full or partial) issued[cite: 4].
```SQL
WITH placed AS (
    SELECT COUNT(DISTINCT oh.order_id_nk) AS placed_orders
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
ref AS (
    SELECT COUNT(DISTINCT rf.order_id_nk) AS refunded_orders
    FROM fact_refund rf
    JOIN dim_date d ON rf.date_key_refund = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    placed_orders,
    refunded_orders,
    ROUND(refunded_orders * 100.0 / NULLIF(placed_orders, 0), 2) AS refund_rate_percent
FROM placed CROSS JOIN ref;
```

### 7) Repeat Customer Rate (90-Day Lookback)
**Definition:** The percentage of active customers with two or more completed orders over the past 90 days[cite: 4].
```SQL
WITH cust_orders AS (
    SELECT 
        oh.customer_sk, 
        COUNT(DISTINCT oh.order_id_nk) AS orders_90d
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN DATE_SUB(CURRENT_DATE(), 90) AND CURRENT_DATE()
      AND oh.status = 'DELIVERED'
    GROUP BY oh.customer_sk
)
SELECT
    COUNT(*) AS active_customers_90d,
    SUM(CASE WHEN orders_90d >= 2 THEN 1 ELSE 0 END) AS repeat_customers_90d,
    ROUND(
        SUM(CASE WHEN orders_90d >= 2 THEN 1 ELSE 0 END) * 100.0 
        / NULLIF(COUNT(*), 0), 2
    ) AS repeat_customer_rate_percent
FROM cust_orders;
```

### 8) Restaurant Acceptance Rate
**Definition:** The percentage of order dispatch requests successfully acknowledged and accepted by partner kitchens[cite: 4].
```SQL
SELECT
    r.restaurant_id_nk,
    r.brand,
    COUNT(*) AS total_offers,
    SUM(CASE WHEN a.accepted_flag = TRUE THEN 1 ELSE 0 END) AS accepted_offers,
    ROUND(
        SUM(CASE WHEN a.accepted_flag THEN 1 ELSE 0 END) * 100.0 
        / NULLIF(COUNT(*), 0), 2
    ) AS acceptance_rate_percent
FROM fact_assignment a
JOIN dim_restaurant r ON a.restaurant_sk = r.restaurant_sk AND r.is_current = TRUE
JOIN dim_date d ON a.date_key = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
GROUP BY r.restaurant_id_nk, r.brand
ORDER BY acceptance_rate_percent DESC;
```

### 9) Rider Utilization Rate
**Definition:** The percentage of total rider shift hours spent actively completing trips (to-pickup, at-pickup, to-dropoff)[cite: 4].
```SQL
SELECT
    COUNT(*) AS shifts,
    SUM(rs.shift_duration_sec) AS total_shift_sec,
    SUM(rs.active_duration_sec) AS total_active_sec,
    ROUND(
        SUM(rs.active_duration_sec) * 100.0 
        / NULLIF(SUM(rs.shift_duration_sec), 0), 2
    ) AS rider_utilization_percent
FROM fact_rider_shift rs
JOIN dim_date d ON rs.date_key = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date;
```

### 10) Order Preparation SLA Hit Rate
**Definition:**The percentage of kitchen orders marked ready on or before the promised preparation timestamp[cite: 4].
```SQL
SELECT
    COUNT(*) AS accepted_orders,
    SUM(CASE WHEN oh.ready_ts IS NOT NULL AND oh.ready_ts <= oh.promised_ready_ts THEN 1 ELSE 0 END) AS prep_sla_hits,
    ROUND(
        SUM(CASE WHEN oh.ready_ts IS NOT NULL AND oh.ready_ts <= oh.promised_ready_ts THEN 1 ELSE 0 END) * 100.0 
        / NULLIF(COUNT(*), 0), 2
    ) AS prep_sla_hit_percent
FROM fact_order_header oh
JOIN dim_date d ON oh.date_key_ordered = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
  AND oh.status IN ('ACCEPTED', 'DELIVERED');
```