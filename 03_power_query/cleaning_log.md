# Data Cleaning Log: Sales

This log documents every data-quality issue found in the raw sales files, how each was fixed in Power Query, and what it means for the business. Full M code is in [`m_code_queries.md`](./m_code_queries.md).

> **Note:** All data is simulated for portfolio purposes.

## Summary

| Metric | Value |
|---|---|
| Raw rows (2024 + 2025 appended) | 7,897 |
| Rows after removing duplicate transactions | 7,822 |
| Return transactions flagged (`Is Return`) | 246 |
| Future-dated transactions flagged | 3 |
| Payment spellings standardised | 29 → 6 methods |
| Blank payment methods remaining | 0 |

## Cleaning principles
1. **Report, don't hide.** Anomalies are flagged, not silently deleted.
2. **No fake defaults.** Missing values are derived from other fields where possible; otherwise left as `null`.
3. **Returns are kept** as positive values with an `Is Return` flag, so Net Sales can be calculated in DAX.
4. **Every imputation is documented** (see the transparency table at the end).

---
## 1. Two source files with different structures
| Field | Details |
|---|---|
| **Issue** | The 2024 and 2025 files use different column names (e.g. `Qty` vs `Quantity`, `Sales Rep` vs `Sales Representative`), and the 2024 file has no `Channel` column. |
| **Fix** | Renamed columns to a common schema, added `Channel = "Unassigned"` for 2024, then appended the two tables with `Table.Combine`. |
| **Impact** | One consistent table: 3,720 (2024) + 4,177 (2025) = 7,897 rows, with no misaligned columns. |
| **Insight** | Inconsistent column names across years would have produced duplicated or shifted columns on append and silently broken year-over-year comparison. Because the 2024 export has no channel field, channel analysis is only reliable for 2025. |
| **Recommendation** | Keep a single, fixed export template across reporting years, and capture sales channel on every transaction from now on so all periods can be compared on the same basis. |

**Append dialog:**
<img width="1682" height="727" alt="Screenshot 2026-10-30 182615" src="https://github.com/user-attachments/assets/641e2af1-64d7-4095-9c42-00410749f5e3" />


**Result after append (7,897 rows):**
<img width="1675" height="749" alt="Screenshot 2026-10-30 182823" src="https://github.com/user-attachments/assets/8c1f3166-964c-4d4d-aa29-5a811bc3c1a2" />




## 2. Duplicate transactions
| Field | Details |
|---|---|
| **Issue** | Duplicate `TransactionID` rows after appending (7,897 rows). |
| **Fix** | Trimmed and standardised IDs, then removed duplicates on `TransactionID`. |
| **Impact** | 7,822 unique transactions (about 75 duplicates removed). |
| **Insight** | Duplicates would double-count revenue and inflate transaction counts. |
| **Recommendation** | Enforce unique transaction IDs at source and reconcile exports before reporting. |

## 3. Dates stored as text in multiple formats
| Field | Details |
|---|---|
| **Issue** | `Txn_Date` was text in at least 5 formats (ISO, dd/mm/yyyy, long and short month names, Excel serial numbers). 1,602 distinct values for a 366-day year; 2 empty strings. |
| **Fix** | Parsed with `try … otherwise` (en-GB, then en-US, then Excel serial) into a true Date column, then set the data type to Date. |
| **Impact** | 100% valid dates, 0 errors; distinct values fell to 734. |
| **Insight** | The same calendar day appeared under many spellings. Left as text, monthly and Actual vs Budget reports would be unreliable. "Valid 99%" on a text column only means non-empty, not a usable date. |
| **Recommendation** | Store dates in ISO format (`yyyy-mm-dd`) at source; parse by a fixed priority order and list unparseable dates for review. |

**Before:**
<img width="1661" height="729" alt="Screenshot 2026-10-30 185454" src="https://github.com/user-attachments/assets/5506ee00-9271-4a3c-b641-17d7d822068f" />



