# Insights and Recommendations

Findings from the 2025 Retail Sales Performance & Budget Variance dashboard. All data is simulated for a Tanzanian multi-store retailer; amounts are in TZS. Every figure below comes from the dashboard measures (see `06_dax/dax_measures.md`).

## Executive summary

2025 was a strong growth year that narrowly missed its budget. Net Sales reached **TZS 1,035.3M**, **21.3% higher than 2024**, but finished **1.8% (TZS 18.5M) below the TZS 1,053.7M budget** (98.2% achievement; 2024 achieved 100.9%). Electronics carried the year, while Home Appliances drove most of the shortfall. Returns and discounts both rose, which trimmed margin quality. Data cleaning also mattered: uncorrected entry errors would have overstated revenue and split the sales rep rankings.

## Key insights

### 1. Growth is strong, but the budget was set higher still
- Net Sales grew 21.3% year on year.
- Budget grew faster than actual sales (TZS 845.7M in 2024 to TZS 1,053.7M in 2025), so achievement fell from 100.9% to 98.2%.
- The gap is small (1.8%), so the issue is a few weak categories and stores, not the whole business.

### 2. Electronics is the engine
- Electronics generated TZS 421.3M, **40.7% of Net Sales**.
- It beat its budget by **TZS 29.3M (+7.5%)** and offset much of the weakness elsewhere.
- The business depends heavily on one category: a slowdown in Electronics would have a large impact.

### 3. Home Appliances is the main budget gap
- Home Appliances finished **10.6% below budget (about TZS 32.5M)**, the largest shortfall of any category.
- Fashion (-6.2%), Groceries (-4.7%) and Stationery (-4.0%) also missed budget.
- Beauty was slightly ahead of budget (+1.3%).

### 4. Sales are concentrated in Dar es Salaam
- Dar es Salaam contributed TZS 369.8M, **35.7% of Net Sales**, more than double the next region (Arusha, TZS 165M).
- Mbeya (TZS 47M) and Kilimanjaro (TZS 44M) are the smallest regions.

### 5. Store performance is uneven
- **Best:** Morogoro Road, +10.3% versus budget. Stone Town (+8.7%), Nzuguni (+7.8%) and Njiro (+5.6%) also beat budget.
- **Weakest:** Mbeya Mjini, -22.5% versus budget. Kariakoo (-8.5%) is also well below budget.
- **Tanga Mjini recorded no sales** in the data. This needs checking: it is either a data issue or a store that did not trade.

### 6. Mobile money dominates payments
- Mobile money (M-Pesa, Airtel Money, Mixx by Yas) accounts for about **57% of Net Sales**.
- M-Pesa alone is the largest channel at TZS 349.4M, ahead of Cash (TZS 230M) and Card (TZS 134M).

### 7. Returns and discounts are creeping up
- Return rate rose from **3.3% to 4.2%**.
- Discount rate rose from **5.7% to 6.1%** of list price.
- Both reduce Net Sales and signal pressure on margin.

### 8. Data quality changed the answers
Power Query cleaning was not cosmetic. Without it, the headline numbers would have been wrong:
- **Blank discounts** hid real price reductions, so the true discount rate would have been understated.
- **Returns stored as positive amounts** would have been added to revenue instead of subtracted, overstating Net Sales.
- **61 different spellings of 15 sales reps** would have split the Top 10 Sales Reps ranking across duplicate names.

## Recommendations

| # | Recommendation | Why (evidence) | Suggested KPI to track |
|---|---|---|---|
| 1 | **Review the Home Appliances plan**: pricing, stock availability and promotions | Largest category gap (-10.6%, about TZS 32.5M) | Home Appliances budget achievement % |
| 2 | **Run a store recovery plan for Mbeya Mjini and Kariakoo** | Weakest stores versus budget (-22.5%, -8.5%) | Store variance % |
| 3 | **Confirm the Tanga Mjini status** (data error or non-trading store) | No sales recorded | Store sales by month |
| 4 | **Replicate what works at Morogoro Road** in comparable stores | Best store versus budget (+10.3%) | Store variance %, sales per rep |
| 5 | **Protect Electronics, and reduce dependence on it**: secure stock and grow the weaker categories | 40.7% of sales and +7.5% versus budget | Category share of Net Sales |
| 6 | **Investigate the rise in returns** by category and store | Return rate up from 3.3% to 4.2% | Return rate % |
| 7 | **Tighten discount approval** to stop the drift above 6% | Discount rate up from 5.7% to 6.1% | Discount rate % |
| 8 | **Prioritise mobile money partnerships** (promotions, fast checkout, reconciliation) | About 57% of Net Sales | Payment method mix |
| 9 | **Set budgets from trend and capacity**, not just growth targets | Budget outgrew actual sales, so achievement fell to 98.2% | Budget achievement %, by category and store |
| 10 | **Use validated dropdowns** for payment method, channel, customer type and sales rep at data entry | Free-text entry produced blank discounts and 61 spellings of 15 reps | Share of records failing validation checks |

## Limitations

- The data is **simulated**, so the findings show how the analysis works, not the performance of a real retailer.
- Variance is measured against a draft budget (`v3`).
- Time-intelligence measures such as year-on-year growth need a contiguous date selection.
- Margin and cost data were not available, so profitability is not assessed.
