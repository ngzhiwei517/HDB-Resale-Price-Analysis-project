# HDB Resale Price Analysis: The Impact of Air Quality and Accessibility in Singapore

## Project Overview

This project explores how **air quality** and **proximity to public facilities** affect **HDB resale flat prices** in Singapore. Using data from 2021–2024, we build a regression model to quantify the relative contribution of environmental and accessibility factors — alongside traditional housing attributes — in explaining resale price variation.

This is a team project for SC3021 Data Science Fundamentals at Nanyang Technological University (NTU), College of Computing and Data Science (CCDS).

---

## Research Questions

1. Do areas with persistently high PSI/PM2.5 levels exhibit lower HDB resale prices?
2. How does accessibility to MRT/LRT stations and schools relate to resale prices?
3. What is the relative contribution of environmental and accessibility factors compared to traditional housing attributes (flat size, storey range, remaining lease) in explaining price variation?

---

## Dataset Sources

## Dataset Sources

> \* = selected for analysis | Datasets without \* were considered but rejected

### Property Price Data
- [DS1 — Resale Flat Prices (Based on Registration Date) from Jan 2017 onwards](https://data.gov.sg/datasets/d_8b84c4ee58e3cfc0ece0d773c8ca6abc/view) `selected`

### Air Quality Data
- [DS2 — Historical Pollutant Standards Index (PSI) 2021](https://data.gov.sg/datasets/d_05b35c51664e1bb6f4dcd78478ae1abe/view) `selected`
- [DS3 — Historical Pollutant Standards Index (PSI) 2022](https://data.gov.sg/datasets/d_d3fb32451d63dc48dc425146ec014516/view) `selected`
- [DS4 — Historical Pollutant Standards Index (PSI) 2023](https://data.gov.sg/datasets/d_10501b71361f97dbbbab82095406c9c5/view) `selected`
- [DS5 — Historical Pollutant Standards Index (PSI) 2024](https://data.gov.sg/datasets/d_9213cd2e4631f7148ab5932a10df9958/view) `selected`

### Location / Amenities Data
- [DS6 — Singapore Train Station Coordinates](https://www.kaggle.com/datasets/yxlee245/singapore-train-station-coordinates) `selected`
- [DS7 — Hospitals with Coordinates](https://www.kaggle.com/datasets/muhdirshath/hospitals-in-singapore) `rejected — see DS7 section below`

### Education Amenities
- [DS8 — General Information of Schools](https://data.gov.sg/datasets/d_688b934f82c1059ed0a6993d2a829089/view) `selected`

### Geographic Mapping Data
- [DS9 — Singapore City Geo-Coordinates](https://www.kaggle.com/datasets/shymammoth/singapore-city-geo-coordinates-more-reliable) `selected`

---

## Data Preparation Pipeline

### Overview

Data preparation follows a structured pipeline of profiling, structuring, cleaning, and enriching steps across all datasets. The diagram below summarises the full pipeline.

We profiled 6 datasets in total, retaining 5 and rejecting 1 (DS7 — Hospital Coordinates), as hospitals are heavily concentrated in central Singapore, resulting in near-zero variation in distance features across HDB towns.

### DS1 — HDB Resale Flat Prices
- **224,000 rows, 11 columns** | Date range: Jan 2017 – Dec 2024 | 26 towns | 7 flat types
- Zero missing values
- Key issue: `storey_range` ("10 TO 12") and `remaining_lease` ("61 years 04 months") stored as strings — numeric extraction required
- **Structuring:** filter to 2021–2024 → extract `storey_low` via string split → extract `remaining_lease_years` via regex → drop `block`, `street_name`, `flat_model`

### DS2–DS5 — Historical PSI Data (2021–2024)
- **202,158 rows** across 4 yearly files | 5 regions | hourly granularity
- `psi_three_hourly` entirely null → dropped | ~5% missing in core pollutant columns
- Central and East regions consistently record higher average PSI than West
- Notable PM2.5 spike in September 2023 from transboundary haze event
- **Structuring:** filter out national rows → retain `pm25_twenty_four_hourly` and `psi_twenty_four_hourly` only → aggregate to monthly mean per region
- **Derivation:** North-East PSI synthesised as average of North and East monthly readings to ensure full regional coverage

### DS6 — MRT/LRT Station Coordinates
- **157 stations** | 120 MRT + 37 LRT | Zero missing values
- Some interchange stations share identical coordinates (expected, not errors)
- 2 stations found with incorrect coordinates pointing outside Singapore → corrected using OneMap API
- **Cleaning:** fix 2 erroneous coordinates via OneMap API | deduplicate interchange stations

### DS7 — Hospital Coordinates ❌ Rejected
- Only 24 hospitals, heavily concentrated in central Singapore
- `dist_to_nearest_hospital` would show near-zero variation across HDB towns → weak regression feature
- **Decision:** excluded from further analysis

### DS8 — Schools
- **337 schools** across 28 towns | Even zone distribution (~83 per zone)
- No coordinate columns — town-level aggregation used instead
- School count varies meaningfully across towns (22 in Woodlands → 1 in Tengah)
- **Structuring:** drop irrelevant columns → group by `dgp_code` → compute `school_count` per town

### DS9 — Singapore Town Geo-Coordinates
- **332 rows** | 55 towns | 5 regions
- Tengah and Hougang have erroneous coordinates (pointing to India and Philippines respectively) → fixed manually
- KALLANG/WHAMPOA and CENTRAL AREA missing entirely → added manually
- 17 rows with missing coordinates confirmed safe to drop after profiling check
- **Pipeline:** Profiling (check 17 missing-coord rows) → Structuring (fix coords, add towns, drop irrelevant cols) → Cleaning (drop 17 rows, compute centroid per town)

---

## Enriching Steps

All datasets are joined into DS1 in three enriching steps:

### Step 1 — Geographic Enrichment (DS9 + DS6)
- DS9 provides `region`, `latitude`, `longitude` per town (centroid)
- DS9 centroids + DS6 station coordinates → Haversine formula → `dist_to_nearest_mrt` per town
- **Note:** As individual flat-level coordinates were unavailable without external geocoding, town-level centroids are used as a spatial approximation. All flats within the same town share the same distance value. This is acknowledged as a limitation.

### Step 2 — School Enrichment (DS8)
- DS8 school counts aggregated by `dgp_code` (town)
- Left join into DS1 by `town` → adds `school_count`

### Step 3 — Air Quality Enrichment (DS2–DS5)
- DS1 town → DS9 region mapping → join monthly PSI table by `region` + `year` + `month_num`
- Adds `avg_pm25` and `avg_psi` per transaction

### Step 4 — Post-join Check
- `isnull().sum()` on merged dataset to detect any nulls introduced by joins
- Impute or drop affected rows as appropriate

---

## Final Dataset Features

| Column | Source | Type |
|--------|--------|------|
| `resale_price` | DS1 | Target variable |
| `flat_type` | DS1 | Categorical |
| `storey_low` | DS1 | Numeric |
| `floor_area_sqm` | DS1 | Numeric |
| `remaining_lease_years` | DS1 | Numeric |
| `town` | DS1 | Categorical |
| `region` | DS9 | Categorical |
| `avg_pm25` | DS2–DS5 | Numeric |
| `avg_psi` | DS2–DS5 | Numeric |
| `dist_to_nearest_mrt` | DS6 + DS9 | Numeric |
| `school_count` | DS8 | Numeric |

---

## Model Evaluation Metrics

- **R-squared** — proportion of price variation explained by the model
- **Mean Squared Error (MSE)** — predictive accuracy of resale price estimates

---

## Known Limitations

- `dist_to_nearest_mrt` is computed at the town-centroid level, not individual flat level — all flats in the same town share the same distance value
- PSI data covers only 4 monitoring regions; North-East PSI is derived as an average of North and East readings
- Analysis restricted to 2021–2024 (post-COVID period) — findings may not generalise to earlier periods

---

## Team

Project submitted for SC3021 Data Science Fundamentals
Nanyang Technological University, CCDS
AY2025/26

---

