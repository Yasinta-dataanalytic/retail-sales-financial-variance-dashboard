# M Code: Budget Targets

Power Query (M) for the `Budget targets` query. Replace `<path>` with the folder that holds `budget_targets_raw.csv`.
This query reads the `Stores` query (merge on `Store Name`), so `Stores` must exist in the same workbook.

See [`cleaning_log_budget.md`](cleaning_log_budget.md) for the reason behind each step.

## Steps at a glance

| Step | Purpose | Log section |
|---|---|---|
| `Removed Top Rows`, `Promoted Headers` | Skip the 3 title lines, use real header | 1 |
| `Trimmed Text`, `Capitalized Each Word` | Fix case and spaces in Store and Category | 3 |
| `Unpivoted Other Columns`, `Renamed Columns` | Months from columns to rows | 2 |
| `Replaced Value` | Remove `TZS` | 4 |
| `Added Custom1`, `Changed Type4` | Create `Budget Status` before any replacement | 6 |
| `Replaced Value1` to `Replaced Value5`, `Changed Type` | Turn text into numbers (placeholders temporarily 0) | 4, 6 |
| `Added Custom`, `Changed Type1` | `Month Start` date | 5 |
| `Replaced Value6`, `Replaced Value7` | Store names to match the `Stores` master | 7 |
| `Merged Queries`, `Expanded Stores`, `Renamed Columns2` | Look up `Store ID` | 7 |
| `Removed Unmatched and Total Rows` | Remove 72 footer rows | 8 |
| `Replaced Value8` to `Replaced Value10` | Category variants (whole-cell) | 9 |
| `Changed Type3`, `Added Custom2`, `Changed Type5` | Value becomes null unless `Budgeted`, whole number | 6, 4 |
| `Removed Columns1`, `Reordered Columns2`, `Renamed Columns3` | Final columns and order | 10 |

<details>
<summary>Full M code (click to expand)</summary>

