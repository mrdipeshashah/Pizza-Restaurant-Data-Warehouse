# PIZZA RESTAURANT: DATA WAREHOUSE & ANALYTICS PIPELINE

# OVERVIEW

This repository serves as the definitive data catalog, schema registry, and operational blueprint for our centralized restaurant analytics pipeline. The architecture live-links operational Google Sheets to a **Google BigQuery** data warehouse to calculate top-line revenue, daily overheads, and bottom-line net profitability across multiple locations.

---

# DATA ARCHITECTURE & PIPELINE DEPENDANCY LINEAGE

This repository outlines the architecture and data lineage mapping for the two primary pipeline anchors in the repository:
1. `3.2_master_business_performance` (Master Reporting Table/View)
2. `5.1_forecast_gross_revenue_365d` (365-Day ML Forecasting Pipeline)

---

## 1. Architecture Mapping: `3.2_master_business_performance`

This view serves as the **core reporting source of truth**. It aggregates raw daily transactions, incorporates theoretical inventory costs, and cleans data using audit scripts to prepare business performance data for Looker Studio dashboards and BigQuery ML training models.

```
[Audit Layer (1.x)] ──► [Aggregation Layer (2.x / 3.x)] ──► [3.2 Master Business Performance]
```

### Dependency Lineage Table

| Target Pipeline Stage | Script / File Name | Upstream Dependency / Purpose | Role in Master Performance (`3.2`) |
| :--- | :--- | :--- | :--- |
| **Audit Layer** | `1.0_audit-daily-costs` | Cost validation | Provides net profit & margin calculation context |
| | `1.1_audit-isodates` | Calendar standardization | Ensures clean date/ISO-week alignment across leap years |
| | `1.2_audit-itemmenu-itemcode` | Menu hierarchy audit | Maps order items to standardized revenue categories |
| | `1.3_audit-monday-holiday-closure` | Operational anomalies | Flags zero-revenue days (e.g., closed Mondays, refurbishments) |
| | `1.4_audit-ordertype-platform` | Channel mapping | Standardizes order channels (Dine-in, Delivery, Takeaway) |
| | `1.5_audit-revenue-location` | Location mapping | Maps raw store tags to canonical `location_id` |
| | `1.6_audit-paymentcost` | Payment fee reconciliation | Standardizes merchant payment processing fee deductions |
| | `1.7_audit_priceversionyear` | Pricing version tracking | Normalizes price changes across financial years |
| **Aggregation Layer** | `2.1_daily_sales_aggregated` | Sales line-item rollups | Collapses transactional records into daily store-level totals |
| | `2.2_theoretical_cogs` | Cost of goods sold | Blends theoretical menu costs into daily revenue totals |
| | `3.1_aggregated_daily_sales` | Intermediate view | Standardizes revenue metrics across order platforms/types |
| **Final Target** | **`3.2_master_business_performance`** | **All Above Views/Tables** | **Combines sales, costs, location mapping, and operational flags into a single master source.** |

---

## 2. Architecture Mapping: `5.1_forecast_gross_revenue_365d`

This pipeline powers the **end-to-end ML forecasting engine**. It reads clean historical daily sales from `3.2`, trains the BigQuery `ARIMA_PLUS` model in `5.0`, and generates a complete 365-day prediction output.

```
[3.2 Master Business Performance] ──► [5.0 Model Training] ──► [5.1 365-Day Forecast Generation]
```

### Dependency Lineage Table

| Pipeline Stage | Script / File Name | Input Source | Primary Function & Impact |
| :--- | :--- | :--- | :--- |
| **Data Baseline** | `3.2_master_business_performance` | Core Business Data | Provides historical daily gross revenue filtered by `location_id` and `date`. |
| **Model Training** | `5.0_train_gross_revenue_forecast` | `3.2_master_business_performance` | Runs `CREATE OR REPLACE MODEL` using `ARIMA_PLUS` on history up to cutoff date. |
| **Validation / Backtest** | `4.0_backtest_2025_model`<br>`4.1_actual_2025_monthly_data` | Trained Model + 2025 Actuals | Evaluates model accuracy (MAPE/RMSE) against actual historical performance. |
| **Forecast Execution** | **`5.1_forecast_gross_revenue_365d`** | `5.0` Trained Model | Executes `ML.FORECAST` with `STRUCT(365 AS horizon)` to project full 12-month timeline. |
| **Final Output** | `t_actual_v_forecasted_revenue` | `5.1` Execution Script | Populates the production BigQuery table powering Looker Studio scorecards and comparative charts. |

