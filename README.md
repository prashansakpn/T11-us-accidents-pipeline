# Road Accident Risk Pipeline (Team 11)

A Databricks lakehouse pipeline that turns 7.7 million raw US road-accident records into clean, trusted tables for the pricing team of a US auto insurer.

**Course:** Data Engineering and Big Data Analytics, PGDM Business Analytics
**Client brief:** Accenture, Data Engineer (Hyderabad)
**Team 11:** Prashansa Naomi Konduru (BA05042), Sandeep Thummalapalli (BA05052), Lakkineni Roshith (BA05032), Leira George (BA05062), Chrislynn Sabu (BA05017), Robin Miranda (BA05048)

## Business problem
An auto insurer prices premiums mainly on the driver's age, car and claim history, and does not account well for where and when customers drive. This pipeline produces clean tables that show which regions, hours and conditions carry the highest share of severe accidents.

Business questions (answered by the gold tables in Checkpoint 4):
1. Which states and cities have the most accidents, and what share of them are severe (severity 3–4)?
2. At which hours, weekdays and months do accidents peak?
3. How do weather and road features change the share of severe accidents?

## Dataset
- **US Accidents (2016–2023)** by Sobhan Moosavi et al., on [Kaggle](https://www.kaggle.com/datasets/sobhanmoosavi/us-accidents)
- 7,728,394 records, 46 columns, 49 US states, February 2016 to March 2023, one CSV of about 3 GB
- **Licence:** CC BY-NC-SA 4.0, for non-commercial learning use only
- **Citation:** Moosavi, S., Samavatian, M. H., Parthasarathy, S., & Ramnath, R. (2019). *A Countrywide Traffic Accident Dataset.* arXiv:1906.05409
- The data itself is **not** stored in this repository (3 GB). Download it from Kaggle.

## Pipeline

```
Kaggle zip (685 MB) → Unity Catalog Volume → BRONZE (raw Delta) → SILVER (clean Delta) → quality log → GOLD (Checkpoint 4) → dashboard
```

| Layer | Table | What it holds |
|---|---|---|
| Bronze | `workspace.accidents.bronze_accidents` | Raw CSV as loaded, every column as text, plus `_ingest_ts` and `_source_file` |
| Silver | `workspace.accidents.silver_accidents` | Typed columns, parsed timestamps, year/month/weekday/hour, duration, 8 weather categories, severe flag |
| Silver | `workspace.accidents.silver_rejects` | Rows that fail validity rules, each with a reason |
| Quality | `workspace.accidents.dq_log` | Row counts, null rates and rule checks for every run |

## Results of the full run (Checkpoint 3)
| Check | Result |
|---|---|
| Bronze rows | 7,728,394 (matches the published count) |
| Duplicate IDs removed | 0 |
| Rows rejected | 0 |
| Silver rows | 7,728,394 (rows in = duplicates + rejects + silver) |
| Null % precipitation / wind speed / visibility / humidity / weather / temperature | 28.51 / 7.39 / 2.29 / 2.25 / 2.24 / 2.12 |

## How to run
1. Download the dataset zip from Kaggle.
2. In a notebook, run:
   ```sql
   %sql
   CREATE SCHEMA IF NOT EXISTS workspace.accidents;
   CREATE VOLUME IF NOT EXISTS workspace.accidents.raw_files;
   ```
3. Upload the zip to the `raw_files` volume (Catalog → workspace → accidents → raw_files → Upload). Unzip it inside Databricks with Python's `zipfile` (see the setup notes in the notebook).
4. Import `notebooks/T11_CP3_Notebook.ipynb`, connect to **Serverless**, and click **Run all** with `SAMPLE_FRACTION = 0.01`.
5. When it works, set `SAMPLE_FRACTION = 1.0` and run from "Step 1: working set" downwards.

## Repository contents
- `notebooks/T11_CP3_Notebook.ipynb`: bronze, silver and quality checks (Checkpoint 3)
- `docs/`: checkpoint submissions (project plan, Agile write-up)



