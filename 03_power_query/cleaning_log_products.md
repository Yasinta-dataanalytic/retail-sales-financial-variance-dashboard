# Products Cleaning Log

Source: `products_raw.csv` → Power Query query `Products` → **60 products**.

| Check | Result |
|---|---|
| Raw rows → clean rows | 63 → 60 |
| Rows removed | 1 blank row and 2 duplicate Product IDs (P009, P032) |
| Category variants → categories | 19 → 6 |
| Missing Unit Cost | 2 products, kept as blank (not 0) |
| Products with list price below cost | 2 (Tablet, Standing Fan) |

**Principle:** nothing is invented. Missing values stay blank, suspicious values are flagged and not changed.

---

## 1. Blank row, duplicates and Product ID format

| Field | Details |
|---|---|
| **Issue** | The file had one blank row and two repeated Product IDs (P009, P032). Some IDs were in lowercase (`p011`, `p012`, `p047`). |
| **Fix** | Removed the blank row, trimmed and converted IDs to upper case, then removed duplicate IDs. |
| **Impact** | 60 unique products, all IDs in the format `P###`. |
| **Insight** | A duplicated product key multiplies sales rows in a data model, and a lowercase ID can fail to match the Sales table in a merge. |
| **Recommendation** | Make Product ID a unique, validated key in the source system. |

**Before (63 rows, 61 distinct values including the blank):**

<img width="1648" height="684" alt="Screenshot 2026-11-01 215706" src="https://github.com/user-attachments/assets/cb523729-0242-44da-8482-bd7f2b7242d8" />

**After (60 unique IDs):**

<img width="1630" height="711" alt="Screenshot 2026-11-01 215815" src="https://github.com/user-attachments/assets/2b31e497-2a39-4364-837b-8b8750eff56a" />

---


## 2. Product Name

| Field | Details |
|---|---|
| **Issue** | Names in inconsistent case (`HEADPHONES`, `ELECTRIC KETTLE`, `stapler`) and with repeated spaces. One name was blank because of the blank row. |
| **Fix** | Trimmed, collapsed double spaces and applied proper case. Proper case then damaged acronyms, apostrophes and units (`Led Tv 43In`, `Men'S Shirt`, `Rice 5Kg`, `Tea Leaves 500G`, `Milk 1L X6`), so these were corrected with explicit replaces (`LED TV 43in`, `Men's Shirt`, `5kg`, `500g`, `x6`). |
| **Impact** | 60 unique product names, all in a consistent format (distinct 60, unique 60). |
| **Insight** | Automatic text formatting damages acronyms, apostrophes and units. The distinct count looked correct (60) even while names were wrong, so the names had to be checked by eye. |
| **Recommendation** | Keep a product master with approved names. |

**Before (63 rows, mixed case):**

<img width="1631" height="696" alt="Screenshot 2026-11-01 221538" src="https://github.com/user-attachments/assets/1aa05e1c-3658-4ef7-9325-8da8bcd29916" />

**After (60 unique names):**

<img width="1650" height="716" alt="Screenshot 2026-11-01 221640" src="https://github.com/user-attachments/assets/f224ae3d-601d-4865-bc1b-7c10fe1740ed" />


---

## 3. Category

| Field | Details |
|---|---|
| **Issue** | 19 spellings for 6 real categories: `Electronics`, `electronics`, `Electronic`, `Home Appliances`, `home appliances`, `Appliances`, `Fashion`, `fashion`, `Groceries`, `groceries` and others. |
| **Fix** | Trimmed, proper case, then a conditional column mapped every `Electronic…` to `Electronics` and every `Appliance…` or `Home Appliance…` to `Home Appliances`. |
| **Impact** | 6 categories: Beauty, Electronics, Fashion, Groceries, Home Appliances, Stationery. |
| **Insight** | Category is the main slicer and the budget grain. Variants would split sales and budget across several labels. |
| **Recommendation** | Use a controlled category list shared by Sales, Products and Budget. |

**Before (19 variants):**

<img width="1664" height="720" alt="Screenshot 2026-11-01 221935" src="https://github.com/user-attachments/assets/041f1a26-d00a-4793-b778-43f11fb04cce" />

**After (6 categories):**

<img width="1681" height="704" alt="Screenshot 2026-11-01 222003" src="https://github.com/user-attachments/assets/554ef823-243c-45cf-90e8-b49f91b1a493" />

---

## 4. Unit Cost and Unit Price