## Entity Relationship & Data Join Rules

### 1. Order Tracker ↔ Business KPI Tracker
* **Join Key:** Composite Primary Key (`location_id` + `date`).
* **Logic & Integrity:** Because performance metrics are aggregated at a daily granularity per restaurant, joining on both fields is mandatory to prevent cross-contamination and fan-out errors across locations.

### 2. Order Tracker ↔ Item Menu Tracker
* **Join Key:** Composite Join Rule matching `price_version_year` to the corresponding menu version year.
* **Logic & Category Mapping:** Includes explicit logic to map item quantities to their descriptive macro categories (e.g., `qty_pizza` $\rightarrow$ `Main Pizza`) to ensure accurate price versioning across financial years.

---

## Data Dictionary & Schema Registry

### 1. Table: `order_tracker`
* **BigQuery Path:** `mrdipeshashah.pizzarestaurant.order_tracker`
* **Purpose:** Captures every individual customer transaction at the point of sale.

| Field Name | Data Type | Mode | Description / Business Rule | Example |
| :--- | :--- | :--- | :--- | :--- |
| `location_id` | `STRING` | NULLABLE | Unique identifier for the branch location. | `LOC_01` |
| `order_no` | `STRING` | NULLABLE | Single-Column Primary Key. Unique per order. | `LOC_01-10001` |
| `date` | `DATE` | NULLABLE | Transaction date. **Must strictly follow ISO 8601 (`YYYY-MM-DD`).** | `2024-01-02` |
| `time` | `STRING` | NULLABLE | 24-hour timestamp of payment (`HH:MM`). Constraints: 12:00 to 22:00. | `12:22` |
| `order_type` | `STRING` | NULLABLE | Category of service. Allowed: `Dine-in`, `Takeaway`, `Delivery`. | `Delivery` |
| `platform` | `STRING` | NULLABLE | Portal used. `Direct` for Dine-in/Takeaway; `Deliveroo`/`UberEats` for Delivery. | `UberEats` |
| `price_version_year` | `INTEGER` | NULLABLE | Menu version year for cost lookups. **Cannot be blank.** | `2024` |
| `qty_starter` | `INTEGER` | NULLABLE | Count of individual starter items included in the order. | `1` |
| `qty_pizza` | `INTEGER` | NULLABLE | Count of main pizzas included in the order. | `3` |
| `qty_dessert` | `INTEGER` | NULLABLE | Count of dessert items included in the order. | `0` |
| `qty_drinks` | `INTEGER` | NULLABLE | Count of beverage items included in the order. | `0` |
| `packaging_cost` | `FLOAT` | NULLABLE | Unit cost of boxes/bags used for the order (e.g., £0.80 for off-premise). | `0.80` |
| `platform_cost` | `FLOAT` | NULLABLE | Marketplace commission fees. Must be £0 for `Direct` orders. | `16.80` |
| `payment_type` | `STRING` | NULLABLE | Financial tender used. Allowed: `Card`, `Cash`. | `Card` |
| `payment_cost` | `FLOAT` | NULLABLE | Bank merchant processing fees. **Must be exactly 0 for Cash.** | `0.84` |
| `order_value_gross` | `FLOAT` | NULLABLE | Total cash hitting the till/bank account. Includes taxes and add-ons. | `56.00` |

### 2. Table: `item_menu_tracker`
* **BigQuery Path:** `mrdipeshashah.pizzarestaurant.item_menu_tracker`
* **Purpose:** Stores baseline ingredient costs and standard prices per year version to compute theoretical Cost of Goods Sold (COGS).

