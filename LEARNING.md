# Learning notes

The concepts behind each module, linked to things you already know, with a short answer you can give in an interview. This file grows as each module is finished.

---

## Module 0: Setup

### Lakehouse
**What it is:** data stored as files in cheap cloud storage (a "lake"), with a layer on top that lets you query and manage it like database tables (a "warehouse").
**You already know:** BigQuery and PostgreSQL are warehouses. A folder of CSVs is a lake. Databricks is both.
**🎤 Interview answer:** *"A lakehouse keeps data in open file formats in cloud storage but adds warehouse features like tables, SQL, transactions and governance on top, so analysts and engineers work from one copy of the data."*

### Unity Catalog and the three-level namespace
**What it is:** Databricks' catalogue of every data asset, addressed as `catalog.schema.table`, e.g. `workspace.dubai_re.bronze_transactions`.
**You already know:** in PostgreSQL you write `schema.table` inside a database. Same idea, one level higher.
**🎤 Interview answer:** *"Unity Catalog is the governance layer. Tables, views and file volumes live under catalog.schema.object, and permissions and lineage are managed in one place."*

### Volume
**What it is:** a governed folder in Unity Catalog for *files*, e.g. the raw CSVs before they become tables. Paths look like `/Volumes/workspace/dubai_re/raw_files/`.

### Idempotent
**What it is:** an operation you can run once or many times with the same result. `CREATE … IF NOT EXISTS` and `mode("overwrite")` are idempotent; `INSERT` and `mode("append")` are not.
**🎤 Interview answer:** *"I design loads to be idempotent so a failed or repeated run never creates duplicates. In this project the bronze load overwrites the table, and the DDL uses IF NOT EXISTS."*

---

## Module 1: Bronze

### Medallion architecture (bronze → silver → gold)
| Layer | Holds | Excel equivalent |
|-------|-------|------------------|
| Bronze | Raw data as it arrived | The raw export tab you never edit |
| Silver | Cleaned, typed, standardised | Power Query output |
| Gold | Business-ready tables and aggregates | The table feeding your PivotTables |

**🎤 Interview answer:** *"Bronze keeps an untouched copy of the source for auditing and reprocessing, silver applies cleaning and typing rules, and gold holds modelled tables built for specific questions. If a cleaning rule is wrong, I fix it and rebuild from bronze without re-extracting."*

### Spark DataFrame
**What it is:** a table split into partitions that are processed in parallel across machines.
**You already know:** a pandas DataFrame. Similar syntax, but pandas runs on one machine and loads everything into memory; Spark spreads the work and only loads what it needs.

### Lazy evaluation: transformations vs actions
- **Transformations** (`read`, `select`, `filter`, `withColumn`, `join`) only add steps to a plan.
- **Actions** (`count`, `show`/`display`, `collect`, `write`) run the plan.
**You already know:** Power Query records applied steps and only runs them on **Close & Load**.
**🎤 Interview answer:** *"Spark builds a plan from transformations and only executes it when an action is called, which lets the optimiser combine steps, push filters down and skip columns it doesn't need."*

### Schema-on-read and why bronze stays as strings
`inferSchema` reads the whole file once just to guess the types, and it can guess wrong (IDs losing leading zeros, `1960.00` becoming a decimal). Bronze stays as text, and silver applies explicit, documented types.
**🎤 Interview answer:** *"I read raw files as strings in bronze to avoid an extra full scan and silent type errors, then cast explicitly in silver, where the rules are versioned and tested."*

### Delta Lake
Parquet files plus a transaction log, which add:
- **ACID transactions:** a write fully succeeds or fully fails
- **Time travel:** query earlier versions (`VERSION AS OF n`), see `DESCRIBE HISTORY`
- **Schema enforcement:** appends with the wrong columns fail
**🎤 Interview answer:** *"Delta adds a transaction log on top of Parquet, which gives ACID writes, schema enforcement and time travel, so data lake tables behave reliably like database tables."*

