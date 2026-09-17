# Sales Performance Dashboard — BI Capstone Project

## Business Question
Build an ongoing, reusable sales performance reporting system — not a
one-time analysis — covering revenue trends, growth, profitability, and
top customers, structured the way a real BI reporting tool would be.

## Dataset
Sample Superstore Dataset (Kaggle) — 9,994 orders, 2014-2017. This
project reuses the same raw data as an earlier flat-table analysis, but
rebuilds it as a proper star schema and BI data model.

---

## Part 1: SQL — Star Schema Design & Build

### Why a star schema
The original data was one flat table where customer, product, and
region details repeated on every order row. A star schema separates
this into a central fact table (measures + keys) surrounded by
dimension tables (descriptive attributes, stored once each).

```sql
CREATE DATABASE IF NOT EXISTS superstore_project;
USE superstore_project;

-- Raw flat table (source data, already existed from an earlier project)
-- orders (row_id, order_id, order_date, ship_date, ship_mode, customer_id,
--         customer_name, segment, country, city, state, postal_code,
--         region, product_id, category, sub_category, product_name,
--         sales, quantity, discount, profit)

-- ============================================
-- DIMENSION TABLES
-- ============================================

-- dim_date: a COMPLETE calendar table (every day, not just order dates)
-- This is critical -- DAX time-intelligence functions like
-- SAMEPERIODLASTYEAR require a gap-free date range to work correctly.
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE,
    year INT,
    month INT,
    month_name VARCHAR(20),
    day INT,
    day_name VARCHAR(20),
    quarter INT
);

INSERT INTO dim_date (date_key, full_date, year, month, month_name, day, day_name, quarter)
SELECT
    YEAR(d) * 10000 + MONTH(d) * 100 + DAY(d) AS date_key,
    d AS full_date,
    YEAR(d) AS year,
    MONTH(d) AS month,
    MONTHNAME(d) AS month_name,
    DAY(d) AS day,
    DAYNAME(d) AS day_name,
    QUARTER(d) AS quarter
FROM (
    SELECT DATE_ADD('2014-01-01', INTERVAL (a.a + (10*b.a) + (100*c.a) + (1000*d.a)) DAY) AS d
    FROM (SELECT 0 a UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
          UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a
    CROSS JOIN (SELECT 0 a UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
          UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b
    CROSS JOIN (SELECT 0 a UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
          UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) c
    CROSS JOIN (SELECT 0 a UNION SELECT 1 UNION SELECT 2) d
) dates
WHERE d <= '2018-12-31';
-- Result: 1,826 rows (every day, 2014-2018)

-- dim_customer
CREATE TABLE dim_customer (
    customer_key INT AUTO_INCREMENT PRIMARY KEY,
    customer_id VARCHAR(20),
    customer_name VARCHAR(100),
    segment VARCHAR(30)
);
INSERT INTO dim_customer (customer_id, customer_name, segment)
SELECT DISTINCT customer_id, customer_name, segment FROM orders;

-- dim_product
-- NOTE: deduplicated by product_id only (not all 4 columns), since some
-- product_ids had slightly different name/category text across orders --
-- deduping on all columns created duplicate keys and inflated the fact table.
CREATE TABLE dim_product (
    product_key INT AUTO_INCREMENT PRIMARY KEY,
    product_id VARCHAR(30),
    product_name VARCHAR(255),
    category VARCHAR(30),
    sub_category VARCHAR(30)
);
INSERT INTO dim_product (product_id, product_name, category, sub_category)
SELECT product_id, MAX(product_name), MAX(category), MAX(sub_category)
FROM orders
GROUP BY product_id;

-- dim_region
CREATE TABLE dim_region (
    region_key INT AUTO_INCREMENT PRIMARY KEY,
    region VARCHAR(20),
    state VARCHAR(50),
    city VARCHAR(50),
    postal_code INT
);
INSERT INTO dim_region (region, state, city, postal_code)
SELECT DISTINCT region, state, city, postal_code FROM orders;

-- ============================================
-- FACT TABLE
-- ============================================

CREATE TABLE fact_orders (
    order_id VARCHAR(20),
    row_id INT,
    customer_key INT,
    product_key INT,
    region_key INT,
    date_key INT,
    ship_date DATE,
    ship_mode VARCHAR(30),
    sales DECIMAL(10,4),
    quantity INT,
    discount DECIMAL(5,2),
    profit DECIMAL(10,4)
);

INSERT INTO fact_orders (order_id, row_id, customer_key, product_key, region_key,
                          date_key, ship_date, ship_mode, sales, quantity, discount, profit)
SELECT
    o.order_id, o.row_id, c.customer_key, p.product_key, r.region_key,
    YEAR(o.order_date)*10000 + MONTH(o.order_date)*100 + DAY(o.order_date) AS date_key,
    o.ship_date, o.ship_mode, o.sales, o.quantity, o.discount, o.profit
FROM orders o
JOIN dim_customer c ON o.customer_id = c.customer_id
JOIN dim_product p ON o.product_id = p.product_id
JOIN dim_region r ON o.region = r.region AND o.state = r.state
                  AND o.city = r.city AND o.postal_code = r.postal_code;

-- Verify: should return 9994 (matches original flat table exactly)
SELECT COUNT(*) FROM fact_orders;
```

