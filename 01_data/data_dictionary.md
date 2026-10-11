# Data Dictionary — Retail Sales & Financial Variance Project

> **Simulated dataset** created for portfolio purposes. All stores, products, people, and figures are fictional. Currency: Tanzanian Shillings (TZS). Period: **Jan 2024 – Dec 2025** (24 months — enough for YoY / time-intelligence work).

## Files

| File | Grain (one row = …) | Approx. rows | Role in model |
|---|---|---|---|
| `sales_transactions_raw.csv` | one sale/return line | ~7,900 (raw, incl. duplicates & blanks) | Fact — Actuals |
| `products_raw.csv` | one product | ~63 (raw) | Dimension — Product |
| `stores_raw.csv` | one store | ~17 (raw) | Dimension — Store |
| `budget_targets_raw.csv` | one store × category, months across columns | 90 data rows | Fact — Budget (needs unpivot) |

## Columns

**sales_transactions_raw.csv**
| Column | Meaning |
|---|---|
| TransactionID | Unique sale line ID (`TXN######`) |
| Txn_Date | Date of sale |
| StoreID | Links to Stores |
| ProductID | Links to Products |
| Qty | Units sold (negative = return) |
| Unit Price | Selling price per unit, TZS |
| Discount | Discount given on the line |
| Payment Method | M-Pesa, Airtel Money, Tigo Pesa, Cash, Card, Bank Transfer |
| Customer Type | Retail, Wholesale, Corporate |
| Sales Rep | Sales representative name |
| Total Amount | Net line value after discount, TZS |
| Notes | Free text |

**products_raw.csv** — Product ID, Product Name, Category, Unit Cost, Unit Price, Supplier, Launch Date, Active?

**stores_raw.csv** — Store ID, Store Name, Region, Store Type, Manager, Open Date, Floor Area, Status

**budget_targets_raw.csv** — Store (by *name*, not ID), Category, then one column per month (Jan-24 … Dec-25) holding the budget target in TZS.

## Suggested data model (star schema)

```
            Dim_Date (build it yourself)
                 |
Dim_Store ---- Fact_Sales ---- Dim_Product
                 |
   Fact_Budget (unpivoted, at Store × Category × Month)
```
Budget is at Store × Category × Month grain, while Sales is at transaction grain — think about how to relate them (hint: shared dimensions / a Month key).

## Suggested workflow
1. **Power Query**: clean each table (one query per file), unpivot budget, load as connection only.
2. **Data Model (Power Pivot)**: build relationships, create Dim_Date.
3. **DAX**: Actual Sales, Budget, Variance, Variance %, YoY %, Margin, Return Rate, Top-N …
4. **PivotTables + Slicers + Charts**: dashboard pages (Overview, Variance, Store/Product drill-down).
5. **VBA**: navigation buttons, Refresh All, export-to-PDF.
6. **Data Validation / Conditional Formatting**: input controls and variance traffic-lights.
