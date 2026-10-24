# ACE E-Commerce Data Platform in Databricks

An end-to-end medallion data platform on Databricks — from raw CSV ingestion through cleaning and enrichment to business-ready analytical tables. Built with PySpark and Unity Catalog, with embedded business insights at every layer.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Data Sources](#data-sources)
3. [Project Structure](#project-structure)
4. [Bronze Layer](#bronze-layer)
5. [Silver Layer](#silver-layer)
6. [Gold Layer](#gold-layer)
7. [Key Metrics](#key-metrics)
8. [How to Run](#how-to-run)

---

## Architecture Overview

The platform follows the **medallion architecture** pattern, organising data into three layers of increasing quality and structure:

| Layer | Purpose | Tables | Row Counts |
| --- | --- | --- | --- |
| **Bronze** | Raw ingestion from source CSVs; column-name cleaning only | 3 | 1,577 orders / 238 categories / 1,097 daily revenue |
| **Silver** | Cleaning, type casting, standardisation, and enrichment | 3 | 1,577 / 238 / 1,097 |
| **Gold** | Business-ready analytical tables with calendar enrichment and reconciliation | 3 | 1,097 daily / 414 customers / 232 categories |

Each layer reads exclusively from the layer beneath it. All tables are managed within Unity Catalog under the `workspace` catalog, ensuring centralised governance and access control.

**Data Period:** 1 July 2023 to 1 July 2026 (1,097 days).

---

## Data Sources

Three raw CSV files are stored in a Unity Catalog volume at `/Volumes/workspace/bronze/source_systems/`:

| File | Description | Rows |
| --- | --- | --- |
| `orders_data.csv` | Order-level sales records (customer, products, coupons, attribution) | 1,577 |
| `categories.csv` | Category-level sales summaries | 238 |
| `revenue.csv` | Daily revenue totals (gross sales, returns, coupons, net sales) | 1,097 |

All source files are read with `header=True`, `inferSchema=True`, and multi-line CSV handling enabled to accommodate embedded commas and quoted values.

---

## Project Structure

```
ace-ecommerce-data-platform-in-databricks/
├── README.md
└── ace_lakehouse_2026/
    ├── Bronze_Layer.ipynb          # Ingests 3 source CSVs into Bronze tables
    ├── Silver_Orders_Data.ipynb    # Cleans and enriches order-level data
    ├── Silver_Category_Data.ipynb  # Cleans category-level data
    ├── Silver_Revenue_Data.ipynb   # Cleans daily revenue data
    └── Gold_Layer.ipynb            # Builds business-ready gold tables
```

Notebooks must be executed in the order shown above, as each layer depends on the one beneath it.

---

## Bronze Layer

**Notebook:** `Bronze_Layer.ipynb`

The Bronze layer ingests the three source CSVs into Unity Catalog with minimal transformation — only column-name cleaning is applied to ensure valid table field names.

### Transformations

1. **Column-name cleaning** — spaces replaced with underscores, `#` replaced with `num`, parentheses and full stops removed. For example, `Order #` becomes `Order_num`.
2. **Schema evolution** — `overwriteSchema` is enabled on every write, so re-runs automatically pick up any source schema changes.

### Output Tables

| Table | Source File | Rows |
| --- | --- | --- |
| `workspace.bronze.orders_data` | `orders_data.csv` | 1,577 |
| `workspace.bronze.categories_data` | `categories.csv` | 238 |
| `workspace.bronze.revenue_data` | `revenue.csv` | 1,097 |

Each table is verified after writing by reading it back and displaying the contents.

---

## Silver Layer

Three notebooks, each cleaning one bronze source and embedding four business-insight sections (KPIs, ranking/trend, deep-dive, efficiency).

| Notebook | Source → Output | Key Transformations |
| --- | --- | --- |
| `Silver_Orders_Data.ipynb` | `bronze.orders_data` → `silver.silver_orders_data` (1,577 rows) | Date to `yyyy-mm-dd`; split revenue into amount + savings; proper-case text; decode HTML entities; derive coupon amounts from codes; snake-case columns |
| `Silver_Category_Data.ipynb` | `bronze.categories_data` → `silver.silver_categories_data` (238 rows) | Trim category names; near-duplicates consolidated in Gold |
| `Silver_Revenue_Data.ipynb` | `bronze.revenue_data` → `silver.silver_revenue_data` (1,097 rows) | Date to `yyyy-mm-dd`; strip stray apostrophes from string money columns; cast to `double` |

---

## Gold Layer

**Notebook:** `Gold_Layer.ipynb`

Builds three business-ready tables from the silver layer, then reconciles each against its source to prove no data was lost or duplicated.

| Table | Rows | Description |
| --- | --- | --- |
| `workspace.gold.gold_daily_revenue_summary` | 1,097 | Daily revenue enriched with year, quarter, month, day of week, and weekend flag |
| `workspace.gold.gold_customer_summary` | 414 | One row per customer: lifetime orders, revenue, average order value, current type (from most recent order), top acquisition channel |
| `workspace.gold.gold_category_performance` | 232 | Consolidated category league table with revenue rankings and cumulative Pareto shares |

---

## Key Metrics

| Metric | Value |
| --- | --- |
| Total orders | 1,577 (1,549 completed, 28 refunded) |
| Total net sales | $271,941.59 |
| Total customers | 414 (185 evolved from New to Returning) |
| Distinct categories | 232 |
| Pareto concentration | 40 categories (17%) drive 80% of revenue |
| Average order value | $165.21 |
| Data period | 1 July 2023 – 1 July 2026 (1,097 days) |

**Data quality handling:** Revenue money columns arrive as strings with stray apostrophes (stripped and cast to `double` in Silver); category names contain duplicates and near-duplicates (consolidated in Gold); HTML entities in product names decoded in Silver; all dates standardised to `yyyy-mm-dd`.

---

## How to Run

Execute notebooks in order. Steps 2–4 are independent and can run in any sequence after Step 1.

1. `Bronze_Layer.ipynb` — ingest source CSVs
2. `Silver_Orders_Data.ipynb` — clean order data
3. `Silver_Category_Data.ipynb` — clean category data
4. `Silver_Revenue_Data.ipynb` — clean revenue data
5. `Gold_Layer.ipynb` — build gold tables and reconcile

All notebooks use `mode('overwrite')` with `overwriteSchema` where needed — fully idempotent and safe to re-run.

**Prerequisites:** Databricks workspace with Unity Catalog, source CSVs in the volume, and PySpark compute.

## Unity Catalog Schema

| Schema | Table | Layer | Rows |
| --- | --- | --- | --- |
| `workspace.bronze` | `orders_data` | Bronze | 1,577 |
| `workspace.bronze` | `categories_data` | Bronze | 238 |
| `workspace.bronze` | `revenue_data` | Bronze | 1,097 |
| `workspace.silver` | `silver_orders_data` | Silver | 1,577 |
| `workspace.silver` | `silver_categories_data` | Silver | 238 |
| `workspace.silver` | `silver_revenue_data` | Silver | 1,097 |
| `workspace.gold` | `gold_daily_revenue_summary` | Gold | 1,097 |
| `workspace.gold` | `gold_customer_summary` | Gold | 414 |
| `workspace.gold` | `gold_category_performance` | Gold | 232 |

---

## Author

**Wasim** — Data and Analytics Engineer, Ace Screen Supply