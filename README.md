# COVID-19 Mortality & Risk Factor Analysis — Mexico

> End-to-end epidemiological analysis of COVID-19 in Mexico combining official SSA surveillance data (2025) with a historical Kaggle dataset (2020–2021). Covers ETL, feature engineering, CFR modelling, and a three-page interactive Power BI dashboard.

  


## Dashboard Preview

### Executive View — KPIs & Temporal Evolution

<img width="2428" height="1364" alt="dashboard_ejecutivo" src="https://github.com/user-attachments/assets/baa1a502-fd2d-4a06-a6b0-64db1480855f" />


### State-Level Analysis

<img width="2428" height="1368" alt="dashboard_estados" src="https://github.com/user-attachments/assets/d1eb02d3-d07b-44de-8f66-984cbb19b9b1" />


### Demographic Analysis

<img width="2424" height="1362" alt="dashboard_demográfico" src="https://github.com/user-attachments/assets/df42fc0f-b964-4c5a-a908-bd760e51cdcd" />


  


## Key Findings


| Metric                         | Value                                 |
| ------------------------------ | ------------------------------------- |
| Confirmed Cases (2025 dataset) | **137,030**                           |
| Hospitalized Patients          | **62,664**                            |
| Intubated Patients             | **3,000**                            |
| Intubation Rate                | **4.69%**                             |
| Global CFR                     | **3.71%**                             |
| State with Highest CFR         | **Guerrero — 9.04%**                  |
| State with Most Cases          | **Ciudad de México — 23.4K**          |
| Highest-Risk Age Group         | **60+ years — CFR 10.31%**            |
| Highest-Risk Comorbidity       | **Chronic Renal Disease — CFR 26.6%** |
| Average Patient Age            | **36 years**                          |
| Patients with 3+ Comorbidities | **9,000**                            |


  


## Project Structure

```
covid19-mortality-analysis/
│
├── dashboard/
│   ├── Dashboard Covid-19 México 2025.pbix # Main dashboard with the information
│
├── data/
│   ├── raw/                          # Source datasets
│   │   ├── COVID19MEXICO.csv         # Official SSA dataset — ~137K records (updated Nov 2025)
│   │   └── Covid Data.csv            # Kaggle historical dataset — 1.05M records (2020–2021)
│   │
│   └── processed/                    # ETL outputs consumed by Power BI
│       ├── fact_covid_2025.csv       # Main fact table for the dashboard
│       ├── dim_estado.csv            # State dimension (INEGI codes + ISO codes)
│       ├── cfr_by_state_final_for_map.csv
│       ├── cfr_by_age_group.csv
│       ├── top5_comorbidities.csv
│       ├── intubation_comparison.csv
│       ├── cases_deaths_time_series.csv
│       └── deaths_by_comorbidity_count.csv
│
├── screenshots/                           # Dashboard screenshots for this README
│   ├── dashboard_ejecutivo.png
│   ├── dashboard_estados.png
│   └── dashboard_demografico.png
│
├── notebooks/
├   ├──Covid-19_Análisis.ipynb           # Main analysis notebook
└── README.md
```

  


## Tech Stack


| Layer               | Tools                                    |
| ------------------- | ---------------------------------------- |
| Data Processing     | Python 3, Pandas, NumPy                  |
| Visualisation (EDA) | Plotly Express                           |
| BI Dashboard        | Power BI Desktop (Bing Maps, custom DAX) |
| Geospatial          | INEGI state codes + ISO 3166-2:MX codes  |


  


## Datasets

### Dataset 1 — Official SSA Surveillance (`COVID19MEXICO.csv`)

- **Source:** Mexico's Ministry of Health (Secretaría de Salud)
- **Records:** 137,000
- **Updated:** November 18, 2025
- **Scope:** Full granularity — state of residence, symptom onset date, admission date, age, all comorbidities, hospitalization, intubation, and death
- **Role:** Sole fact table driving the Power BI dashboard and all its date/state/age filters

### Dataset 2 — Historical Kaggle (`Covid Data.csv`)

