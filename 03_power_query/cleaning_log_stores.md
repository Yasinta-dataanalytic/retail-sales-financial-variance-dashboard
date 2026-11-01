# Stores Cleaning Log

Source: `stores_raw.csv` → Power Query query `Stores`. Result: **15 stores (S01–S15)**.

| Check | Result |
|---|---|
| Unique stores after cleaning | 15 |
| Active / Closed | 14 / 1 |
| Rows removed | 1 duplicate (S03) and 1 blank row |
| Regions | 9 |
| Store types | 3 (Mall, Flagship, Outlet) |

**Principle:** every change is documented, nothing is deleted without a reason, and information that matters later (the closure date) is kept as a new column.

---

## 1. Headers not recognised

| Field | Details |
|---|---|
| **Issue** | The CSV loaded with generic headers (`Column1` … `Column8`). The real headers were in the first row. |
| **Fix** | Promoted the first row to headers. |
| **Impact** | Every later step can reference columns by name. |
| **Insight** | Raw exports lose header information when loaded as plain text. |
| **Recommendation** | Export master data with a fixed header row and a documented schema. |

**Before:**

<img width="1875" height="720" alt="Screenshot 2026-11-01 120504" src="https://github.com/user-attachments/assets/45af626c-b70f-4b16-85cd-2f23e5746558" />


---

## 2. Duplicate and blank rows

| Field | Details |
|---|---|
| **Issue** | Store S03 (Mikocheni) appeared twice with identical values, and one row was completely blank. |
| **Fix** | Removed duplicates on `Store ID`, then filtered out blank IDs. |
| **Impact** | One row per store (15 stores), so joins from Sales and Budget cannot duplicate transactions. |
| **Insight** | A duplicate key in a dimension table multiplies sales rows in a data model and overstates revenue. |
| **Recommendation** | Make `Store ID` a unique, mandatory key in the source system. |

**After (15 unique stores):**
<img width="1603" height="652" alt="Screenshot 2026-11-01 123636" src="https://github.com/user-attachments/assets/0794c141-3aae-44d0-8c50-11742bce8a41" />

---


## 3. Region spelling, abbreviations and hidden spaces
## 3. Region spelling, abbreviations and hidden spaces

| Field | Details |
|---|---|
| **Issue** | Region had 13 distinct values for 9 real regions: mixed case (`ARUSHA`, `mwanza`, `DODOMA`, `kilimanjaro`), abbreviations (`Znz`, `MORO`) and spaces that were invisible in the table. |
| **Fix** | Applied proper case, replaced `Znz` → Zanzibar and `Moro` → Morogoro, trimmed leading and trailing spaces, and standardised the spelling to `Dar es Salaam`. Proper case alone does not remove spaces, so duplicate-looking values stayed separate until the extra trim was added. |
| **Impact** | Distinct regions reduced from 13 to **9**: Dar es Salaam, Arusha, Mwanza, Dodoma, Mbeya, Zanzibar, Kilimanjaro, Morogoro and Tanga. |
| **Insight** | Hidden spaces split totals silently: a regional chart would show the same region twice with different values. Checking the distinct count after cleaning caught it. |
| **Recommendation** | Use a dropdown of official regions in the source system, and validate the distinct count after every text-cleaning step. |

**Before (13 distinct values):**

<img width="1692" height="705" alt="Screenshot 2026-11-01 125244" src="https://github.com/user-attachments/assets/38a4274e-12d8-42fb-bbd3-21316577f031" />

**After (9 distinct values):**

<img width="1658" height="683" alt="Screenshot 2026-11-01 125423" src="https://github.com/user-attachments/assets/2780bb9c-86fb-40b0-82c6-13995dd88c4e" />

---

## 4. Store Type

## 4. Store Type

| Field | Details |
|---|---|
| **Issue** | Store Type had 6 spelling variants for 3 real types: `OUTLET`, `Outlet`, `Outlet Store`, `FLAGSHIP`, `flagship` and `Mall`. A blank row added an empty value. |
| **Fix** | Trimmed, applied proper case, merged `Outlet Store` into `Outlet`, then trimmed again to remove the space left by the replacement. |
| **Impact** | 3 store types remain: **Outlet, Flagship and Mall** (15 stores, no blanks). |
| **Insight** | Store type is a core slicer. Variants would show one format as several categories and split any comparison between flagship, outlet and mall stores. |
| **Recommendation** | Use a controlled list of store types in the source system. |

**Before (6 variants plus a blank):**
<img width="1636" height="689" alt="Screenshot 2026-11-01 125551" src="https://github.com/user-attachments/assets/f3143610-73a8-4458-86e7-833b3224206a" />


**After (3 types):**

<img width="1652" height="707" alt="Screenshot 2026-11-01 125612" src="https://github.com/user-attachments/assets/ccb614d5-8044-4b69-98d9-1b45b549f667" />


