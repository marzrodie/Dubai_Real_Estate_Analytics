# Data dictionary

This is based on the publisher's attribute sheet, corrected and extended with what the data actually contains (see [`data_profile.md`](data_profile.md)). Only English columns are listed. Each has an Arabic twin ending in `_ar` (14 in total) that stays in bronze and is dropped in silver.

"Blank %" is measured on sales rows.

## Identifiers

| Column | Actual content | Blank % | Notes |
|--------|----------------|--------:|-------|
| `transaction_id` | Text like `1-11-2022-16850` | 0 | Unique. Structure is `{trans_group_id}-{procedure_id}-{year}-{sequence}` (not documented) |
| `procedure_id` | Integer | 0 | Code for `procedure_name_en` |
| `trans_group_id` | 1 = Sales, 2 = Mortgages, 3 = Gifts | 0 | |
| `property_type_id`, `property_sub_type_id`, `reg_type_id`, `area_id` | Integer codes | 0 / 14.8 / 0 / 0 | Lookup codes for the matching `_en` columns |
| `project_number` | Number stored as text with decimals (`1960.00`) | 20.4 | Developer project number; cast to integer |

## Transaction

| Column | Actual content | Blank % | Notes |
|--------|----------------|--------:|-------|
| `instance_date` | Date, **YYYY-MM-DD** | 0 | The attribute sheet says DD-MM-YYYY, which is wrong (DQ1). 4 rows have Hijri years (DQ2) |
| `trans_group_en` | Sales / Mortgages / Gifts | 0 | The sheet swaps the `_en` and `_ar` descriptions (DQ4) |
| `procedure_name_en` | 60+ procedure types | 0 | Core sales = `Sell`, `Sell - Pre registration`, `Delayed Sell` |
| `reg_type_en` | `Off-Plan Properties` / `Existing Properties` | 0 | |
| `no_of_parties_role_1` | Integer | 0.1 | Sellers / mortgagors |
| `no_of_parties_role_2` | Integer | 0.1 | Buyers / mortgagees |
| `no_of_parties_role_3` | Integer | 0.1 | Lessees / usufructuaries |
| `load_timestamp` | ISO timestamp | 0 | **Not documented.** When the extract was produced (DQ3) |

## Money and size

| Column | Actual content | Blank % | Notes |
|--------|----------------|--------:|-------|
| `actual_worth` | AED, 2 decimals | 0 | Transaction value. For mortgages it's the mortgage amount, not a sale price |
| `procedure_area` | Square metres | 0 | Multiply by 10.7639 for sq ft |
| `meter_sale_price` | AED per sq m | 0 | Equals `actual_worth / procedure_area` (DQ11) |
| `rent_value` | AED | 100 | Not rent: the finance value in lease-to-own deals (DQ12) |
| `meter_rent_price` | AED per sq m | 100 | As above |

## Property

| Column | Actual content | Blank % | Notes |
|--------|----------------|--------:|-------|
| `property_type_en` | Unit / Villa / Land / Building | 0 | |
| `property_sub_type_en` | Flat, Villa, Office, Hotel Apartment, Shop, Hotel Rooms… | 14.8 | Mostly blank for land |
| `property_usage_en` | Residential, Commercial, Hospitality, Other… | 0 | Contains the Arabic value `أخرى` ("Other") (DQ5) |
| `rooms_en` | `Studio`, `1 B/R` … `9 B/R`, `PENTHOUSE`, `Office`, `Shop`… | 16.5 | Mixes bedroom counts and other labels (DQ9) |
| `has_parking` | `0` / `1` | 0 | |

## Location

| Column | Actual content | Blank % | Notes |
|--------|----------------|--------:|-------|
| `area_name_en` | Official DLD land area (244 values) | 0 | Not market names: JVC is `Al Barsha South Fourth` (DQ8) |
| `master_project_en` | Master community name (161 values), e.g. `Jumeirah Village Circle`, `Dubai Marina` | 10.9 | The sheet calls it "master developer name", but the values are master communities |
| `project_name_en` | Developer project (3,443 values) | 20.4 | e.g. `SOBHA ONE`, `Binghatti Skyrise` |
| `building_name_en` | Developer's building name | 22.9 | |
| `nearest_metro_en` / `nearest_mall_en` / `nearest_landmark_en` | Names only, no distance | 33.4 / 34.1 / 23.2 | DQ14 |

