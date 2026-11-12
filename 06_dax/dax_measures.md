# DAX Measures

All measures live in the Power Pivot data model (`Fact_Sales` and `Fact_Budget` calculation areas). They are built on the star schema described in [`../05_data_model/data_model.md`](../05_data_model/data_model.md).

Every figure below was **validated against an independent Python calculation** from the cleaned CSV files.

## Calculated columns on `Fact_Sales`

Three columns were added so profit can be summed row by row. They use the existing relationship `Fact_Sales[ProductID]` to `Dim_Product[Product ID]`.

| Column | Formula | Purpose |
|---|---|---|
| `Unit Cost` | `=RELATED(Dim_Product[Unit Cost])` | Cost of one unit, looked up from the product master |
| `Line Profit` | `=IF(OR(ISBLANK(Fact_Sales[Unit Cost]), ISBLANK(Fact_Sales[Total Amount])), BLANK(), Fact_Sales[Total Amount] - Fact_Sales[Quantity] * Fact_Sales[Unit Cost])` | Profit of one sales line. Blank when cost or amount is unknown |
| `Costed Amount` | `=IF(ISBLANK(Fact_Sales[Unit Cost]), BLANK(), Fact_Sales[Total Amount])` | Revenue of lines that have a known cost (denominator for margin) |

## Sales measures

| Measure | Formula | Business question |
|---|---|---|
| `Gross Sales` | `CALCULATE(SUM(Fact_Sales[Total Amount]), Fact_Sales[Is Return] = FALSE(), Fact_Sales[Date Status] = "Valid")` | How much did we sell before returns? |
| `Returns` | Same as `Gross Sales` with `Is Return = TRUE()` | How much was returned? |
| `Net Sales` | `[Gross Sales] - [Returns]` | What revenue did we keep? |
| `Return Rate %` | `DIVIDE([Returns], [Gross Sales])` | How much of what we sell comes back? |
| `Transactions` | `CALCULATE(COUNTROWS(Fact_Sales), Fact_Sales[Is Return] = FALSE(), Fact_Sales[Date Status] = "Valid", Fact_Sales[Total Amount] <> BLANK())` | How many sales were made? |
| `Avg Transaction Value` | `DIVIDE([Gross Sales], [Transactions])` | How big is a typical sale? |
| `Units Sold` | `CALCULATE(SUM(Fact_Sales[Quantity]), Fact_Sales[Is Return] = FALSE(), Fact_Sales[Date Status] = "Valid", Fact_Sales[Total Amount] <> BLANK())` | How many items did we sell? |

## Budget measures

| Measure | Formula | Business question |
|---|---|---|
| `Budget` | `SUM(Fact_Budget[Value])` | What was the target? |
| `Variance` | `IF(ISBLANK([Budget]), BLANK(), [Net Sales] - [Budget])` | How far are we from the target (TZS)? Blank when no budget exists |
| `Variance %` | `DIVIDE([Variance], [Budget])` | How far are we from the target (%)? |
| `Budget Achievement %` | `DIVIDE([Net Sales], [Budget])` | What share of the target did we reach? |

## Time intelligence measures

These need `Dim_Date` marked as the date table and must be used with `Year` or `Month` from `Dim_Date`.

| Measure | Formula | Business question |
|---|---|---|
| `Net Sales PY` | `CALCULATE([Net Sales], SAMEPERIODLASTYEAR(Dim_Date[Date]))` | What did we sell in the same period last year? |
| `YoY Growth %` | `IF(HASONEVALUE(Dim_Date[Year]), DIVIDE([Net Sales] - [Net Sales PY], [Net Sales PY]), BLANK())` | Are we growing? Blank unless exactly one year is selected, so it can never compare two years with one |
| `Net Sales YTD` | `TOTALYTD([Net Sales], Dim_Date[Date])` | How much have we sold so far this year? |

## Discount measures

Answer the project question "how much revenue is lost to discounts?". Only valid, non-return lines with a `Total Amount` are used.

