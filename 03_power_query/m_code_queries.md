# Sales: Power Query M Code

All transformations applied to the `Sales` query (2024 and 2025 transaction files appended). Each section shows the purpose and the M code used.

> **Note:** Source file paths are replaced with placeholders.

---

## 1. Helper function: `fxNum`
**Purpose:** Strip currency text ("TZS", "Tsh", commas) and convert to a number. Returns `null` if the value cannot be parsed.

```m
(x as any) as nullable number =>
    try Number.From(Text.Select(Text.From(x), {"0".."9", ".", "-"})) otherwise null
```

---

## 2. Append 2024 + 2025
**Purpose:** The two yearly files use different column names. Columns were renamed to match, then appended into one table.

```m
## 2. Append 2024 + 2025
**Purpose:** The 2024 file has no `Channel` column, so one is added (value: "Unassigned"). Column names are then renamed to match the 2025 file, and the two tables are appended.

```m
// 2024: add Channel (not present in the 2024 file)
#"Added Custom" = Table.AddColumn(#"Changed Type", "Channel", each "Unassigned"),

// 2024: align column names with 2025
#"Renamed Columns" = Table.RenameColumns(#"Added Custom",{{"Sales Rep", "Sales Representative"}, {"Qty", "Quantity"}}),

// Append 2025 transactions (the "Sales 2025" query is cleaned separately)
#"Appended Query" = Table.Combine({#"Renamed Columns", #"Sales 2025"}),
```
---

## 3. Date parsing (mixed dd/mm/yyyy and mm/dd/yyyy)
**Purpose:** Convert text dates to a true Date. Tries UK format first, then US, then an Excel serial number. Unparseable values become `null`.

```m
try Date.FromText([Txn_Date],"en-GB")
otherwise try Date.FromText([Txn_Date],"en-US")
otherwise try Date.From(Number.From([Txn_Date]))
otherwise null
```

---

## 4. Discount normalisation
**Purpose:** Convert "10%", "10" and "0.1" to a decimal (0.10). A missing discount is treated as 0.

```m
let d = try Number.From(Text.Trim(Text.Replace(Text.From([Discount]), "%", ""))) otherwise null
in if d = null then 0 else if d > 1 then d / 100 else d
```

---

## 5. Unit Price Clean
**Purpose:** Remove currency text and convert to a number.

```m
## 5. Unit Price Clean
**Purpose:** Use the recorded unit price. If it is missing, back-calculate it from Total, Quantity and Discount: Total ÷ (Quantity × (1 − Discount)). If neither is possible, leave `null` (no fake default).

```m
let
    p = fxNum([Unit Price]),
    q = fxNum([Quantity]),
    t = fxNum([Total Amount]),
    d = [Discount Clean]
in
    if p <> null and p <> 0 then Number.Abs(p)
    else if q <> null and q <> 0 and t <> null and (1 - d) > 0
         then Number.Round(Number.Abs(t) / (Number.Abs(q) * (1 - d)), 0)
    else null
```

---

## 6. Quantity Clean
**Purpose:** Use the recorded quantity. If it is missing, back-calculate it from Total, Unit Price and Discount. If neither is possible, leave `null` (no fake default).

```m
let
    num = (x) => try Number.From(Text.Select(Text.From(x), {"0".."9", ".", "-"})) otherwise null,
    p = num([Unit Price Clean]),
    q = num([Quantity]),
    t = num([Total Amount]),
    d = if [Discount Clean] = null then 0 else [Discount Clean]
in
    if q <> null and q <> 0 then Number.Abs(q)
    else if p <> null and p <> 0 and t <> null and (1 - d) > 0
         then Number.Round(Number.Abs(t) / (Number.Abs(p) * (1 - d)), 0)
    else null
```

---

## 7. Total Clean
**Purpose:** Use the recorded total. If it is missing, calculate Quantity × Unit Price × (1 − Discount).

```m
let
    num = (x) => try Number.From(Text.Select(Text.From(x), {"0".."9", ".", "-"})) otherwise null,
    t = num([Total Amount]), q = [Quantity Clean], p = [Unit Price Clean],
    d = if [Discount Clean] = null then 0 else [Discount Clean]
in
    if t <> null and t <> 0 then Number.Abs(t)
    else if q <> null and p <> null then Number.Round(q * p * (1 - d), 0)
    else null
```

---

## 8. Is Return flag
**Purpose:** Returns are kept as positive values and flagged, so Net Sales can be calculated later in DAX.