### Lineage / audit columns
`_source_file` (from Spark's hidden `_metadata` column) and `_ingested_at` record where and when each row was loaded.
**🎤 Interview answer:** *"Every bronze row carries its source file and load timestamp, so any figure can be traced back to a specific extract."*

---

## Module 2: Silver

### What silver is for
Turning *what the source sent* into *what the business means*: proper types, fixed values, standard categories and analysis flags, with every rule written in code.
**🎤 Interview answer:** *"In silver I apply the cleaning rules that came out of profiling: explicit types, fixes for known bad values, standardised categories and flags. I flag problems rather than delete rows, so totals still reconcile with the raw layer."*

### ANSI mode and `try_cast`
In ANSI mode (the default on Databricks), `CAST('abc' AS DOUBLE)` fails the job instead of returning NULL. `try_cast` returns NULL for values it can't convert. You then **count the NULLs** to prove nothing was lost.
**You already know:** Excel's `IFERROR(VALUE(A2), "")`, or PostgreSQL raising `invalid input syntax for type numeric`.
**🎤 Interview answer:** *"I use try_cast so one malformed value can't stop a 1.8M-row job, then I check NULL counts before and after to prove the casting didn't silently drop data. Price, area and date had zero failures."*

### Nulls in disguise
A blank can be a real `NULL` or an empty string `""`. `COUNT(col)` and `COALESCE` treat them differently, so check which one you have before cleaning.

### `when` / `otherwise`
PySpark's `CASE WHEN … THEN … ELSE … END`, and Excel's `IF()`. A `when` without `otherwise` returns NULL when nothing matches, which is how `bedrooms` becomes NULL for offices.

### NULL vs 0
0 is a real value (a studio has 0 bedrooms). NULL means "not applicable" or "unknown" (an office has no bedroom count). Mixing them up breaks averages and counts.

### Regular expressions
`^\d+ B/R$` = start, one or more digits, space, `B/R`, end. `rlike` tests a match; `regexp_extract(col, pattern, 1)` pulls out the part in brackets.

### Flag, don't delete (outliers)
Outliers are marked `is_price_outlier` (below the 0.5th or above the 99.5th percentile of AED per sq ft, *within each property type*), never removed. Each analysis chooses whether to exclude them, and you can show results both ways.
**🎤 Interview answer:** *"I defined outliers relative to each property type, because land and flats have completely different price levels. That flagged about 1% of sales. I keep them in the table and exclude them in analysis, so the decision is visible and reversible."*

### `percentile_approx`
An exact percentile sorts every value. The approximate version is far cheaper and accurate to a tiny fraction of a percentile, which is plenty for an outlier threshold.

### Broadcast joins and join fan-out
`F.broadcast(small_df)` sends a small lookup table to every worker, so the big table never has to be shuffled. **Fan-out** is when the lookup has duplicate keys and the join silently multiplies rows.
**🎤 Interview answer:** *"I check the row count after every join. If a lookup has duplicate keys, the join multiplies rows and inflates every total downstream."*

### Data quality gate
A list of yes/no tests that runs automatically and **raises an error** if any fail, so a bad table never reaches a dashboard. Mine tests row count vs bronze, unique IDs, no NULL dates or prices, valid ranges, and flag consistency. Industry tools: dbt tests, Great Expectations, Databricks DLT expectations.

### When not to partition
Partitioning splits a table into folders (e.g. by year). On a small table (well under 1 TB) it creates many tiny files and makes queries slower. Not partitioning was a deliberate choice here.

### Median vs average for prices
One AED 50M penthouse moves an average a lot but barely moves the median. Price reporting in real estate uses medians.

---

## Module 3: Gold

### Star schema, fact and dimension
- **Fact:** events you measure (sales), holding measures plus keys. Long and narrow.
- **Dimension:** descriptions you filter and group by (date, location, property). Short and wide.
- **Star:** fact in the middle, each dimension joined directly to it. (A *snowflake* would chain dimensions to other dimensions, which Power BI handles less well.)
**You already know:** a transactions sheet plus lookup sheets, with relationships instead of XLOOKUPs.
**🎤 Interview answer:** *"I modelled the sales as a star schema: one fact table at the grain of one core sale, with date, location and property dimensions on integer surrogate keys. It's the shape Power BI's VertiPaq engine is built for: small dimensions to filter, one narrow fact to aggregate."*

### Grain
What one fact row represents. Mine: *one core sale registered from 2015 onwards*. State it first; mixing grains (e.g. sales and monthly targets in one table) is the classic modelling mistake.

### Surrogate keys (and why hash them)
One integer standing for a combination of descriptive columns. I used `xxhash64` over the NULL-safe, delimited values:
- **Deterministic:** the same location gets the same key on every re-run.
- **No lookup needed:** the fact computes its keys independently.
- **NULL-safe:** Spark's hash skips NULLs, so I replace them with a marker first; otherwise `(A, NULL)` and `(NULL, A)` would collide.
**🎤 Interview answer:** *"Power BI relationships need a single column, and integer joins are faster than multi-column text joins. I hashed the natural key so the keys are stable across re-runs, and handled NULLs explicitly to avoid collisions."*

### Date dimension
One row per calendar day, generated with `explode(sequence(start, end, INTERVAL 1 DAY))`. Needed for time intelligence (YoY, YTD), so that months with zero sales still appear, and to hold flags like `is_complete_month`.
**🎤 Interview answer:** *"The date table gives a continuous calendar for time intelligence and lets me flag the partial last month once, so no chart or forecast ever compares five days with a full month."*

### Parameterising
Derive values from the data (`DATA_END = max(transaction_date)`) instead of hard-coding them. A new extract then updates everything automatically.

### Referential integrity and anti joins
Every fact key must exist in its dimension. `fact.join(dim, "key", "left_anti")` returns the orphans; the test expects zero.
**You already know:** rows where XLOOKUP returns `#N/A`; in SQL, `WHERE NOT EXISTS`.
**🎤 Interview answer:** *"My gold gate tests that every dimension key is unique and that the fact has no orphan keys, using anti joins. Orphans would fall into a Blank bucket in Power BI and silently disappear from filtered visuals."*

### Conditional aggregation
`SUM(CASE WHEN cond THEN 1 ELSE 0 END)` is COUNTIFS in SQL. `MEDIAN(CASE WHEN cond THEN x END)` gives a filtered median in the same query, because non-matching rows become NULL and are ignored.

### Business categories defined once
`segment` and `bedroom_band` live in `dim_property`, so SQL, Python, Excel and Power BI all use the same groups. `bedroom_sort` fixes Power BI's alphabetical sorting (**Sort by column**).

### When `.toPandas()` is safe
It moves every row onto one machine's memory: fine for a few thousand aggregated rows, never for the 1M-row fact. *"Aggregate in Spark, then move small results to pandas."*

---

## Module 4: Q1 price hotspots (SQL window functions)

### GROUP BY vs window functions
`GROUP BY` collapses rows; a window function computes across related rows **and keeps every row**. `FUNCTION() OVER (PARTITION BY … ORDER BY …)`.
**You already know:** `=IF(A3=A2, C3/C2-1, "")` in a sorted sheet. `PARTITION BY` is the "same group" test, `ORDER BY` is the sort, `LAG` is "the row above".
**🎤 Interview answer:** *"GROUP BY returns one row per group; a window function adds a calculated column while keeping the detail rows, so I can compare each year with the previous one, or rank within a segment."*

### ROW_NUMBER vs RANK vs DENSE_RANK
With a tie for 2nd: `ROW_NUMBER` 1,2,3,4 · `RANK` 1,2,2,4 · `DENSE_RANK` 1,2,2,3.
**🎤 Interview answer:** *"ROW_NUMBER when I need exactly one row per group, like the latest record; RANK for league tables where ties share a place; DENSE_RANK when I want the top N distinct values."*

### The LAG gap trap
`LAG` returns the previous **row**, not the previous **year**. If a year is missing, you silently compare across two years. Fix: `WHERE prev_year = year - 1`. (In my data, 285 rows would have been mislabelled.)

### QUALIFY
Filters on a window function's result (`HAVING` is for aggregates, `QUALIFY` is for windows). Window functions can't go in `WHERE`, because `WHERE` runs before they're calculated. Supported in Databricks, Snowflake and BigQuery; in PostgreSQL, use a sub-query or CTE.

### WINDOW clause and CTEs
`WINDOW w AS (PARTITION BY … ORDER BY …)` names a window once for several functions. A CTE (`WITH x AS (…)`) names a sub-query so the logic reads top to bottom.

### Minimum-sample rule
55% of 2025 groups had fewer than 50 sales, but together they were under 3% of sales. Requiring 50+ sales in both years removes noise while keeping almost the whole market.
**🎤 Interview answer:** *"Before ranking, I checked how many sales sat behind each median. I set a 50-sale minimum, which removed half the groups but under 3% of sales. It's a documented judgement call, parameterised so I can test its sensitivity."*

### Pivot with conditional aggregation
`MAX(CASE WHEN year = 2021 THEN x END)` puts each year's value in its own column: a PivotTable in SQL.

### CAGR
`(end / start) ^ (1 / years) − 1`. A 60% rise over 4 years is about 12.5% a year (compounding), not 15%.

### Like-for-like comparison
Compare the same months in both years (Jan–Oct 2026 vs Jan–Oct 2025), never a partial year against a full one, so season and length don't distort the change.

### Mix (composition) effect
A median can jump because *different* homes sold, not because homes gained value. In Dubai the main driver is off-plan vs ready share. Motor City 2-bed apartments showed +97% in 2025, but the off-plan share went from about 25% to 65%, and ready-only prices rose about 27%.
**🎤 Interview answer:** *"My top riser nearly doubled, which made me suspicious. Splitting by off-plan vs ready showed most of it was a change in mix, not appreciation. I flagged groups whose off-plan share moved 25+ points and reported the ranking with and without them. The proper fix would be a repeat-sales index, which needs a unit ID this dataset doesn't have."*

### Market breadth
The share of groups rising, and the median change across groups. It shows whether a rise is widespread or driven by a few hotspots.

---

## Module 5: Q2 off-plan vs ready (Python statistics)

### Three questions: is it real, how big, how sure?
Hypothesis test → p-value; effect size; confidence interval. With hundreds of thousands of rows nearly everything is "significant", so effect size and interval matter most.
**🎤 Interview answer:** *"Significance only tells me a difference probably isn't chance. With 580k sales nearly everything is significant, so I report the size of the effect and its confidence interval, because that's what a business decision depends on."*

### p-value
The probability of seeing a difference at least this large **if the null hypothesis were true**. It is *not* the probability the null is true, and says nothing about size.

### Mann–Whitney U vs t-test
A t-test compares means and assumes roughly normal data. Mann–Whitney compares ranks: no normality assumption, robust to extreme values, and consistent with median-based reporting. Prices per sq ft are right-skewed (2025 apartments: mean ~1,900 vs median ~1,730).
**🎤 Interview answer:** *"Prices are skewed and I report medians, so I used Mann–Whitney, a rank-based test that doesn't assume normality or get pulled around by extreme sales."*

### Effect size: common-language effect size (CLES)
`U / (n₁ × n₂)`: the probability that a random off-plan sale is priced above a random ready one (50% = no difference). JVC 1-bed 2025: about 72%.

### Confounding
A third variable that affects both the "treatment" (off-plan or ready) and the outcome (price). Here, location: off-plan volume sits in cheaper communities. Fix: compare within matched groups (community × bedrooms × year), or control for the variable in a regression.

### Simpson's paradox
A trend that reverses when you aggregate. Naive off-plan premium: about 56% → 24% (2021 → 2026). Like-for-like: about 19% → 32%. Both are "true"; only one answers the question.
**🎤 Interview answer:** *"The naive premium halved, which looked like buyers were cooling on off-plan. But off-plan volume had shifted to cheaper communities. Comparing within the same community and unit type, the premium actually rose from about 19% to 32%. Reporting the naive number would have given the wrong headline."*

### Weighted median
The value where cumulative weight reaches half the total. It summarises per-group results the way the *typical buyer* experiences them, not the typical group.

### Multiple comparisons and Benjamini–Hochberg
Run 120 tests at 5% and you expect about 6 false positives even if nothing is real. BH controls the **false discovery rate** (the share of "significant" results that are false alarms); Bonferroni (0.05 ÷ tests) is stricter and loses more real findings.

### Bootstrap confidence interval
Resample your data with replacement thousands of times, recompute the statistic each time, and take the middle 95%. It needs no formula, so it works for medians and weighted medians. Resample **the unit you generalise over**: sales within one group; groups for the market-wide premium (a cluster bootstrap). Set a **seed** for reproducibility.
**🎤 Interview answer (for a non-technical manager):** *"I re-ran the calculation 2,000 times on slightly different random versions of the data. 95% of the answers fell between 20% and 45%, so that's the range we can be fairly confident about."*

### Fixed-effects regression with cluster-robust errors
`log(price) ~ off_plan + C(group)`: one intercept per group means β only compares within groups; `exp(β) − 1` is the premium. Cluster-robust standard errors account for sales in the same group not being independent. Result: about 33% (CI 28–38%), agreeing with the weighted median of about 30%.

### Virtual environments and reproducibility
`.venv` + `requirements.txt` + a fixed random seed + data recreated from documented code = anyone can rerun the analysis and get the same numbers.

---

## PySpark ↔ pandas ↔ SQL cheat sheet (grows each module)

| Task | PySpark | pandas | SQL |
|------|---------|--------|-----|
| Read CSV | `spark.read.option("header","true").csv(path)` | `pd.read_csv(path)` | — |
| Column names | `df.columns` | `df.columns` | — |
| Types | `df.printSchema()` | `df.dtypes` | `\d table` (psql) |
| First rows | `df.limit(5)` / `display(df)` | `df.head(5)` | `SELECT … LIMIT 5` |
| Pick columns | `df.select("a", "b")` | `df[["a", "b"]]` | `SELECT a, b` |
| Count rows | `df.count()` | `len(df)` | `SELECT COUNT(*)` |
| Add column | `df.withColumn("x", expr)` | `df["x"] = …` | `SELECT *, expr AS x` |
| Distinct values | `df.select("a").distinct()` | `df["a"].unique()` | `SELECT DISTINCT a` |
| Save table | `df.write.saveAsTable("t")` | `df.to_sql("t", con)` | `CREATE TABLE t AS …` |
| Run SQL from Python | `spark.sql("SELECT …")` | `pd.read_sql("SELECT …", con)` | — |
| Read a table | `spark.table("cat.schema.t")` | `pd.read_sql_table("t", con)` | `SELECT * FROM t` |
| Rename | `F.col("a").alias("b")` | `df.rename(columns={"a": "b"})` | `a AS b` |
| Safe cast | `F.expr("try_cast(a AS DOUBLE)")` | `pd.to_numeric(df.a, errors="coerce")` | `TRY_CAST(a AS DOUBLE)` |
| Parse date | `F.to_date("a", "yyyy-MM-dd")` | `pd.to_datetime(df.a, format="%Y-%m-%d")` | `TO_DATE(a, 'YYYY-MM-DD')` |
| If / else | `F.when(cond, x).otherwise(y)` | `np.where(cond, x, y)` | `CASE WHEN … END` |
| First non-null | `F.coalesce("a", "b")` | `df.a.fillna(df.b)` | `COALESCE(a, b)` |
| In a list | `F.col("a").isin([...])` | `df.a.isin([...])` | `a IN (...)` |
| Regex match | `F.col("a").rlike(r"^\d+")` | `df.a.str.match(r"^\d+")` | `a ~ '^\d+'` (Postgres) |
| Group and aggregate | `df.groupBy("a").agg(F.sum("b"))` | `df.groupby("a").b.sum()` | `GROUP BY a` |
| Percentile | `F.percentile_approx("b", 0.995)` | `df.b.quantile(0.995)` | `PERCENTILE_CONT(0.995)` |
| Join | `df.join(other, on="k", how="left")` | `df.merge(other, on="k", how="left")` | `LEFT JOIN … USING (k)` |
| Filter | `df.filter("a > 5")` | `df[df.a > 5]` | `WHERE a > 5` |
| Month start | `F.trunc("d", "month")` | `df.d.dt.to_period("M")` | `DATE_TRUNC('month', d)` |
| Max value | `df.agg(F.max("d")).first()[0]` | `df.d.max()` | `SELECT MAX(d)` |
| Distinct rows | `df.select(...).distinct()` | `df[[...]].drop_duplicates()` | `SELECT DISTINCT …` |
| Hash key | `F.xxhash64(F.concat_ws("||", ...))` | `pd.util.hash_pandas_object(df)` | `xxhash64(…)` (Spark SQL) |
| Rows with no match | `df.join(dim, "k", "left_anti")` | `df[~df.k.isin(dim.k)]` | `WHERE NOT EXISTS (…)` |
| Generate dates | `explode(sequence(d1, d2, INTERVAL 1 DAY))` | `pd.date_range(d1, d2)` | `generate_series` (Postgres) |
| Smallest of values | `F.least("a", F.lit(5))` | `np.minimum(df.a, 5)` | `LEAST(a, 5)` |
| Spark → pandas | `df.toPandas()` (small data only) | — | — |
| Create table from query | `spark.sql("CREATE OR REPLACE TABLE t AS SELECT …")` | — | `CREATE TABLE t AS SELECT …` |
| Previous row in group | `F.lag("x").over(Window.partitionBy("g").orderBy("yr"))` | `df.groupby("g").x.shift(1)` | `LAG(x) OVER (PARTITION BY g ORDER BY yr)` |
| Rank in group | `F.rank().over(w)` | `df.groupby("g").x.rank(method="min", ascending=False)` | `RANK() OVER (PARTITION BY g ORDER BY x DESC)` |
| Share of total | `F.col("x") / F.sum("x").over(Window.partitionBy())` | `df.x / df.x.sum()` | `x / SUM(x) OVER ()` |
| Filter on a window | `.filter(F.col("rk") <= 10)` after `withColumn` | `df[df.rk <= 10]` | `QUALIFY rk <= 10` |
| Pivot years to columns | `df.groupBy("g").pivot("yr").agg(F.max("x"))` | `df.pivot_table(index="g", columns="yr", values="x")` | `MAX(CASE WHEN yr = 2021 THEN x END)` |
| Power | `F.pow("a", 0.25)` | `df.a ** 0.25` | `POWER(a, 0.25)` |
| Count matching rows | `F.count_if(cond)` / `F.sum(cond.cast("int"))` | `(cond).sum()` | `COUNT_IF(cond)` |
| Read Parquet | `spark.read.parquet(path)` | `pd.read_parquet(path)` | — |
| Named aggregations | `.agg(F.median("x").alias("median"), F.count("*").alias("n"))` | `.agg(median="median", n="size")` | `MEDIAN(x) AS median, COUNT(*) AS n` |
| Long → wide | `.groupBy(...).pivot("market")` | `.unstack("market")` | `MAX(CASE WHEN market = … END)` |
| Proportion of True | `F.avg(cond.cast("int"))` | `(df.a == "x").mean()` | `AVG(CASE WHEN … THEN 1.0 ELSE 0 END)` |
| Map values from a lookup | `df.join(lookup, "k")` | `df.k.map(series)` | `JOIN … ON` |
| Loop over groups | — | `for key, g in df.groupby("k"):` | — |