| Measure | Formula | Business question |
|---|---|---|
| `Discount Given` | `SUMX(FILTER(Fact_Sales, Fact_Sales[Is Return] = FALSE() && Fact_Sales[Date Status] = "Valid" && NOT(ISBLANK(Fact_Sales[Total Amount]))), Fact_Sales[Quantity] * Fact_Sales[Unit Price] * Fact_Sales[Discount])` | How much revenue did we give away in discounts (TZS)? |
| `Discount Rate %` | `DIVIDE([Discount Given], SUMX(<same FILTER>, Fact_Sales[Quantity] * Fact_Sales[Unit Price]))` | What share of list-price sales was discounted? |

## Profit measures (supplementary, use with caution)

Profit is **not** one of the core project questions. The measures are kept in the model as an extra, with the data-quality caveat described below. They are not used as KPI cards on the Overview page.

| Measure | Formula | Business question |
|---|---|---|
| `Gross Profit` | `CALCULATE(SUM(Fact_Sales[Line Profit]), Is Return = FALSE(), Date Status = "Valid") - CALCULATE(SUM(Fact_Sales[Line Profit]), Is Return = TRUE(), Date Status = "Valid")` | How much did we earn after product cost? |
| `Costed Net Sales` | Same pattern with `SUM(Fact_Sales[Costed Amount])` | Net sales of lines with a known cost |
| `Gross Margin %` | `DIVIDE([Gross Profit], [Costed Net Sales])` | How much of each shilling is profit? |
| `Gross Profit PY` | `CALCULATE([Gross Profit], SAMEPERIODLASTYEAR(Dim_Date[Date]))` | Profit in the same period last year |
| `Profit YoY %` | `IF(HASONEVALUE(Dim_Date[Year]), DIVIDE([Gross Profit] - [Gross Profit PY], [Gross Profit PY]), BLANK())` | Is profit growing? Blank unless one year is selected |

(`Is Return` and `Date Status` stand for `Fact_Sales[Is Return]` and `Fact_Sales[Date Status]`.)

## Dashboard card measures

Used by the "Top Region" and "Top Category" highlight cards on the Overview page. They deliberately **ignore the slicer of their own dimension** (`ALL`), so the card keeps showing the best region (or category) overall while the other slicers (Year, Category / Region) still apply.

| Measure | Formula | Business question |
|---|---|---|
| `Top Region` | `CONCATENATEX(TOPN(1, ADDCOLUMNS(ALL(Dim_Store[Region]), "@ns", [Net Sales]), [@ns], DESC), Dim_Store[Region], ", ")` | Which region sells the most? |
| `Top Region Sales` | `MAXX(ALL(Dim_Store[Region]), [Net Sales])` | How much did it sell? |
| `Top Category` | `CONCATENATEX(TOPN(1, ADDCOLUMNS(ALL(Dim_Category[Category]), "@ns", [Net Sales]), [@ns], DESC), Dim_Category[Category], ", ")` | Which category sells the most? |
| `Top Category Sales` | `MAXX(ALL(Dim_Category[Category]), [Net Sales])` | How much did it sell? |
| `Selected Region` | `IF(HASONEVALUE(Dim_Store[Region]), VALUES(Dim_Store[Region]), "All Regions")` | Which single region is selected in the slicer? |

On the dashboard, one card switches between the two views with plain worksheet formulas: when `Selected Region` is `"All Regions"` it shows `Top Region` and `Top Region Sales`, otherwise it shows the selected region and its `Net Sales`.

## Validated results

