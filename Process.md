# Project Notes: Rearc Data Quest (Databricks Edition)

This is a quick rundown of how I set up this Bronze/Silver/Gold data pipeline on Databricks using Spark Declarative Pipelines (SDP).

---

### Data Sources & Where Things Land

I pulled data from two places:
1. **BLS Productivity Time-Series**: Pulled straight from `https://download.bls.gov/pub/time.series/pr/`, which includes 12 raw text files (data, series metadata, and various lookup tables like measure, sector, and class). Note: BLS will hand you a 403 Forbidden error if you don't include a proper `User-Agent` with contact info, per their data access rules.
2. **DataUSA Population API**: Grabs US population estimates (ACS 1-year) going from 2013 to 2024 (skipping 2020).

Everything gets dumped into a Unity Catalog Volume at `/Volumes/rearc_quest/bronze/raw_data/`, accompanied by an `ingestion_manifest.json` that acts as a quick audit trail for what was pulled and when.

---

### Pipeline Design

* **Ingestion**: Kept it idempotent so it checks if a file already exists in the volume before downloading. It also dynamically parses the BLS directory listing HTML, meaning if they tweak or add a file, it won't break the script.
* **Bronze**: Landed the raw text files and JSON into Delta tables with minimal changes (mostly just whitespace trimming) and added basic expectation checks for non-null primary keys.
* **Silver**: Cleaned up types (casting years to integers, values to doubles, population to bigints), dropped duplicates, and built a joined reference table (`silver.bls_series_enriched`) that stitches together all the BLS lookup tables. This creates human-readable labels like `Sector - Class - Measure (Duration, Seasonality)` so you don't have to decode raw IDs manually.
* **Gold**: Built three tables to answer the quest questions:
  * *Q1 (`gold.population_stats`)*: Mean and std dev of US population from 2013–2018.
  * *Q2 (`gold.best_year_per_series`)*: Finds the peak year for each series using a window function over summed quarterly values.
  * *Q3 (`gold.series_q01_population`)*: Pulls a specific series (`PRS30006032`) for Q1 and joins it with the corresponding year's population.

*Note on SQL vs. PySpark:* I went with **Spark SQL as the primary language** for the Gold layer because it makes window functions and joins look cleaner and easier for anyone else to review. I left PySpark alternatives as commented-out code blocks in case procedural code is preferred.

---

### Real Talk / Hurdles

* **BLS Whitespace:** Those text files are padded with heavy whitespace (e.g., trailing spaces on series IDs and leading spaces on values). Had to clean those thoroughly before casting or joining, otherwise types and keys got messed up.
* **JSON Flattening:** Unnesting the DataUSA API payload required explicitly exploding the nested `data` array before dropping it into the table.
* **File Choice:** I used `pr.data.0.Current` (around 38k rows) instead of `pr.data.1.AllData` (77k rows) as the main driver because it has the active current data analysts usually care about, though both land in bronze anyway.

---

### Production Caveats

If this were going to a real production environment instead of a tech quest, I’d spend time hardening a few areas:
* **Schema Evolution:** Right now, it's pretty implicit; if BLS shifts a column, it'll get dropped silently. Need explicit schema evolution or validation policies.
* **Scale & Partitioning:** Small dataset right now, but for real scale, I'd partition the bronze tables by year and look at Auto Loader.
* **Monitoring:** Hooking up actual alerting on data quality expectation failures rather than just checking the pipeline UI.

*(Used Databricks Assistant to help scaffold out the boilerplate for the ingestion script and initial pipeline structure, then cleaned and tweaked the logic from there.)*