### Validation: star schema reproduces the flat-table results exactly

```sql
SELECT d.year, d.month_name, ROUND(SUM(f.sales), 2) AS total_sales
FROM fact_orders f
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.year, d.month_name, d.month
ORDER BY d.year, d.month;
-- Confirmed to match the original flat-table monthly totals exactly
```

---

## Part 2: Power BI — DAX Measures

```dax
Total Sales = SUM(fact_orders[sales])

Total Profit = SUM(fact_orders[profit])

Total Orders = DISTINCTCOUNT(fact_orders[order_id])

Avg Order Value = DIVIDE([Total Sales], [Total Orders])

Gross Margin % = DIVIDE([Total Profit], [Total Sales])

-- Time intelligence: requires dim_date marked as an official Date Table
Sales PY = CALCULATE([Total Sales], SAMEPERIODLASTYEAR(dim_date[full_date]))

YoY Growth % = DIVIDE([Total Sales] - [Sales PY], [Sales PY])

-- Running total using explicit filter context (manual equivalent of
-- what SAMEPERIODLASTYEAR does automatically)
Running Total Sales =
CALCULATE(
    [Total Sales],
    FILTER(ALL(dim_date), dim_date[full_date] <= MAX(dim_date[full_date]))
)
```

**Key troubleshooting note:** the first version of `dim_date` only
contained dates that appeared in actual orders (with gaps on days with
no sales). This caused `SAMEPERIODLASTYEAR` and the running total to
return blank for every row except the grand total. Rebuilding `dim_date`
as a complete, gap-free calendar table fixed it -- a genuinely important
BI lesson: **date dimensions need every date in the range, not just the
dates present in the data.**

---

## Part 3: Power Query (M) — Transformation Steps

Applied as recorded steps in the Power Query Editor (not written by
hand, but generated from UI actions and preserved so they re-run on
every data refresh):

- **dim_product**: removed stray blank/index columns (`_1`, `Column1`)
  left over from CSV export; added a custom column
  `Category_Full = [category] & " - " & [sub_category]`
- **fact_orders**: added a conditional column `Sales_Tier` — `"High"`
  if `sales >= 500`, else `"Standard"`

This is the core difference between a one-time Python cleaning script
and a real BI pipeline: every step here is saved and automatically
re-applied whenever new data is loaded, rather than needing to be
re-run manually.

---

## Dashboard

**KPI cards:** Total Orders (5,009), Total Sales ($2.30M), Total Profit
($286.40K), Avg Order Value ($458.61), Gross Margin % (12.47%)

**Supporting visuals:** Year-over-year growth table, cumulative running
total chart, Top 10 Customers by sales (led by Sean Miller at $25,043
lifetime value — cross-validated against an earlier SQL ranking query)

---

## Key Findings

- Gross margin of 12.47% is thin for retail — driven largely by
  Furniture, which generates high revenue but near-zero profit
  (established in an earlier flat-table analysis, confirmed again here)
- Revenue grew from $484,248 (2014) to $733,215 (2017), average YoY
  growth of ~15.7% (matches the flat-table Python/SQL analysis exactly)
- Sean Miller is the single highest lifetime-value customer at $25,043

## What Makes This a BI Project, Not Just an Analysis

- Built on a proper star schema (fact + 4 dimension tables), not one
  flat file
- Uses native DAX time-intelligence and filter-context measures instead
  of pre-calculated values from Python
- Power Query steps are saved and repeatable, not manual one-time edits
- Framed and laid out as an ongoing reporting dashboard (KPI cards +
  supporting detail) rather than a single-finding report

## Tools Used

MySQL (star schema design and build) → Power BI (data modeling, DAX,
Power Query, dashboard)