**After:**
<img width="1664" height="695" alt="Screenshot 2026-10-30 190640" src="https://github.com/user-attachments/assets/683dc0a8-269c-46a4-ad3b-28b75ce2e55b" />


## 4. Future-dated transactions
| Field | Details |
|---|---|
| **Issue** | 734 distinct dates versus 731 days in 2024–2025; the maximum date was 16 Aug 2052. |
| **Fix** | Added `Date Status` (`Valid` / `Future date - review`). Rows were flagged, not deleted. |
| **Impact** | 3 rows flagged: Txn001079 (2052), Txn001143 (2042), Txn006076 (2052). Combined list value about TZS 374,000 (about 0.02% of net sales). |
| **Insight** | Dates fall outside the 2024–2025 date table, so these rows would vanish from monthly and yearly views while still counting in totals. They come from three different sales representatives, pointing to a missing validation control. The years look like transposed digits (2025 → 2052, 2024 → 2042). |
| **Recommendation** | Reject future dates at point of entry; ask stores to confirm correct dates before any correction; report these rows separately. |

<img width="1692" height="751" alt="Screenshot 2026-10-30 192927" src="https://github.com/user-attachments/assets/31734381-e8fb-44ba-a95f-abbcbd49d9f0" />


## 5. Inconsistent IDs and text
| Field | Details |
|---|---|
| **Issue** | Mixed case and stray spaces in IDs (e.g. `p025` vs `P025`, `s15` vs `S15`). |
| **Fix** | `Trim` and `Capitalize Each Word` on ID and text columns. |
| **Impact** | Products and stores match their dimension tables. |
| **Insight** | Case variants would split one product or store into several, breaking joins to Products and Stores. |
| **Recommendation** | Use system-generated, validated IDs instead of free text. |

## 6. Discount in mixed formats


| Field | Details |
|---|---|
| **Issue** | Discounts recorded as `10%`, `10`, `0.1`, or blank. |
| **Fix** | Normalised to a decimal (e.g. 0.10). Blank discount initially treated as 0, then refined in Section 7. |
| **Impact** | Consistent `Discount Clean` used in all derived calculations. |
| **Insight** | Mixed scales (10 vs 0.1) would overstate discounts by 100× in revenue calculations. |
| **Recommendation** | Capture discount as a validated percentage field. |

> Note: the blank = 0% assumption was later refined. See Section 7 (Discount Reconciliation).

## 7. Discount Reconciliation

| Field | Details |
|---|---|
| **Issue** | 17% of rows had a blank Discount. These were first treated as 0%. An Excel reconciliation check showed 666 of 7,780 complete rows (8.6%) where Quantity × Unit Price × (1 − Discount) did not equal Total Amount. |
| **Diagnosis** | 572 of the mismatches had Discount = 0. In Power Query I calculated the implied discount (1 − Total / (Quantity × Unit Price)). Values clustered at standard tiers (5%, 10%, 15%, 20%, 25%), showing that real discounts had been applied to the Total but were missing from the Discount field. |
| **Fix** | Discount is inferred from Total only when the implied discount is exactly 5%, 10%, 15%, 20% or 25%. Every row gets a `Discount Status`: **Recorded**, **Inferred from Total**, or **Blank or zero - assumed 0**. |
| **Impact** | Mismatches fell from 666 (8.6%) to 194 (2.5%). Mismatches with Discount = 0 fell from 572 to 100. |
| **Insight** | Missing discounts were not random blanks. They hid real price reductions, so treating every blank as 0% would have overstated list-price revenue and understated discounting. |
| **Recommendation** | Make Discount a mandatory field in the POS/order system, and require a reason code for any price override. |