| Field Name | Data Type | Mode | Description / Business Rule | Example |
| :--- | :--- | :--- | :--- | :--- |
| `item_code` | `STRING` | NULLABLE | Single-Column Primary Key. Format: `[Type]-[YYYY]`. | `PZ-2024` |
| `location_id` | `STRING` | NULLABLE | Scope of menu pricing. Marked as `ALL` for macro fallback logic. | `ALL` |
| `category` | `STRING` | NULLABLE | Macro category name. Written for human readability (`Main Pizza`). | `Main Pizza` |
| `qty_per_unit` | `INTEGER` | NULLABLE | Standard serving unit ratio multiplier. Defaults to 1. | `1` |
| `std_unit_cost` | `FLOAT` | NULLABLE | **Weighted Average Ingredient Cost** to produce one unit. | `5.00` |
| `std_unit_price` | `FLOAT` | NULLABLE | Base menu item price before application platform inflation. | `16.50` |
| `status` | `STRING` | NULLABLE | Lifecycle state of that row. Values: `Active`, `Retired`. | `Retired` |
| `notes` | `STRING` | NULLABLE | Descriptive engineering or pricing remarks. | `Standard` |

### 3. Table: `business_kpi_tracker`
* **BigQuery Path:** `mrdipeshashah.pizzarestaurant.business_kpi_tracker`
* **Purpose:** Tracks fixed and variable daily operational expenses.

| Field Name | Data Type | Mode | Description / Business Rule | Example |
| :--- | :--- | :--- | :--- | :--- |
| `location_id` | `STRING` | NULLABLE | Part of Composite Primary Key. Branch location incurring the costs. | `LOC_01` |
| `date` | `DATE` | NULLABLE | Part of Composite Primary Key. **Must strictly follow ISO 8601 (`YYYY-MM-DD`).** | `2024-01-02` |
| `labor_cost` | `FLOAT` | NULLABLE | Combined daily wage payroll cost for scheduled floor and kitchen staff. | `250.00` |
| `marketing_spend` | `FLOAT` | NULLABLE | Daily localized advertising or social media promo spend. | `15.00` |
| `utilities` | `FLOAT` | NULLABLE | Pro-rated daily fixed costs for power, gas, water, and broadband. | `20.00` |
| `wastage` | `FLOAT` | NULLABLE | Financial value of ruined ingredients or voided kitchen errors. | `54.52` |

---

## Data Quality Guardrails & Business Logic Rules

The pipeline enforces several strict operational rules to protect financial metrics from distortion:

1.  **The Cash Processing Fee Rule:** Bank merchant fees (`payment_cost`) are strictly banned on cash orders. All cash rows must have a transaction cost of exactly `0`.
2.  **The Calendar Constraint (Monday Closures):** Operations are closed on Mondays. The pipeline assumes zero revenue and flags anomalies if transaction data appears on a Monday.
3.  **The Price Year Fallback Rule:** If `price_version_year` is left blank or a new year menu hasn't been generated yet, SQL calculations apply a `COALESCE` safety fallback to default to the most recent active year's costs rather than letting ingredient costs drop to `NULL` (which deceptively inflates profit metrics to 100%).
4.  **The Menu Multiplication Rule:** A new row must be created in the `item_menu_tracker` for every new operating year, even if ingredient prices do not change. This isolates historical profit records from inflation spikes.

---

## Future Data Roadmap (Things Worth Adding)

As the database scales toward a comprehensive £1.6m business evaluation, the following data schemas should be structured and integrated:

* **Discount & Promotion Tracker:** A separate log for discount voucher codes, loyalty rewards, and manager comps. This bridges the gap between menu `std_unit_price` and actual `order_value_gross`.
* **Actual vs. Theoretical Inventory (Stock Variance):** A table tracking physical stock takes (e.g., kilos of flour/cheese left in the walk-in). This allows us to compare *Actual Food Cost* against the *Theoretical Food Cost* calculated via our menu tracker to pinpoint kitchen theft, over-portioning, or undeclared waste.
* **Labor Efficiency Matrix (`shift_id`):** Breaking daily `labor_cost` down into hourly staff clock-ins. This unlocks **Sales per Labor Hour (SPLH)** metrics, telling us exactly when a site is over or under-staffed.

# Pizza Restaurant Group - Data Pipeline Architecture & Data Dictionary

This repository contains the database views and schemas for a three-layer BigQuery analytical pipeline. The architecture transforms raw transactional data and daily KPI metrics into a clean, time-intelligent financial model optimized for Looker Studio dashboards.

---

## Core Architecture Overview

