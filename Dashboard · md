# Overview Dashboard

A single-page, interactive Excel dashboard built on the Power Pivot data model and the DAX measures in [`../06_dax/dax_measures.md`](../06_dax/dax_measures.md). It answers one question for management: **are we meeting budget, and where is the gap?**

![Dashboard overview](../07_screenshots/dashboard_overview.png)

*Screenshot: Year 2025, all regions and categories selected. Data is simulated; amounts are in TZS. Budget is draft v3.*

---

## How to use it

| Control | What it does |
|---|---|
| **Year** slicer | One year at a time (multi-select is off). Year-on-year growth needs exactly one year. |
| **Month** slicer | Filters every card and chart to the chosen months (short names, Jan to Dec). |
| **Region** slicer | Filters KPIs and charts to one or more regions. |
| **Category** slicer | Filters KPIs and charts to one or more product categories. |

All slicers are connected to every PivotTable behind the dashboard (Report Connections), so every card and chart reacts to every slicer.

---

## KPI cards

| Card | Measure | Value (2025) |
|---|---|---|
| Net Sales | `Net Sales` | TZS 1,035.3M |
| Budget Achievement | `Budget Achievement %` | 98.2% |
| Variance % | `Variance %` | -1.8% (TZS -18.5M) |
| YoY Growth | `YoY Growth %` | +21.3% |
| Return Rate | `Return Rate %` | 4.2% |
| Discount Rate | `Discount Rate %` | 6.1% |
| Top Region | `Top Region`, `Top Region Sales` | Dar es Salaam, TZS 369.8M |
| Top Category | `Top Category`, `Top Category Sales` | Electronics, TZS 421.3M |

The Top Region card switches to the selected region and its Net Sales when a single region is chosen in the slicer, and returns to the best region when the slicer is cleared.

---

## Charts

| Chart | Question it answers |
|---|---|
| Net Sales by Region | Where do we sell most? |
| Variance % by Store (Actual vs Budget) | Which stores beat or miss budget? |
| Variance % by Category | Which categories drive the gap? |
| Net Sales by Payment Method | How do customers pay? |
| Net Sales vs Budget by Month | Are we on track through the year? |
| Top 10 Sales Reps | Who sells the most? |
| Key Insights panel | What should the reader take away? |

---

## Design decisions

| Decision | Reason |
|---|---|
| One-page layout with slicers on the left | Filters are always visible; no navigation needed |
| Navy for actuals, red for budget and negative variance | Two colours carry meaning; the rest stays neutral |
| Bars, not pie or donut charts | Easier to compare 6 payment methods and 6 categories |
| Funnel and map charts rejected | A funnel implies a sequence that categories do not have; map charts cannot be PivotCharts and need geography recognition |
| Data labels instead of axes on bar charts | Fewer elements, exact values visible |
| Short month names | Axis labels stay horizontal and readable |
| Store variance axis capped at -30% | Tanga Mjini (-100%, no sales against a budget) would otherwise flatten every other bar |

---

## Key insights (2025)

1. **Growth is strong, but the budget was missed.** Net Sales grew 21.3% year on year to TZS 1,035.3M, yet fell 1.8% (TZS 18.5M) short of the TZS 1,053.7M budget.
2. **One category carries the result.** Electronics beat its budget by TZS 29.3M (+7.5%) and delivers 40.7% of Net Sales. It offset shortfalls in Home Appliances (-TZS 32.5M, -10.6%), Fashion (-6.2%), Groceries (-4.7%) and Stationery (-4.0%).
3. **Sales are concentrated in Dar es Salaam**, which generates 35.7% of Net Sales. Morogoro Road (+10.3%) and Stone Town (+8.7%) beat budget; Kariakoo (-8.5%) and Mbeya Mjini (-22.5%) fell short. Tanga Mjini recorded no sales.
4. **Mobile money dominates payments.** M-Pesa, Airtel Money and Mixx by Yas together account for about 57% of Net Sales, led by M-Pesa at TZS 349.4M.
5. **Returns and discounts are rising.** Return rate increased from 3.3% to 4.2% and the discount rate from 5.7% to 6.1%.

## Recommendations

- Review the Home Appliances budget and stock plan; it is the largest gap against target.
- Investigate Mbeya Mjini and Kariakoo, and confirm the status of Tanga Mjini.
- Look into the causes of the higher return rate (product quality, return policy).
- Track discount depth to protect margin as sales grow.

---

## Known behaviour and limitations

- **Budget is draft v3**, so every variance figure inherits that risk.
- **Year-on-year needs one year selected.** With several years selected the card is blank by design.
- **Time intelligence needs a contiguous date selection.** If the Year slicer is cleared while the Month slicer has only some months selected, `SAMEPERIODLASTYEAR` can return an error. Clear the Month slicer first, or keep one year selected.
- **Profit and units are not shown.** Quantity errors in the source (350 and 500 units on lines worth one unit, and missing quantities) distort them; see the caveats in `dax_measures.md`.
- **Tanga Mjini** has a budget but no sales, so it shows -100% in the store chart.
- Data is simulated, so the findings illustrate the method, not a real business.