**Known limitation:** 194 rows (2.5%) still do not reconcile: 94 with a recorded discount and 100 with an implied discount outside the standard tiers. There is no evidence to show whether Total or Discount is wrong, so these values were **not changed**. Total Amount is used as recorded.
## 8. Unit Price
| Field | Details |
|---|---|
| **Issue** | `Unit Price` was stored as text with inconsistent formatting (`TZS 8,000`, `Tsh 96,500`, `10000`, `8000`), so the same price appeared under several spellings (558 distinct text values). In addition, 620 rows (8%) were empty and some values were negative. |
| **Fix** | Helper function `fxNum` strips all non-numeric characters and converts the value to a number; negative values are converted to absolute values. If the price is missing, it is derived as Total ÷ (Quantity × (1 − Discount)). Values that cannot be derived are left as `null`. One derived price (60) came from a placeholder quantity of 999 and was set to `null`; the recorded total was kept (see section 8). |
| **Impact** | `Unit Price Clean` is now a numeric (currency) column: 99% valid, 0 errors, and only 29 rows (0.4%) still empty, down from 620 (7.9%). Distinct values fell from 558 text variants to 147 numeric prices. About 95% of the missing prices were recovered. |
| **Insight** | Currency text would have blocked every numeric calculation, and 8% missing prices would have understated revenue and margin. Most gaps were recoverable because Total, Quantity and Discount were usually still recorded. Prices below list that reconcile with quantity, discount and total (e.g. 1,750 × 6 × 0.85 = 8,925) were treated as valid promotional prices. The cleaned column also showed that a derived price is only as reliable as the fields it comes from: one bad quantity produced an impossible price of 60 for a product costing 6,500. |
| **Recommendation** | Store prices as numeric fields without currency text, make unit price a required field, and validate recorded prices against the product price list so promotional or out-of-range prices are flagged at entry. |

**Before cleaning** (raw `Unit Price`, 7,897 rows before duplicates were removed):

<img width="1609" height="669" alt="Screenshot 2026-10-30 203259" src="https://github.com/user-attachments/assets/6702b5a9-6990-4373-b1ad-715b6a27f44f" />


**After cleaning** (`Unit Price Clean`, 7,822 rows):

<img width="1688" height="729" alt="Screenshot 2026-10-30 203635" src="https://github.com/user-attachments/assets/62625d52-eb40-4044-8c84-5dd12d68f09a" />


---

## 9. Quantity
| Field | Details |
|---|---|
| **Issue** | `Qty` was stored as text with 65 distinct values; 131 rows (about 2%) were empty and some were negative (e.g. −1, −2, −5). The raw maximum was 999, a placeholder-like quantity on a single transaction with a total of 60,000. |
| **Fix** | Absolute value of the recorded quantity; if missing, derived as Total ÷ (Unit Price × (1 − Discount)); otherwise left as `null`. No default value (such as 1) is assumed. The single row with quantity 999 had its quantity and derived unit price set to `null`; its recorded total (60,000) was kept, so revenue is unaffected. |
| **Impact** | `Quantity Clean`: 99% valid, 0 errors, under 1% empty (down from 2%). The 999 outlier was neutralised so it cannot inflate total units sold. |
| **Insight** | Dropping blank-quantity rows would understate sales volume, while inventing a default would fabricate it. The 999 row is almost certainly an entry error: dividing its total by the product's list price (7,500) gives about 8 units, and 999 units would imply a price of 60 against a unit cost of 6,500. Negative quantities appear to be returns recorded as negative numbers; returns are now identified by the `Is Return` flag rather than by sign, so they are not mistaken for sales. |
| **Recommendation** | Make quantity a required field and reject placeholder values (such as 999) at entry; monitor the blank-quantity rate monthly by store so recurring gaps can be traced to specific outlets. |

**Before cleaning** (`Qty`, 7,897 rows before duplicates were removed: 131 empty, negatives, maximum 999):
<img width="1271" height="705" alt="Screenshot 2026-10-30 211401" src="https://github.com/user-attachments/assets/e578c906-931f-4f68-90d3-7e55392417b2" />



**After cleaning** (`Quantity Clean`, 7,822 rows):
<img width="1643" height="661" alt="Screenshot 2026-10-30 211838" src="https://github.com/user-attachments/assets/c0003049-3bb0-40e5-8b62-11cb7180beb3" />


