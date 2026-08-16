# wsl_2025_2026_databricks
A Databricks / PySpark pipeline that ingests match data for the **Women's Super League (WSL)** from two complementary sources — [Sportmonks](https://www.sportmonks.com/) and [FotMob](https://www.fotmob.com/) — and builds an analytical Gold layer that links per-minute **pressure indices** to **shot creation events**. The pipeline concludes with testing a series of multi-level models and exploratory visualisations.


---
 
## Repository structure
 
```
.
├── 01_ingest_wsl_fixtures.ipynb          # Exploratory single-fixture ingest from Sportmonks API (optional)
├── 02_ingest_fixture_details.ipynb       # Bulk ingest of all season fixtures (batched API calls)
├── 03_landing_to_bronze.ipynb            # Sportmonks raw JSON → Delta bronze table
├── 04_bronze_to_silver.ipynb             # Sportmonks bronze table → 7 normalised silver tables
├── 05_fotmob_landing_to_bronze.ipynb     # FotMob raw JSON → Delta bronze table
├── 06_fotmob_bronze_to_silver.ipynb      # FotMob bronze table → silver table (matches, shots, xG, player stats)
├── 07_match_source_mapping.ipynb         # Cross-source match & team ID reconciliation
├── 08_build_match_momentum_corrected_… . # Gold: per-minute pressure × shot metrics table
├── 09_feature_engineering_corrected.ipynb# Feature engineering with anti-leakage design
├── 10_multilevel_time_series.ipynb       # GEE models: pressure → shot probability
├── 11_visualisations.ipynb               # Pressure dynamics, shot quality, forest & 
└── 11_visualisations.ipynb               # Pressure dynamics, shot quality, forest & heatmap plots
```
 
---
