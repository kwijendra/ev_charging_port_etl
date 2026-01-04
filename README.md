# EV Charging Points in Central London – Data Engineering Take-Home Assignment

## Overview
This repository contains ETL solution for data on EV charging points in Central London.

The project:
- Sources public EV charging infrastructure data
- Automates ingestion into BigQuery using Airflow DAG
- Applies transformations and data modelling
- Enables analytics and dashboarding for key business insights

---

## 1. Data Sourcing

### Dataset: Open Charge Map
- **Source:** Open Charge Map (OCM) Public API
- **Access:** REST API (JSON)
- **Project Repository & Documentation:** https://openchargemap.org/site/develop/api, https://github.com/openchargemap/ocm-docs/blob/master/Model/schema/ocm-openapi-spec.yaml

### Justification
**Strengths**
- Publicly available and well-documented
- Good coverage of EV charging infrastructure in London
- Regularly updated by operators and community contributors
- Provides key attributes such as:
  - Latitude / longitude
  - Operator / network
  - Charger power ratings
  - Operational status

**Limitations**
- Community-contributed data can contain inconsistencies
- Nested JSON structure requires careful parsing
- No official boundary for “Central London”

Despite these limitations, Open Charge Map is well-suited for building automated, analytics-focused data pipelines.

---

## 2. Data Pipeline

### Architecture
Open Charge Map API
↓
Apache Airflow (Cloud Composer)
↓
Python ETL
↓
BigQuery (Curated Analytics Table)


### Technology Choices
- **Python** for data extraction and transformation
- **Apache Airflow (Cloud Composer)** for orchestration and scheduling
- **BigQuery** for analytics-ready storage

---

## 2.1 Extract
- Data is programmatically fetched from the Open Charge Map API using API Key (API key is stored securely in GCP Secret Manager which is not implemented)
- Extraction is triggered by an Airflow DAG
- Pipeline is designed to run on a daily schedule

---

## 2.2 Transform

For this project, data is extracted considering a 5km radius from the central london lattitude and longitude location. Central London is approximated using a latitude/longitude bounding box derived from commonly recognised central areas.

**Bounding box used:**
- Latitude: `51.48 → 51.54`
- Longitude: `-0.15 → 0.02`

This method is:
- Simple and deterministic
- Easy to document and reproduce
- Commonly used in analytics pipelines

---

### Handling Nested JSON
The Open Charge Map API returns nested JSON objects.

- **AddressInfo**
  - Used to extract latitude, longitude, and location metadata

- **Connections**
  - Array describing individual charging connectors and their power ratings

Defensive access patterns are used to safely handle missing or null nested fields.

---

### Charger Speed Classification
Charger speeds are derived from connector `PowerKW` values using UK government definitions:

| Charger Speed | Power Range |
|--------------|-------------|
| Slow | 3–6 kW |
| Fast | 7–22 kW |
| Rapid | 25–100 kW |
| Ultra-Rapid | 100+ kW |

Source: UK Government EV Charging Infrastructure Statistics (January 2024)
https://www.gov.uk/government/statistics/electric-vehicle-charging-device-statistics-january-2024/electric-vehicle-public-charging-infrastructure-statistics-january-2024

---

### Data Standardisation
The curated dataset ensures:
- A unique identifier per charging point
- Latitude and longitude are present
- Standardised operator / network names
- Charger speed category
- Operational status (operational vs non-operational)
- Sensible defaults for missing or inconsistent values

---

## 2.3 Load (BigQuery)

### Target Table
ev_charging.central_london_chargers

**Design highlights**
- Analytics-ready denormalised schema
- Partitioned by `data_date` and write data on overwrite mode
- Optimised for BI tools and SQL analysis by adding clusters for operator and charger_speed

### Automation
- The Airflow DAG runs daily at 02:00 UTC
- Each run loads data for `execution_date` as `data_date`

Cron schedule:
0 2 * * *

---

## 3. Setup Instructions

### Prerequisites
- GCP project with BigQuery enabled
- Cloud Composer environment
- Open Charge Map API key
- BigQuery dataset created

### 1. Create BigQuery Objects
Run the SQL scripts in the `sql/` directory:
- `create_dataset.sql`
- `create_table.sql`

### 2. Configure Secrets
- Store the Open Charge Map API key in GCP Secret Manager. (Currently hardcoded in the code)
- Grant the Cloud Composer service account access to the secret

### 3. Deploy Code
- Place the Airflow DAG in the `dags/` directory
- Place ETL logic in the `etl/` directory
- Airflow will automatically detect and schedule the DAG

---

## 4. Analytics & Insights

The curated BigQuery table supports several meaningful analyses as specified in the EV_Charging_Port_-_Central_London.pdf dashboard.