```m
[Notes] <> null and Text.StartsWith(Text.Upper(Text.Trim(Text.From([Notes]))), "RETURN")
```

---

## 9. Payment Method standardisation
**Purpose:** Trim and capitalise, then map about 29 raw spellings to 6 methods. Card types are merged into "Card". Tigo Pesa is merged into "Mixx by Yas" (rebrand).

```m
## 9. Payment Method standardisation
**Purpose:** The raw data has about 29 spellings for 6 payment methods. Text is trimmed and capitalised first (steps `Trimmed Text2`, `Capitalized Each Word1`), then mapped with prefix rules. All card types are merged into "Card". Tigo Pesa is merged into "Mixx by Yas" (rebrand). Already-clean values (for example "M-Pesa") pass through the `else` branch unchanged.

```m
#"Added Conditional Column" = Table.AddColumn(#"Capitalized Each Word1", "Payment Method Clean",
    each if Text.StartsWith([Payment Method], "Cash") then "Cash"
    else if Text.StartsWith([Payment Method], "Visa") then "Card"
    else if Text.StartsWith([Payment Method], "Master") then "Card"
    else if Text.StartsWith([Payment Method], "Debit") then "Card"
    else if Text.StartsWith([Payment Method], "Credit") then "Card"
    else if Text.StartsWith([Payment Method], "Card") then "Card"
    else if Text.StartsWith([Payment Method], "Airtel") then "Airtel Money"
    else if Text.StartsWith([Payment Method], "Mixx") then "Mixx by Yas"
    else if Text.StartsWith([Payment Method], "Tigo") then "Mixx by Yas"
    else if Text.StartsWith([Payment Method], "Mpesa") then "M-Pesa"
    else if Text.StartsWith([Payment Method], "M Pesa") then "M-Pesa"
    else if Text.StartsWith([Payment Method], "Bank") then "Bank Transfer"
    else if Text.StartsWith([Payment Method], "Direct") then "Bank Transfer"
    else [Payment Method]),
```

| Raw spellings (examples) | Standardised to |
|---|---|
| M-Pesa, M PESA, mpesa, m-pesa | M-Pesa |
| Airtel Money, AirtelMoney, Airtel-Money | Airtel Money |
| Tigo Pesa, TigoPesa, Tigo-Pesa, Mixx by Yas | Mixx by Yas |
| Card, Visa, Master Card, Debit Card, Credit Card | Card |
| Cash, Cash Payment | Cash |
| Bank Transfer, Bank Trf, Direct Deposit | Bank Transfer |
```

## 10. Full query (Advanced Editor)

<details>
<summary>Click to expand</summary>

