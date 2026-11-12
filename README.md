# End-to-End Retail Sales Performance & Financial Variance Dashboard

An end-to-end Excel analytics project: cleaning deliberately messy retail data with **Power Query**, building a **Power Pivot data model**, writing **DAX measures** for Actual vs Budget variance, and presenting an interactive one-page dashboard. All data is **simulated** for a Tanzanian multi-store retailer (TZS).

![Dashboard overview](07_screenshots/dashboard_overview.png)

> **Status:** Data cleaning, the star-schema data model, DAX measures and the Overview dashboard are complete. VBA automation and the final write-up are in progress.

---

## 1. Business problem

A retail chain with 15 stores across Tanzania records sales in several systems and spreadsheets. Management cannot answer basic questions reliably because the data is inconsistent.

The project answers questions such as:
- What are net sales by store, region, category and month?
- How do actual sales compare with budget, and where are the biggest variances?
- How much revenue is lost to returns and discounts?
- Which payment methods, channels and customer types drive sales?
- Which sales representatives and stores perform best?

---

## 2. Tools and skills

| Area | Tools |
|---|---|
| Data cleaning | Excel Power Query (M language) |
| Data model | Power Pivot, star schema, `Dim_Date`, `Dim_Store`, `Dim_Product`, `Dim_Category` |
| Calculations | DAX measures (Net Sales, Budget, Variance, YoY, discounts, dashboard cards) |
| Presentation | PivotTables, PivotCharts, slicers with Report Connections, conditional formatting, `GETPIVOTDATA` KPI cards |
| Documentation | GitHub, Markdown, cleaning logs with before/after screenshots |

---

## 3. Data

| File | Description |
|---|---|
| `sales_2024` and `sales_2025` | Transaction-level sales (two source files appended) |
| `stores_raw.csv` | Store master data |
| `products_raw.csv` | Product master data |
| `budget_targets_raw.csv` | Budget targets |

The raw files contain realistic problems: mixed date formats, currency text in numeric fields (`TZS`, `Tsh`), inconsistent spellings, duplicates, blank IDs, placeholder values, missing amounts and typos. See [`01_data/data_dictionary.md`](01_data/data_dictionary.md).

---

## 4. Cleaning summary

Full detail, with Issue / Fix / Impact / Insight / Recommendation for every step, is in [`03_power_query/cleaning_log.md`](03_power_query/cleaning_log.md). The M code is in [`03_power_query/m_code_queries.md`](03_power_query/m_code_queries.md).

| Area | Result |
|---|---|
| Rows | 7,897 raw rows reduced to **7,822** clean rows (70 duplicates and 5 blank-ID rows removed) |
| Dates | Mixed formats parsed into one date type; future-dated rows **flagged** in `Date Status`, not deleted |
| Unit Price, Quantity, Total | Currency text removed; missing values derived **only** when arithmetic-verifiable; placeholder values set to null |
| Discount | Mixed formats normalised; 472 hidden discounts inferred from Total, each labelled in `Discount Status` |
| Returns | Kept as positive values with an `Is Return` flag (246 return rows) |
| Payment, Customer Type, Channel | Dozens of spellings reduced to standard values |
| Sales Representative | 61 spelling variants reduced to **15** representatives |
| Unknown product code | `P999` kept and documented as an orphan product code |

**Output:** [`02_data_clean/sales_clean.csv`](02_data_clean/sales_clean.csv) (7,822 rows, 16 columns), with a 100-row preview in [`sales_clean_sample.csv`](02_data_clean/sales_clean_sample.csv).

### Data quality principles
1. **Flag, do not hide.** Questionable rows are labelled, not silently removed.
2. **No fake defaults.** A value is derived only when the arithmetic proves it.
3. **Document every assumption** in the cleaning log.

### Validation
An Excel check tests whether `Quantity × Unit Price × (1 − Discount) = Total Amount` on the 7,780 rows where all three values are present. Mismatches fell from 666 (8.6%) to **194 (2.5%)** after discount recovery. The remaining rows are left unchanged because nothing shows which field is wrong.

---

## 5. Data model and DAX

A star schema with `Fact_Sales` (7,822 rows) and `Fact_Budget` (2,160 rows) connected to `Dim_Store` (15), `Dim_Product` (60), `Dim_Category` (6) and `Dim_Date` (731 days) through 7 relationships. See [`05_data_model/data_model.md`](05_data_model/data_model.md).

