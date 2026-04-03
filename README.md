# HDB Resale Price Analysis: The Impact of Air Quality and Accessibility in Singapore

![Python](https://img.shields.io/badge/Python-3.x-blue)
![XGBoost](https://img.shields.io/badge/XGBoost-R²%3D0.953-green)
![Dataset](https://img.shields.io/badge/Dataset-109%2C393%20transactions-orange)
![Period](https://img.shields.io/badge/Period-2021--2024-yellow)

## 📋 Project Overview

This project explores how **air quality** and **proximity to public facilities** affect **HDB resale flat prices** in Singapore. Using data from 2021–2024, we build and compare four regression models to quantify the relative contribution of environmental and accessibility factors — alongside traditional housing attributes — in explaining resale price variation.

> **SC3021 Data Science Project — Group 4**
> 
> Hau Jia Qi | Ng Zhi Wei | Toh En Qi
> 
> Nanyang Technological University, College of Computing and Data Science

---

## 🔍 Research Questions

1. Do areas with persistently high PSI/PM2.5 levels exhibit lower HDB resale prices?
2. How does accessibility to MRT/LRT stations and schools relate to resale prices?
3. What is the relative contribution of environmental and accessibility factors compared to traditional housing attributes (flat size, storey range, remaining lease) in explaining price variation?

---

## 📁 Dataset Sources

| ID | Dataset | Source |
|----|---------|--------|
| DS1 | HDB Resale Flat Prices (Jan 2017 onwards) | [data.gov.sg](https://data.gov.sg/datasets/d_8b84c4ee58e3cfc0ece0d773c8ca6abc/view) |
| DS2 | Historical PSI Data 2021 | [data.gov.sg](https://data.gov.sg/datasets/d_05b35c51664e1bb6f4dcd78478ae1abe/view) |
| DS3 | Historical PSI Data 2022 | [data.gov.sg](https://data.gov.sg/datasets/d_d3fb32451d63dc48dc425146ec014516/view) |
| DS4 | Historical PSI Data 2023 | [data.gov.sg](https://data.gov.sg/datasets/d_10501b71361f97dbbbab82095406c9c5/view) |
| DS5 | Historical PSI Data 2024 | [data.gov.sg](https://data.gov.sg/datasets/d_9213cd2e4631f7148ab5932a10df9958/view) |
| DS6 | Singapore Train Station Coordinates | [Kaggle](https://www.kaggle.com/datasets/yxlee245/singapore-train-station-coordinates) |
| DS7 | Hospital Coordinates *(Rejected)* | [Kaggle](https://www.kaggle.com/datasets/muhdirshath/hospitals-in-singapore) |
| DS8 | General Information of Schools | [data.gov.sg](https://data.gov.sg/datasets/d_688b934f82c1059ed0a6993d2a829089/view) |
| DS9 | Singapore City Geo-Coordinates | [Kaggle](https://www.kaggle.com/datasets/shymammoth/singapore-city-geo-coordinates-more-reliable) |

> ⚠️ **DS7 (Hospitals) was rejected** — only 24 hospitals, heavily concentrated in central Singapore, resulting in near-zero variation in distance features across HDB towns.

---

## 🔧 Data Preparation Pipeline

### Overview
Data preparation follows a structured pipeline of profiling, structuring, cleaning, and enriching steps across all datasets. We profiled 6 datasets in total, retaining 5 and rejecting 1.

### DS1 — HDB Resale Flat Prices
- **224,000 rows, 11 columns** | Jan 2017 – Dec 2024 | 26 towns | 7 flat types
- Zero missing values
- Key issue: `storey_range` and `remaining_lease` stored as strings — numeric extraction required
- **Steps:** filter to 2021–2024 → extract `storey_low` → extract `remaining_lease_years` → drop irrelevant columns

### DS2–DS5 — Historical PSI Data (2021–2024)
- **202,158 rows** across 4 yearly files | 5 regions | hourly granularity
- `psi_three_hourly` entirely null → dropped | ~5% missing in core pollutant columns
- **Steps:** filter national rows → retain PM2.5 and PSI columns → aggregate to monthly mean per region
- **Derivation:** North-East PSI synthesised as average of North and East readings

### DS6 — MRT/LRT Station Coordinates
- **157 stations** | 120 MRT + 37 LRT | Zero missing values
- 2 stations had incorrect coordinates → corrected using OneMap API (Singapore Land Authority)

### DS8 — Schools
- **337 schools** across 28 towns | Even zone distribution (~83 per zone)
- No coordinate columns — town-level aggregation used instead
- **Steps:** drop irrelevant columns → group by `dgp_code` → compute `school_count` per town

### DS9 — Singapore Town Geo-Coordinates
- **332 rows** | 55 towns | 5 regions
- Tengah and Hougang had erroneous coordinates → fixed manually
- KALLANG/WHAMPOA and CENTRAL AREA missing → added manually
- **Steps:** fix coordinates → add missing towns → drop irrelevant columns → compute centroid per town

---

## 🔗 Enriching Steps

| Step | Description |
|------|-------------|
| Step 1 — Geographic | DS9 centroids × DS6 stations → Haversine formula → `dist_to_nearest_mrt`; extract `region`, `latitude`, `longitude` |
| Step 2 — School | DS8 school counts aggregated by town → left join to DS1 by `town` → adds `school_count` |
| Step 3 — Air Quality | DS2–5 monthly PSI → join to DS1 by `region` + `year` + `month` → adds `mean_pm25`, `mean_psi` |
| Step 4 — Post-join | `isnull().sum()` on merged dataset; impute or drop null rows |

---

## 📊 Final Dataset Features

| Column | Source | Type | Role |
|--------|--------|------|------|
| `resale_price` | DS1 | Numeric | Target variable |
| `flat_type` | DS1 | Categorical | Encoded as ordinal `flat_type_num` |
| `storey_low` | DS1 | Numeric | Predictor |
| `floor_area_sqm` | DS1 | Numeric | Predictor |
| `remaining_lease_years` | DS1 | Numeric | Predictor |
| `town` | DS1 | Categorical | Grouping |
| `region` | DS9 | Categorical | Grouping + join key |
| `mean_pm25` | DS2–DS5 | Numeric | Environmental feature |
| `mean_psi` | DS2–DS5 | Numeric | Dropped (r=0.97 with PM2.5) |
| `dist_to_nearest_mrt` | DS6 + DS9 | Numeric | Accessibility feature |
| `school_count` | DS8 | Numeric | Accessibility feature |

---

## 📈 Analysis

### Descriptive Analytics
- Summary statistics and price distribution across flat types and towns
- Monthly median resale price time series by region (2021–2024)
- PM2.5 and PSI trends by region — seasonal haze spikes identified
- Pearson correlation heatmap across all numeric features

### Diagnostic Analytics
- OLS scatter plots: each feature vs `resale_price`
- Pearson and Spearman correlation table with divergence analysis
- **Mann-Whitney U Test:** MRT proximity vs resale price
  - Result: statistically significant (p = 1.94 × 10⁻⁷), but negligible effect size at bivariate level due to regional confounding — resolved by multivariate XGBoost
- **One-Way ANOVA:** PM2.5 and resale price across five regions
  - PM2.5: F = 2,818.58, p ≈ 0.00 ✓
  - Price: F = 1,505.80, p ≈ 0.00 ✓

### Predictive Analytics

| Model | Features | R² (Test) | RMSE (SGD) | CV R² |
|-------|----------|-----------|------------|-------|
| Model 1 — Baseline Linear Regression | Housing only | 0.592 | 113,105 | 0.508 |
| Model 2 — Enriched Linear Regression | All features | 0.609 | 110,742 | 0.461 |
| Model 3 — Decision Tree (depth=8) | All features | 0.774 | 84,267 | 0.481 |
| **Model 4 — XGBoost (Tuned) ✓ Best** | All features | **0.953** | **38,479** | **0.951** |

**Best hyperparameters (GridSearchCV, 360 fits):**
max_depth=8 | n_estimators=400 | learning_rate=0.1 | subsample=0.9 | colsample_bytree=0.7

> **Key insight:** Pre-tuning XGBoost had CV R² = 0.529 vs test R² = 0.924 — a large gap indicating overfitting. GridSearchCV resolved this completely: tuned model achieves test R² = 0.953 and CV R² = 0.951, confirming reliable generalisation.

---

## 🏆 Key Findings

| Rank | Feature | XGBoost Importance | Insight |
|------|---------|-------------------|---------|
| 1 | Flat Type | 0.485 | Space is the primary price driver |
| 2 | **MRT Distance** | **0.150** | **Connectivity outranks floor area — despite appearing weakest in bivariate correlation (r = -0.032), multivariate modelling reveals it as the 2nd strongest independent driver** |
| 3 | Floor Area (sqm) | 0.118 | Physical size still important |
| 4 | School Count | 0.093 | Education access adds moderate value |
| 5 | Remaining Lease | 0.059 | Lease length matters |
| 6 | Storey Level | 0.050 | Moderate influence |
| 7 | Air Quality PM2.5 | 0.044 | Minimal effect within Singapore's narrow pollution range |

> **Note on MRT distance:** Weak bivariate correlation (r = -0.032) does not mean MRT is unimportant — regional confounding suppressed the signal. XGBoost's multivariate non-linear modelling correctly identifies it as the 2nd strongest price driver. This is one of our most important analytical findings.

---

## 💡 Recommendations

| Stakeholder | Recommendation |
|-------------|----------------|
| **Current Homeowners** | Factor in MRT proximity when pricing; highlight school count and remaining lease as secondary value drivers |
| **Future Homeowners** | A smaller flat in a well-connected town may offer better long-term value than a larger flat in a poorly connected area; north and east regions tend to have slightly lower PM2.5 |
| **Property Agents** | Use flat type and MRT distance as primary pricing anchors; avoid over-emphasising air quality as a price justification given its weak current signal |
| **Policymakers / Urban Planners** | Expand MRT coverage to north and west regions to reduce spatial price inequality; prioritise environmental monitoring in west region towns with above-average PM2.5 and below-median prices |
| **General Public** | Use the interactive visualisations in the notebook to explore price trends across towns and years |

---

## ⚠️ Known Limitations

- `dist_to_nearest_mrt` computed at town-centroid level — all flats in the same town share the same distance value; true block-level effect likely stronger
- PSI data covers only 4 monitoring regions; North-East PSI derived as average of North and East
- Analysis restricted to 2021–2024 (post-COVID); findings may not generalise to earlier periods
- Key amenities (shopping malls, parks, hospitals) excluded from final model
- Spatial autocorrelation between towns not corrected for
- School count is static and does not reflect school quality or actual proximity

---

## 🔒 Ethical Considerations

### Data Ethics
Our datasets were originally collected for administrative and environmental monitoring purposes, not housing price prediction. Repurposing them raises concerns around intended use — model findings on MRT proximity and school density premiums could be misused to justify aggressive pricing in affordable towns, contributing to gentrification. Our analysis is correlational, not causal.

### Data Privacy
The HDB resale dataset contains transaction-level records that do not include personal identifiers, but could allow re-identification in edge cases such as low-transaction towns or rare flat types. All individual-level data remained internal to the team and analysis was conducted on aggregated representations only.

---

## 🛠️ Tech Stack

| Library | Purpose |
|---------|---------|
| `pandas`, `numpy` | Data manipulation and numerical computing |
| `matplotlib`, `seaborn` | Static visualisation |
| `plotly` | Interactive visualisation |
| `scikit-learn` | Linear Regression, Decision Tree, GridSearchCV |
| `xgboost` | Gradient boosting regressor |
| `joblib` | Model serialisation |
| `scipy` | Hypothesis testing (Mann-Whitney U, ANOVA) |