- **Source:** [Kaggle — COVID-19 Mexico Dataset](https://www.kaggle.com/)
- **Records:** 1,048,575 (first pandemic wave, 2020–2021)
- **Limitations:** No residence state or symptom onset date columns; row count falls exactly at Excel's limit — potential truncation
- **Role:** Cleaned and exported separately as `historical_2020_2021_clean.csv`; intentionally **not merged** into the main fact table (see notebook Section 2.2 for the full rationale)

> **Why the datasets are kept separate:** Merging them caused KPI cards for Hospitalized and Intubated to silently include the 1.04M historical rows (which have no date or patient status), while Confirmed Cases remained filtered correctly. The two universes responded differently to the same dashboard filters, producing silent inconsistencies.

  


## Notebook Walkthrough (`Covid-19_Análisis.ipynb`)

### 0 — Configuration & Mappings

Sets up reproducible paths (`PROJECT_ROOT / data / raw|processed`), the official SSA numeric-to-name state dictionary for all 32 states + CDMX, the INEGI-to-ISO 3166-2:MX code mapping (required by Power BI's Bing Maps choropleth), and two column-translation dictionaries (Spanish → English for the SSA dataset; English → standardised English for the Kaggle dataset).

### 1 — Data Loading & Memory Optimisation

Reads both CSVs with `dtype=category` on high-cardinality string columns (sector, entity codes, nationality) to reduce in-memory footprint before transformation.

### 2 — Harmonisation, Cleaning & Unification

**2.1 Source-aware date parsing** — `df1` dates are in ISO `YYYY-MM-DD`; `df2` dates are in `DD/MM/YYYY` with `9999-99-99` as a sentinel for "alive". Parsing happens independently before any merge to avoid pandas guessing a single ambiguous format across both formats.

**2.2 Sentinel code cleanup** — SSA uses codes `97` (not applicable), `98` (unknown), and `99` (not specified) in binary comorbidity columns. All three are replaced with `NaN` before boolean analysis.

**2.3 Scope decision** — `df1` becomes `df_recent` (the Power BI fact table); `df2` becomes `df_historical` (exported separately). See the note above.

### 3 — Feature Engineering

- `DECEASED` — binary flag derived from whether `DEATH_DATE` is non-null
- `AGE_GROUP` — four cohorts: `0-17`, `18-35`, `36-59`, `60+`
- `NUM_COMORBIDITIES` — count of active comorbidities per patient across 10 tracked conditions
- `COMORBIDITY_GROUP` — binned into `0`, `1`, `2`, `3+` comorbidities

### 4 — CFR Metrics


| Section                                    | Output                                              |
| ------------------------------------------ | --------------------------------------------------- |
| 4.1 CFR by State                           | `cfr_by_state_final_for_map.csv` + `dim_estado.csv` |
| 4.2 CFR by Age Cohort                      | `cfr_by_age_group.csv`                              |
| 4.3 Specific CFR by Comorbidity            | `top5_comorbidities.csv`                            |
| 4.4 Intubation Rate                        | `intubation_comparison.csv`                         |
| 4.5 Time Series (daily cases, deaths, CFR) | `cases_deaths_time_series.csv`                      |
| 4.6 CFR by Comorbidity Count               | `deaths_by_comorbidity_count.csv`                   |


### 5 — Master Export

Exports a clean `fact_covid_2025.csv` containing only the columns needed by Power BI, keeping the file size minimal and the data model explicit.

### 6 — Validation

Shape check, `df.info()`, and a column audit to confirm all expected fields are present before connecting the CSVs to Power BI.

  


## Dashboard Pages

### Page 1 — Ejecutivo (Executive)

Global KPI cards (Confirmed Cases, Hospitalized, Intubated, Intubation Rate, Global CFR), a time-series area chart of daily cases vs. deaths (Jan–Nov 2025), a choropleth map of CFR by state, a horizontal bar chart of CFR by the top 5 comorbidities, and a bar chart of CFR by age group. Fully filterable by date range, age group, state, and sex.

### Page 2 — Análisis por Estado (State Analysis)

Highlights the state with the highest CFR (Guerrero, 9.04%), the state with the most cases (CDMX, 23.4K), and the most hospitalizations (CDMX, 7,992). Includes a Top 10 CFR ranking bar chart, a Top 10 cases ranking bar chart, and a detailed table showing cases, deaths, hospitalizations, and CFR for the 10 highest-mortality states. National CFR average card: **3.71%**.

### Page 3 — Análisis Demográfico (Demographic Analysis)

A combined bar + line chart showing case volume and CFR curve across individual ages (0–100+). A donut chart of case distribution by sex (Women 55.63% / Men 44.37%), with Women having a higher CFR (4.72%). A horizontal bar chart of the Top 5 comorbidity-specific CFRs: Chronic Renal (26.6%), COPD (23.2%), Diabetes (21.0%), Cardiovascular (19.7%), Hypertension (18.5%). Summary KPIs: age group 60+ leads CFR at 10.31%; average patient age is 36.

  


## How to Run

**1. Clone the repo**

```bash
git clone https://github.com/WillyyyFernandez/covid19-mortality-analysis.git
cd covid19-mortality-analysis
```

**2. Install dependencies**

```bash
pip install pandas numpy plotly jupyter
```

**3. Add the raw datasets**

Place both source files inside `data/raw/` (not included in the repo due to size):

- `COVID19MEXICO.csv` — download from [datos.gob.mx](https://datos.gob.mx/busca/dataset/informacion-referente-a-casos-covid-19-en-mexico)
- `Covid Data.csv` — download from Kaggle

**4. Run the notebook**

```bash
jupyter notebook "Covid-19_Análisis.ipynb"
```

Run all cells sequentially. Processed CSVs will be written to `data/processed/`.

**5. Connect to Power BI**

Open Power BI Desktop and point the data sources to the files in `data/processed/`. The star schema uses `fact_covid_2025.csv` as the fact table and `dim_estado.csv` as the state dimension (join key: `ENTIDAD_RES` ↔ `State_Code`).

  


## Notes & Caveats

- The 2025 dataset reflects confirmed COVID-19 cases registered under Mexico's USMER epidemiological surveillance network and may under-represent the true case count.
- `DEATH_DATE` being null is used as the proxy for "survived" — records with an unknown outcome are treated as alive, which may slightly underestimate CFR.
- Binary comorbidity columns encoded as `97`, `98`, or `99` by SSA are treated as missing (`NaN`) rather than as "not present," which may slightly deflate comorbidity-specific CFRs.
- The Kaggle dataset rows cap at 1,048,575 (Excel's row limit). It is recommended to verify against the original source that no records were truncated during export.

  


## Author

**William Fernández**
Data Engineering & AI student | Mexico

[GitHub](https://github.com/WillyyyFernandez)
