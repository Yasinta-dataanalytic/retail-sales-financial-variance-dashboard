# Stores

Query `Stores` cleans `stores_raw.csv` into 15 stores. Steps are grouped by purpose; the full query is at the end.

## S1. Remove duplicate and blank rows
```m
#"Removed Duplicates" = Table.Distinct(#"Promoted Headers", {"Store ID"}),
#"Filtered Rows" = Table.SelectRows(#"Removed Duplicates1", each [Store ID] <> null and [Store ID] <> "")
```
Keeps the first row for each `Store ID`, then removes rows with a blank ID.

## S2. Region, Store Type and Manager text cleaning
```m
#"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text",{{"Store Name", Text.Proper, type text}, {"Region", Text.Proper, type text}}),
#"Replaced Value" = Table.ReplaceValue(#"Capitalized Each Word","Znz","Zanzibar",Replacer.ReplaceText,{"Region"}),
#"Replaced Value1" = Table.ReplaceValue(#"Replaced Value","Moro","Morogoro",Replacer.ReplaceText,{"Region"}),
#"Replaced Value2" = Table.ReplaceValue(#"Capitalized Each Word1","Store","",Replacer.ReplaceText,{"Store Type"}),
#"Trimmed Text3" = Table.TransformColumns(#"Changed Type2",{{"Store Type", Text.Trim, type text}, {"Region", Text.Trim, type text}}),
#"Replaced Value6" = Table.ReplaceValue(#"Trimmed Text3","Dar Es Salaam","Dar es Salaam",Replacer.ReplaceText,{"Region"})
```
> Caution: `Replacer.ReplaceText` replaces part of a value. If the data later contains `Morogoro`, the `Moro` rule would turn it into `Morogorogoro`. An exact-match replacer (`Replacer.ReplaceValue`) is safer for new data.

## S3. Open Date in mixed formats
```m
#"Added Custom" = Table.AddColumn(#"Trimmed Text2", "Custom", each try Date.FromText([Open Date],"en-GB") otherwise Date.FromText([Open Date],"en-US"))
```
Tries day-first, then month-first. Ambiguous `01/07/2024` is read as 1 July 2024.

## S4. Floor Area to a number
```m
#"Inserted Lowercased Text" = Table.AddColumn(#"Changed Type", "lowercase", each Text.Lower([Floor Area]), type text),
#"Replaced Value3" = Table.ReplaceValue(#"Inserted Lowercased Text","m2","",Replacer.ReplaceText,{"lowercase"}),
#"Replaced Value4" = Table.ReplaceValue(#"Replaced Value3","sqm","",Replacer.ReplaceText,{"lowercase"}),
#"Changed Type1" = Table.TransformColumnTypes(#"Renamed Columns1",{{"Floor Area", Int64.Type}})
```

## S5. Closed Month and Status
```m
#"Added Custom1" = Table.AddColumn(#"Changed Type1", "Closed Month", each if Text.Contains([Status], "Dec 2024") then #date(2024,12,1) else null),
#"Replaced Value5" = Table.ReplaceValue(#"Added Custom1","(Dec 2024)","",Replacer.ReplaceText,{"Status"}),
#"Capitalized Each Word3" = Table.TransformColumns(#"Replaced Value5",{{"Status", Text.Proper, type text}})
```
`Closed Month` is created **before** the text is removed, so the closure date is not lost.

## S6. Hidden spaces in Store ID, Status and Manager
```m
#"Trimmed Text4" = Table.TransformColumns(#"Replaced Value6",{{"Status", Text.Trim, type text}, {"Store ID", Text.Trim, type text}}),
#"Replaced Value7" = Table.ReplaceValue(#"Trimmed Text4","  "," ",Replacer.ReplaceText,{"Manager"})
```
7 of 15 Store IDs had leading or trailing spaces, which would silently break relationships with the Sales table. `Status` kept a space after `(Dec 2024)` was removed, and one manager name had a double space.

<details>
<summary>Full Advanced Editor query (Stores)</summary>

