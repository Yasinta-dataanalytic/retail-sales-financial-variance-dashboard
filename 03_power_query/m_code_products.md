# Products

Query `Products` cleans `products_raw.csv` into 60 products. Steps are grouped by purpose; the full query is at the end.

## P1. Headers, blank rows and duplicate Product IDs
```m
#"Promoted Headers" = Table.PromoteHeaders(#"Changed Type", [PromoteAllScalars=true]),
#"Filtered Rows" = Table.SelectRows(#"Promoted Headers", each ([Product ID] <> "")),
#"Trimmed Text" = Table.TransformColumns(#"Filtered Rows",{{"Product ID", Text.Trim, type text}}),
#"Removed Duplicates" = Table.Distinct(#"Trimmed Text", {"Product ID"})
```
`Product ID` is trimmed **before** removing duplicates, so `P001 ` and `P001` count as the same product.

## P2. Product Name, Category and Supplier
```m
#"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text1",{{"Product Name", Text.Proper, type text}}),
#"Added Conditional Column" = Table.AddColumn(#"Capitalized Each Word1", "Custom", each if Text.StartsWith([Category], "Electronic") then "Electronics" else if Text.StartsWith([Category], "Appliances") then "Home Appliances" else if Text.StartsWith([Category], "Home Appliance") then "Home Appliances" else [Category]),
#"Replaced Value7" = Table.ReplaceValue(#"Changed Type3","Led Tv 43In","LED TV 43in",Replacer.ReplaceText,{"Product Name"})
```
Proper case turns acronyms into `Led Tv`, so `LED TV 43in` is corrected by an explicit replace.

## P3. Unit Cost and Unit Price (currency text to numbers)
```m
#"Replaced Value" = Table.ReplaceValue(#"Renamed Columns","TZS","",Replacer.ReplaceText,{"Unit Cost"}),
#"Replaced Value1" = Table.ReplaceValue(#"Replaced Value","Tsh","",Replacer.ReplaceText,{"Unit Cost"}),
#"Replaced Value2" = Table.ReplaceValue(#"Replaced Value1","",null,Replacer.ReplaceValue,{"Unit Cost"}),
#"Changed Type with Locale" = Table.TransformColumnTypes(#"Trimmed Text3", {{"Unit Cost", Currency.Type}}, "en-TZ")
```
Missing costs stay `null` (not `0`), so margin is never overstated. `Unit Price` is cleaned the same way.

## P4. Launch Date (text, mixed formats and Excel serial numbers)
```m
#"Added Custom" = Table.AddColumn(#"Capitalized Each Word2", "Custom", each try Date.FromText(Text.From([Launch Date]), "en-GB")
otherwise try Date.FromText(Text.From([Launch Date]), "en-US")
otherwise try Date.From(Number.From([Launch Date]))
otherwise null)
```
Tries day-first, then month-first, then treats a plain number (for example `44479`) as an Excel serial date.

## P5. Active? flag
```m
#"Added Conditional Column1" = Table.AddColumn(#"Renamed Columns1", "Custom", each if Text.StartsWith([#"Active?"], "Y") then "Yes" else if Text.StartsWith([#"Active?"], "1") then "Yes" else if Text.StartsWith([#"Active?"], "TRUE") then "Yes" else if Text.StartsWith([#"Active?"], "N") then "No" else if Text.StartsWith([#"Active?"], "FALSE") then "No" else if Text.StartsWith([#"Active?"], "0") then "No" else [#"Active?"])
```

## P6. Price Below Cost flag
```m
#"Added Custom1" = Table.AddColumn(#"Replaced Value7", "Price Below Cost", each if [Unit Price] <> null and [Unit Cost] <> null and [Unit Price] < [Unit Cost] then "Yes" else "No")
```
Values are **not changed**: products whose list price is below cost are only flagged.

<details>
<summary>Full Advanced Editor query (Products)</summary>