---
## 9b. Unknown product code
| Field | Details |
|---|---|
| **Issue** | `ProductID` held 61 distinct codes against 60 products in the product master. The extra code, `P999`, does not exist in the Products table and appears on 6 transactions. |
| **Fix** | Rows were kept and flagged for review, not deleted or reassigned to another product. |
| **Impact** | 6 rows flagged (Txn000900, Txn002760, Txn003496, Txn003571, Txn006367, Txn006869) across 5 different stores, with a combined value of TZS XXX,XXX. |
| **Insight** | An unknown product code breaks the link to the Products and category tables, so these sales would drop out of category-level reporting while still counting in overall totals. The six rows span five different stores and widely different unit prices (from about 16,000 to 550,500), which suggests `P999` is used as a catch-all code when a product cannot be found, rather than a single mistyped item. |
| **Recommendation** | Restrict product codes in the sales system to the product master list so unknown items cannot be saved, and add a "new product request" step for items that are missing from the master. Review the flagged rows with the stores to assign the correct products. |

**Flagged transactions:**
<img width="1265" height="276" alt="Screenshot 2026-10-30 212815" src="https://github.com/user-attachments/assets/6e395bae-13ff-44dd-8447-5a52bd1e2828" />


## 10. Total Amount
| Field | Details |
|---|---|
| **Issue** | `Total Amount` was stored as text with inconsistent formatting (`TZS 80,000`, `80,000`, `50000`, `Tsh 991,…`), giving 3,834 distinct text values. In addition, 319 rows (4%) were empty and some values were negative. |
| **Fix** | Helper function `fxNum` strips all non-numeric characters and converts the value to a number; negative values are converted to absolute values. If the total is missing, it is calculated as Quantity × Unit Price × (1 − Discount). Where neither the total nor its components are available, the value is left as `null`. |
| **Impact** | `Total Clean` is now a numeric (currency) column: 99% valid, 0 errors, and only 28 rows (under 1%) still empty, down from 319 (4%). Distinct values fell from 3,834 text variants to 1,372 numeric amounts. About 91% of the missing totals were recovered. |
| **Insight** | Currency text would have blocked every sum and average, and 4% missing totals would have understated revenue. Most gaps were recoverable because Quantity, Unit Price and Discount were usually still recorded. The 28 rows that remain empty have no price and no total, so their revenue cannot be reconstructed without inventing values; they are reported as unknown instead. The largest total (TZS 8,812,000) equals 4 units at the highest unit price (2,203,000), so it is a valid bulk sale and not an error. |
| **Recommendation** | Store totals as numeric fields without currency text, and add a reconciliation check (Quantity × Unit Price × (1 − Discount) = Total) in the sales system so missing or inconsistent totals are caught at entry. |

**Before cleaning** (raw `Total Amount`, 7,897 rows before duplicates were removed):

<img width="1264" height="710" alt="Screenshot 2026-10-30 213511" src="https://github.com/user-attachments/assets/4bafccca-11a0-4c3a-856b-b6f5c86147a1" />


**After cleaning** (`Total Clean`, 7,822 rows):

<img width="1649" height="725" alt="Screenshot 2026-10-30 213555" src="https://github.com/user-attachments/assets/141d6f91-cde1-4582-8872-e973fd143659" />



## 11. Returns
| Field | Details |
|---|---|
| **Issue** | Returns appeared as negative quantities/amounts and as `RETURN - …` text in `Notes`. |
| **Fix** | Created `Is Return` from Notes; values kept positive. An early version took absolute values without a flag and inflated revenue by about 7.8%; this was corrected. |
| **Impact** | 246 returns flagged. Net Sales = Gross − Returns (DAX). |
| **Insight** | Summing all totals treats returns as sales, overstating revenue by about 7.8%. |
| **Recommendation** | Record returns as a distinct transaction type, not in free-text notes. |

