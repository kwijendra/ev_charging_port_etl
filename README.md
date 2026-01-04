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
Few key findings and insights from the analysis are as below.

1: Central London has high charger availability, but speed is a bottleneck

Although over 50% of charging points are operational, a large proportion of chargers fall into the “slow” or “unknown speed” categories, while rapid and fast chargers form a much smaller share that indicates that infrastructure availability has grown faster than infrastructure quality. For urban areas like Central London the requirement could be for quick top ups rather than overnight charging and the lack of rapid chargers could limit EV adoption.

2: Market concentration among a few operators

The data shows that a small number of operators control a significant share of charging points in Central London.

3: Data quality issues 

A high percentage of charging points are marked as “unknown” in both status accounting for over 40% of records and this can negatively impact EV route planning apps, Consumer trust and City-level transport planning. Improving data completeness may be as impactful as installing new chargers.

An important data quality issue identified in the analysis is the lack of standardization in categorical fields, particularly the charging point status, where semantically identical values such as “unknown” and “Unknown” are treated as separate categories. This type of inconsistencies leads to inaccurate aggregations, misleading visualizations, and reduced reliability of analytical insights. Such issues can be effectively addressed at the ETL transformation stage by applying categorical normalization techniques, including case standardization, controlled value mappings, and validation rules. Addressing this during ETL ensures cleaner, more consistent datasets, reduces downstream data cleansing effort in BI tools, and improves the overall robustness of analytics and decision-making.