The transformation pipeline utilizes a **Composite Primary Key (`location_id` + `date`)** to connect decoupled data streams without creating duplication or nested calculation loops.

1. **Layer 1 (Audit & Validation)**: Acts as a quality gate catching operational anomalies and financial leaks.
2. **Layer 2, View 1 (`2.1_v_int_daily_sales_aggregated`)**: The Aggregation Engine. Condenses raw tickets to a daily level.
3. **Layer 2, View 2 (`2.2_v_int_theoretical_cogs`)**: The Matrix Cost Engine. Computes dynamic food costs with location-based overrides.
4. **Layer 3, Master View (`3.1_v_master_business_performance`)**: The Performance Engine. Integrates fixed overheads, pre-computes bottom-line profit margins, and injects extensive time-intelligence fields.

---

## Data Dictionary & Schema Specification

### Layer 2, View 1: `v_int_daily_sales_aggregated`
* **Role:** Compresses row-by-row transaction logs down to daily location-based volume and revenue snapshots.
* **GitHub File Reference:** `2_intermediate_layer/2.1_v_int_daily_sales_aggregated.sql`

| Field Name | Data Type | Mode | Description / Logic | Production Example |
| :--- | :--- | :--- | :--- | :--- |
| `location_id` | `STRING` | NULLABLE | Unique identifier for the restaurant branch. Part of the composite key. | `LOC_01` |
| `date` | `DATE` | NULLABLE | Operational date of the business transactions. Part of the composite key. | `2024-01-02` |
| `price_version_year` | `INTEGER` | NULLABLE | `MAX(price_version_year)` extracted from daily orders to pair with menu pricing editions. | `2024` |
| `total_qty_pizza` | `INTEGER` | NULLABLE | Aggregated volume sum of all pizzas sold (`SUM(COALESCE(qty_pizza, 0))`). | `94` |
| `total_qty_starter` | `INTEGER` | NULLABLE | Aggregated volume sum of all starters sold (`SUM(COALESCE(qty_starter, 0))`). | `41` |
| `total_qty_dessert` | `INTEGER` | NULLABLE | Aggregated volume sum of all desserts sold (`SUM(COALESCE(qty_dessert, 0))`). | `0` |
| `total_qty_drinks` | `INTEGER` | NULLABLE | Aggregated volume sum of all drinks sold (`SUM(COALESCE(qty_drinks, 0))`). | `0` |
| `total_gross_revenue` | `FLOAT` | NULLABLE | Raw top-line incoming funds processed through the till system. | `1817.50` |
| `total_delivery_commission` | `FLOAT` | NULLABLE | Accumulated platform channel fees (e.g., Deliveroo, UberEats). | `160.05` |
| `total_card_merchant_fees` | `FLOAT` | NULLABLE | Processing fees applied to card transactions. | `17.30` |
| `total_packaging_cost` | `FLOAT` | NULLABLE | Dynamic cost computed based on takeout boxes used. | `37.60` |
| `total_orders_count` | `INTEGER` | NULLABLE | Total number of unique orders processed during the operational window. | `47` |

---

### Layer 2, View 2: `v_int_theoretical_cogs`
* **Role:** Connects day-by-day sales volumes against the menu cost matrices to calculate theoretical ingredient outlays.
* **GitHub File Reference:** `2_intermediate_layer/2.2_v_int_theoretical_cogs.sql`

| Field Name | Data Type | Mode | Description / Logic | Production Example |
| :--- | :--- | :--- | :--- | :--- |
| `location_id` | `STRING` | NULLABLE | Inherited branch locator. Part of the composite key. | `LOC_01` |
| `date` | `DATE` | NULLABLE | Inherited calendar date. Part of the composite key. | `2024-01-02` |
| `total_theoretical_cogs` | `FLOAT` | NULLABLE | Total blended menu ingredient costs across all categories combined. | `531.50` |
| `cogs_pizza` | `FLOAT` | NULLABLE | Pizza volume multiplied by ranked pizza unit cost. | `470.00` |
| `cogs_starter` | `FLOAT` | NULLABLE | Starter volume multiplied by ranked starter unit cost. | `61.50` |
| `cogs_dessert` | `FLOAT` | NULLABLE | Dessert volume multiplied by ranked dessert unit cost. | `0.0` |
| `cogs_drinks` | `FLOAT` | NULLABLE | Drinks volume multiplied by ranked drink unit cost. | `0.0` |

