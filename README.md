# HDB Resale Price Analysis: The Impact of Air Quality and Accessibility in Singapore

## Project Overview

This project explores how **air quality** and **proximity to public facilities** affect **HDB resale flat prices** in Singapore. Using data from 2021–2024, we build and compare four regression models to quantify the relative contribution of environmental and accessibility factors — alongside traditional housing attributes — in explaining resale price variation.

---

## Research Questions

1. Do areas with persistently high PSI/PM2.5 levels exhibit lower HDB resale prices?
2. How does accessibility to MRT/LRT stations and schools relate to resale prices?
3. What is the relative contribution of environmental and accessibility factors compared to traditional housing attributes (flat size, storey range, remaining lease) in explaining price variation?

---

## Dataset Sources

| ID | Dataset | Source |
|----|---------|--------|
| DS1 | HDB Resale Flat Prices (Jan 2017 onwards) | [data.gov.sg](https://data.gov.sg/datasets/d_8b84c4ee58e3cfc0ece0d773c8ca6abc/view) |
| DS2 | Historical PSI Data 2021 | [data.gov.sg](https://data.gov.sg/datasets/d_05b35c51664e1bb6f4dcd78478ae1abe/view) |
| DS3 | Historical PSI Data 2022 | [data.gov.sg](https://data.gov.sg/datasets/d_d3fb32451d63dc48dc425146ec014516/view) |
| DS4 | Historical PSI Data 2023 | [data.gov.sg](https://data.gov.sg/datasets/d_10501b71361f97dbbbab82095406c9c5/view) |
| DS5 | Historical PSI Data 2024 | [data.gov.sg](https://data.gov.sg/datasets/d_9213cd2e4631f7148ab5932a10df9958/view) |
| DS6 | Singapore Train Station Coordinates | [Kaggle](https://www.kaggle.com/datasets/yxlee245/singapore-train-station-coordinates) |
| DS7 | Hospital Coordinates ❌ Rejected | [Kaggle](https://www.kaggle.com/datasets/muhdirshath/hospitals-in-singapore) |
| DS8 | General Information of Schools | [data.gov.sg](https://data.gov.sg/datasets/d_688b934f82c1059ed0a6993d2a829089/view) |
| DS9 | Singapore City Geo-Coordinates | [Kaggle](https://www.kaggle.com/datasets/shymammoth/singapore-city-geo-coordinates-more-reliable) |

> DS7 (Hospitals) was rejected — only 24 hospitals, heavily concentrated in central Singapore, resulting in near-zero variation in distance features across HDB towns.

---

## Data Preparation Pipeline

### Overview

Data preparation follows a structured pipeline of profiling, structuring, cleaning, and enriching steps across all datasets. We profiled 6 datasets in total, retaining 5 and rejecting 1 (DS7 — Hospital Coordinates), as hospitals are heavily concentrated in central Singapore, resulting in near-zero variation in distance features across HDB towns.

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
- **Pipeline:** Profiling → Structuring (fix coords, add towns, drop irrelevant cols) → Cleaning (drop 17 rows, compute centroid per town)

---

## Enriching Steps

| Step | Description |
|------|-------------|
| Step 1 — Geographic | DS9 centroids × DS6 stations → Haversine formula → `dist_to_nearest_mrt`; extract `region`, `latitude`, `longitude` per town |
| Step 2 — School | DS8 school counts aggregated by town → left join to DS1 by `town` → adds `school_count` |
| Step 3 — Air Quality | DS2–5 monthly PSI → join to DS1 by `region` + `year` + `month` → adds `mean_pm25`, `mean_psi` |
| Step 4 — Post-join | `isnull().sum()` on merged dataset; impute or drop null rows |

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
| `mean_pm25` | DS2–DS5 | Numeric |
| `mean_psi` | DS2–DS5 | Numeric |
| `dist_to_nearest_mrt` | DS6 + DS9 | Numeric |
| `school_count` | DS8 | Numeric |

---

## Analysis

### Predictive Analytics

| Model | Features | R² (Test) | RMSE (SGD) | CV R² |
|-------|----------|-----------|------------|-------|
| Model 1 — Baseline Linear Regression | Housing only | 0.592 | ~107,000 | ~0.591 |
| Model 2 — Enriched Linear Regression | All features | 0.609 | ~105,000 | ~0.608 |
| Model 3 — Decision Tree (depth=8) | All features | 0.774 | ~83,000 | ~0.770 |
| Model 4 — XGBoost (Tuned) ✅ Best | All features | **0.953** | **38,479** | **0.951** |

**Best model hyperparameters:** `max_depth=8`, `n_estimators=400`, `learning_rate=0.1`, `subsample=0.9`, `colsample_bytree=0.7`

---

## Key Findings

| Rank | Feature | Importance | Insight |
|------|---------|------------|---------|
| 1 | Flat Type | 0.485 | Space is the primary price driver |
| 2 | MRT Distance | 0.150 | Connectivity outranks floor area |
| 3 | Floor Area (sqm) | 0.118 | Physical size still important |
| 4 | School Count | 0.093 | Education access adds moderate value |
| 5 | Remaining Lease | 0.078 | Lease length matters |
| 6 | Air Quality PM2.5 | 0.044 | Minimal effect within Singapore's narrow pollution range |
| 7 | Storey Level | 0.032 | Least impactful housing attribute |

---

## Recommendations

| Stakeholder | Recommendation |
|-------------|----------------|
| **Current Homeowners** | Factor in MRT proximity when pricing; highlight school count and remaining lease as secondary value drivers |
| **Future Homeowners** | Evaluate towns holistically; a smaller flat in a well-connected town may offer better value than a larger flat in a poorly connected area |
| **Property Agents** | Use flat type and MRT distance as primary pricing anchors; avoid over-emphasising air quality as a price justification |
| **Policymakers / Urban Planners** | Expand MRT coverage to north and west regions to reduce spatial price inequality and improve liveability |
| **General Public** | Use the interactive visualisations to explore price trends across towns and years without requiring technical background |

---

## Ethical Considerations

### Data Ethics
Our datasets were originally collected for administrative and environmental monitoring purposes, not housing price prediction. Repurposing them raises concerns around intended use — our model's findings on MRT proximity and school density premiums could be misused to justify aggressive pricing in affordable towns, contributing to gentrification. We also acknowledge our analysis is correlational, not causal, and that treating all schools equally in our school count feature may introduce bias.

### Data Privacy and Security
The HDB resale dataset contains transaction-level records (block, street, flat type, month) that do not include personal identifiers, but could allow re-identification in edge cases such as low-transaction towns or rare flat types. Our geospatial enrichment adds further locational detail, increasing this risk marginally. All individual-level data remained internal to the team and analysis was conducted on aggregated representations only.

---

## Known Limitations

- `dist_to_nearest_mrt` computed at town-centroid level — all flats in the same town share the same distance value
- PSI data covers only 4 monitoring regions; North-East PSI derived as average of North and East
- Analysis restricted to 2021–2024 (post-COVID); findings may not generalise to earlier periods
- Key amenities (shopping malls, parks, hospitals) excluded from final model
- Spatial autocorrelation between towns not corrected for

---

## Tech Stack
```
Python 3.x
├── pandas, numpy          — data manipulation
├── matplotlib, seaborn    — static visualisation
├── plotly                 — interactive visualisation
├── scikit-learn           — Linear Regression, Decision Tree, GridSearchCV
├── xgboost                — gradient boosting regressor
├── joblib                 — model serialisation
└── scipy                  — hypothesis testing (Mann-Whitney U, ANOVA)
```

---

