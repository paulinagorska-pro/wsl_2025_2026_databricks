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
└── 11_visualisations.ipynb               # Pressure dynamics, shot quality, forest & heatmap plots
```

---
 
## Data sources
 
| Source | What it provides | Access |
|---|---|---|
| **Sportmonks v3 API** | Fixture metadata, lineups, events, statistics, pressure indices | API token (stored in Databricks Secrets) |
| **FotMob** | Shot map (xG, xGoT, shot type, position), player ratings, match-level xG | JSON files pre-downloaded to Unity Catalog Volumes |
 
---
 
## Medallion architecture
 
```
Landing (raw JSON on Volumes)
    │
    ├─── Sportmonks ──▶  bronze.fixtures_raw           (VARIANT payload)
    │                         │
    │                         ├──▶ silver.fixtures
    │                         ├──▶ silver.fixture_teams
    │                         ├──▶ silver.teams
    │                         ├──▶ silver.pressure
    │                         ├──▶ silver.events
    │                         ├──▶ silver.lineups
    │                         └──▶ silver.statistics
    │
    └─── FotMob ──────▶  bronze.fotmob_matches_raw     (VARIANT payload)
                              │
                              ├──▶ silver.fotmob_matches
                              ├──▶ silver.fotmob_shots
                              ├──▶ silver.fotmob_team_xg
                              ├──▶ silver.fotmob_player_stats_long
                              └──▶ silver.fotmob_player_match
                                        │
               silver.match_source_map ─┤ (cross-source ID bridge)
               silver.team_source_map  ─┤
                                        │
                              gold.match_momentum           (minute-level spine)
                              gold.match_momentum_features  (anti-leakage feature set)
                              gold.match_momentum_temporal  (additional shot-history features)
                              gold.variance_partitioning_icc
```
 
---
 
## Key analytical concepts
 
### Pressure index
A per-minute, per-team index provided by Sportmonks that quantifies territorial and ball-pressure intensity. Values are bounded but can occasionally exceed 100 (documented and investigated in notebook 13).
 
### Anti-leakage feature design (notebook 09)
All pressure features use **only observations from minutes ≤ t−1** as predictors for events in minute t. The current-minute pressure value is deliberately excluded from the feature table to prevent data leakage in any downstream model.
 
### Own-goal handling (notebook 08)
Own goals are credited to the **opponent's** `goal_count` and excluded from the shooting team's `shot_count` and xG aggregates, matching football analytics conventions.

---
 
## Prerequisites
 
- **Databricks Runtime** with Unity Catalog and PySpark
- **Unity Catalog** namespace `wsl_analytics` with schemas `landing`, `bronze`, `silver`, `gold`, and `reference`
- **Databricks Secret Scope** named `sportmonks` with key `api-token`
- Python packages (installed inline in relevant notebooks): `requests`, `openpyxl`, `statsmodels`, `patsy`, `scipy`, `matplotlib`, `pandas`, `numpy`
---
 
## Setup
 
1. Obtain a Sportmonks API token and store it in a Databricks Secret Scope:
```
   scope: sportmonks
   key:   api-token
```
2. Pre-download FotMob match JSON files to:
```
   /Volumes/wsl_analytics/landing/fotmob_raw/match_details/league_id=9227/season=2025-2026/
```
3. Place the manual cross-source match mapping spreadsheet at:
```
   /Volumes/wsl_analytics/reference/manual_mappings/matching.xlsx
```
4. Run notebooks in order (02 → 11). Notebooks 01–02 are for **ingestion only** and write to Landing; subsequent notebooks assume the upstream tables already exist.
---
 
## Configuration constants
 
Each notebook declares its constants at the top of the first cell. The values below reflect the 2025–2026 WSL season:
 
| Constant | Value | Description |
|---|---|---|
| `LEAGUE_ID` | `45` | Sportmonks league ID for WSL |
| `SEASON_ID` | `26097` | Sportmonks season ID |
| `FOTMOB_LEAGUE_ID` | `9227` | FotMob league ID for WSL |
 
---
 
## Notebook summaries
 
| # | Notebook | Output |
|---|---|---|
| 01 | Single-fixture exploratory ingest | JSON file in Landing |
| 02 | Bulk season ingest (batched, with retry) | JSON batch files in Landing |
| 03 | Landing → bronze (Sportmonks) | `bronze.fixtures_raw` |
| 04 | Bronze → silver (Sportmonks) | 7 silver tables |
| 05 | Landing → bronze (FotMob) | `bronze.fotmob_matches_raw` |
| 06 | Bronze → silver (FotMob) | 5 silver tables |
| 07 | Cross-source mapping | `silver.match_source_map`, `silver.team_source_map` |
| 08 | Build Gold momentum table | `gold.match_momentum` |
| 09 | Feature engineering | `gold.match_momentum_features` |
| 10 | Temporal features + GEE models | `gold.match_momentum_temporal`, model output |
| 11 | Visualisations | Inline matplotlib figures |
 
---
 
## Notes on data quality
 
- **Pressure coverage**: not all minutes in every fixture have a pressure reading. `NULL` pressure values are preserved as such in the Gold layer (they are **not** imputed to zero, which would conflate absence of data with absence of pressure activity).
- **FotMob shot map availability**: coverage depends on FotMob's `coverageLevel` field. Matches with limited coverage may have incomplete shot data.

---
 
*League: Women's Super League (WSL) — Season 2025–2026*
