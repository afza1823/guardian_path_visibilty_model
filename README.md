# 🛡️ GuardianPath — Time-Aware Safe Pedestrian Route Navigation

> *A machine-learning framework that turns Jane Jacobs' natural surveillance theory into a practical, explainable routing system — built entirely on free OpenStreetMap data.*

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python)
![Streamlit](https://img.shields.io/badge/Streamlit-1.24%2B-FF4B4B?logo=streamlit)
![XGBoost](https://img.shields.io/badge/XGBoost-1.7%2B-green)
![SHAP](https://img.shields.io/badge/SHAP-Explainable_AI-orange)
![License](https://img.shields.io/badge/License-Academic-lightgrey)

---

## 📌 What Is This?

Every mainstream navigation app optimises for distance or speed — none of them care whether the route *feels safe to walk*. A street bustling with shops at noon can be deserted at midnight, yet Google Maps will send you down it either way.

**GuardianPath** fills that gap. It computes a time-aware safety score for every road segment using five interpretable features derived from OpenStreetMap POI data, trains an XGBoost regressor to approximate the scoring formula at millisecond speed, and embeds the predictions into a modified Dijkstra search to produce safety-optimised routes alongside the conventional shortest path. A SHAP explainability layer accompanies every recommendation so users can see *why* a route was chosen.

### Key Results

| Metric | Value |
|--------|-------|
| Model R² Score | **0.9937** |
| Mean Absolute Error | **0.008** |
| Training Samples | **12,000** (500 nodes × 24 hours) |
| First Query Latency | ~10–15 s |
| Subsequent Queries | <5 s |
| GPU Required | **No** |

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   User Query (Streamlit)                 │
│              Start → Destination + Time of Day           │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────▼────────────┐
          │   1. Data Collection    │
          │   OSMnx + Overpass API  │
          │   Road graph + POIs     │
          └────────────┬────────────┘
                       │
          ┌────────────▼────────────┐
          │ 2. Feature Engineering  │
          │  5 safety features per  │
          │  node × 24 hours        │
          │  → 12,000 labelled      │
          │    training samples     │
          └────────────┬────────────┘
                       │
          ┌────────────▼────────────┐
          │  3. XGBoost Prediction  │
          │  Trained regressor      │
          │  replaces rule-based    │
          │  formula at runtime     │
          └────────────┬────────────┘
                       │
          ┌────────────▼────────────┐
          │   4. Safe Routing       │
          │   Modified Dijkstra     │
          │   + SHAP Explanations   │
          └────────────┬────────────┘
                       │
          ┌────────────▼────────────┐
          │   Interactive Map       │
          │   Shortest (red) vs.    │
          │   Safe (green dashed)   │
          └─────────────────────────┘
```

---

## ✨ Features

- **Time-Aware Scoring** — Safety scores change with the hour; the system differentiates between a bustling 10 AM street and a deserted midnight corridor
- **Five Interpretable Safety Features** — Guardian proximity, active-hour flag, active POI density, anchor presence, night penalty
- **XGBoost ML Model** — Achieves R² = 0.9937 in approximating the hand-crafted visibility formula
- **SHAP Explainability** — Every route recommendation comes with feature-level explanations (e.g., "Near Guardian: 95% proximity to police/hospital")
- **Live GPS + Geocoding** — Browser geolocation, address search, and manual coordinate entry
- **Real-Time POI Analysis** — Time-band filtering (morning/day/evening/night) with OSM `opening_hours` parsing
- **Interactive Streamlit Dashboard** — Folium-based map with dual-route overlay, POI clusters, and route comparison metrics
- **Zero Proprietary Data** — Everything runs on free OpenStreetMap data

---

## 📁 Project Structure

```
safepath_export/
│
├── app/                            # Core application modules
│   ├── streamlit_app.py            # Main Streamlit dashboard (entry point)
│   ├── router.py                   # Safe routing engine (Dijkstra + edge cost)
│   ├── feature_engineer.py         # 5 safety features computation
│   ├── visibility_model.py         # XGBoost model wrapper
│   ├── train_model.py              # Model training pipeline
│   ├── shap_explainer.py           # SHAP-based route explanations
│   ├── config.py                   # City bounds, POI tags, weights
│   ├── osm_api.py                  # OpenStreetMap data fetching
│   ├── realtime_poi.py             # Time-aware POI activity detection
│   ├── opening_hours_parser.py     # OSM opening_hours tag parser
│   ├── geocoder.py                 # Address geocoding (Nominatim)
│   ├── map_visualizer.py           # Folium map rendering
│   └── live_location.py            # GPS, privacy masking, visibility gauge
│
├── models/
│   └── visibility_model.pkl        # Pre-trained XGBoost model
│
├── data/
│   └── ml_training_dataset.xlsx    # 12,000 labelled training samples
│
├── cache/                          # Cached road graphs and POI data
│
├── train_visibility_model.py       # Standalone training script
├── requirements.txt                # Python dependencies
│
├── research_paper/                 # Academic manuscripts
│   ├── IEEE_Submissions/           # IEEE OJ-ITS formatted paper
│   └── MDPI_Submissions/          # MDPI Smart Cities formatted paper
│
└── README.md                       # ← You are here
```

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10 or higher
- pip package manager
- Internet connection (for OSM data fetching on first run)

### Installation

```bash
# 1. Clone or download the project
cd safepath_export

# 2. Create a virtual environment (recommended)
python -m venv .venv

# Windows
.venv\Scripts\activate

# macOS/Linux
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt
```

### Running the Application

```bash
# Launch the Streamlit dashboard
streamlit run app/streamlit_app.py
```

The app will open at `http://localhost:8501`. On first launch, it loads the Bengaluru road network (~4 seconds) and fetches POIs via the Overpass API.

### Training the Model (Optional)

The pre-trained model is included at `models/visibility_model.pkl`. To retrain:

```bash
python train_visibility_model.py
```

This generates 12,000 labelled samples (500 nodes × 24 hours), trains the XGBoost regressor, and saves the updated model.

---

## 🧪 How It Works

### The Five Safety Features

| # | Feature | Symbol | Description |
|---|---------|--------|-------------|
| 1 | Guardian Proximity | *P* | Normalised distance to nearby guardian POIs (cafes, pharmacies, shops) via KD-tree |
| 2 | Active Hour Flag | *A* | Binary: is it between 7 AM and 10 PM? |
| 3 | Active POI Density | *D* | Proximity score scaled by time (100% day, 30% night) |
| 4 | Anchor Presence | *N* | Proximity to 24/7 institutions (police, hospitals, pharmacies) |
| 5 | Night Penalty | *NP* | Fixed −0.3 deduction during 10 PM – 6 AM |

### Visibility Formula

```
V = clip(0.2 + 0.5P + 0.2A + 0.15D + 0.1N − NP + ε, 0, 1)
```

Where `ε ~ N(0, 0.1)` adds Gaussian noise to prevent model memorisation.

### Edge Cost Function

For an edge with physical length `ℓ`:

```
Cost(u,v) = ℓ × (1 + 4 · b²)
```

Where `b` blends a heuristic danger score with the XGBoost prediction. A well-lit, busy edge costs roughly its physical length; a dark, deserted edge gets inflated by up to 5×.

---

## 📊 Experimental Results

Tested on the Bengaluru urban road network (12.95°N–13.01°N, 77.56°E–77.65°E).

### Night Scenario (22:00)
- **Route**: Good Hope School → Mount Carmel College
- Safe route is 2% longer but 3% safer
- Active POI density: 15.6 POIs/km (safe) vs 14.0 (shortest)

### Day Scenario (10:00 AM)
- **Route**: Coles Park → Mount Carmel College
- Both routes score similarly (~87–88% safety)
- Engine recommends shortest route — "Both routes are safe at this time"

### Temporal Sensitivity
| Metric | Night (22:00) | Day (10:00) | Ratio |
|--------|:---:|:---:|:---:|
| Active POIs | 136 | 744 | 5.5× |
| Active Density (POIs/km) | 14.0 | 95.4 | 6.8× |
| Guardian Facilities | 88 | 443 | 5.0× |

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| **ML Model** | XGBoost (regression) |
| **Explainability** | SHAP (SHapley Additive exPlanations) |
| **Road Network** | OSMnx + NetworkX |
| **Geospatial Data** | OpenStreetMap (Overpass API) |
| **Spatial Index** | SciPy KD-tree |
| **Data Processing** | Pandas, GeoPandas, NumPy |
| **Routing Algorithm** | Modified Dijkstra (NetworkX) |
| **Web Interface** | Streamlit + Folium |
| **Geocoding** | Nominatim (OpenStreetMap) |

---

## 📝 Research Papers

This project has been submitted to two peer-reviewed journals:

1. **IEEE Open Journal of Intelligent Transportation Systems** — `research_paper/IEEE_Submissions/`
2. **MDPI Smart Cities** — `research_paper/MDPI_Submissions/`

Both papers follow the same conversational scholarly tone with full mathematical formulations, experimental validation, and SHAP analysis.

---

## ⚠️ Known Limitations

1. **Cumulative scoring bias** — Dijkstra favours routes with dense POI clusters near endpoints over evenly-distributed surveillance. A min-link formulation would be better.
2. **Bengaluru-only** — The road graph and model are trained for Bengaluru; other cities require re-downloading and retraining.
3. **Proxy-based safety** — Labels come from a rule-based formula, not actual crime data. The system measures indicators of safety, not empirically verified safety outcomes.
4. **OSM data quality** — OpenStreetMap coverage and `opening_hours` tags vary; missing data falls back to time-band heuristics.

---

## 🔮 Future Work

- 📱 Mobile app with GPS turn-by-turn navigation
- 🔄 Bayesian fusion of rule-derived scores with real crime/incident data
- 🌧️ Weather penalty (rain suppresses foot traffic like nightfall)
- 🌍 On-demand graph loading for any city via bounding box

---

## 👤 Authors

- **Afza Ruheen** — Developer & Researcher | [ORCID: 0009-0007-2177-1671](https://orcid.org/0009-0007-2177-1671) | m24cs01@mccblr.edu.in
- **Shaila Mary J.** — Project Mentor | Department of Computer Science, Mount Carmel College, Bengaluru

---

## 📄 License

This project is developed for academic research purposes at Mount Carmel College, Bengaluru, India.

---

## 🙏 Acknowledgments

- The [OpenStreetMap](https://www.openstreetmap.org) community for maintaining the geospatial data infrastructure
- Jane Jacobs, for the half-century-old insight that streets with "eyes" are safer streets
