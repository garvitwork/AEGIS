# 🛰️ AEGIS — AI-Driven Planetary Defense Intelligence Platform

**An end-to-end data engineering + AI + BI project that prioritizes which near-Earth asteroids need urgent re-observation, using real NASA/JPL data, MySQL, Python ML, and Power BI.**

![Status](https://img.shields.io/badge/status-complete-brightgreen) ![Power BI](https://img.shields.io/badge/PowerBI-Dashboard-yellow) ![Python](https://img.shields.io/badge/Python-ETL%20%2B%20ML-blue) ![MySQL](https://img.shields.io/badge/MySQL-Database-orange)

---

## Problem Statement

Planetary defense agencies (NASA's PDCO, ESA's NEOCC) track over 35,000 known Near-Earth Objects (NEOs), but **telescope observation time is scarce and expensive**. Not every asteroid can be re-observed regularly, yet many objects — including potentially hazardous ones — have thin, outdated orbital data (short observation arcs, few tracked positions). An object with an uncertain orbit is precisely the kind that could surprise us.

There is currently no simple, data-driven decision-support tool that combines:
- Impact risk (JPL Sentry risk list)
- Orbital uncertainty (observation arc length, number of observations)
- Orbital anomaly (unusual orbital geometry worth scientific attention)

...into a single **prioritized observation watchlist**. This project builds exactly that: a system that tells you *which asteroids most urgently need telescope time*, and why.

This is a real, active problem discussed in the planetary defense and astrodynamics community — not a solved Kaggle dataset.

---

##  Project Goal

Build a full pipeline — from raw NASA/JPL API data to a polished, interactive Power BI report — that:
1. Ingests real close-approach, orbital, and risk data for NEOs
2. Cleans and models it into a proper relational (star) schema in MySQL
3. Applies machine learning (anomaly detection + a weighted composite risk score) to surface which objects deserve attention
4. Visualizes everything in an interactive, drillthrough-enabled Power BI report designed like a planetary-defense command center

---

## 🗂️ Data Sources

| Source | Data | Access |
|---|---|---|
| **NASA NeoWs (Near Earth Object Web Service)** | Close-approach events: velocity, miss distance, estimated diameter, hazard flag | [api.nasa.gov](https://api.nasa.gov) — free API key |
| **JPL CNEOS Sentry System** | Impact probability, Palermo Scale, Torino Scale for ~1,500 risk-listed objects | [ssd-api.jpl.nasa.gov/sentry.api](https://ssd-api.jpl.nasa.gov/doc/sentry.html) — open, no key |
| **JPL Small-Body Database (SBDB)** | Orbital elements: eccentricity, semi-major axis, inclination, orbital period, observation arc, number of observations | [ssd-api.jpl.nasa.gov/sbdb.api](https://ssd-api.jpl.nasa.gov/doc/sbdb.html) — open, no key |

---

## 🏗️ Architecture

```
NASA NeoWs API ─┐
JPL Sentry API ─┼──► Python (requests) ──► MySQL (raw tables) ──► Python (pandas + scikit-learn)
JPL SBDB API   ─┘                              │                          │
                                               ▼                          ▼
                                     MySQL star schema         ML outputs written back to MySQL
                                    (dim_neo, fact_close_approach)
                                                │
                                                ▼
                                        Power BI (DAX + visuals)
                                                │
                                                ▼
                                  Interactive 3-page report + drillthrough
```

---

## 🧱 Database Design (MySQL)

**Raw layer** (ingested as-is from APIs):
- `raw_neo_feed` — daily close-approach records from NeoWs
- `raw_sentry_risk` — impact risk data from JPL Sentry
- `raw_orbital_elements` — orbital elements from SBDB

**Modeled layer** (star schema):
- `dim_neo` — one row per unique object: orbital elements, risk fields, ML outputs
- `fact_close_approach` — one row per close-approach event, linked to `dim_neo`

Key engineering decision: NASA's two APIs use **different ID systems** (NeoWs numeric `id` vs. JPL's `spkid`). These were reconciled using a normalized `designation_num` field extracted from each object's name/designation string, which is the one identifier consistent across both NASA systems.

---

## 🐍 Python Pipeline

1. **Ingestion** — scheduled pulls from all three APIs into raw MySQL tables, with rate-limiting and error handling
2. **Cleaning & merging** — deduplication, type normalization (e.g. fractional JPL dates, mixed string/numeric IDs), designation-based joins across ID systems
3. **Feature engineering** — observation arc length, number of observations, lunar-distance-normalized miss distance
4. **Machine learning**:
   - **Isolation Forest** (unsupervised anomaly detection) on orbital features (eccentricity, semi-major axis, inclination, orbital period) to flag objects with unusual orbital geometry
   - **Weighted composite risk score** combining hazard flag, normalized impact probability, Torino Scale, and anomaly score into a single 0–100 `ml_risk_score`
5. **Write-back** — all ML outputs (`is_anomaly`, `anomaly_score`, `ml_risk_score`) written back into MySQL for direct BI consumption

---

## Power BI Report

**Page 1 — Command Center Overview**
KPI cards (Total NEOs Tracked, Hazardous NEOs, Avg ML Risk Score, Anomalous Objects), Risk Tier donut chart, close-approach distance trend, orbital anomaly scatter plot.

**Page 2 — Orbital Anomaly & Observation Priority**
Priority-ranked watchlist table, anomaly scatter (eccentricity vs. inclination), observation-coverage scatter (data arc vs. number of observations vs. risk), approach-count-by-risk-tier bar chart.

**Page 3 — Object Profile (Drillthrough)**
Click any object in the priority table to open its full profile: risk gauge, anomaly gauge, orbit classification, catalog-percentile comparison bars, diameter, orbital period in years, impact probability status, and its individual close-approach history and trend.

### Key DAX Measures
```DAX
Total NEOs Tracked = DISTINCTCOUNT(dim_neo[neo_id])

High Priority Watchlist Count = 
CALCULATE(DISTINCTCOUNT(dim_neo[neo_id]), dim_neo[ml_risk_score] >= 30)

Risk Percentile Rank = 
VAR SelectedScore = SELECTEDVALUE(dim_neo[ml_risk_score])
VAR TotalObjects = CALCULATE(COUNTROWS(dim_neo), ALL(dim_neo))
VAR ObjectsBelow = 
    CALCULATE(COUNTROWS(dim_neo), ALL(dim_neo), dim_neo[ml_risk_score] <= SelectedScore)
RETURN DIVIDE(ObjectsBelow, TotalObjects) * 100
```
*(Full DAX measure list in `/dax/measures.dax`)*

---

## ⚠️ Challenges Faced & How They Were Solved

| Challenge | Root Cause | Solution |
|---|---|---|
| JPL Sentry `id` field rejected by MySQL as integer | Sentry IDs are alphanumeric codes (e.g. `bJ79X00B`), not numbers | Changed column type to `VARCHAR` |
| Fractional dates from JPL Sentry broke `DATE` columns | JPL returns dates like `2020-10-3.80160` | Stored as `VARCHAR`, stripped fractional part in Python |
| `sb-group=neo` API parameter returned 400 errors | JPL's SBDB Query API uses a different filter syntax (`sb-cdata`), not a `sb-group` param | Rewrote query using correct SBDB filter syntax |
| Foreign key violations joining feed data to orbital data | NASA NeoWs (`id`) and JPL SBDB (`spkid`) are **two separate ID systems** with almost zero overlap | Built a normalized `designation_num` join key extracted from each object's name/designation string, common to both systems |
| `GROUP BY` errors under MySQL strict mode | `ONLY_FULL_GROUP_BY` requires every selected column to be aggregated or grouped | Wrapped non-aggregated columns in `AVG()`/`MAX()` and expanded the `GROUP BY` clause |
| `ml_risk_score` silently returned `NULL` for every row | A Python `NaN or 1` fallback evaluated to `NaN` (since `NaN` is truthy in Python), silently poisoning every score | Replaced with an explicit `pd.isna()` check before division |
| Power BI line chart collapsed to hierarchy levels instead of real dates | Power BI auto-converts date fields into a Year/Quarter/Month/Day hierarchy by default | Selected the plain date field via the field-well dropdown instead of the hierarchy |
| Small sample size caused near-zero ID overlap between tables | Initial data pull only covered 7 days / 40 records | Backfilled 6 months of close-approach data (~850+ records) via a scheduled weekly-chunk pull, respecting API rate limits |

---

## 🛠️ Tech Stack

- **Data Sources:** NASA NeoWs API, JPL CNEOS Sentry API, JPL SBDB API
- **Database:** MySQL
- **ETL / AI:** Python (`requests`, `pandas`, `numpy`, `scikit-learn`, `mysql-connector-python`)
- **ML Models:** Isolation Forest (anomaly detection), custom weighted composite risk scoring
- **BI / Visualization:** Power BI Desktop (DAX, star schema modeling, drillthrough reports)

---

##  How to Run This Project

1. Clone this repository
2. Get a free NASA API key at [api.nasa.gov](https://api.nasa.gov)
3. Create a MySQL database named `aegis_neo` and run the schema scripts in `/sql/schema.sql`
4. Update `DB_CONFIG` and `API_KEY` in the notebooks under `/notebooks/`
5. Run notebooks in order:
   - `Day1_Ingestion.ipynb`
   - `Day2_DataModeling.ipynb`
   - `Day3_AI_Models.ipynb`
6. Open `AEGIS_Report.pbix` in Power BI Desktop and connect it to your local `aegis_neo` MySQL database

---

## 📈 Future Improvements

- Automate daily ingestion via a scheduled Python job (cron / Airflow) instead of manual notebook runs
- Add a supervised model once more labeled "confirmed re-observation outcome" data is available
- Incorporate observatory-specific visibility windows (via JPL's Observability API) to recommend *which* telescope should track *which* object *when*
- Publish the Power BI report to Power BI Service with scheduled refresh

---

## 👤 Author 
- Garvit Gupta
- https://www.linkedin.com/in/garvit-gupta-87875a226/ (Linkedin Profile)
- https://garvitwork.github.io/portfolio/  (Portfolio Website)

Built as an end-to-end portfolio project demonstrating data engineering, applied machine learning, and business intelligence skills using real-world scientific data.