## Bronze additions

| Column | Definition |
|--------|------------|
| `_source_file` | CSV file the row came from |
| `_ingested_at` | When the row was loaded into Databricks |

## Silver table: `workspace.dubai_re.silver_transactions`

1,814,844 rows (every transaction, all groups) · 36 columns. Built by `databricks/02_silver_clean.ipynb`.

| Column | Type | From bronze | Rule |
|--------|------|-------------|------|
| `transaction_id` | string | `transaction_id` | Unchanged; tested unique |
| `transaction_date` | date | `instance_date` | `to_date(…, 'yyyy-MM-dd')` |
| `transaction_month` | date | derived | First day of the month |
| `is_valid_date` | boolean | derived | Year ≥ 1960 (4 rows false) |
| `transaction_group` | string | `trans_group_en` | Sales / Mortgages / Gifts |
| `procedure` | string | `procedure_name_en` | |
| `is_core_sale` | boolean | derived | Procedure in Sell, Sell - Pre registration, Delayed Sell (1,314,686 rows) |
| `registration_type` | string | `reg_type_en` | |
| `is_off_plan` | boolean | derived | `registration_type = 'Off-Plan Properties'` |
| `property_type` | string | `property_type_en` | Unit / Villa / Land / Building |
| `property_sub_type` | string | `property_sub_type_en` | |
| `property_usage` | string | `property_usage_en` | `أخرى` → `Other` |
| `rooms_raw` | string | `rooms_en` | Kept for reference |
| `bedrooms` | int | derived | Studio = 0, `n B/R` = n, otherwise NULL |
| `has_parking` | boolean | `has_parking` | `'1'` → true |
| `price_aed` | double | `actual_worth` | |
| `area_sqm` | double | `procedure_area` | |
| `area_sqft` | double | derived | `area_sqm × 10.7639` |
| `price_per_sqm` | double | `meter_sale_price` | |
| `price_per_sqft` | double | derived | `price_aed / area_sqft` when area > 0 |
| `is_price_outlier` | boolean | derived | Core sales only: below the 0.5th or above the 99.5th percentile of price per sq ft within property type (about 13,100 rows). NULL for non-core rows |
| `community` | string | derived | `master_project`, or `dld_area` if blank (373 communities in core sales) |
| `master_project` | string | `master_project_en` | |
| `dld_area` | string | `area_name_en` | |
| `area_id` | int | `area_id` | |
| `project` | string | `project_name_en` | |
| `project_number` | int | `project_number` | `'1960.00'` → 1960 |
| `building` | string | `building_name_en` | |
| `nearest_metro`, `nearest_mall`, `nearest_landmark` | string | `nearest_*_en` | |
| `parties_role_1`, `parties_role_2`, `parties_role_3` | int | `no_of_parties_role_*` | |
| `_source_file`, `_ingested_at` | string, timestamp | bronze | Lineage |

**Dropped in silver:** all 14 `_ar` columns, the five `*_id` codes that duplicate a text column (`procedure_id`, `trans_group_id`, `property_type_id`, `property_sub_type_id`, `reg_type_id`), `rent_value`, `meter_rent_price` and `load_timestamp`.

## Gold tables (star schema)

Built by `databricks/03_gold_model.ipynb`. Scope: core sales, valid dates, from 1 January 2015.

### `fact_sales`: one row per core sale (1,015,297 rows)

| Column | Type | Definition |
|--------|------|------------|
| `transaction_id` | string | Unique sale ID |
| `date_key` | int | `yyyyMMdd` → `dim_date` |
| `location_key` | bigint | xxhash64(community, dld_area) → `dim_location` |
| `property_key` | bigint | xxhash64(property_type, property_sub_type, property_usage, bedrooms) → `dim_property` |
| `is_off_plan` | boolean | Off-plan registration |
| `market` | string | `Off-plan` / `Ready` |
| `price_aed`, `area_sqft`, `price_per_sqft` | double | Measures |
| `is_price_outlier` | boolean | From silver |
| `has_parking` | boolean | |
| `project`, `building` | string | Drill-down detail |

