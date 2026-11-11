# Module 0: Setup

**Goal:** a Databricks workspace with your two CSV files uploaded, and this project on GitHub.
**Time:** about 45 minutes, most of it waiting for the upload.

---

## Step 1: Sign up for Databricks Free Edition

Sign-up page: **<https://login.databricks.com/?dbx_source=docs&intent=CE_SIGN_UP>**

- Sign up with your personal Gmail (or "Continue with Google").
- Make sure you're on **Free Edition**, not the **Free Trial**. The trial is a 14-day business evaluation; Free Edition doesn't expire as long as you use it.
- Databricks creates your workspace straight away.

> **Why we're doing this**
> Databricks is a "lakehouse" platform: it stores data as files (cheap, like a data lake) but lets you query them like database tables (like a warehouse). Many large companies in the region run their data teams on it, often on Azure. Free Edition gives you the same notebooks, Spark, SQL editor and Unity Catalog, just with smaller compute.

---

## Step 2: Five-minute tour

Use the left sidebar to find these:

| Sidebar item | What it is | Excel / SQL equivalent |
|--------------|------------|------------------------|
| **Workspace** | Your folders and notebooks | Your Documents folder |
| **Catalog** | Every table, schema and volume | The list of databases in pgAdmin |
| **SQL Editor** | Write and run SQL | pgAdmin's query tool |
| **Compute** | The engine that runs your code. On Free Edition it's **serverless**, so there's nothing to set up | The PostgreSQL server you start before querying |

---

## Step 3: Create a schema and a volume

Open **SQL Editor**, paste this, and click **Run all**:

```sql
CREATE SCHEMA IF NOT EXISTS workspace.dubai_re
  COMMENT 'Dubai real estate transactions portfolio project';

CREATE VOLUME IF NOT EXISTS workspace.dubai_re.raw_files
  COMMENT 'Raw CSV downloads from the Dubai Data portal';
```

> **Why we're doing this**
> Unity Catalog organises everything in three levels: **catalog → schema → table/volume**, written as `workspace.dubai_re.bronze_transactions`.
> - `workspace` is the catalog Free Edition gives you (think: the whole filing cabinet).
> - `dubai_re` is our project's schema (one drawer). It's the same idea as a PostgreSQL schema.
> - A **volume** is a governed folder for *files* that aren't tables yet: our CSVs.
>
> `IF NOT EXISTS` makes the script safe to run twice. Writing code that can be re-run without breaking anything is called **idempotent**, and you'll hear that word in data engineering interviews.

**Check:** go to **Catalog → workspace**. You should see `dubai_re` with a volume called `raw_files` inside it.

---

## Step 4: Upload the two CSVs

1. **Catalog → workspace → dubai_re → Volumes → raw_files**
2. Click **Upload to this volume**.
3. Select both files:
   - `transactions_2026-11-06_21-44-39_0001.csv`
   - `transactions_2026-11-06_21-44-39_0002.csv`
4. Wait. They're 582 MB each, so this depends on your internet speed. Keep the tab open.

Upload only the two CSVs. The attribute sheet and the licence PDF are already in the repo under `docs/source/`.

**Check:** the volume shows two files of about 555 MiB each. (The browser shows MiB, which is why 582 MB appears as roughly 555.)

---

## Step 5: Put the project on GitHub

1. On GitHub, create a **new public repository** called `Dubai_Real_Estate_Analytics`. Don't add a README, `.gitignore` or licence, because this folder already has them.
2. In VS Code, open the `Dubai_Real_Estate_Analytics` folder, then open the terminal (`` Ctrl+` ``) and run:

```bash
git init
git add .
git commit -m "Set up project structure, data profile and data dictionary"
git branch -M main
git remote add origin https://github.com/marzrodie/Dubai_Real_Estate_Analytics.git
git push -u origin main
```

> **Why we're doing this**
> Hiring managers look at your commit history as well as the final files. Small, clear commits after each module ("Add bronze ingestion notebook") show how you work. The raw CSVs stay out of Git: GitHub blocks files over 100 MB, and the licence asks you to point people to the original source. `.gitignore` handles this.

---

## Step 6: Import the Module 1 notebook into Databricks

1. **Workspace → Home** (your user folder)
2. Click the **⋮** menu (or right-click) **→ Import**
3. Choose **File** and select `databricks/01_bronze_ingest.ipynb` from this repo
4. Open it. At the top right, make sure the compute says **Serverless** and shows connected.

---

## ✅ Module 0 checklist

- [ ] Free Edition workspace created
- [ ] `workspace.dubai_re` schema and `raw_files` volume exist
- [ ] Both CSVs are in the volume
- [ ] GitHub repo created and first commit pushed
- [ ] `01_bronze_ingest` notebook imported and connected to serverless compute

Next: **Module 1, Bronze ingestion** → open the notebook and work through it top to bottom.
