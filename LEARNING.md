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
