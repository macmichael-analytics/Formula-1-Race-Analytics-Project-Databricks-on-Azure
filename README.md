## Formula 1 Race Analytics — Databricks on Azure

End-to-end data engineering project built on Azure + Databricks to ingest Formula 1 race data from the Ergast API, store it in Parquet, and provide SQL analysis and dashboards (Dominant Drivers, Dominant Teams) and Databricks Dashboards. This repository contains notebooks, pipeline definitions, sample code, and instructions to reproduce the solution.

![charles-leclerc-ferrari-sf-23-](https://github.com/vedanthv/data-engineering-projects/assets/44313631/4e8c3e14-0652-4ebc-b418-3e906526c6e4)
<img src = "https://github.com/macmichael-analytics/Formula-1-Race-Analytics-Project-Databricks-on-Azure/project-images/f1-car.jpg">

# Formula 1 Race Analytics — Databricks on Azure

> End-to-end data engineering project built on Azure + Databricks to ingest Formula 1 race data from the Ergast API, store it in Parquet, and provide SQL analysis and dashboards (Dominant Drivers, Dominant Teams) and Databricks Dashboards. This repository contains notebooks, pipeline definitions, sample code, and instructions to reproduce the solution.

---

## Architecture Overview


<img src = "https://github.com/vedanthv/data-engineering-projects/blob/main/formula-1-analytics-engg/static/formula1-solution-architecture.png">

High-level flow:

1. **Extract** data from the Ergast API.
2. **ADLS Raw Layer** — raw JSON payloads stored as-is (optional).
3. **Ingest** — apply schema, add audit columns, write Parquet to **ADLS Ingested Layer**.
4. **Transform** — optional aggregation / joins; produce presentation tables in **ADLS Presentation Layer**.
5. **Analyze / Report** — Databricks SQL queries and Databricks Dashboards (or Power BI direct to ADLS/Databricks SQL Warehouse).

All orchestration shown in the architecture can be implemented via ADF (Azure Data Factory) / Azure Data Factory pipelines or Databricks Jobs.

---

## Goals & Requirements

* Extract data from Ergast API and store in Parquet.
* Ingested data must have a correct schema and audit columns (ingest_ts, source, batch_id, is_current).
* Data must support incremental loads.
* Provide SQL-based analysis for **Dominant Drivers** and **Dominant Teams**.
* Create Databricks Dashboards for visualization.

---

## Prerequisites

* Azure subscription with:

  * ADLS Gen2 storage account
  * Databricks workspace
  * (Optional) Azure Data Factory
* Databricks cluster (runtime with PySpark / pandas)
* GitHub repo connected to Databricks (recommended)
* Python 3.8+ for local scripts

---

## Repository Structure

```
/ (root)
├─ README.md
├─ notebooks/
│  ├─ 01_ingest_ergast.py (Databricks notebook, PySpark)
│  ├─ 02_transform_presentation.py
│  └─ 03_analysis_sql.sql
├─ pipelines/
│  └─ adf_pipeline.json (example ADF pipeline definition)
├─ infra/
│  └─ arm_template.json (optional resources)
├─ sql/
│  ├─ dominant_drivers.sql
│  └─ dominant_teams.sql
├─ scripts/
│  └─ fetch_ergast.py (local fetch + store to ADLS)
└─ docs/
   └─ architecture.png
```

---

## Data Source — Ergast API

Ergast API provides historical Formula One data in JSON/XML. Use the endpoints for `drivers`, `constructors`, `results`, `races`, `qualifying`, etc.

**Example endpoints**:

* `http://ergast.com/api/f1/2023/results.json` (races/results by year)
* `http://ergast.com/api/f1/drivers.json`

Notes: Ergast paginates results — your ingestion logic should loop over pages until no data returned.

---

## Ingest: Requirements & Implementation

### Schema

Define explicit schema for each entity to ensure Parquet files have consistent columns. Example minimal schema for `results` table (PySpark schema):

```python
from pyspark.sql.types import StructType, StructField, IntegerType, StringType, DoubleType

results_schema = StructType([
    StructField('raceId', IntegerType(), False),
    StructField('driverId', IntegerType(), True),
    StructField('constructorId', IntegerType(), True),
    StructField('position', IntegerType(), True),
    StructField('points', DoubleType(), True),
    StructField('grid', IntegerType(), True),
    # ... add all required fields
])
```

### Audit columns (add at ingestion time)

* `ingest_ts` (timestamp when data was ingested)
* `source` (e.g., `ergast_api`)
* `batch_id` (unique id for this batch e.g., run id or timestamp)
* `is_current` (boolean, useful if maintaining slowly changing snapshot)

Add them as columns before writing to Parquet.

### Parquet & Partitioning

Write Parquet files partitioned by natural partitions to speed queries. Suggested partitions:

* `year` (race year)
* optionally `season` or `raceId` for finer partitioning

Example write:

```python
df.write.mode('append').partitionBy('year').parquet('/mnt/adls/ingested/results/')
```

> **Recommendation:** Although the requirement says Parquet, consider using Delta Lake on Databricks for easier incremental merges and ACID guarantees. If Delta is allowed, use `df.write.format('delta')` and `MERGE` for upserts.

### Incremental Load Strategy (parquet-only)

If sticking strictly to Parquet:

1. Use **partition-based incremental** loads — append only new partitions (e.g., new year or raceId).
2. Maintain a simple metadata table (a small Parquet/JSON file) that tracks last processed `raceId` or `ingest_ts`.
3. For updates to existing partitions, either rewrite the partition or switch to Delta for safe upserts.

If using Delta (recommended):

* Use `MERGE INTO` on a Delta table to upsert by `primary_key` (e.g., `raceId + driverId`).

### Sample incremental logic (pseudo)

* Query metadata store for `last_race_processed`.
* Call Ergast for races > last_race_processed.
* For each page result produce DataFrame, apply schema, add audit columns.
* Write partition (append new partition) or MERGE if Delta.
* Update metadata store with latest `raceId`.

---

## Databricks Notebooks (Suggested)

### `01_ingest_ergast.py` (PySpark notebook)

* fetch pages from Ergast API (use `requests` in a driver or run as Python job)
* parse JSON
* create DataFrame with enforced schema
* add audit columns
* write Parquet partitioned by `year`

### `02_transform_presentation.py`

* Read ingested Parquet
* Clean/normalize joins (drivers, constructors, races)
* Produce presentation tables: `race_results`, `driver_career_stats`, `team_season_stats`
* Write presentation tables to ADLS presentation layer (Parquet/Delta)

### `03_analysis_sql.sql`

Contains SQL queries used by Databricks SQL / dashboards.

---

## SQL Analysis Examples

### Dominant Drivers (by wins / points)

```sql
-- Top drivers by career wins
SELECT d.driverId, d.familyName, SUM(CASE WHEN r.position = 1 THEN 1 ELSE 0 END) AS wins,
       SUM(r.points) as total_points
FROM presentation.race_results r
JOIN presentation.drivers d ON r.driverId = d.driverId
GROUP BY d.driverId, d.familyName
ORDER BY wins DESC, total_points DESC
LIMIT 20;
```

### Dominant Teams (by championships / wins)

```sql
SELECT constructorId, constructorName,
       SUM(CASE WHEN position = 1 THEN 1 ELSE 0 END) AS wins,
       SUM(points) AS total_points
FROM presentation.race_results
GROUP BY constructorId, constructorName
ORDER BY wins DESC
LIMIT 20;
```

---

## Databricks Dashboards

1. Create Databricks SQL queries (use `dominant_drivers.sql` and `dominant_teams.sql`).
2. Save each query to the SQL editor and create visualizations: bar charts for wins, line charts for points over seasons.
3. Combine visualizations into a Databricks Dashboard and schedule refreshes.
4. Optionally expose the result via a Databricks SQL Warehouse for Power BI to directly connect.

---

## Orchestration (ADF / Databricks Jobs)

* Option A: Use Azure Data Factory pipeline to orchestrate: fetch -> run Databricks notebook -> load -> update metadata.
* Option B: Use Databricks Jobs + job clusters and schedule notebooks in order.

Example ADF pipeline tasks:

1. Web activity / Azure Function to fetch new race list.
2. ForEach activity to call Databricks notebook per race or per batch.
3. Stored procedure / Copy activity to update metadata.

---

## CI / CD

* Use GitHub Actions or Azure DevOps to push notebooks to Databricks Repos or to deploy ARM templates.
* Example: GitHub Action that lints notebooks, converts `.py` to Databricks notebook format, and calls Databricks REST API to upload.

---

## How to run locally (quickstart)

1. Clone the repo.
2. Configure environment variables for ADLS credentials and Databricks token in `~/.env`.
3. Run `python scripts/fetch_ergast.py --out /mnt/adls/raw/` to fetch raw JSON (or run inside Databricks).
4. Run the Databricks notebook `01_ingest_ergast.py` as a job to process raw JSON to Parquet.

---

## Sample Notebook Snippet — Ingestion (PySpark)

```python
import requests
from pyspark.sql import SparkSession
from pyspark.sql.functions import current_timestamp, lit

spark = SparkSession.builder.getOrCreate()

# fetch sample page
resp = requests.get('http://ergast.com/api/f1/2023/results.json?limit=1000')
data = resp.json()
# parse JSON into rows (flatten) then create DataFrame with schema
# df = spark.createDataFrame(parsed_rows, schema=results_schema)

df = df.withColumn('ingest_ts', current_timestamp()) \
       .withColumn('source', lit('ergast_api')) \
       .withColumn('batch_id', lit('20251129_01'))

# write parquet partitioned by year
(df
 .write
 .mode('append')
 .partitionBy('year')
 .parquet('/mnt/adls/ingested/results/'))
```

---

## Notes & Recommendations

* **Delta Lake**: strongly recommended for production to simplify incremental upserts and provide ACID guarantees.
* **Testing**: include unit tests for parsing logic and integration tests for end-to-end ingestion.
* **Monitoring**: add logging, Databricks job alerts, and ADF monitoring for failures.

---

## License

MIT

---

## Next steps / Enhancements

* Add Power BI sample report pointing to Databricks SQL Warehouse.
* Add more advanced analyses (driver form, head-to-head, tyre strategy effects).
* Add streaming ingestion for live telemetry (if additional source available).

---

If you want, I can also add the actual Databricks notebook `.py` files for the `notebooks/` folder and a sample `adf_pipeline.json` in the `pipelines/` folder. Let me know and I will generate them in this repo.

## A Brief Introduction to Formula 1

**Cars**: Formula 1 cars are cutting-edge, single-seat racing machines with advanced aerodynamics and powerful engines. They are designed for maximum speed and agility.

**Races**: F1 races take place on a variety of tracks, including purpose-built circuits and temporary street circuits in cities around the world.

**Teams**: Multiple teams, each with two drivers, compete in the championship. Prominent teams include Mercedes, Ferrari, Red Bull Racing, and McLaren.

**Drivers**: F1 attracts some of the world's best racing talents, and drivers like Lewis Hamilton, Sebastian Vettel, and Max Verstappen have become household names.

**Points and Championships**: Drivers and teams earn points based on their performance in each race. At the end of the season, the driver with the most points wins the Drivers' Championship, and the team with the most points wins the Constructors' Championship.

## How does the Formula 1 Season Work

A Formula 1 championship season refers to a specific year in which a series of Formula 1 races are held, and points are accumulated by drivers and teams to determine the champions in various categories. Here's an explanation of how a Formula 1 championship season works:

**Race Calendar**: Each Formula 1 season typically consists of a calendar of races, known as Grand Prix events. These races are held at various locations around the world, ranging from traditional circuits to temporary street tracks.

**Teams and Drivers**: Multiple teams participate in the championship, with each team fielding two drivers. These drivers compete throughout the season to earn points for themselves and their teams.

**Points System**: Formula 1 uses a points system to determine the championship standings. The points are awarded to drivers based on their finishing positions in each race. The points system can vary slightly over the years, but a common system is to award points to the top 10 finishers, with the winner earning the most points (e.g., 25 points) and the 10th-place finisher earning the fewest (e.g., 1 point).

**Drivers' Championship**: The primary focus of a Formula 1 season is the Drivers' Championship. Drivers accumulate points from each race throughout the season. The driver with the most points at the end of the season is crowned the Drivers' Champion and often receives the prestigious "World Champion" title.

**Constructors' Championship**: In addition to the Drivers' Championship, there is also a Constructors' Championship. This championship considers the combined points earned by both drivers of each team. The team with the most points at the end of the season wins the Constructors' Championship.

## Project Requirements

### Data Ingestion 

- Extract Data from the Ergast API.

- Ingested Data must have the correct schema applied.

- Ingested Data must have the audit columns.

- Ingested data must be in the Parquet Format.

- Analyse the ingested data via SQL

- Ingested Data must be able to handle incremental load.

### Data Transformation

- Join key information required to report anything.

- Transformed Data must be stored in column format.

### Analysis Requirements

- Dominant Drivers

- Dominant Teams

- Create Databricks Dashboards

### Scheduling Requirements

- Schedule to run at 10pm every Friday

- Ability to monitor pipelines.

- Rerun failed pipelines.

- Set up Alerts on Failures.

## Solution Architecture

<img src = "https://github.com/vedanthv/data-engineering-projects/blob/main/formula-1-analytics-engg/static/formula1-solution-architecture.png">


### Visualizations

#### Dominant Drivers

![image](https://github.com/vedanthv/data-engineering-portfolio/assets/44313631/aef0dacf-3cd8-494d-b1d7-e2249a5f7652)

### Dominant Teams

![image](https://github.com/vedanthv/data-engineering-portfolio/assets/44313631/3995a206-950d-470b-a525-8f1845ed7cca)

