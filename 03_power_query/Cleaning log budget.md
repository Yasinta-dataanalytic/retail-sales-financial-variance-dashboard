# Cleaning Log: Budget Targets

**Source file:** `budget_targets_raw.csv` (Finance dept, FY2024/25, "draft v3, NOT FINAL")
**Query:** `Budget targets` (Power Query) | **Output:** [`02_data_clean/budget_clean.csv`](../02_data_clean/budget_clean.csv)
**M code:** [`m_code_budget.md`](m_code_budget.md)

| Metric | Raw | Clean |
|---|---|---|
| Layout | Wide report: 93 data lines x 24 month columns, 3 title lines on top | Long table: 1 row per store, category and month |
| Rows | 93 lines (2,232 store-category-month cells) | **2,160 rows** (15 stores x 6 categories x 24 months) |
| Store names | 18 distinct (incl. `Total`, `Note...`, spelling variants) | **15** stores, each with a `Store ID` |
| Categories | 9 distinct spellings | **6** categories |
| Value problems | `TZS` text, thousand separators, `-`, `TBC`, `N/A`, blanks | Whole numbers, **58 nulls** each with a reason in `Budget Status` |

**Row count check:** 93 lines x 24 months = 2,232 rows after unpivot. The merge with `Stores` returned the same 2,232 rows (many-to-one join on a unique key), so the merge did not duplicate anything. Removing 72 footer rows gave 2,160 = 15 x 6 x 24.

---

## 1. Report layout: title rows above the header

| Field | Details |
|---|---|
| **Issue** | The file starts with a report title, a source note (`Finance dept draft v3 \| NOT FINAL`) and a blank line before the real header (`Store, Category, Jan-24 ...`). |
| **Fix** | `Table.Skip(Source, 3)` then `Promote Headers`. |
| **Impact** | Without this the first row of data would be a title and every column would be named `Column1...Column26`. |
| **Insight** | The note says the budget is a **draft and not final**. Every variance figure built on it inherits that risk. |
| **Recommendation** | Request the approved version from Finance and re-run the query (the steps are reusable). Show "Budget: draft v3" on the dashboard. |

## 2. Wide to long: months in columns

| Field | Details |
|---|---|
| **Issue** | Each month is its own column (24 columns), which cannot be related to a date table or summed by month. |
| **Fix** | `Unpivot Other Columns` on `Store` and `Category`; columns renamed to `Month Year Budget` and `Value`. |
| **Impact** | 93 lines became 2,232 rows, one per store, category and month. This is the grain the data model needs. |
| **Insight** | Sales are recorded per transaction, budget per store, category and month. Both can now be compared at that level. |
| **Recommendation** | Ask Finance to deliver the budget in long format to avoid the unpivot step. |

## 3. Store and Category text formatting

| Field | Details |
|---|---|
| **Issue** | Mixed case (`DODOMA CENTRAL`, `posta`, `Dodoma Central `), trailing spaces (`STATIONERY `, `Stone Town `). |
| **Fix** | `Trim` then `Capitalize Each Word` on `Store` and `Category`. |
| **Impact** | Spelling variants of the same store collapse into one; this is required for the Store ID lookup in section 7. |
| **Insight** | Case and hidden spaces alone created many "different" stores. |
| **Recommendation** | Use a dropdown (data validation) for store and category when Finance enters the budget. |

## 4. Currency text and thousand separators in `Value`

| Field | Details |
|---|---|
| **Issue** | Amounts were stored as text: `TZS 1,150,000`, `1,290,000`, `1060000`. |
| **Fix** | Removed `TZS`, trimmed spaces, converted to a number (`Currency.Type`, later whole number `Int64.Type`). |
| **Impact** | Values can be summed. After conversion `Min` = 40,000, `Max` = 7,310,000, 0 errors. |
| **Insight** | Amounts are whole shillings, so whole number is the right type (no decimals needed). |
| **Recommendation** | Keep currency in the column header (`Value (TZS)`) rather than in each cell. |

## 5. Month labels to real dates