```m
let
    Source = Csv.Document(File.Contents("C:\Users\USER\Downloads\sales_2024_raw.csv"),[Delimiter=",", Columns=12, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",{{"TransactionID", type text}, {"Txn_Date", type text}, {"StoreID", type text}, {"ProductID", type text}, {"Qty", type text}, {"Unit Price", type text}, {"Discount", type number}, {"Payment Method", type text}, {"Customer Type", type text}, {"Sales Rep", type text}, {"Total Amount", type text}, {"Notes", type text}}),
    #"Added Custom" = Table.AddColumn(#"Changed Type", "Channel", each "Unassigned"),
    #"Renamed Columns" = Table.RenameColumns(#"Added Custom",{{"Sales Rep", "Sales Representative"}, {"Qty", "Quantity"}}),
    #"Appended Query" = Table.Combine({#"Renamed Columns", #"Sales 2025"}),
    #"Trimmed Text" = Table.TransformColumns(#"Appended Query",{{"ProductID", Text.Trim, type text}, {"StoreID", Text.Trim, type text}, {"TransactionID", Text.Trim, type text}}),
    #"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text",{{"ProductID", Text.Proper, type text}, {"StoreID", Text.Proper, type text}, {"TransactionID", Text.Proper, type text}}),
    #"Removed Duplicates" = Table.Distinct(#"Capitalized Each Word", {"TransactionID"}),
    #"Filtered Rows" = Table.SelectRows(#"Removed Duplicates", each [TransactionID] <> null and [TransactionID] <> ""),
    #"Trimmed Text1" = Table.TransformColumns(#"Filtered Rows",{{"Txn_Date", Text.Trim, type text}}),
    #"Added Custom1" = Table.AddColumn(#"Trimmed Text1", "Custom", each try Date.FromText([Txn_Date],"en-GB")
otherwise try Date.FromText([Txn_Date],"en-US") otherwise try Date.From(Number.From([Txn_Date])) otherwise null),
    #"Renamed Columns1" = Table.RenameColumns(#"Added Custom1",{{"Custom", "Transaction Date"}}),
    #"Reordered Columns" = Table.ReorderColumns(#"Renamed Columns1",{"TransactionID", "Transaction Date", "Txn_Date", "StoreID", "ProductID", "Quantity", "Unit Price", "Discount", "Payment Method", "Customer Type", "Sales Representative", "Total Amount", "Notes", "Channel"}),
    #"Removed Columns" = Table.RemoveColumns(#"Reordered Columns",{"Txn_Date"}),
    #"Added Custom2" = Table.AddColumn(#"Removed Columns", "Custom", each let d = try Number.From(Text.Trim(Text.Replace(Text.From([Discount]), "%", ""))) otherwise null
     in if d = null then 0 else if d > 1 then d / 100 else d),
    #"Reordered Columns1" = Table.ReorderColumns(#"Added Custom2",{"TransactionID", "Transaction Date", "StoreID", "ProductID", "Quantity", "Unit Price", "Custom", "Discount", "Payment Method", "Customer Type", "Sales Representative", "Total Amount", "Notes", "Channel"}),
    #"Renamed Columns2" = Table.RenameColumns(#"Reordered Columns1",{{"Custom", "Discount clean"}}),
    #"Removed Columns1" = Table.RemoveColumns(#"Renamed Columns2",{"Discount"}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Removed Columns1",{{"Discount clean", type number}}),
    #"Added Custom3" = Table.AddColumn(#"Changed Type1", "Custom", each [Notes] <> null and Text.StartsWith(Text.Upper(Text.Trim(Text.From([Notes]))), "RETURN")),
    #"Renamed Columns3" = Table.RenameColumns(#"Added Custom3",{{"Discount clean", "Discount Clean"}, {"Custom", "Return"}}),
    #"Added Custom4" = Table.AddColumn(#"Renamed Columns3", "Custom", each let p = fxNum([Unit Price]), q = fxNum([Quantity]), t = fxNum([Total Amount]), d = [Discount Clean]
in if p <> null and p <> 0 then Number.Abs(p)
   else if q <> null and q <> 0 and t <> null and (1 - d) > 0
        then Number.Round(Number.Abs(t) / (Number.Abs(q) * (1 - d)), 0)
   else null),
    #"Reordered Columns2" = Table.ReorderColumns(#"Added Custom4",{"TransactionID", "Transaction Date", "StoreID", "ProductID", "Quantity", "Custom", "Unit Price", "Discount Clean", "Payment Method", "Customer Type", "Sales Representative", "Total Amount", "Notes", "Channel", "Return"}),
    #"Renamed Columns4" = Table.RenameColumns(#"Reordered Columns2",{{"Custom", "Unit Price Clean"}}),
    #"Removed Columns2" = Table.RemoveColumns(#"Renamed Columns4",{"Unit Price"}),
    #"Changed Type2" = Table.TransformColumnTypes(#"Removed Columns2",{{"Unit Price Clean", Currency.Type}}),
    #"Added Custom5" = Table.AddColumn(#"Changed Type2", "Quantity clean", each let
    num = (x) => try Number.From(Text.Select(Text.From(x), {"0".."9", ".", "-"})) otherwise null,
    p = num([Unit Price Clean]),
    q = num([Quantity]),
    t = num([Total Amount]),
    d = if [Discount Clean] = null then 0 else [Discount Clean]
in
    if q <> null and q <> 0 then Number.Abs(q)
    else if p <> null and p <> 0 and t <> null and (1 - d) > 0
         then Number.Round(Number.Abs(t) / (Number.Abs(p) * (1 - d)), 0)
    else null),
    #"Renamed Columns5" = Table.RenameColumns(#"Added Custom5",{{"Quantity clean", "Quantity Clean"}}),
    #"Removed Columns3" = Table.RemoveColumns(#"Renamed Columns5",{"Quantity"}),
    #"Reordered Columns3" = Table.ReorderColumns(#"Removed Columns3",{"TransactionID", "Transaction Date", "StoreID", "Quantity Clean", "ProductID", "Unit Price Clean", "Discount Clean", "Payment Method", "Customer Type", "Sales Representative", "Total Amount", "Notes", "Channel", "Return"}),
    #"Added Custom6" = Table.AddColumn(#"Reordered Columns3", "Total Clean", each let
    num = (x) => try Number.From(Text.Select(Text.From(x), {"0".."9", ".", "-"})) otherwise null,
    t = num([Total Amount]),
    q = [Quantity Clean],
    p = [Unit Price Clean],
    d = if [Discount Clean] = null then 0 else [Discount Clean]
in
    if t <> null and t <> 0 then Number.Abs(t)
    else if q <> null and p <> null then Number.Round(q * p * (1 - d), 0)
    else null),
    #"Removed Columns4" = Table.RemoveColumns(#"Added Custom6",{"Total Amount"}),
    #"Reordered Columns4" = Table.ReorderColumns(#"Removed Columns4",{"TransactionID", "Transaction Date", "StoreID", "Quantity Clean", "ProductID", "Unit Price Clean", "Discount Clean", "Total Clean", "Payment Method", "Customer Type", "Sales Representative", "Notes", "Channel", "Return"}),
    #"Changed Type3" = Table.TransformColumnTypes(#"Reordered Columns4",{{"Total Clean", Currency.Type}, {"Quantity Clean", Int64.Type}}),
    #"Reordered Columns5" = Table.ReorderColumns(#"Changed Type3",{"TransactionID", "Transaction Date", "StoreID", "ProductID", "Quantity Clean", "Unit Price Clean", "Discount Clean", "Total Clean", "Payment Method", "Customer Type", "Sales Representative", "Notes", "Channel", "Return"}),
    #"Trimmed Text2" = Table.TransformColumns(#"Reordered Columns5",{{"Payment Method", Text.Trim, type text}, {"Customer Type", Text.Trim, type text}, {"Sales Representative", Text.Trim, type text}}),
    #"Capitalized Each Word1" = Table.TransformColumns(#"Trimmed Text2",{{"Payment Method", Text.Proper, type text}, {"Customer Type", Text.Proper, type text}, {"Sales Representative", Text.Proper, type text}}),
    #"Added Conditional Column" = Table.AddColumn(#"Capitalized Each Word1", "Payment Method Clean", each if Text.StartsWith([Payment Method], "Cash") then "Cash" else if Text.StartsWith([Payment Method], "Visa") then "Card" else if Text.StartsWith([Payment Method], "Master") then "Card" else if Text.StartsWith([Payment Method], "Debit") then "Card" else if Text.StartsWith([Payment Method], "Credit ") then "Card" else if Text.StartsWith([Payment Method], "Card") then "Card" else if Text.StartsWith([Payment Method], "Airtel") then "Airtel Money" else if Text.StartsWith([Payment Method], "Mixx") then "Mixx by Yas" else if Text.StartsWith([Payment Method], "Tigo") then "Mixx by Yas" else if Text.StartsWith([Payment Method], "Mpesa") then "M-Pesa" else if Text.StartsWith([Payment Method], "M Pesa") then "M-Pesa" else if Text.StartsWith([Payment Method], "Bank") then "Bank Transfer" else if Text.StartsWith([Payment Method], "Direct") then "Bank Transfer" else [Payment Method]),
    #"Reordered Columns6" = Table.ReorderColumns(#"Added Conditional Column",{"TransactionID", "Transaction Date", "StoreID", "ProductID", "Quantity Clean", "Unit Price Clean", "Discount Clean", "Total Clean", "Payment Method", "Payment Method Clean", "Customer Type", "Sales Representative", "Notes", "Channel", "Return"}),
    #"Removed Columns5" = Table.RemoveColumns(#"Reordered Columns6",{"Payment Method"}),
    #"Added Conditional Column1" = Table.AddColumn(#"Removed Columns5", "Customer Clean Type", each if Text.StartsWith([Customer Type], "Whole") then "Wholesale" else if Text.StartsWith([Customer Type], "Corp") then "Corpoate" else if Text.StartsWith([Customer Type], "Retail") then "Retail" else [Customer Type]),
    #"Reordered Columns7" = Table.ReorderColumns(#"Added Conditional Column1",{"TransactionID", "Transaction Date", "StoreID", "ProductID", "Quantity Clean", "Unit Price Clean", "Discount Clean", "Total Clean", "Payment Method Clean", "Customer Clean Type", "Customer Type", "Sales Representative", "Notes", "Channel", "Return"}),
    #"Removed Columns6" = Table.RemoveColumns(#"Reordered Columns7",{"Customer Type"}),
    #"Trimmed Text3" = Table.TransformColumns(#"Removed Columns6",{{"Notes", Text.Trim, type text}}),
    #"Capitalized Each Word2" = Table.TransformColumns(#"Trimmed Text3",{{"Notes", Text.Proper, type text}})
in
    #"Capitalized Each Word2"
    ```

</details>