## 12. Customer Type
| Field | Details |
|---|---|
| **Issue** | The customer segment was written in 13 different ways for only 3 real types (`Retail`, `RETAIL`, `retail`, `Retail Customer`, `Whole sale`, `wholesale`, `WHOLESALE`, `Corp`, `Corporate`, and others), plus 5 blank values. |
| **Fix** | Trimmed and capitalised the text, then mapped it with prefix rules to **Retail**, **Wholesale** and **Corporate**. |
| **Impact** | 3 clean customer types across 7,822 rows and no blank values remaining (down from 14 distinct values). |
| **Insight** | Retail is by far the largest segment, at roughly three quarters of transactions, with wholesale (about 15%) and corporate (about 10%) making up the rest. Split spellings would have scattered each segment's revenue across several labels and made the smaller segments look weaker and harder to track. |
| **Recommendation** | Use a fixed dropdown of customer types in the POS instead of free text. Review revenue per transaction by segment, since wholesale and corporate customers are fewer but typically buy larger baskets, so their share of revenue may be much higher than their share of transactions. |

**Before cleaning** (14 distinct values, 5 blanks, rows before duplicates were removed):

<img width="1687" height="737" alt="Screenshot 2026-10-30 215401" src="https://github.com/user-attachments/assets/6594822a-3e27-4529-9688-a41818ec557f" />


**After cleaning** (3 distinct values, 7,822 rows):
<img width="1664" height="743" alt="Screenshot 2026-10-30 215724" src="https://github.com/user-attachments/assets/f3b38834-47fb-43ba-98d6-7568ee7f4805" />


## 13. Payment Method
| Field | Details |
|---|---|
| **Issue** | The same payment method was written in many ways: 29 different spellings for 6 real methods (`M PESA`, `mpesa`, `m-pesa`, `Tigo-Pesa`, `Bank Trf`, `Direct Deposit`, `Cash Payment`, and Visa / Master Card / Debit / Credit), plus 5 blank values. |
| **Fix** | Trimmed and capitalised the text, then mapped it with prefix rules. All card types were merged into **Card**, and Tigo Pesa was merged into **Mixx by Yas** (the same network after its rebrand). |
| **Impact** | 6 clean payment methods across 7,822 rows, with no blank values remaining (the 5 blank values appear to belong to the blank-ID rows removed in section 2). |
| **Insight** | After standardisation, M-Pesa accounts for roughly one in three transactions and mobile money overall for about 57%, followed by cash (about 20%), card (about 15%) and bank transfer (about 7%). Split spellings would have fragmented each method's share and hidden this concentration. In particular, Tigo Pesa and Mixx by Yas are one network under old and new names, and only together does it show its true size, similar to Airtel Money (about 12% each). |
| **Recommendation** | Use a fixed dropdown of payment methods in the POS instead of free text, so spelling variants cannot be created. Because most revenue flows through mobile money, review merchant fees and have a fallback plan for network outages, and reconcile cash takings daily given that about one in five sales is paid in cash. |

**Cleaned payment mix (approximate share of transactions):**

| Payment method | Share |
|---|---|
| M-Pesa | about 33% |
| Cash | about 20% |
| Card | about 15% |
| Airtel Money | about 12% |
| Mixx by Yas | about 12% |
| Bank Transfer | about 7% |

**Before cleaning** (30 distinct values including 5 blanks, rows before duplicates were removed):

<img width="1679" height="726" alt="Screenshot 2026-10-30 214316" src="https://github.com/user-attachments/assets/80083448-5d49-4879-8970-9a531fd48d4b" />


**After cleaning** (6 distinct values, 7,822 rows):
<img width="1670" height="704" alt="Screenshot 2026-10-30 214801" src="https://github.com/user-attachments/assets/dff37399-3919-443a-a784-7499b8239605" />