All measures are in [`06_dax/dax_measures.md`](06_dax/dax_measures.md), and every figure was **validated against an independent Python calculation** from the cleaned files.

| Measure | 2024 | 2025 |
|---|---|---|
| Net Sales | 853.6M | 1,035.3M |
| Budget | 845.7M | 1,053.7M |
| Budget Achievement | 100.9% | 98.2% |
| Return Rate | 3.3% | 4.2% |
| Discount Rate | 5.7% | 6.1% |

---

## 6. Dashboard

A one-page interactive Overview with Year, Month, Region and Category slicers, eight KPI and highlight cards, six charts and a Key Insights panel. Layout, design decisions and limitations are documented in [`08_dashboard/dashboard.md`](08_dashboard/dashboard.md). The workbook is in [`04_excel_workbook/`](04_excel_workbook/).

---

## 7. Key findings (2025)

1. **Growth is strong, but the budget was missed.** Net Sales grew 21.3% to TZS 1,035.3M, yet fell 1.8% (TZS 18.5M) short of budget.
2. **One category carries the result.** Electronics beat budget by TZS 29.3M (+7.5%) and delivers 40.7% of Net Sales, offsetting shortfalls in Home Appliances (-10.6%), Fashion (-6.2%), Groceries (-4.7%) and Stationery (-4.0%).
3. **Sales are concentrated in Dar es Salaam** (35.7% of Net Sales). Morogoro Road (+10.3%) is the best store against budget and Mbeya Mjini (-22.5%) the weakest.
4. **Mobile money dominates payments**: M-Pesa, Airtel Money and Mixx by Yas make up about 57% of Net Sales.
5. **Returns and discounts are rising**: return rate from 3.3% to 4.2%, discount rate from 5.7% to 6.1%.
6. **Data quality matters.** Blank discounts hid real price reductions, returns stored as positive amounts would have overstated revenue, and 61 spellings of 15 sales reps would have split rankings.

### Recommendations
- Review the Home Appliances budget and stock plan; it is the largest gap against target.
- Investigate Mbeya Mjini and Kariakoo, and confirm the status of Tanga Mjini.
- Look into the causes of the higher return rate.
- Track discount depth to protect margin as sales grow.
- Use validated dropdowns for payment method, channel and customer type to prevent free-text errors.

---

## 8. Repository structure

```text
retail-sales-financial-variance-dashboard/
├── 01_data/              Raw CSV files and data dictionary
├── 02_data_clean/        Cleaned outputs (sales_clean.csv and a 100-row sample)
├── 03_power_query/       M code and cleaning logs
├── 04_excel_workbook/    retail_sales_dashboard.xlsx
├── 05_data_model/        Star schema and relationships
├── 06_dax/               dax_measures.md
├── 07_screenshots/       Cleaning screenshots and dashboard_overview.png
├── 08_dashboard/         dashboard.md
└── README.md
```

---

## 9. Roadmap

- [x] Project design and simulated messy data
- [x] Clean and document Sales, Stores, Products and Budget in Power Query
- [x] Build `Dim_Date` and the star-schema data model
- [x] DAX measures: Net Sales, Budget, Variance, YoY, discounts
- [x] Interactive Overview dashboard with slicers
- [ ] VBA automation
- [ ] Final insights and recommendations write-up

---

## 10. Known limitations

- Data is simulated, so findings illustrate method, not a real business.
- Budget is **draft v3**; every variance figure inherits that risk.
- 194 sales rows (2.5%) do not reconcile arithmetically and are kept as recorded.
- Quantity errors in the source (350 and 500 units on lines worth one unit, and 22 missing quantities) distort profit and units sold, so those are kept out of the dashboard KPIs. See [`06_dax/dax_measures.md`](06_dax/dax_measures.md).
- Rows dated after 2025-12-31 are flagged, not corrected.
- The orphan product code `P999` does not exist in the product master.
- `Channel` is `Unassigned` for most 2024 rows because the field was not captured in that period.
- Tanga Mjini has a budget but no sales.

---

## 11. Author

**Yasinta**, Dar es Salaam, Tanzania. Sales and Marketing professional with a finance and banking degree, transitioning into data analytics.

Related project: SQL / BigQuery churn analysis for Airtel home broadband subscribers.
