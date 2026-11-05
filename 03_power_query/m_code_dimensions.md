# M Code: Dimension Tables (`Dim_Date`, `Dim_Category`)

These two tables do not exist in the raw files. They are created in Power Query and loaded into the Power Pivot data model with the cleaned fact and dimension tables.

## Why they are needed

| Table | Reason |
|---|---|
| `Dim_Date` | Sales have a transaction date and the budget has a month. One shared calendar lets a single Year/Month slicer filter both, and makes time-intelligence DAX possible. |
| `Dim_Category` | Sales know a category only through the product, the budget knows it directly. One category table filters both. |

## `Dim_Date`

Covers 1 Jan 2024 to 31 Dec 2025 (731 days), the period of the sales and budget data. Three sales rows dated after 2025-12-31 are flagged in `Date Status` and fall outside this calendar on purpose.

```m
let
    StartDate = #date(2024, 1, 1),
    EndDate = #date(2025, 12, 31),
    DayCount = Duration.Days(EndDate - StartDate) + 1,
    Dates = List.Dates(StartDate, DayCount, #duration(1, 0, 0, 0)),
    ToTable = Table.FromList(Dates, Splitter.SplitByNothing(), {"Date"}, null, ExtraValues.Error),
    Typed = Table.TransformColumnTypes(ToTable, {{"Date", type date}}),
    AddYear = Table.AddColumn(Typed, "Year", each Date.Year([Date]), Int64.Type),
    AddQuarter = Table.AddColumn(AddYear, "Quarter", each "Q" & Text.From(Date.QuarterOfYear([Date])), type text),
    AddMonthNumber = Table.AddColumn(AddQuarter, "Month Number", each Date.Month([Date]), Int64.Type),
    AddMonth = Table.AddColumn(AddMonthNumber, "Month", each Date.ToText([Date], "MMMM", "en-US"), type text),
    AddMonthShort = Table.AddColumn(AddMonth, "Month Short", each Date.ToText([Date], "MMM", "en-US"), type text),
    AddYearMonth = Table.AddColumn(AddMonthShort, "Year-Month", each Date.ToText([Date], "yyyy-MM", "en-US"), type text),
    AddMonthStart = Table.AddColumn(AddYearMonth, "Month Start", each Date.StartOfMonth([Date]), type date),
    AddDayName = Table.AddColumn(AddMonthStart, "Day Name", each Date.ToText([Date], "dddd", "en-US"), type text),
    AddDayNumber = Table.AddColumn(AddDayName, "Day of Week", each Date.DayOfWeek([Date], Day.Monday) + 1, Int64.Type),
    AddDayType = Table.AddColumn(AddDayNumber, "Day Type", each if Date.DayOfWeek([Date], Day.Monday) >= 5 then "Weekend" else "Weekday", type text)
in
    AddDayType
```

## `Dim_Category`

Built from the cleaned `Products` query, so it always has the same categories as the product master (6).

```m
let
    Source = Table.SelectColumns(Products, {"Category"}),
    Distinct = Table.Distinct(Source),
    Sorted = Table.Sort(Distinct, {{"Category", Order.Ascending}}),
    Typed = Table.TransformColumnTypes(Sorted, {{"Category", type text}})
in
    Typed
```
[Dim_Category profile](../07_screenshots/power_query/dimensions/dim_category_profile.png).
## Checks

| Table | Expected rows |
|---|---|
| `Dim_Date` | 731 (366 days in 2024 + 365 days in 2025) |
| `Dim_Category` | 6 |
