# Pizza Restaurant Group: Data Warehouse & Analytics Pipeline

This repository serves as the definitive data catalog, schema registry, and operational blueprint for our centralized restaurant analytics pipeline. The architecture live-links operational Google Sheets to a **Google BigQuery** data warehouse to calculate top-line revenue, daily overheads, and bottom-line net profitability across multiple locations.

---

## 🏛 Architecture & Core Philosophy: Volume vs. Value

The database is split across three distinct trackers to separate **operational volume** from **financial value**. 

* **Volume = Category Precision:** Derived by mapping order quantities against the `item_menu_tracker` baseline costs. Used for operational planning (ingredient prep, stock ordering), productivity monitoring, and high-level menu trend analysis.
* **Revenue = Financial Truth:** The `order_value_gross` field inside the `order_tracker` serves as the absolute source of truth. It accounts for every single penny charged to the customer (including modifiers, delivery app markups, or discounts) and **will 100% match cash/card deposits in the bank account.**

### Relational Schema Map & Core Keys

[order_tracker]                                      [business_kpi_tracker]

order_no (Single Primary Key)

location_id   ======= COMPOSITE JOIN ====>    - location_id \ (Composite PK)

date          =======/                             - date        /

price_version_year  =

| COMPOSITE JOIN ===>   [item_menu_tracker]

[Category Quantities] =/                           - item_code (Single PK)
- category

1.  **Order Tracker ↔ Business KPI Tracker:** Connect via a **Composite Primary Key** on `location_id` + `date`. Because metrics are aggregated daily per restaurant, both fields are required to prevent data cross-contamination.
2.  **Order Tracker ↔ Item Menu Tracker:** Connect via a **Composite Join Rule** matching `price_version_year` to the menu year version, alongside hardcoded mappings of item quantities to their descriptive macro categories (e.g., `qty_pizza` maps to `Main Pizza`).

---

## 📊 Data Dictionary & Schema Registry

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

## 🛡 Data Quality Guardrails & Business Logic Rules

The pipeline enforces several strict operational rules to protect financial metrics from distortion:

1.  **The Cash Processing Fee Rule:** Bank merchant fees (`payment_cost`) are strictly banned on cash orders. All cash rows must have a transaction cost of exactly `0`.
2.  **The Calendar Constraint (Monday Closures):** Operations are closed on Mondays. The pipeline assumes zero revenue and flags anomalies if transaction data appears on a Monday.
3.  **The Price Year Fallback Rule:** If `price_version_year` is left blank or a new year menu hasn't been generated yet, SQL calculations apply a `COALESCE` safety fallback to default to the most recent active year's costs rather than letting ingredient costs drop to `NULL` (which deceptively inflates profit metrics to 100%).
4.  **The Menu Multiplication Rule:** A new row must be created in the `item_menu_tracker` for every new operating year, even if ingredient prices do not change. This isolates historical profit records from inflation spikes.

---

## 🔮 Future Data Roadmap (Things Worth Adding)

As the database scales toward a comprehensive £1.6m business evaluation, the following data schemas should be structured and integrated:

* **Discount & Promotion Tracker:** A separate log for discount voucher codes, loyalty rewards, and manager comps. This bridges the gap between menu `std_unit_price` and actual `order_value_gross`.
* **Actual vs. Theoretical Inventory (Stock Variance):** A table tracking physical stock takes (e.g., kilos of flour/cheese left in the walk-in). This allows us to compare *Actual Food Cost* against the *Theoretical Food Cost* calculated via our menu tracker to pinpoint kitchen theft, over-portioning, or undeclared waste.
* **Labor Efficiency Matrix (`shift_id`):** Breaking daily `labor_cost` down into hourly staff clock-ins. This unlocks **Sales per Labor Hour (SPLH)** metrics, telling us exactly when a site is over or under-staffed.
