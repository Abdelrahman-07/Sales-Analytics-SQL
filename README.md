# 📊 Data Warehouse Analytics — SQL Analysis Portfolio

A structured SQL analytics project built on a fictional retail data warehouse, demonstrating progressively advanced analytical techniques — from basic time-series queries all the way to reusable business-intelligence views used in reporting dashboards.

---

## 📁 Project Structure

```
├── 00_init_database.sql          # Database setup, schema creation, and data loading
├── 01_change_over_time_analysis.sql   # Trend and seasonality analysis
├── 02_cumulative_analysis.sql         # Running totals and moving averages
├── 03_performance_analysis.sql        # Year-over-Year product performance
├── 04_data_segmentation.sql           # Customer and product segmentation
├── 05_part_to_whole_analysis.sql      # Category contribution to total sales
├── 06_report_customers.sql            # Customer KPI report view
└── 07_report_products.sql             # Product KPI report view
```

---

## 🗄️ Database Schema

The project uses a **star schema** inside a `gold` layer (following a Medallion Architecture convention), with the following tables:

### `gold.dim_customers`
| Column | Type | Description |
|---|---|---|
| `customer_key` | int | Surrogate key |
| `customer_id` | int | Source system ID |
| `customer_number` | nvarchar | Business identifier |
| `first_name`, `last_name` | nvarchar | Customer name |
| `country` | nvarchar | Country of residence |
| `marital_status` | nvarchar | Marital status |
| `gender` | nvarchar | Gender |
| `birthdate` | date | Date of birth |
| `create_date` | date | Record creation date |

### `gold.dim_products`
| Column | Type | Description |
|---|---|---|
| `product_key` | int | Surrogate key |
| `product_id` | int | Source system ID |
| `product_number` | nvarchar | Business identifier |
| `product_name` | nvarchar | Product name |
| `category`, `subcategory` | nvarchar | Product classification |
| `cost` | int | Unit cost |
| `product_line` | nvarchar | Product line |
| `start_date` | date | Product availability start |

### `gold.fact_sales`
| Column | Type | Description |
|---|---|---|
| `order_number` | nvarchar | Unique order identifier |
| `product_key` | int | FK to dim_products |
| `customer_key` | int | FK to dim_customers |
| `order_date` | date | Date of order |
| `shipping_date` | date | Date shipped |
| `due_date` | date | Expected delivery date |
| `sales_amount` | int | Total sale value |
| `quantity` | tinyint | Units sold |
| `price` | int | Unit price |

---

## 📜 Script Breakdown

### `00_init_database.sql` — Database Initialization
Sets up the entire analytics environment from scratch.

- Drops and recreates the `DataWarehouseAnalytics` database safely
- Creates the `gold` schema
- Defines all three tables (`dim_customers`, `dim_products`, `fact_sales`)
- Loads data via `BULK INSERT` from CSV files

> ⚠️ **Warning:** Running this script drops and recreates the database. All existing data will be lost. Ensure backups exist before execution.

---

### `01_change_over_time_analysis.sql` — Trend Analysis
Tracks how sales metrics evolve across time using three different date-grouping techniques.

**Key metrics computed:**
- Total sales revenue per period
- Distinct customer count
- Total quantity sold

**SQL techniques demonstrated:**
- `YEAR()` / `MONTH()` for manual date decomposition
- `DATETRUNC(month, ...)` for native date-level grouping (sorts correctly as a date)
- `FORMAT(date, 'yyyy-MMM')` for human-readable labels (sorts as a string — use with care)

**Use case:** Identifying monthly or yearly sales trends, detecting seasonality patterns.

---

### `02_cumulative_analysis.sql` — Running Totals & Moving Averages
Builds on time-series data to compute cumulative metrics using window functions.

**Key metrics computed:**
- Monthly/yearly total sales
- Running total of sales over time
- Moving average of price across periods

**SQL techniques demonstrated:**
- Subquery to pre-aggregate by year
- `SUM() OVER (ORDER BY order_date)` — unbounded running total
- `AVG() OVER (ORDER BY order_date)` — cumulative moving average

**Use case:** Tracking growth trajectories and understanding long-term revenue momentum.

---

### `03_performance_analysis.sql` — Year-over-Year Product Performance
Benchmarks each product's annual sales against its own historical average and the prior year.

**Key metrics computed:**
- Current year sales per product
- Average sales across all years (per product)
- Deviation from historical average (`diff_avg`)
- Previous year sales (`py_sales`)
- Year-over-year change (`diff_py`)

**SQL techniques demonstrated:**
- CTE (`WITH`) for clean query structuring
- `AVG() OVER (PARTITION BY product_name)` — per-product average across years
- `LAG() OVER (PARTITION BY product_name ORDER BY order_year)` — prior year lookup
- `CASE` expressions for labeling trends (`Above Avg`, `Increase`, `Decrease`, etc.)

**Use case:** Identifying top-performing products, spotting declines early, and driving product strategy decisions.

---

### `04_data_segmentation.sql` — Customer & Product Segmentation
Groups entities into meaningful business segments using rule-based `CASE` logic.

**Product segmentation** — by unit cost:
| Segment | Cost Range |
|---|---|
| Below 100 | cost < 100 |
| 100–500 | 100 ≤ cost ≤ 500 |
| 500–1000 | 500 ≤ cost ≤ 1000 |
| Above 1000 | cost > 1000 |