| Measure | 2024 | 2025 | Total |
|---|---|---|---|
| Gross Sales | | | 1,962,926,998 |
| Returns | | | 74,025,948 |
| Net Sales | 853,648,585 | 1,035,252,465 | 1,888,901,050 |
| Return Rate % | 3.30% | 4.16% | 3.77% |
| Transactions | 3,556 | 3,989 | 7,545 |
| Avg Transaction Value | 248,253 | 270,779 | 260,163 |
| Units Sold | 16,216 | 17,466 | 33,682 |
| Budget | 845,740,000 | 1,053,730,000 | 1,899,470,000 |
| Variance | 7,908,585 | -18,477,535 | -10,568,950 |
| Budget Achievement % | 100.9% | 98.2% | 99.4% |
| YoY Growth % | blank (no 2023 data) | 21.3% | |
| Discount Given | 56,335,600 | 74,419,625 | 130,755,225 |
| Discount Rate % | 5.7% | 6.1% | 5.9% |
| Costed Net Sales (supplementary) | 831,967,860 | 1,001,891,635 | 1,833,859,495 |
| Gross Profit (supplementary) | 111,527,560 | 231,760,335 | 343,287,895 |
| Gross Margin % (supplementary) | 13.4% | 23.1% | 18.7% |

The profit figures match an independent Python calculation of the same logic. See the data-quality caveat below: the logic itself is affected by quantity errors in the source.

Net Sales by category matches the budget view: Electronics 797,730,559, Home Appliances 472,190,800, Groceries 230,248,873, Fashion 197,729,027, Stationery 108,622,716, Beauty 81,060,175, blank (product `P999`) 1,318,900.

## Design decisions and assumptions

| Decision | Reason |
|---|---|
| Returns are subtracted, not stored as negative rows | The source stores returns as positive rows with `Is Return = TRUE`. Gross, Returns and Net stay visible separately. |
| Only `Date Status = "Valid"` rows are counted | 3 rows have dates up to 2052 and fall outside `Dim_Date`. |
| `Variance` is blank when there is no budget | Null budget means unknown, not zero (see the budget cleaning log). |
| Profit uses only lines with a known unit cost | Products `P039` and `P051` have no cost and `P999` is not in the product master (259 sales lines). Including their revenue without cost would inflate margin. `Costed Net Sales` covers about 97% of `Net Sales`. |
| A returned item is assumed to go back to stock | Its cost is reversed together with its revenue. |
| `Unit Cost` is one fixed value per product | Margin changes come from product mix, price and discount, not from supplier cost changes. |
| Profit is **Gross Profit** | Operating costs (rent, salaries) are not in the data. |

## Things to read carefully

- **Grand Total of `YoY Growth %` is misleading** (121.3% in a PivotTable without a year filter). It compares both years with the 2024 total. Use it only with `Year` or `Month` on rows or a `Year` slicer.
- **Time intelligence needs a contiguous date selection.** `SAMEPERIODLASTYEAR` returns an error when the selected dates have a gap, for example when the Year slicer is cleared while the Month slicer has only a few months selected (Jan 2024 and Jan 2025). Keep one year selected, or clear the Month slicer first.
- **`Net Sales YTD` grand total** shows the last year-end, not a sum across years.
- **Profit is affected by quantity errors in the source.** Profit is `Total Amount - Quantity x Unit Cost`, so a wrong `Quantity` distorts it directly.
  - 5 lines have a quantity of 350 or 500. At least three (for example `Txn000179` and `Txn000976`, product `P013`, 350 and 500 units but a total of 66,975, the price of one unit) look like typing errors. Each one adds roughly TZS 18M of cost.
  - 12 lines with a known cost have no `Quantity` (TZS 8.75M of sales). The model treats their cost as zero, so their full revenue counts as profit.
  - Without these lines the margin is about **18.0% in 2024 and 22.0% in 2025**, against 13.4% and 23.1% in the model. The large margin jump seen in the model is mostly caused by the quantity errors.
  - `Units Sold` is inflated by about 2,050 units for the same reason (33,682 in the model, about 31,986 with the five lines set to one unit).
  - These lines were not changed in Power Query because the correct quantity cannot be proven from the data. Recommendation: confirm them with the source system before using profit or units in decisions.
- **Discount figures** use `Quantity x Unit Price x Discount`. 187 lines have a `Total Amount` that does not match list price after the recorded discount, so the discount shown is the recorded one, not the gap between list price and amount paid.
- **Budget is draft v3**, so every variance figure inherits that risk.