```m
let
    Source = Csv.Document(File.Contents("<path>\stores_raw.csv"),[Delimiter=",", Columns=8, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Removed Duplicates" = Table.Distinct(#"Promoted Headers", {"Store ID"}),
    #"Removed Duplicates1" = Table.Distinct(#"Removed Duplicates", {"Store ID"}),
    #"Filtered Rows" = Table.SelectRows(#"Removed Duplicates1", each [Store ID] <> null and [Store ID] <> ""),
    #"Trimmed Text" = Table.TransformColumns(#"Filtered Rows",{{"Store Name", Text.Trim, type text}}),
    #"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text",{{"Store Name", Text.Proper, type text}, {"Region", Text.Proper, type text}}),
    #"Replaced Value" = Table.ReplaceValue(#"Capitalized Each Word","Znz","Zanzibar",Replacer.ReplaceText,{"Region"}),
    #"Replaced Value1" = Table.ReplaceValue(#"Replaced Value","Moro","Morogoro",Replacer.ReplaceText,{"Region"}),
    #"Trimmed Text1" = Table.TransformColumns(#"Replaced Value1",{{"Store Type", Text.Trim, type text}}),
    #"Capitalized Each Word1" = Table.TransformColumns(#"Trimmed Text1",{{"Store Type", Text.Proper, type text}}),
    #"Replaced Value2" = Table.ReplaceValue(#"Capitalized Each Word1","Store","",Replacer.ReplaceText,{"Store Type"}),
    #"Capitalized Each Word2" = Table.TransformColumns(#"Replaced Value2",{{"Manager", Text.Proper, type text}}),
    #"Trimmed Text2" = Table.TransformColumns(#"Capitalized Each Word2",{{"Manager", Text.Trim, type text}}),
    #"Added Custom" = Table.AddColumn(#"Trimmed Text2", "Custom", each try Date.FromText([Open Date],"en-GB")
otherwise Date.FromText([Open Date],"en-US")),
    #"Reordered Columns" = Table.ReorderColumns(#"Added Custom",{"Store ID", "Store Name", "Region", "Store Type", "Manager", "Open Date", "Custom", "Floor Area", "Status"}),
    #"Removed Columns" = Table.RemoveColumns(#"Reordered Columns",{"Open Date"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns",{{"Custom", "Open Date"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{{"Open Date", type date}}),
    #"Inserted Lowercased Text" = Table.AddColumn(#"Changed Type", "lowercase", each Text.Lower([Floor Area]), type text),
    #"Replaced Value3" = Table.ReplaceValue(#"Inserted Lowercased Text","m2","",Replacer.ReplaceText,{"lowercase"}),
    #"Replaced Value4" = Table.ReplaceValue(#"Replaced Value3","sqm","",Replacer.ReplaceText,{"lowercase"}),
    #"Removed Columns1" = Table.RemoveColumns(#"Replaced Value4",{"Floor Area"}),
    #"Reordered Columns1" = Table.ReorderColumns(#"Removed Columns1",{"Store ID", "Store Name", "Region", "Store Type", "Manager", "Open Date", "lowercase", "Status"}),
    #"Renamed Columns1" = Table.RenameColumns(#"Reordered Columns1",{{"lowercase", "Floor Area"}}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Renamed Columns1",{{"Floor Area", Int64.Type}}),
    #"Added Custom1" = Table.AddColumn(#"Changed Type1", "Closed Month", each if Text.Contains([Status], "Dec 2024") then #date(2024,12,1) else null),
    #"Replaced Value5" = Table.ReplaceValue(#"Added Custom1","(Dec 2024)","",Replacer.ReplaceText,{"Status"}),
    #"Capitalized Each Word3" = Table.TransformColumns(#"Replaced Value5",{{"Status", Text.Proper, type text}}),
    #"Renamed Columns2" = Table.RenameColumns(#"Capitalized Each Word3",{{"Floor Area", "Floor Area (sqm)"}}),
    #"Changed Type2" = Table.TransformColumnTypes(#"Renamed Columns2",{{"Closed Month", type date}}),
    #"Trimmed Text3" = Table.TransformColumns(#"Changed Type2",{{"Store Type", Text.Trim, type text}, {"Region", Text.Trim, type text}}),
    #"Replaced Value6" = Table.ReplaceValue(#"Trimmed Text3","Dar Es Salaam","Dar es Salaam",Replacer.ReplaceText,{"Region"}),
    #"Trimmed Text4" = Table.TransformColumns(#"Replaced Value6",{{"Status", Text.Trim, type text}, {"Store ID", Text.Trim, type text}}),
    #"Replaced Value7" = Table.ReplaceValue(#"Trimmed Text4","  "," ",Replacer.ReplaceText,{"Manager"})
in
    #"Replaced Value7"
```

</details>