| Field | Details |
|---|---|
| **Issue** | Months were labels like `Jan-24`, which are text, not dates. |
| **Fix** | New column `Month Start` = `Date.FromText("01-" & [Month Year Budget], [Format="dd-MMM-yy", Culture="en-US"])`, type `date`. Original label column removed. |
| **Impact** | 24 distinct dates (1 Jan 2024 to 1 Dec 2025), 0 errors. Links directly to `Dim_Date`. |
| **Insight** | The budget covers two calendar years (2024 and 2025) although the title says FY2024/25. |
| **Recommendation** | Confirm with Finance whether the fiscal year is the calendar year, and document it. |

## 6. Blank cells and placeholders in `Value` (no fake zeros)

| Field | Details |
|---|---|
| **Issue** | 58 cells have no usable budget: **36 are blank** and **22 contain `-`, `TBC` or `N/A`**. A first cleaning pass turned all of them into `0`, which says "budget is zero" when the truth is "budget is unknown". |
| **Fix** | A new column `Budget Status` is created **before** any replacement: `Budgeted` (2,102), `Store not yet open (blank in source)` (36), `Placeholder in source` (22). `Value` is set to **null** when the status is not `Budgeted`. |
| **Impact** | Zero count fell from 58 (plus 72 footer zeros) to **0**. `Value` now has **58 nulls**, each explained. Total budgeted amount: **TZS 1,899,470,000** (2024: 845.7M, 2025: 1,053.7M). |
| **Insight** | All 36 blanks belong to **Morogoro Road (S14), January to June 2024**: the store opened on **1 July 2024**, so there was nothing to budget. The 22 placeholders are spread across 12 stores and are genuinely missing. |
| **Recommendation** | In DAX, show "No budget" instead of a variance when `Value` is null. Ask Finance to supply the 22 missing figures. Do not impute them. |

## 7. Store name variants and Store ID lookup

| Field | Details |
|---|---|
| **Issue** | Two names did not exist in the `Stores` master: `Mlimani City Mall` (master: `Mlimani City`) and `Rock City` (master: `Rock City Mall`). Before the fix, rows with these names got an empty `Store ID`. |
| **Fix** | Replace Values using **whole-cell matching** (`Replacer.ReplaceValue`), then `Merge` with `Stores` on `Store Name` (left outer, many-to-one) and expand `Store ID`. |
| **Impact** | `Store ID`: Empty 0, Distinct **15**, Min S01, Max S15. Merge returned 2,232 rows before and after, so no duplicates were created. |
| **Insight** | Replacing part of a name (instead of the whole cell) produced `Rock City Mall Mall` during development. Whole-cell replacement avoids this. |
| **Recommendation** | Use `Store ID` instead of store names in every Finance file; names change, IDs do not. |

## 8. Footer rows: blank line, TOTAL and a note

| Field | Details |
|---|---|
| **Issue** | The file ends with a blank line, a `TOTAL` line (no values) and `Note: Tanga budget under review`. After unpivot they became **72 rows** (3 lines x 24 months) with no store, no category and no value. |
| **Fix** | Rows with no `Store ID` after the merge are removed: `Table.SelectRows(..., each [Store ID] <> null)`. This is a rule based on the data, not a hard-coded row count. |
| **Impact** | 2,232 rows to **2,160**. `Store ID`, `Store Name` and `Category` now have 0 empty values. |
| **Insight** | The `TOTAL` line is empty, so it could not be used to reconcile the detail rows to a Finance total. |
| **Recommendation** | Keep totals out of data tables. Compute them with DAX so they always match the detail. |

## 9. Category spelling variants

| Field | Details |
|---|---|
| **Issue** | 9 spellings for 6 categories: `Electronic` (1 line), `Home Appliance` (1 line) and `Appliances` (2 lines). |
| **Fix** | Whole-cell replacements: `Electronic` to `Electronics`; `Home Appliance` to `Home Appliances`; `Appliances` to `Home Appliances`. |
| **Impact** | `Category` Distinct 9 to **6**: Beauty, Electronics, Fashion, Groceries, Home Appliances, Stationery. |
| **Insight** | **Assumption:** `Appliances` means `Home Appliances`, because the product master has only one appliance category. This is documented, not hidden. |
| **Recommendation** | Confirm the assumption with Finance. Maintain a category master list and validate entries against it. |

## 10. Final structure

