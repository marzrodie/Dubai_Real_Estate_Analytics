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
