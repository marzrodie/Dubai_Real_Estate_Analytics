# Data profile: Dubai real estate transactions

Profiled on 9 November 2026 against the full extract (both CSV files, loaded with DuckDB, all columns read as text). These figures are the baseline. Every notebook's check cells compare against them.

## 1. Shape

| Item | Value |
|------|-------|
| Files | `transactions_2026-11-06_21-44-39_0001.csv` (907,423 rows, 582 MB) and `..._0002.csv` (907,421 rows, 582 MB) |
| Total rows | **1,814,844** |
| Columns | **47**: 46 documented plus an undocumented `load_timestamp` (one value: `2026-11-06T21:40:53Z`) |
| English / Arabic | 14 columns are Arabic copies of English columns |
| Unique key | `transaction_id` is unique across both files. There are no duplicate rows and no overlap between the files |
| CSV format | Comma-separated, every value in double quotes, UTF-8 with no BOM. 13,319 rows contain commas inside quoted values; no values contain embedded newlines or quote characters |

## 2. Transaction groups

| Group | Rows | Main procedures |
|-------|------|-----------------|
| Sales | 1,387,509 | Sell - Pre registration (off-plan) 639,812 · Sell (existing) 518,433 · Delayed Sell 156,441 · Lease to Own Registration 29,890 |
| Mortgages | 359,126 | Mortgage Registration 248,095 · Lease to Own Registration 29,887 · Modify Mortgage 23,000 · Delayed Mortgage 21,069 |
| Gifts | 68,209 | Grant 60,020 |

**Core sales definition used in this project:** `procedure_name_en` in (`Sell`, `Sell - Pre registration`, `Delayed Sell`) = **1,314,686 rows**. These three procedures line up exactly with registration type: every `Sell - Pre registration` is off-plan, and every `Sell` and `Delayed Sell` is an existing property.

## 3. Time coverage

| Year | All rows | Sales rows |
|------|---------:|-----------:|
| Before 1998 | 1,058 | 53 |
| 1998–2006 | 31,340 | 16,581 |
| 2007 | 14,845 | 9,082 |
| 2008 | 32,155 | 23,633 |
| 2009 | 79,189 | 64,497 |
| 2013 | 77,972 | 64,556 |
| 2020 | 49,286 | 33,427 |
| 2021 | 82,822 | 59,728 |
| 2022 | 120,014 | 95,781 |
| 2023 | 165,260 | 132,021 |
| 2024 | 224,395 | 179,142 |
| 2025 | 266,991 | 214,553 |
| 2026 (to 5 Nov) | 180,708 | 137,556 |

Plan: analysis windows start in **2015** (enough history for seasonality while staying relevant). **November 2026 is a partial month** (data ends 5 Nov) and is excluded from monthly trends and the forecast.

## 4. Residential flats: off-plan share and price (core sales)

| Year | Flat sales | Off-plan share | Median AED per sq ft |
|------|-----------:|---------------:|---------------------:|
| 2015 | 23,207 | 43.5% | 1,104 |
| 2017 | 26,595 | 61.2% | 1,155 |
| 2019 | 23,177 | 62.9% | 1,196 |
| 2020 | 19,726 | 53.2% | 1,036 |
| 2021 | 35,152 | 47.7% | 1,148 |
| 2022 | 61,433 | 56.3% | 1,501 |
| 2023 | 91,506 | 62.0% | 1,528 |
| 2024 | 133,085 | 69.8% | 1,638 |
| 2025 | 163,045 | 74.4% | 1,737 |
| 2026 YTD | 104,175 | 75.1% | 1,722 |

⚠️ The median mixes off-plan and existing flats, so it moves when the mix changes, not only when prices change. Q2 separates the two.

## 5. Something to investigate: the 2026 slowdown

Monthly core sales, all property types:

