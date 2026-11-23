# Road Accident Risk Pipeline (Team 11)

A Databricks lakehouse pipeline that turns 7.7 million raw US road-accident records into clean, trusted tables, an EDA dashboard and two machine-learning models for the pricing team of a US auto insurer.

**Course:** Data Engineering and Big Data Analytics, PGDM Business Analytics
**Client brief:** Accenture, Data Engineer (Hyderabad)
**Team 11:** Prashansa Naomi Konduru (BA05042), Sandeep Thummalapalli (BA05052), Lakkineni Roshith (BA05032), Leira George (BA05062), Chrislynn Sabu (BA05017), Robin Miranda (BA05048)

## Business problem
An auto insurer prices premiums mainly on the driver's age, car and claim history, and barely accounts for where and when customers drive. We built the data pipeline that lets its pricing team answer three questions:

1. **Where?** Which states and cities have the most accidents, and what share of them are severe (severity 3–4)?
2. **When?** At which hours, weekdays and months do accidents peak?
3. **In what conditions?** How do weather and road features change the share of severe accidents?

## Dataset
- **US Accidents (2016–2023)** by Sobhan Moosavi et al., on [Kaggle](https://www.kaggle.com/datasets/sobhanmoosavi/us-accidents)
- 7,728,394 records, 46 columns, 49 US states, January 2016 to March 2023, one CSV of about 3 GB (685 MB zipped)
- **Licence:** CC BY-NC-SA 4.0, for non-commercial learning use only
- **Citation:** Moosavi, S., Samavatian, M. H., Parthasarathy, S., & Ramnath, R. (2019). *A Countrywide Traffic Accident Dataset.* arXiv:1906.05409; and *Accident Risk Prediction based on Heterogeneous Sparse Data.* arXiv:1909.09638
- The raw data is **not** stored in this repository. Download it from Kaggle.

## Pipeline

```
Kaggle zip ─► Unity Catalog Volume ─► BRONZE ─► SILVER ─► GOLD ─► Dashboard + ML models
                                        │          │         │            (tracked in MLflow)
                                        └──────────┴─────────┴──► dq_log (quality checks on every run)
```

| Notebook | Layer | What it does |
|---|---|---|
| `T11_01_Bronze_Silver.ipynb` | Bronze, Silver | Loads the raw CSV as-is (bronze); types columns, parses mixed date formats, adds time features and 8 weather groups, removes duplicates, separates invalid rows (silver); logs quality checks |
| `T11_02_Gold_EDA.ipynb` | Gold | Clusters silver by year and state; builds 7 gold tables; checks they reconcile to silver; draws 6 EDA charts |
| `T11_03_ML_Models.ipynb` | Models | Model A: severity risk (classification). Model B: monthly accident volume per state (regression). Metrics saved to `gold_model_metrics`, runs logged in MLflow |

The three notebooks are set up as one Databricks Job, **US Accident Job Flow**: three serverless tasks in a chain (`bronze_silver` → `Gold_Eda` → `ML_Models`), each starting only when the one before it succeeds.

### Tables (`workspace.accidents`)
| Layer | Tables |
|---|---|
| Bronze | `bronze_accidents` |
| Silver | `silver_accidents`, `silver_rejects` |
| Gold | `gold_kpis`, `gold_state_summary`, `gold_city_summary`, `gold_time_patterns`, `gold_weather_severity`, `gold_road_feature_severity`, `gold_monthly_state_volume`, `gold_ml_severity_sample`, `gold_model_metrics` |
| Quality | `dq_log` |

### Dashboard
Databricks AI/BI dashboard **Team 11 · US Accidents EDA**, built on 7 SQL queries over the gold tables: headline numbers, accidents per year, hour × weekday heatmap, top 15 states, severe share by weather, and the Model A and Model B test metrics. A two-page export is in `T11_CP4_Dashboard.pdf`.

## Key results
**Data quality:** 7,728,394 rows in and out of silver; 0 duplicate IDs; 0 rows failing validity rules; all 5 gold reconciliation checks PASS. Missing values kept as blanks: precipitation 28.51%, wind speed 7.39%, other weather fields about 2%.

**Performance and tracking:** clustering silver by year and state took the same test query (California, 2022) from 4.79 s to 0.79 s. All 4 model runs are logged in MLflow.

**Findings**
- 19.46% of accidents are severe (1,504,047 of 7,728,394).
- California has the most accidents (1,741,433), but New York (23.2%), Virginia (22.9%) and Texas (21.9%) have higher severe shares.
- Accidents peak in weekday rush hours (7–8 am and 3–5 pm); Tuesday 7 am is the busiest hour (118,920 accidents).
- Rain has the highest severe share (23.1% vs 19.5% overall). Junctions raise the severe share by 7.9 points; crossings lower it by 14.0 points.

**Model A: severity risk** (test set of 153,765 accidents)

| Model | ROC-AUC | PR-AUC | Precision | Recall | F1 |
|---|---|---|---|---|---|
| Always "not severe" (baseline) | 0.500 | 0.195 | 0.000 | 0.000 | 0.000 |
| Logistic regression | 0.736 | 0.391 | 0.341 | 0.676 | 0.453 |
| Random forest | **0.743** | **0.409** | 0.387 | 0.578 | **0.464** |

Training ROC-AUC 0.748 vs test 0.743, so no overfitting. The top input is whether precipitation was recorded, which is a data-collection effect, not a cause of severity.

**Model B: monthly accident volume per state** (tested on April 2022 to March 2023)

| Model | R² | MAE (accidents per state-month) |
|---|---|---|
| Naive: next month = this month | 0.843 | 873.4 |
| Random forest | 0.837 | 859.3 |
| Linear regression | **0.849** | **841.3** |

MAPE is not used to judge the forecast: states with very few accidents in some months turn small misses into huge percentages.

## How to run (Databricks Free Edition)
1. Download the dataset zip from Kaggle.
2. In a notebook, run:
   ```sql
   %sql
   CREATE SCHEMA IF NOT EXISTS workspace.accidents;
   CREATE VOLUME IF NOT EXISTS workspace.accidents.raw_files;
   ```
3. Upload the zip to the `raw_files` volume (Catalog → workspace → accidents → raw_files → Upload) and unzip it inside Databricks with Python's `zipfile`.
4. Import the three notebooks and connect to **Serverless**.
5. Run `T11_01_Bronze_Silver` with `SAMPLE_FRACTION = 0.01` to test, then `1.0` for the full run. Then run `T11_02_Gold_EDA`, then `T11_03_ML_Models`.
6. Or run the Databricks Job: three notebook tasks in that order, on serverless compute.

All samples and splits use a fixed seed (42), so re-runs give the same results.

## Repository contents
| File | What it is |
|---|---|
| `T11_01_Bronze_Silver.ipynb` | Notebook 01, with outputs of the full run |
| `T11_02_Gold_EDA.ipynb` | Notebook 02, with outputs |
| `T11_03_ML_Models.ipynb` | Notebook 03, with outputs |
| `figures/figures/` | The 14 charts produced by notebooks 02 and 03 |
| `T11_CP4_Presentation.pdf` | Project presentation |
| `T11_CP4_Dashboard.pdf` | Export of the Databricks EDA dashboard |
| `T11_CP4_Governance_Note.pdf` | One-page data governance note |
| `T11_CP3_Agile_Writeup.pdf` | Checkpoint 3: Agile approach and Sprint 3 retrospective |
| `T11_CP2_Plan.pdf` | Checkpoint 2: project plan |

## Limitations
- The data counts accidents, not traffic volume, so we compare severe shares rather than raw risk between places or hours.
- Recorded accidents rise every year while the severe share falls sharply, which points to changes in reporting rather than safer roads.
- Severity measures impact on traffic, not injuries. 2023 holds only January to March.
