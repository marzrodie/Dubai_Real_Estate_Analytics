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

## Columns added by this project

| Column | Layer | Definition |
|--------|-------|------------|
| `_source_file` | Bronze | CSV file the row came from |
| `_ingested_at` | Bronze | When the row was loaded into Databricks |
| `community` | Silver | `master_project_en`, or `area_name_en` when that is blank |
| `bedrooms` | Silver | 0 for Studio, 1–9 for `n B/R`, null otherwise |
| `area_sqft`, `price_per_sqft` | Silver | Converted from sq m |
| `is_core_sale` | Silver | `procedure_name_en` in (`Sell`, `Sell - Pre registration`, `Delayed Sell`) |
| `is_price_outlier` | Silver | Outside the 0.5th–99.5th percentile of price per sq ft within property type |