| Month | Off-plan | Existing | Value (AED bn) |
|-------|---------:|---------:|---------------:|
| 2025-10 | 12,952 | 6,379 | 56.8 |
| 2025-12 | 13,398 | 5,423 | 62.3 |
| 2026-01 | 10,837 | 5,675 | 64.1 |
| 2026-02 | 10,553 | 6,128 | 60.2 |
| 2026-03 | 9,585 | 3,969 | 43.3 |
| 2026-05 | 7,126 | 2,723 | 27.7 |
| 2026-07 | 9,410 | 4,104 | 33.3 |
| 2026-09 | 7,251 | 3,662 | 28.5 |
| 2026-10 | 6,496 | 4,729 | 32.1 |

From March 2026, monthly sales value roughly halves compared with late 2025 and early 2026, and existing-property sales fall harder than off-plan.

**Before treating this as a finding:**
1. Compare a few months with the DLD's own published monthly figures or news coverage, to rule out a data lag.
2. Download the dataset again in a few weeks and check whether recent months grow (late registrations being backfilled).
3. Only then explain it. The forecast in Q5 must handle this structural break and not assume 2025's growth continues.

## 6. Data quality findings

| # | Finding | Scale | How we handle it |
|---|---------|-------|------------------|
| DQ1 | `instance_date` is **YYYY-MM-DD**, not DD-MM-YYYY as the attribute sheet says | All rows | Parse with explicit format `yyyy-MM-dd` in silver |
| DQ2 | Impossible years 1416–1422 (Hijri years entered as Gregorian) | 4 rows | Flag and exclude |
| DQ3 | `load_timestamp` column is not documented | 1 column | Keep in bronze as lineage; drop in silver |
| DQ4 | Attribute sheet swaps the descriptions of `master_project_en`/`_ar` and `trans_group_en`/`_ar` | Documentation | Noted in data dictionary |
| DQ5 | Arabic value `أخرى` ("Other") appears in `property_usage_en` | 4,432 sales rows | Map to `Other` |
| DQ6 | **Lease-to-own deals are recorded twice**: once under Sales and once under Mortgages, with linked IDs (`1-110-2018-822` and `2-110-2018-822`) | ~30k pairs | Excluded by the core sales definition |
| DQ7 | `transaction_id` has a hidden structure: `{group}-{procedure}-{year}-{sequence}` | All rows | Documented; useful for the DQ6 link |
| DQ8 | `area_name_en` holds official DLD land names (e.g. Jumeirah Village Circle is `Al Barsha South Fourth`); `master_project_en` holds the market names people recognise but is blank for 10.9% of sales | Sales | Build a `community` column: master project, falling back to area name |
| DQ9 | `rooms_en` mixes bedroom counts with other labels (`Office`, `Shop`, `PENTHOUSE`, `GYM`, `Single Room`); blank for 16.5% of sales (mostly land and villas) | Sales | Standardise to `bedrooms` (0 = studio) plus a `unit_label` column |
| DQ10 | Extreme prices: units up to AED 660M; 540 unit sales below AED 500 per sq m (about AED 46 per sq ft) | <0.1% | Flag in silver; use medians and percentile filters |
| DQ11 | `meter_sale_price` equals `actual_worth / procedure_area` to within 1% on all but 20 sales | Sales | Trust it; recompute per sq ft (1 sq m = 10.7639 sq ft) |
| DQ12 | `rent_value` and `meter_rent_price` are 98% blank and relate to lease-to-own finance, not rent | All rows | Drop in silver |
| DQ13 | No property or unit ID | — | Can't track repeat sales of the same unit or link a mortgage to its sale. Stated as a limitation |
| DQ14 | Nearest metro, mall and landmark are names, not distances, and about a third are blank | Sales | Used only as descriptive labels |

## 7. Blank rates for key columns (sales rows)

| Column | Blank |
|--------|------:|
| actual_worth, procedure_area, meter_sale_price, instance_date, area_name_en, reg_type_en, property_type_en | 0% |
| master_project_en | 10.9% |
| property_sub_type_en | 14.8% |
| rooms_en | 16.5% |
| project_name_en | 20.4% |
| building_name_en | 22.9% |
| nearest_landmark_en | 23.2% |
| nearest_metro_en | 33.4% |
| nearest_mall_en | 34.1% |