| Column | Type | Notes |
|---|---|---|
| `Store ID` | text | S01 to S15 |
| `Store Name` | text | 15 values, matches `Stores` |
| `Category` | text | 6 values |
| `Budget Status` | text | `Budgeted`, `Store not yet open (blank in source)`, `Placeholder in source` |
| `Month Start` | date | 2024-01-01 to 2025-12-01 |
| `Value` | whole number | Budget in TZS; null when no budget |

---

## Validation (independent check)

The clean table was rebuilt from the raw CSV with a separate script and compared with the Power Query result:

| Check | Power Query | Independent rebuild |
|---|---|---|
| Rows | 2,160 | 2,160 |
| Stores / categories | 15 / 6 | 15 / 6 |
| Null values | 58 | 58 (36 + 22) |
| Zero values | 0 | 0 |
| Min / Max | 40,000 / 7,310,000 | 40,000 / 7,310,000 |
| Duplicate store-category-month rows | n/a | 0 |
| Distinct non-null values | 335 (incl. null) | 334 + null |

## Findings for the dashboard

- **Budget growth:** the 2025 budget (TZS 1,053.7M) is **24.6% higher** than 2024 (TZS 845.7M). Part of the gap is S14 having no budget for half of 2024.
- **Seasonality:** the median December budget is **1.46x** the average of January to November, so monthly variance must be read against a seasonal budget, not a flat one.
- **Largest single budget:** TZS 7,310,000, Kariakoo Electronics, December 2024.
- **Baseline shift in 2025:** 12 store-category pairs have a January 2025 budget more than 30% above December 2024 (for example Dodoma Central Electronics), which is not what seasonality alone would produce. Worth confirming with Finance.

## Known limitations

- The source is **draft v3, not final**.
- **Tanga Mjini (S15)** closed in December 2024 but has 72 budget rows for 2025 (TZS 19.4M, 1.8% of the 2025 budget). The source itself notes "Tanga budget under review". They are kept and flagged here rather than deleted.
- 22 budget cells are missing in the source and stay null.
- The `Appliances` to `Home Appliances` mapping is an assumption.
- The query depends on the `Stores` query (merge on `Store Name`).

---

## Screenshots

| Section | Before | After |
|---|---|---|<img width="1669" height="705" alt="Screenshot 2026-11-02 233824" src="https://github.com/user-attachments/assets/2ea21846-c9b1-414d-baf1-65aacce81e69" />

| 1 and 8 (footer rows) | ![footer rows before](../<img width="1647" height="723" alt="Screenshot 2026-11-02 233851" src="https://github.com/user-attachments/assets/dae334e6-503d-4e85-9d34-8cb15083d908" />
) | ![rows after](../
) |
| 7 (Store ID) | ![unmatched store names](../<img width="1643" height="705" alt="Screenshot 2026-11-02 235150" src="https://github.com/user-attachments/assets/256f4912-04cf-4634-8837-0cef2b72c3f3" />
) | ![store id after](../<img width="1677" height="724" alt="Screenshot 2026-11-02 235217" src="https://github.com/user-attachments/assets/2f7db84a-de4e-443d-bb92-77fb765a6689" />
) |
| 9 (Category) | ![category before](../<img width="1657" height="720" alt="Screenshot 2026-11-02 235552" src="https://github.com/user-attachments/assets/6bd4c842-803f-4764-a71c-6bbf7d61660b" />
) | ![category after](../<img width="1647" height="706" alt="Screenshot 2026-11-02 235508" src="https://github.com/user-attachments/assets/39e873f2-6105-4ec2-a776-540ec0b1becc" />
) |
| 6 (Value) | ![value zeros before](../<img width="1671" height="721" alt="Screenshot 2026-11-02 235856" src="https://github.com/user-attachments/assets/6a54a6ae-7a2c-4b7c-9d81-412dd851d898" />
) | ![value after](../<img width="1628" height="685" alt="Screenshot 2026-11-02 235925" src="https://github.com/user-attachments/assets/9fb82c59-9245-46d4-bdaa-9dae14b67fe0" />
) |
| 6 (Budget Status) | | ![budget status](../<img width="1678" height="727" alt="Screenshot 2026-11-03 000236" src="https://github.com/user-attachments/assets/02176175-9f31-42de-86be-0ce0cfa1753e" />
) |