---


## 5. Manager names

| Field | Details |
|---|---|
| **Issue** | Manager names were typed in inconsistent case: `PETER LYIMO`, `SALMA SANGA`, `JOSEPH KIMARO`, `MICHAEL KESSY` in capitals and `david nnko` in lowercase. The raw data also contained a blank row and a repeated manager (Grace Mushi, from the duplicate S03 row). |
| **Fix** | Trimmed spaces and applied proper case; the duplicate and blank rows were removed in the earlier steps. |
| **Impact** | 15 managers for 15 stores, all in one consistent format, no blanks (distinct count 15, unique 15). |
| **Insight** | The count of distinct names did not change, so this was a formatting issue, not a duplication issue. It still matters: `PETER LYIMO` and `Peter Lyimo` would be treated as different people when matching with HR or sales records. Each manager runs exactly one store. |
| **Recommendation** | Take manager names from the HR master list instead of typing them by hand. |

**Before (17 rows, mixed case, blank row):**

<img width="1674" height="710" alt="Screenshot 2026-11-01 130307" src="https://github.com/user-attachments/assets/df049de3-fd94-43db-8e34-9201b934b5bf" />

**After (15 unique managers, proper case):**

<img width="1651" height="707" alt="Screenshot 2026-11-01 130405" src="https://github.com/user-attachments/assets/41f57baf-eaec-4fdf-bb47-4e1099bdca77" />

---

## 6. Open Date in mixed formats

| Field | Details |
|---|---|
| **Issue** | Dates in several formats: `2019/03/12`, `01-Jun-2017`, `02-20-2021` (month first), `August 10, 2020`, `01/07/2024`. |
| **Fix** | Parsed with `try Date.FromText(..., "en-GB") otherwise Date.FromText(..., "en-US")`, then set the type to Date. |
| **Impact** | All 15 dates parsed with no errors. |
| **Assumption** | `01/07/2024` (S14) is ambiguous. It is read as **1 July 2024** (day first). |
| **Insight** | Ambiguous slash dates are a silent risk because day and month can be swapped without any error. |
| **Recommendation** | Store dates in ISO format (`yyyy-mm-dd`). |

---

## 7. Floor Area with units

| Field | Details |
|---|---|
| **Issue** | Area stored as text with different units or none: `320 sqm`, `450m2`, `280 SQM`, `160`. |
| **Fix** | Lowercased, removed `sqm` and `m2`, converted to a whole number, and renamed the column `Floor Area (sqm)`. |
| **Impact** | A numeric column that can be used to calculate sales per square metre. |
| **Insight** | Sales density can now be compared between stores of different sizes. |
| **Recommendation** | Store area as a number and put the unit in the column name. |

---

## 8. Status and the closed store

| Field | Details |
|---|---|
| **Issue** | Status had 3 spellings of the same value (`Active`, `active`, `ACTIVE`), and one store was recorded as `Closed (Dec 2024)`, with the closure date written inside the text. |
| **Fix** | Before removing the text, created a new column `Closed Month` (1 Dec 2024, type Date). Then removed `(Dec 2024)` and applied proper case, leaving two values: `Active` and `Closed`. |
| **Impact** | 14 active stores and 1 closed store (S15, Tanga Mjini). The closure date is kept as a real date, so no information is lost. |
| **Insight** | S15 closed in December 2024, yet a 2025 budget exists for it. This must be handled in the data model, so budget variance for 2025 is not distorted. |
| **Recommendation** | Keep a closure date in the master data and stop budgeting for closed stores. |

**Before (3 spellings of Active, plus `Closed (Dec 2024)`):**

<img width="1650" height="681" alt="Screenshot 2026-11-01 131302" src="https://github.com/user-attachments/assets/91f3f125-c8c4-4c76-9376-4d007340a7f4" />

**After (2 values):**

<img width="1648" height="700" alt="Screenshot 2026-11-01 131357" src="https://github.com/user-attachments/assets/ab641afc-201a-4257-b695-c28302920e84" />


**Closed Month created for the closed store (S15, Tanga Mjini):**

<img width="1661" height="719" alt="Screenshot 2026-11-01 131837" src="https://github.com/user-attachments/assets/c157d349-5f97-47ee-a3c1-fe4bfd0207db" />

---

## Final validation

| Check | Result |
|---|---|
| Store ID is unique | Yes (15 distinct IDs) |
| Blank Store IDs | 0 |
| Date parsing errors | 0 |
| Floor Area is numeric | Yes |
| Closed stores | 1 (S15) |

## Known limitations and assumptions
- `01/07/2024` is read as day/month (1 July 2024).
- `Outlet Store` is treated as the same type as `Outlet`.
- S15 is closed, but a 2025 budget still exists for it (handled in the data model).