```m
let
    Source = Csv.Document(File.Contents("<path>\products_raw.csv"),[Delimiter=",", Columns=8, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Column1", type text}, {"Column2", type text}, {"Column3", type text}, {"Column4", type text}, {"Column5", type text}, {"Column6", type text}, {"Column7", type text}, {"Column8", type text}}),
    #"Promoted Headers" = Table.PromoteHeaders(#"Changed Type", [PromoteAllScalars=true]),
    #"Filtered Rows" = Table.SelectRows(#"Promoted Headers", each ([Product ID] <> "")),
    #"Trimmed Text" = Table.TransformColumns(#"Filtered Rows",{{"Product ID", Text.Trim, type text}}),
    #"Removed Duplicates" = Table.Distinct(#"Trimmed Text", {"Product ID"}),
    #"Trimmed Text1" = Table.TransformColumns(#"Removed Duplicates",{{"Product Name", Text.Trim, type text}}),
    #"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text1",{{"Product Name", Text.Proper, type text}}),
    #"Trimmed Text2" = Table.TransformColumns(#"Capitalized Each Word",{{"Category", Text.Trim, type text}}),
    #"Capitalized Each Word1" = Table.TransformColumns(#"Trimmed Text2",{{"Category", Text.Proper, type text}}),
    #"Added Conditional Column" = Table.AddColumn(#"Capitalized Each Word1", "Custom", each if Text.StartsWith([Category], "Electronic") then "Electronics" else if Text.StartsWith([Category], "Appliances") then "Home Appliances" else if Text.StartsWith([Category], "Home Appliance") then "Home Appliances" else [Category]),
    #"Removed Columns" = Table.RemoveColumns(#"Added Conditional Column",{"Category"}),
    #"Reordered Columns" = Table.ReorderColumns(#"Removed Columns",{"Product ID", "Product Name", "Custom", "Unit Cost", "Unit Price", "Supplier", "Launch Date", "Active?"}),
    #"Renamed Columns" = Table.RenameColumns(#"Reordered Columns",{{"Custom", "Catergory"}}),
    #"Replaced Value" = Table.ReplaceValue(#"Renamed Columns","TZS","",Replacer.ReplaceText,{"Unit Cost"}),
    #"Replaced Value1" = Table.ReplaceValue(#"Replaced Value","Tsh","",Replacer.ReplaceText,{"Unit Cost"}),
    #"Replaced Value2" = Table.ReplaceValue(#"Replaced Value1","",null,Replacer.ReplaceValue,{"Unit Cost"}),
    #"Trimmed Text3" = Table.TransformColumns(#"Replaced Value2",{{"Unit Cost", Text.Trim, type text}}),
    #"Changed Type with Locale" = Table.TransformColumnTypes(#"Trimmed Text3", {{"Unit Cost", Currency.Type}}, "en-TZ"),
    #"Replaced Value3" = Table.ReplaceValue(#"Changed Type with Locale","TZS","",Replacer.ReplaceText,{"Unit Price"}),
    #"Replaced Value4" = Table.ReplaceValue(#"Replaced Value3","Tsh","",Replacer.ReplaceText,{"Unit Price"}),
    #"Changed Type with Locale1" = Table.TransformColumnTypes(#"Replaced Value4", {{"Unit Price", Currency.Type}}, "en-TZ"),
    #"Trimmed Text4" = Table.TransformColumns(#"Changed Type with Locale1",{{"Supplier", Text.Trim, type text}}),
    #"Capitalized Each Word2" = Table.TransformColumns(#"Trimmed Text4",{{"Supplier", Text.Proper, type text}}),
    #"Added Custom" = Table.AddColumn(#"Capitalized Each Word2", "Custom", each try Date.FromText(Text.From([Launch Date]), "en-GB") 
otherwise try Date.FromText(Text.From([Launch Date]), "en-US") 
otherwise try Date.From(Number.From([Launch Date])) 
otherwise null),
    #"Changed Type1" = Table.TransformColumnTypes(#"Added Custom",{{"Custom", type date}}),
    #"Reordered Columns1" = Table.ReorderColumns(#"Changed Type1",{"Product ID", "Product Name", "Catergory", "Unit Cost", "Unit Price", "Supplier", "Custom", "Launch Date", "Active?"}),
    #"Removed Columns1" = Table.RemoveColumns(#"Reordered Columns1",{"Launch Date"}),
    #"Renamed Columns1" = Table.RenameColumns(#"Removed Columns1",{{"Custom", "Launch Date"}}),
    #"Added Conditional Column1" = Table.AddColumn(#"Renamed Columns1", "Custom", each if Text.StartsWith([#"Active?"], "Y") then "Yes" else if Text.StartsWith([#"Active?"], "1") then "Yes" else if Text.StartsWith([#"Active?"], "TRUE") then "Yes" else if Text.StartsWith([#"Active?"], "N") then "No" else if Text.StartsWith([#"Active?"], "FALSE") then "No" else if Text.StartsWith([#"Active?"], "0") then "No" else [#"Active?"]),
    #"Removed Columns2" = Table.RemoveColumns(#"Added Conditional Column1",{"Active?"}),
    #"Renamed Columns2" = Table.RenameColumns(#"Removed Columns2",{{"Custom", "Active?"}}),
    #"Replaced Value5" = Table.ReplaceValue(#"Renamed Columns2","  "," ",Replacer.ReplaceText,{"Supplier"}),
    #"Trimmed Text5" = Table.TransformColumns(#"Replaced Value5",{{"Product ID", Text.Trim, type text}}),
    #"Replaced Value6" = Table.ReplaceValue(#"Trimmed Text5","  "," ",Replacer.ReplaceText,{"Product Name"}),
    #"Changed Type2" = Table.TransformColumnTypes(#"Replaced Value6",{{"Active?", type text}, {"Unit Price", type number}}),
    #"Renamed Columns3" = Table.RenameColumns(#"Changed Type2",{{"Catergory", "Category"}}),
    #"Changed Type3" = Table.TransformColumnTypes(#"Renamed Columns3",{{"Unit Cost", type number}}),
    #"Replaced Value7" = Table.ReplaceValue(#"Changed Type3","Led Tv 43In","LED TV 43in",Replacer.ReplaceText,{"Product Name"}),
    #"Added Custom1" = Table.AddColumn(#"Replaced Value7", "Price Below Cost", each if [Unit Price] <> null and [Unit Cost] <> null and [Unit Price] < [Unit Cost] then "Yes" else "No"),
    #"Changed Type4" = Table.TransformColumnTypes(#"Added Custom1",{{"Price Below Cost", type text}})
in
    #"Changed Type4"
```

</details>