### `dim_date`: one row per day, 2015-01-01 to the end of the last data year (4,383 rows)
`date_key`, `date`, `year`, `quarter`, `month`, `month_name`, `month_start`, `year_month`, `is_complete_month` (month ended on or before the last data date).

### `dim_location`: one row per community × DLD area (341 rows)
`location_key`, `community`, `dld_area`. One community can span several DLD areas (318 distinct communities).

### `dim_property`: one row per property type × sub-type × usage × bedrooms (74 rows)
`property_key`, `property_type`, `property_sub_type`, `property_usage`, `bedrooms`, plus:
- `segment`: Apartment · Villa / Townhouse · Hotel apartment · Office · Retail · Land · Building · Other
- `bedroom_band`: Studio · 1 BR … 4 BR · 5+ BR · N/A
- `bedroom_sort`: 0–5, 99 for N/A (for sorting in Power BI)

### Aggregates (also exported to `data/gold/*.csv`)
| Table | Grain | Columns |
|-------|-------|---------|
| `agg_market_monthly` | Complete month (142 rows) | `month_start`, `sales`, `value_aed_bn`, `off_plan_sales`, `ready_sales`, `median_apartment_aed_sqft` (excl. outliers) |
| `agg_price_community_yearly` | Year × community × segment × bedroom band, apartments and villas only, excl. outliers (~5,700 rows) | `year`, `is_full_year`, `community`, `segment`, `bedroom_band`, `bedroom_sort`, `sales`, `median_aed_sqft`, `median_price_aed`, `off_plan_pct` |

## Q1 results tables (Module 4)

Built by `databricks/04_q1_price_hotspots.ipynb` from `agg_price_community_yearly` and the star schema. Apartments and villas/townhouses, outliers excluded, minimum 50 sales in each compared period. Exported to `data/gold/*.csv`.

| Table | Grain | Key columns |
|-------|-------|-------------|
| `q1_yoy` | Year × community × segment × bedroom band, consecutive years only | `prev_median_aed_sqft`, `median_aed_sqft`, `yoy_pct`, `prev_sales`, `sales`, `prev_off_plan_pct`, `off_plan_pct`, `mix_shift_pts`, `is_mix_shift` (off-plan share moved by 25+ points) |
| `q1_growth_2021_2025` | Community × segment × bedroom band | `median_2021`, `median_2025`, `sales_2021`, `sales_2025`, `total_growth_pct`, `cagr_pct` |
| `q1_momentum_2026` | Community × segment × bedroom band | Jan to last complete month, 2025 vs 2026: `sales_ytd_2025/2026`, `median_ytd_2025/2026`, `ytd_change_pct`, `yoy_2025_pct`, `trend` (Still rising / Turning down / Recovering / Still falling; NULL if not in `q1_yoy`) |

## Q2 extract and results (Module 5)

**Extract:** `data/extracts/q2_apartments.parquet`, built by `databricks/05_export_q2_extract.ipynb`. Apartment core sales, 2021 to the last complete month, outliers excluded (582,612 rows). Not in Git.
Columns: `transaction_id`, `month_start`, `year`, `community`, `bedroom_band`, `bedroom_sort`, `market`, `price_aed`, `area_sqft`, `price_per_sqft`.

**Results** (`data/gold/`, written by `python/05_q2_offplan_vs_ready.ipynb`):

| File | Grain | Key columns |
|------|-------|-------------|
| `q2_off_plan_share.csv` | Year | `off_plan_share_pct` |
| `q2_premium_by_group.csv` | Year × community × bedroom band with 30+ off-plan **and** 30+ ready sales | `median_off_plan`, `median_ready`, `n_off_plan`, `n_ready`, `premium_pct`, `weight` |
| `q2_premium_by_year.csv` | Year | `groups`, `sales_covered`, `coverage_pct`, `like_for_like_premium_pct` (sales-weighted median), `ci_low`, `ci_high` (95% cluster bootstrap), `naive_premium_pct` |
| `q2_tests_2025.csv` | 2025 groups | Group columns plus `p_value` (Mann–Whitney U), `p_adjusted` (Benjamini–Hochberg), `significant` |
