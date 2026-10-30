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
| **Fix** | Normalised to a decimal; blank discount treated as 0. |
| **Impact** | Consistent `Discount Clean` used in all derived calculations. |
| **Insight** | Mixed scales (10 vs 0.1) would overstate discounts by 100× in revenue calculations. |
| **Recommendation** | Capture discount as a validated percentage field. |
## 7. Unit Price
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

## 8. Quantity
| Field | Details |
|---|---|
| **Issue** | `Qty` was 2% empty (98% valid) and included negative values (e.g. −2, −5). One transaction carried a placeholder-like quantity of 999 against a total of 60,000. |
| **Fix** | Absolute value of the recorded quantity; if missing, derived as Total ÷ (Unit Price × (1 − Discount)); otherwise left as `null`. No default value (such as 1) is assumed. The single row with quantity 999 had its quantity and derived unit price set to `null`; its recorded total (60,000) was kept, so revenue is unaffected. |
| **Impact** | `Quantity Clean`: 99% valid, 0 errors, under 1% empty (down from 2%). The 999 row was neutralised so it cannot inflate total units sold. |
| **Insight** | Dropping blank-quantity rows would understate sales volume, while inventing a default would fabricate it. The 999 row is almost certainly an entry error: dividing its total by the product's list price (7,500) gives about 8 units, and 999 units would imply a price of 60 against a unit cost of 6,500. Negative quantities appear to be returns recorded as negative numbers; returns are now identified by the `Is Return` flag rather than by sign, so they are not mistaken for sales. |
| **Recommendation** | Make quantity a required field and reject placeholder values (such as 999) at entry; monitor the blank-quantity rate monthly by store so recurring gaps can be traced to specific outlets. |

**Before cleaning** (`Quantity`: 98% valid, 2% empty):



**After cleaning** (`Quantity Clean`: 99% valid, under 1% empty):

📷 *(drag 04d_quantity_after.png here)*

---

## 8b. Unknown product code
| Field | Details |
|---|---|
| **Issue** | `ProductID` held 61 distinct codes against 60 products in the product master. The extra code, `P999`, does not exist in the Products table. |
| **Fix** | Rows were kept and flagged for review, not deleted or reassigned. |
| **Impact** | `[ADD: number of P999 rows and their total value]` |
| **Insight** | An unknown product code breaks the link to the Products and category tables, so these sales would drop out of category-level reporting while still counting in overall totals. |
| **Recommendation** | Restrict product codes in the sales system to the product master list, and review the flagged rows with the stores. |


## 9. Total Amount
| Field | Details |
|---|---|
| **Issue** | Totals with text, negative values or blanks. |
| **Fix** | Used the recorded total; if missing, calculated Quantity × Unit Price × (1 − Discount); otherwise `null`. |
| **Impact** | `[ADD: Total Clean valid / empty % from your screenshot]` |
| **Insight** | Recovering totals from their components keeps revenue complete without inventing values. |
| **Recommendation** | Add a reconciliation check (Quantity × Price × (1 − Discount) = Total) in the source system. |

📷 *(drag the Total screenshot here)*

## 10. Returns
| Field | Details |
|---|---|
| **Issue** | Returns appeared as negative quantities/amounts and as `RETURN - …` text in `Notes`. |
| **Fix** | Created `Is Return` from Notes; values kept positive. An early version took absolute values without a flag and inflated revenue by about 7.8%; this was corrected. |
| **Impact** | 246 returns flagged. Net Sales = Gross − Returns (DAX). |
| **Insight** | Summing all totals treats returns as sales, overstating revenue by about 7.8%. |
| **Recommendation** | Record returns as a distinct transaction type, not in free-text notes. |

## 11. Payment Method
| Field | Details |
|---|---|
| **Issue** | 29 spellings for 6 methods (`M PESA`, `mpesa`, `Tigo-Pesa`, `Bank Trf`, `Direct Deposit`, `Cash Payment`, Visa/Master/Debit/Credit). |
| **Fix** | Trim, capitalise, then prefix-rule mapping. All card types merged into **Card**; Tigo Pesa merged into **Mixx by Yas** (rebrand). |
| **Impact** | 6 clean methods, 0 blank values remaining. |
| **Insight** | Split labels would fragment each method's share in payment-mix analysis; Tigo Pesa and Mixx by Yas are the same network under old and new names. |
| **Recommendation** | Use a fixed dropdown of payment methods in the POS rather than free text. |

📷 *(drag the Payment distribution screenshot here)*

## 12. Customer Type
| Field | Details |
|---|---|
| **Issue** | `[ADD: variants found, e.g. Whole / Wholesale, Corp / Corporate]` |
| **Fix** | Conditional column mapping variants to Retail, Wholesale, Corporate. |
| **Impact** | `[ADD: distinct values after cleaning]` |
| **Insight** | Inconsistent labels would split segment revenue across several categories. |
| **Recommendation** | Fix the customer-type list at source. |

## 13. Channel
| Field | Details |
|---|---|
| **Issue** | 2024 file has no channel, and the placeholder was first entered as "Unassgned" (typo). `[ADD: any 2025 variants found]` |
| **Fix** | Corrected to "Unassigned" at the source step; standardised labels. |
| **Impact** | Clean channel list; 2024 channel is explicitly "Unassigned". |
| **Insight** | Channel analysis is only reliable for 2025. |
| **Recommendation** | Capture channel on every transaction going forward. |

## 14. Notes
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
