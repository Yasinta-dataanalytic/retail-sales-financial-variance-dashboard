# End-to-End Retail Sales Performance & Financial Variance Dashboard

An end-to-end Excel analytics project: cleaning deliberately messy retail data with **Power Query**, building a **Power Pivot data model**, writing **DAX measures** for Actual vs Budget variance, automating with **VBA**, and presenting an interactive dashboard. All data is **simulated** for a Tanzanian multi-store retailer (TZS).

> **Status:** Sales data cleaning is complete and documented. Stores, Products and Budget cleaning, the data model, DAX, VBA and the dashboard are in progress.

---

## 1. Business problem

A retail chain with 15+ stores across Tanzania records sales in several systems and spreadsheets. Management cannot answer basic questions reliably because the data is inconsistent.

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
| Data model | Power Pivot, star schema, `Dim_Date`, `Dim_Store`, `Dim_Category` |
| Calculations | DAX measures (Gross Sales, Returns, Net Sales, Budget, Variance) |
| Automation | VBA |
| Presentation | PivotTables, slicers, conditional formatting, data validation |
| Documentation | GitHub, Markdown, cleaning log with before/after screenshots |

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

## 4. Sales cleaning summary

Full detail, with Issue / Fix / Impact / Insight / Recommendation for every step, is in [`03_power_query/cleaning_log.md`](03_power_query/cleaning_log.md). The M code is in [`03_power_query/m_code_queries.md`](03_power_query/m_code_queries.md).

| Area | Result |
|---|---|
| Rows | 7,897 raw rows reduced to **7,822** clean rows (70 duplicates and 5 blank-ID rows removed) |
| Source files | 2024 and 2025 appended into one `Sales` query |
| Dates | Mixed formats parsed into one date type; future-dated rows **flagged** in `Date Status`, not deleted |
| Unit Price, Quantity, Total | Currency text removed; missing values derived **only** when arithmetic-verifiable; the placeholder quantity `999` and a placeholder price were set to null |
| Discount | Mixed formats (`10%`, `10`, `0.1`) normalised; 472 hidden discounts inferred from Total, each labelled in `Discount Status` |
| Returns | Kept as positive values with an `Is Return` flag (246 return rows) |
| Payment Method | Dozens of spellings reduced to 6 standard methods |
| Customer Type | Reduced to 3 clean values: Retail, Wholesale, Corporate |
| Channel | Standardised to In-store, WhatsApp, Online, Phone and Unassigned |
| Sales Representative | 61 spelling variants reduced to **15** representatives (case, spaces and repeated inner spaces fixed) |
| Unknown product code | `P999` kept and documented as an orphan product code |

**Output:** [`02_data_clean/sales_clean.csv`](02_data_clean/sales_clean.csv) (7,822 rows, 16 columns). GitHub cannot preview large CSV files as a table, so a 100-row preview is in [`sales_clean_sample.csv`](02_data_clean/sales_clean_sample.csv).

### Data quality principles
1. **Flag, do not hide.** Questionable rows are labelled, not silently removed.
2. **No fake defaults.** A value is derived only when the arithmetic proves it.
3. **Document every assumption** in the cleaning log.

### Validation: reconciliation check
An Excel check tests whether `Quantity × Unit Price × (1 − Discount) = Total Amount` on the 7,780 rows where all three values are present.

| Stage | Mismatches |
|---|---|
| Before discount recovery | 666 (8.6%) |
| After discount recovery | **194 (2.5%)** |

The remaining 194 rows are left unchanged because there is no evidence of which field is wrong. This is documented as a known limitation.

---

## 5. Key findings so far

- Blank discounts were not random. They hid real price reductions, so treating every blank as 0% would have understated discounting.
- Returns are recorded as positive amounts and must be separated from sales to avoid overstating revenue.
- The same sales representative appeared under many spellings, which would have split rankings and understated top performers.
- Free-text entry for payment method, channel and customer type caused most of the inconsistency; validated dropdowns would prevent it.

---

## 6. Repository structure

```text
retail-sales-financial-variance-dashboard/
├── 01_data/              Raw CSV files and data dictionary
├── 02_data_clean/        Cleaned outputs (sales_clean.csv and a 100-row sample)
├── 03_power_query/       m_code_queries.md and cleaning_log.md
├── 07_screenshots/       Before/after screenshots of each cleaning step
└── README.md
```

Further folders (data model, DAX, VBA, dashboard) will be added as each phase is completed.

---

## 7. Roadmap

- [x] Project design and simulated messy data
- [x] Sales cleaning in Power Query, documented
- [x] Clean Sales export
- [ ] Clean and document Stores, Products and Budget
- [ ] Build `Dim_Date` and the star-schema data model
- [ ] DAX measures: Gross Sales, Returns, Net Sales, Budget, Variance %
- [ ] VBA automation
- [ ] Interactive dashboard with slicers, conditional formatting and data validation
- [ ] Final insights and recommendations

---

## 8. Known limitations

- Data is simulated, so findings illustrate method, not a real business.
- 194 sales rows (2.5%) do not reconcile arithmetically and are kept as recorded.
- Rows dated after 2025-12-31 are flagged, not corrected.
- The orphan product code `P999` does not exist in the product master.
- `Channel` is `Unassigned` for most 2024 rows because the field was not captured in that period.

---

## 9. Author

**Yasinta** — Dar es Salaam, Tanzania. Sales and Marketing professional with a finance and banking degree, transitioning into data analytics.

Related project: SQL / BigQuery churn analysis for Airtel home broadband subscribers.
