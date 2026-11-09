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
| `YoY Growth %` | `DIVIDE([Net Sales] - [Net Sales PY], [Net Sales PY])` | Are we growing? |
| `Net Sales YTD` | `TOTALYTD([Net Sales], Dim_Date[Date])` | How much have we sold so far this year? |

## Profit measures

| Measure | Formula | Business question |
|---|---|---|
| `Gross Profit` | `CALCULATE(SUM(Fact_Sales[Line Profit]), Is Return = FALSE(), Date Status = "Valid") - CALCULATE(SUM(Fact_Sales[Line Profit]), Is Return = TRUE(), Date Status = "Valid")` | How much did we earn after product cost? |
| `Costed Net Sales` | Same pattern with `SUM(Fact_Sales[Costed Amount])` | Net sales of lines with a known cost |
| `Gross Margin %` | `DIVIDE([Gross Profit], [Costed Net Sales])` | How much of each shilling is profit? |
| `Gross Profit PY` | `CALCULATE([Gross Profit], SAMEPERIODLASTYEAR(Dim_Date[Date]))` | Profit in the same period last year |
| `Profit YoY %` | `DIVIDE([Gross Profit] - [Gross Profit PY], [Gross Profit PY])` | Is profit growing? |

(`Is Return` and `Date Status` stand for `Fact_Sales[Is Return]` and `Fact_Sales[Date Status]`.)

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
| Costed Net Sales | 831,967,860 | 1,001,891,635 | 1,833,859,495 |
| Gross Profit | 111,527,560 | 231,760,335 | 343,287,895 |
| Gross Margin % | 13.4% | 23.1% | 18.7% |

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
- **`Net Sales YTD` grand total** shows the last year-end, not a sum across years.
- **Margin jump from 13.4% to 23.1%** is large. By category, Home Appliances moved from about -1.4% (2024) to 25.0% (2025), which explains most of it. Worth investigating before drawing conclusions.
- **Budget is draft v3**, so every variance figure inherits that risk.
