# Production-Grade Batch Pipeline — Weather Data (Snowflake + dbt + Airflow)

An automated data pipeline that pulls real-time weather data from a public API, loads it into Snowflake, transforms it through layered dbt models, and orchestrates the entire workflow with Apache Airflow. Built to mirror how modern data teams actually run batch pipelines in production — with proper data modeling, automated testing, and failure handling.

---

## What This Project Does

1. Pulls hourly weather data from the [Open-Meteo API](https://open-meteo.com/) (free, no API key required) for several US cities
2. Loads raw JSON responses into a Snowflake staging table
3. Transforms the raw data through three dbt model layers — staging, intermediate, and mart
4. Runs automated data quality tests at each layer to catch bad data before it reaches the final tables
5. Orchestrates the entire pipeline with an Airflow DAG that runs on a schedule, retries on failure, and sends an alert if something breaks

---

## Why I Built This

A lot of student projects stop at "load data, run a query." This project goes further by showing the full production pipeline pattern: ingestion, transformation, testing, and orchestration. The dbt layered architecture is the industry standard at companies using Snowflake, and Airflow is one of the most common orchestration tools in the field. I wanted to show I understand not just how to analyze data, but how to build a reliable system that delivers clean data on a schedule without manual intervention.

---

## Architecture

```
Open-Meteo API (free weather data — no key required)
        |
        v
Python ingestion script (requests + Snowflake connector)
        |
        v
Snowflake RAW schema (raw JSON, no transformations)
        |
        v
dbt STAGING layer (type casting, renaming, light cleaning)
        |
        v
dbt INTERMEDIATE layer (business logic, aggregations)
        |
        v
dbt MART layer (final analytics-ready tables)
        |
        v
Airflow DAG (schedules, retries, failure alerting)
```

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| Python | Ingestion script — hits API, loads to Snowflake |
| Open-Meteo API | Free weather data source (no API key) |
| Snowflake (free trial) | Cloud data warehouse |
| dbt Core | Transformation layer — SQL models + testing |
| Apache Airflow (Docker) | Pipeline orchestration and scheduling |
| Docker | Runs Airflow locally without cloud costs |

---

## dbt Model Architecture

This is the part of the project I'm most proud of. Instead of one big transformation query, dbt encourages you to break transformations into layers where each one has a clear job:

```
RAW (Snowflake)
└── STAGING (dbt)
    Clean the raw data: cast types, rename columns,
    filter out nulls, standardize values.
    No business logic here — just clean inputs.
        |
        v
    INTERMEDIATE (dbt)
    Apply business logic: calculate derived metrics,
    join related tables, build reusable aggregations.
        |
        v
    MART (dbt)
    Final analytics-ready tables. These are what
    a BI tool or analyst would actually query.
```

### dbt Tests Added

```yaml
# Every column that shouldn't be null is tested
- not_null: [city, temperature_c, recorded_at]

# Values must be within realistic ranges
- accepted_values:
    column: weather_condition
    values: ['clear', 'cloudy', 'rain', 'snow', 'fog']

# No duplicate readings for the same city + timestamp
- unique:
    combination_of: [city, recorded_at]

# Source freshness check — fail if data is >2 hours old
sources:
  freshness:
    warn_after: {count: 1, period: hour}
    error_after: {count: 2, period: hour}
```

---

## Airflow DAG

The DAG runs every hour and orchestrates four tasks in sequence:

```
ingest_weather_data
        |
        v
dbt_run (transforms raw → staging → intermediate → mart)
        |
        v
dbt_test (runs all data quality tests)
        |
        v
notify_on_failure (sends alert if any task fails)
```

Retry logic: each task retries 3 times with a 5-minute delay before marking as failed. This handles transient issues like API timeouts or brief network interruptions without requiring manual intervention.

---

## Project Structure

```
weather-batch-pipeline/
├── README.md
├── requirements.txt
├── docker-compose.yml          # Spins up Airflow locally
├── ingestion/
│   └── load_weather.py         # API pull + Snowflake load script
├── dbt_project/
│   ├── dbt_project.yml
│   ├── models/
│   │   ├── staging/
│   │   │   └── stg_weather_raw.sql
│   │   ├── intermediate/
│   │   │   └── int_weather_daily.sql
│   │   └── mart/
│   │       └── mart_weather_summary.sql
│   ├── tests/
│   │   └── schema.yml          # Column-level tests
│   └── sources.yml             # Source freshness config
└── airflow/
    └── dags/
        └── weather_pipeline.py # Airflow DAG definition
```

---

## How to Run

```bash
# Clone the repo
git clone https://github.com/yourusername/weather-batch-pipeline
cd weather-batch-pipeline

# Install Python dependencies
pip install -r requirements.txt

# Set Snowflake credentials as environment variables
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password

# Run the ingestion script manually to test
python ingestion/load_weather.py

# Run dbt transformations
cd dbt_project
dbt run
dbt test

# Start Airflow locally via Docker
cd ..
docker-compose up airflow-init
docker-compose up
# Open http://localhost:8080 to see the Airflow UI
```

---

## Key Learnings

- **Why dbt layers matter**: Putting all your SQL in one giant query is hard to debug and impossible to test. Splitting into staging, intermediate, and mart means each layer has one job, and when something breaks you know exactly where to look.
- **Data quality testing**: Before this project I thought testing was optional. After watching dbt catch a batch of readings where the API returned null temperatures during a brief outage, I understand why schema tests are non-negotiable in production pipelines.
- **Airflow retry logic**: Without retries, a single API timeout would fail the entire daily run and require manual intervention. Three retries with a 5-minute delay handles the vast majority of transient failures automatically.
- **Snowflake free tier**: The 30-day free trial with $400 in credits is more than enough to build and test a pipeline like this. I used approximately $3 of credits across the entire development process.

---

## Cost

- Snowflake: free 30-day trial ($400 credits — used ~$3)
- Open-Meteo API: completely free, no rate limits at this volume
- Airflow: runs locally via Docker at no cost

---

*Tools: Python · Snowflake · dbt Core · Apache Airflow · Docker · Open-Meteo API*