| Field | Details |
|---|---|
| **Issue** | Prices stored as text with currency labels (`Tsh 999,500`, `TZS 1,762,000`) and thousands separators. Two products had no Unit Cost. |
| **Fix** | Removed `TZS` and `Tsh`, trimmed, and converted to numbers using the Tanzanian locale. Missing costs were kept as **blank**, not replaced with 0. |
| **Impact** | Numeric columns (Unit Cost from 2,000 to 1,762,000; no zeros). 2 products have no cost, so their margin cannot be calculated. |
| **Insight** | A cost of 0 would show a false 100% margin. Leaving it blank keeps the gap visible. |
| **Recommendation** | Make Unit Cost mandatory for every active product. |

**Unit Cost before (currency text):**
<img width="1684" height="720" alt="Screenshot 2026-11-01 222234" src="https://github.com/user-attachments/assets/8ca77806-ed57-415a-9743-650c41086e1f" />


**Unit Cost after (numbers, 2 blank):**
<img width="1657" height="675" alt="Screenshot 2026-11-01 222431" src="https://github.com/user-attachments/assets/65dfbf45-8b48-4ba9-950f-e9aef4528b67" />


---

## 5. Launch Date

| Field | Details |
|---|---|
| **Issue** | Dates in many formats (`2024-03-18`, `August 10, 2020`, `19/12/2020`, `10-16-2019`, `2020/07/27`) and some as Excel serial numbers (`44479`, `43393`). |
| **Fix** | Tried day-first, then month-first, then treated plain numbers as Excel serial dates. Result typed as Date. |
| **Impact** | All 60 dates parsed, none blank, ranging from 2018-02-02 to 2024-07-18. `44479` became 2021-10-10 and `43393` became 2018-10-20. |
| **Assumption** | Ambiguous slash dates such as `01/02/2023` are read as day/month (1 February 2023). |
| **Insight** | Serial numbers and ambiguous day/month order are silent risks: they load without any error but can give wrong dates. |
| **Recommendation** | Store dates in ISO format (`yyyy-mm-dd`). |

**Before (mixed formats and serial numbers):**
<img width="1654" height="706" alt="Screenshot 2026-11-01 222738" src="https://github.com/user-attachments/assets/78025022-b681-497b-89bc-d936720a0011" />


**After (typed dates):**

<img width="1671" height="730" alt="Screenshot 2026-11-01 222803" src="https://github.com/user-attachments/assets/a089652c-50f3-4972-994e-e7b3cd4beaa5" />

---

## 6. Active? flag

| Field | Details |
|---|---|
| **Issue** | The same status written as `Y`, `1`, `TRUE`, `N`, `0`, `FALSE`. |
| **Fix** | Mapped all values to `Yes` or `No`. |
| **Impact** | Two clean values. |
| **Recommendation** | Use a single Yes/No field. |

---

## 7. Supplier

| Field | Details |
|---|---|
| **Issue** | Supplier names with inconsistent case and repeated spaces. |
| **Fix** | Trimmed, collapsed double spaces and applied proper case. |
| **Impact** | 8 consistent supplier names. |
| **Recommendation** | Keep suppliers in a master list. |

---

## 8. Price Below Cost (flag)

| Field | Details |
|---|---|
| **Issue** | Two products have a list price **below** their unit cost: Tablet (Electronics) and Standing Fan (Home Appliances). |
| **Fix** | Values were **not changed**. A flag column `Price Below Cost` (Yes/No) was added. |
| **Impact** | 2 of 60 products are flagged. Their margin will be negative in profitability analysis. |
| **Insight** | Selling at list price loses money on each unit. This may be a data-entry error or a real pricing problem. Sales of these products will be checked in the margin analysis. |
| **Recommendation** | Confirm both prices with the pricing team and add a check that blocks a list price below cost. |

<img width="1648" height="689" alt="Screenshot 2026-11-01 223019" src="https://github.com/user-attachments/assets/2364bf0f-0c47-46d9-aec2-e85c017799fc" />

---

## Final validation

| Check | Result |
|---|---|
| Product ID is unique | Yes (60 distinct) |
| Blank Product IDs | 0 |
| Category values | 6 |
| Launch Date errors or blanks | 0 |
| Zero Unit Cost | 0 (2 blank) |
| Active? values | Yes / No |

## Known limitations
- 2 products have no Unit Cost, so margin is unavailable for them.
- Ambiguous slash dates are read as day/month.
- Product code `P999` appears in Sales but not in this table (see the Sales log, section 8b).
- The 2 duplicate Product IDs were removed by keeping the first row.