## 14. Channel
| Field | Details |
|---|---|
| **Issue** | The 2024 file has no sales channel, so a placeholder ("Unassigned") was added; the 2025 file records channel as free text with 10 spellings for only 4 real channels (`In-store`, `In Store`, `instore`, `Whatsapp order`, `online`, `Phone Order`, and others). |
| **Fix** | Mapped every spelling to a standard list with a keyword rule (case-insensitive): **In-store**, **WhatsApp**, **Online**, **Phone**. The 2024 rows stay **Unassigned**. |
| **Impact** | 5 clean channel values (4 channels plus Unassigned) across 7,822 rows, down from 10 distinct spellings, with no unmapped values. |
| **Insight** | Roughly 47% of transactions (all of 2024) have no channel, so channel analysis is only reliable for 2025. Among 2025 transactions, in-store sales dominate at roughly 62%, with WhatsApp and online each around 14% and phone orders about 10%. Without standardising, the same channel would have been split across several labels and the digital channels would have looked smaller than they are. |
| **Recommendation** | Capture channel on every transaction from the start of each year using a fixed dropdown (In-store, WhatsApp, Online, Phone). Track WhatsApp and online sales as a share of revenue, since together they are over a quarter of known-channel sales. |

**Before cleaning** (10 distinct values):
<img width="1640" height="715" alt="Screenshot 2026-10-30 220637" src="https://github.com/user-attachments/assets/00481322-e01d-483c-8c06-ba5546e68c80" />



**After cleaning** (5 distinct values, 7,822 rows):
<img width="1670" height="747" alt="Screenshot 2026-10-30 221113" src="https://github.com/user-attachments/assets/247a8454-0586-449b-8795-643a1a90ad75" />

## 15.Sales Representative
| **Issue** | Sales representative names had inconsistent capitalisation, leading/trailing spaces, and repeated spaces inside names (e.g. "Ali  Mrema"). Standard Trim did not remove the inner spaces, so the same person was counted as several reps. |
| **Fix** | Trimmed, collapsed repeated inner spaces into one, and applied proper case. |
| **Impact** | Distinct representatives reduced from 61 to 30. |
| **Insight** | Without cleaning, rep-level sales rankings would split one person's sales across several names and understate top performers. |
| **Recommendation** | Use a validated dropdown of employee names instead of free-text entry. |

**Before cleaning** (61 distinct values, 5 blanks, rows before duplicates were removed):

<img width="1628" height="660" alt="Screenshot 2026-10-30 221619" src="https://github.com/user-attachments/assets/0eeae33b-fe78-4ed4-b418-a705c9e73a05" />


**After cleaning** (30 distinct values, 7,822 rows):
<img width="1673" height="706" alt="Screenshot 2026-10-31 172633" src="https://github.com/user-attachments/assets/e4f5c2b4-b63b-4022-86ed-0c59740d927f" />

## 16. Notes
| Field | Details |
|---|---|
| **Issue** | Notes is about 83% blank. |
| **Fix** | Trimmed; blanks kept as `null`. |
| **Impact** | No imputation; `Is Return` was already derived from it. |
| **Insight** | A blank note means no comment, not missing data. |
| **Recommendation** | Keep as optional free text; use structured fields for anything analysed. |

---

## Imputation transparency

| Field | Rule | Rows affected |
|---|---|---|
| Quantity | Derived from Total ÷ (Unit Price × (1 − Discount)) | `[ADD]` |
| Unit Price | Derived from Total ÷ (Quantity × (1 − Discount)) | `[ADD]` |
| Total Amount | Derived from Quantity × Unit Price × (1 − Discount) | `[ADD]` |
| Unresolvable values | Left `null` | under 1% per field |

## Final validation checks

| Check | Result |
|---|---|
| Row count after cleaning | 7,822 |
| Error cells in key columns | 0% |
| `Is Return = TRUE` | 246 |
| Blank payment methods | 0 |
| Net Sales (Gross − Returns) | approx. TZS 1.89 bn `[verify]` |

## Limitations
- 3 future-dated rows await confirmation from stores.
- Under 1% of quantities and prices could not be derived and remain `null`.
- 2024 transactions have no channel information.
