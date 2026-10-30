# End-to-End Retail Sales Performance & Financial Variance Dashboard

### Sales & Financial Performance Analytics

An Excel-based analytics solution that takes messy retail sales data through cleaning, modelling and automation to a decision-ready dashboard comparing **actual performance against budget**.

> **Note:** All data in this project is **simulated** (a fictional Tanzanian retail chain with 15 stores) and was deliberately made messy to practise real-world data cleaning. No real company or customer data is used.

> **Status:** 🚧 In progress. Sales data cleaning is complete and documented; the other tables, data model, DAX, VBA and dashboard are in progress (see roadmap below).

---

## Business problem
A retail chain needs to know, month by month, whether stores and product categories are meeting their budget, where the gaps come from, and what to do about them. The raw data arrives as messy yearly exports with inconsistent formats, so it must be cleaned and modelled before any reliable analysis is possible.

## Tools & skills demonstrated
| Area | Tools / techniques |
|---|---|
| Data cleaning | Power Query (M): custom functions, `try … otherwise` parsing, conditional columns, Append, Unpivot, Merge |
| Data modelling | Power Pivot star schema (fact tables + date, store and category dimensions) |
| Measures | DAX: Gross Sales, Returns, Net Sales, Actual vs Budget variance |
| Automation | VBA / macros |
| Reporting | Interactive Excel dashboard with slicers |
| Documentation | Cleaning log (Issue → Fix → Impact → Insight → Recommendation), versioned M code |

## Data cleaning highlights (Sales)
Seven thousand eight hundred rows of raw transactions arrived in two yearly files with different structures. Every issue was documented with evidence, a fix and a business recommendation in the [cleaning log](./03_power_query/cleaning_log.md).

| Issue | Before | After |
|---|---|---|
| Two yearly files appended; duplicates and blank rows removed | 7,897 rows | **7,822 rows** |
| Dates stored as text in 5+ formats | 3,262 distinct values | 734 (731 days + 3 flagged) |
| Future-dated transactions (e.g. year 2052) | undetected | **3 flagged**, not deleted |
| Payment method spellings | 29 spellings | **6 methods** |
| Customer type spellings | 13 spellings | **3 types** |
| Sales channel spellings | 10 values | **5 values** |
| Sales representative spellings | 61 names | **30 people** |
| Missing unit prices | 8% empty | 0.4% empty |
| Missing totals | 4% empty | 0.4% empty |
| Returns recorded as negatives | mixed into sales | **246 flagged** with `Is Return` |

### Cleaning principles
1. **Report, don't hide.** Anomalies are flagged, not silently deleted.
2. **No fake defaults.** Missing values are derived from other fields where possible, otherwise left as `null`.
3. **Returns are kept** as positive values with an `Is Return` flag so Net Sales can be calculated in DAX.
4. **Every assumption is documented** (for example, blank discounts treated as 0%).

## Repository structure
```
├── 01_data/                 Raw (simulated) datasets and data dictionary
├── 03_power_query/
│   ├── cleaning_log.md      Every data issue, fix, impact, insight and recommendation
│   └── m_code_queries.md    All Power Query M code used
├── 07_screenshots/          Evidence: before/after profiles, dashboard
└── README.md
```
Folders for the data model, DAX measures, VBA and dashboard will be added as each stage is completed.

## Roadmap
- [x] Simulated messy datasets and data dictionary
- [x] Sales cleaning (Power Query), documented
- [ ] Stores, Products and Budget cleaning documented
- [ ] Date, Store and Category dimensions
- [ ] Power Pivot data model and relationships
- [ ] DAX measures (Gross Sales, Returns, Net Sales, Budget variance)
- [ ] VBA automation
- [ ] Interactive dashboard and final insights

## Author
**Yasinta**, transitioning from Sales & Marketing into Data Analytics. Dar es Salaam, Tanzania.