---

### Layer 3, Master View: `v_master_business_performance`
* **Role:** The ultimate presentation table. Combines performance metrics from the intermediate layer along with operational overheads and advanced time-intelligence parameters.
* **Data Flow Note:** This view explicitly inherits fields from `2.1` and `2.2` via a composite join to eliminate calculation duplication.
* **GitHub File Reference:** `3_master_reporting_layer/3.1_v_master_business_performance.sql`

#### Inherited Fields (Brought forward automatically)
* 🔹 **Structural Elements:** `location_id`, `date`, `price_version_year`, `total_orders_count` *(From 2.1)*
* 🔹 **Gross Financial Streams:** `total_gross_revenue`, `total_delivery_commission`, `total_card_merchant_fees`, `total_packaging_cost` *(From 2.1)*
* 🔹 **Ingredient Cost Breakdowns:** `total_theoretical_cogs`, `cogs_pizza`, `cogs_starter`, `cogs_dessert`, `cogs_drinks` *(From 2.2)*

#### Extended Time-Intelligence & Operational Cost Elements

| Field Name | Data Type | Mode | Description / Logic | Production Example |
| :--- | :--- | :--- | :--- | :--- |
| `day_name` | `STRING` | NULLABLE | Day name format string (`FORMAT_DATE('%A')`). | `Tuesday` |
| `week_starting_monday` | `DATE` | NULLABLE | Anchors date to previous Monday boundary (`DATE_TRUNC(..., WEEK(MONDAY))`). | `2024-01-01` |
| `month_name` | `STRING` | NULLABLE | Full calendar month string name identifier (`FORMAT_DATE('%B')`). | `January` |
| `week_number` | `INTEGER` | NULLABLE | ISO standard calendar numerical week index value (1-53). | `1` |
| `month_number` | `INTEGER` | NULLABLE | Numerical chronological index of month (1-12). | `1` |
| `calendar_year` | `INTEGER` | NULLABLE | Clean four-digit numerical base reporting year. | `2024` |
| `year_month` | `STRING` | NULLABLE | Dash-separated year and month index string key. | `2024-01` |
| `day_type` | `STRING` | NULLABLE | Flags date based on index value: `Weekend` (Sat/Sun) vs `Weekday`. | `Weekday` |
| `quarter` | `STRING` | NULLABLE | Maps quarter segment string directly (`Q1`, `Q2`, `Q3`, `Q4`). | `Q1` |
| `year_quarter` | `STRING` | NULLABLE | Combined year and quarter tracking index code. | `2024-Q1` |
| `overhead_labor_cost` | `FLOAT` | NULLABLE | Direct labor expenses extracted via composite match to `business_kpi_tracker`. | `250.00` |
| `overhead_marketing_spend` | `FLOAT` | NULLABLE | Marketing capital disbursed on this day. | `15.00` |
| `overhead_utilities_cost` | `FLOAT` | NULLABLE | Variable/fixed utility overhead threshold. | `20.00` |
| `overhead_wastage_cost` | `FLOAT` | NULLABLE | Real dollar value of wasted inventory reported by branch management. | `54.52` |
| `total_net_sales` | `FLOAT` | NULLABLE | **Calculated Metric:** `total_gross_revenue - total_delivery_commission - total_card_merchant_fees - total_packaging_cost`. | `1602.55` |
| `total_net_profit` | `FLOAT` | NULLABLE | **Calculated Metric:** `total_net_sales - total_theoretical_cogs - sum of overhead costs`. | `731.53` |

---

## 🛠️ Engineering Integration Guidance

* **Looker Studio Optimization:** Do not use custom calculated fields for revenue subtractions or net margins inside Looker Studio. Use the pre-calculated `total_net_sales` and `total_net_profit` metrics directly to keep dashboard loading response under 1 second.
* **Aggregations:** When displaying metrics inside scorecard widgets, ensure `total_net_sales` and `total_net_profit` use `SUM` as their default aggregation mode. Time intelligence dimensions (`day_type`, `year_month`) should be used as secondary filters/slicers.