**Customer segmentation** — by spending and tenure:
| Segment | Criteria |
|---|---|
| VIP | Lifespan ≥ 12 months AND total spending > €5,000 |
| Regular | Lifespan ≥ 12 months AND total spending ≤ €5,000 |
| New | Lifespan < 12 months |

**SQL techniques demonstrated:**
- CTE for modular aggregation
- `DATEDIFF(month, ...)` for lifespan calculation
- Nested subquery for clean segment-level aggregation

**Use case:** Targeting marketing campaigns, prioritizing customer retention efforts, and managing product portfolio tiers.

---

### `05_part_to_whole_analysis.sql` — Category Contribution Analysis
Determines which product categories drive the most revenue and quantifies their share of total sales.

**Key metrics computed:**
- Total sales per category
- Overall sales (all categories combined)
- Percentage contribution of each category

**SQL techniques demonstrated:**
- CTE for category-level aggregation
- `SUM() OVER ()` (no partition) for a grand total in every row
- `CAST(... AS FLOAT)` to avoid integer division truncation
- `ROUND(..., 2)` for clean percentage output

**Use case:** Informing inventory and budgeting decisions, identifying revenue-critical categories.

---

### `06_report_customers.sql` — Customer KPI Report View
A production-ready SQL view (`gold.report_customers`) that consolidates all relevant customer metrics into a single queryable object.

**Computed fields:**
| Field | Description |
|---|---|
| `age_group` | Bucketed age ranges (Under 20, 20–29, ..., 50 and above) |
| `customer_segment` | VIP / Regular / New (based on lifespan + spending) |
| `recency` | Months since last purchase |
| `avg_order_value` | Total sales ÷ total orders |
| `avg_monthly_spend` | Total sales ÷ lifespan in months |

**Structure:**
- **CTE 1 — `base_query`:** Joins `fact_sales` with `dim_customers`, computes age from birthdate
- **CTE 2 — `customer_aggregation`:** Aggregates orders, sales, quantity, products, and lifespan per customer
- **Final SELECT:** Applies segmentation and KPI formulas

**Use case:** Powers customer-facing dashboards, CRM feeds, and churn/retention analysis.

---

### `07_report_products.sql` — Product KPI Report View
A production-ready SQL view (`gold.report_products`) that summarizes all key product performance metrics.

**Computed fields:**
| Field | Description |
|---|---|
| `product_segment` | High-Performer (>50K), Mid-Range (≥10K), Low-Performer |
| `recency_in_months` | Months since last sale |
| `avg_selling_price` | Average revenue per unit sold |
| `avg_order_revenue` | Total sales ÷ total orders |
| `avg_monthly_revenue` | Total sales ÷ lifespan in months |

**Structure:**
- **CTE 1 — `base_query`:** Joins `fact_sales` with `dim_products`
- **CTE 2 — `product_aggregations`:** Aggregates orders, customers, sales, quantity, lifespan, and avg price per product
- **Final SELECT:** Applies segmentation and KPI formulas; handles division-by-zero with `CASE` and `NULLIF`

**Use case:** Powers product performance dashboards, informs pricing strategy, and highlights underperforming SKUs.

---

## 🧰 SQL Techniques Reference

| Technique | Scripts Used In |
|---|---|
| `DATETRUNC()`, `FORMAT()`, `DATEPART()` | 01, 06, 07 |
| `SUM() OVER()`, `AVG() OVER()` | 02, 03, 05 |
| `LAG()` window function | 03 |
| Common Table Expressions (CTEs) | 03, 04, 05, 06, 07 |
| `CASE` expressions | 03, 04, 06, 07 |
| `DATEDIFF()` | 04, 06, 07 |
| `NULLIF()` for division-by-zero safety | 07 |
| `BULK INSERT` | 00 |
| SQL Views (`CREATE VIEW`) | 06, 07 |
| Star Schema joins (fact ↔ dimension) | 01–07 |

---

## ⚙️ Setup & Usage

### Prerequisites
- Microsoft SQL Server 2019+ (or Azure SQL Database)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- CSV source files for `gold.dim_customers`, `gold.dim_products`, and `gold.fact_sales`

### Steps

1. **Update file paths** in `00_init_database.sql`:
   ```sql
   FROM 'C:\your\actual\path\gold.dim_customers.csv'
   ```

2. **Run scripts in order:**
   ```
   00_init_database.sql       ← Run first (creates DB and loads data)
   01 through 05              ← Run in any order for ad-hoc analysis
   06_report_customers.sql    ← Creates the customer report view
   07_report_products.sql     ← Creates the product report view
   ```

3. **Query the report views:**
   ```sql
   SELECT * FROM gold.report_customers;
   SELECT * FROM gold.report_products;
   ```

---

## 📌 Notes

- All scripts target the `gold` schema, consistent with a **Medallion Architecture** (Bronze → Silver → Gold) data warehouse pattern.
- The `gold` layer represents curated, business-ready data.
- Scripts 06 and 07 use `IF OBJECT_ID(...) IS NOT NULL DROP VIEW` guards, making them safe to re-run without errors.
- Integer arithmetic is used throughout — for financial precision in production, consider `DECIMAL` or `FLOAT` casting where needed.

---

## 🪪 Author

Built as part of a hands-on SQL analytics portfolio demonstrating data warehousing, window functions, segmentation logic, and KPI reporting using Microsoft SQL Server.