```m
let
    Source = Csv.Document(File.Contents("<path>\budget_targets_raw.csv"),[Delimiter=",", Columns=26, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Removed Top Rows" = Table.Skip(Source,3),
    #"Promoted Headers" = Table.PromoteHeaders(#"Removed Top Rows", [PromoteAllScalars=true]),
    #"Trimmed Text" = Table.TransformColumns(#"Promoted Headers",{{"Store", Text.Trim, type text}, {"Category", Text.Trim, type text}}),
    #"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text",{{"Store", Text.Proper, type text}, {"Category", Text.Proper, type text}}),
    #"Unpivoted Other Columns" = Table.UnpivotOtherColumns(#"Capitalized Each Word", {"Store", "Category"}, "Attribute", "Value"),
    #"Renamed Columns" = Table.RenameColumns(#"Unpivoted Other Columns",{{"Attribute", "Month Year Budget"}}),
    #"Replaced Value" = Table.ReplaceValue(#"Renamed Columns","TZS","",Replacer.ReplaceText,{"Value"}),
    #"Added Custom1" = Table.AddColumn(#"Replaced Value", "Budget Status", each if [Value] = null or Text.Trim([Value]) = "" then "Store not yet open (blank in source)" else if List.Contains({"-", "TBC", "N/A"}, Text.Trim([Value])) then "Placeholder in source" else "Budgeted"),
    #"Changed Type4" = Table.TransformColumnTypes(#"Added Custom1",{{"Budget Status", type text}}),
    #"Replaced Value1" = Table.ReplaceValue(#"Changed Type4","-","0",Replacer.ReplaceText,{"Value"}),
    #"Replaced Value2" = Table.ReplaceValue(#"Replaced Value1","","0",Replacer.ReplaceValue,{"Value"}),
    #"Replaced Value3" = Table.ReplaceValue(#"Replaced Value2","TBC","0",Replacer.ReplaceText,{"Value"}),
    #"Trimmed Text1" = Table.TransformColumns(#"Replaced Value3",{{"Value", Text.Trim, type text}}),
    #"Replaced Value4" = Table.ReplaceValue(#"Trimmed Text1","","0",Replacer.ReplaceValue,{"Value"}),
    #"Replaced Value5" = Table.ReplaceValue(#"Replaced Value4","N/A","0",Replacer.ReplaceText,{"Value"}),
    #"Changed Type" = Table.TransformColumnTypes(#"Replaced Value5",{{"Value", Currency.Type}}),
    #"Added Custom" = Table.AddColumn(#"Changed Type", "Month Start", each Date.FromText("01-" & [Month Year Budget], [Format="dd-MMM-yy", Culture="en-US"])),
    #"Changed Type1" = Table.TransformColumnTypes(#"Added Custom",{{"Month Start", type date}}),
    #"Reordered Columns" = Table.ReorderColumns(#"Changed Type1",{"Store", "Category", "Month Start", "Month Year Budget", "Value"}),
    #"Changed Type2" = Table.TransformColumnTypes(#"Reordered Columns",{{"Value", type number}}),
    #"Replaced Value6" = Table.ReplaceValue(#"Changed Type2","Mlimani City Mall","Mlimani City",Replacer.ReplaceValue,{"Store"}),
    #"Replaced Value7" = Table.ReplaceValue(#"Replaced Value6","Rock City","Rock City Mall",Replacer.ReplaceValue,{"Store"}),
    #"Removed Columns" = Table.RemoveColumns(#"Replaced Value7",{"Month Year Budget"}),
    #"Renamed Columns1" = Table.RenameColumns(#"Removed Columns",{{"Store", "Store Name"}}),
    #"Merged Queries" = Table.NestedJoin(#"Renamed Columns1", {"Store Name"}, Stores, {"Store Name"}, "Stores", JoinKind.LeftOuter),
    #"Expanded Stores" = Table.ExpandTableColumn(#"Merged Queries", "Stores", {"Store ID"}, {"Stores.Store ID"}),
    #"Renamed Columns2" = Table.RenameColumns(#"Expanded Stores",{{"Stores.Store ID", "Store ID"}}),
    #"Reordered Columns1" = Table.ReorderColumns(#"Renamed Columns2",{"Store ID", "Store Name", "Category", "Month Start", "Value"}),
    #"Removed Unmatched and Total Rows" = Table.SelectRows(#"Reordered Columns1", each [Store ID] <> null),
    #"Replaced Value8" = Table.ReplaceValue(#"Removed Unmatched and Total Rows","Electronic","Electronics",Replacer.ReplaceValue,{"Category"}),
    #"Replaced Value9" = Table.ReplaceValue(#"Replaced Value8","Home Appliance","Home Appliances",Replacer.ReplaceValue,{"Category"}),
    #"Replaced Value10" = Table.ReplaceValue(#"Replaced Value9","Appliances","Home Appliances",Replacer.ReplaceValue,{"Category"}),
    #"Changed Type3" = Table.TransformColumnTypes(#"Replaced Value10",{{"Value", Int64.Type}}),
    #"Added Custom2" = Table.AddColumn(#"Changed Type3", "Budget", each if [Budget Status] = "Budgeted" then [Value] else null),
    #"Changed Type5" = Table.TransformColumnTypes(#"Added Custom2",{{"Budget", Int64.Type}}),
    #"Removed Columns1" = Table.RemoveColumns(#"Changed Type5",{"Value"}),
    #"Reordered Columns2" = Table.ReorderColumns(#"Removed Columns1",{"Store ID", "Store Name", "Category", "Budget Status", "Month Start", "Budget"}),
    #"Renamed Columns3" = Table.RenameColumns(#"Reordered Columns2",{{"Budget", "Value"}})
in
    #"Renamed Columns3"
```

</details>

## Notes

- `Replacer.ReplaceValue` matches the **whole cell**; `Replacer.ReplaceText` replaces **part of the text**. Names and categories use the first, so `Electronics` is never turned into `Electronicss`.
- `Budget Status` is created before the placeholders are replaced with `0`. Afterwards the original information would be lost.
- The footer rows are removed by a rule (`Store ID <> null`), not by a fixed number of rows, so the query keeps working if the source changes length.
