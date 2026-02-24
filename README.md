# 🟢 Blinkit Bistro — Smart Expansion Strategy via Zomato Data

> **Location Intelligence project** that uses Zomato restaurant data and a custom-built scoring model to identify the **best cities, localities, and cuisines** for Blinkit to launch its new "Bistro" food service — backed by Python feature engineering and a 4-page interactive Power BI dashboard.

---

## 📌 Business Problem

**Blinkit** (India's quick commerce leader) wants to expand into the food service space with **"Bistro by Blinkit"** — a restaurant/dark kitchen model. The challenge:

> *Which cities, localities, and cuisines give Bistro the highest chance of success?*

This project answers that question using **Zomato's restaurant data** — analysing real demand signals (ratings, votes, delivery availability, competition density) to build a **Blinkit Bistro Success Score** and recommend the top launch locations across 7 major Indian cities.

---

## 🏗️ Project Architecture

```
Zomato Raw Data (CSV)
        ↓
Python (Jupyter Notebook)
  ├── Data Cleaning & Filtering (India only)
  ├── Feature Engineering
  │     ├── Demand Score        = Rating × Votes
  │     ├── Feasibility Score   = (Votes × Rating) / (Total Restaurants + 1)
  │     ├── Saturation Index    = Total Restaurants / Avg Restaurants in City
  │     └── Blinkit Bistro Success Score (weighted composite — MinMaxScaler)
  ├── Reverse Geocoding → Pincode (via geopy / Nominatim)
  └── Export → BLINKITBISTRO.CSV, LOCATIONS.CSV, with_cuisine.csv
        ↓
Power BI Dashboard (BLINKITBISTRO.pbix)
  ├── Page 1: Overview & Key Insights (KPI cards)
  ├── Page 2: City & Locality Intelligence (map + bar charts)
  ├── Page 3: Cuisine Analysis (cost, rating, votes, treemap)
  └── Page 4: Recommended Store Locations (ranked table + map)
```

---

## 🧠 Custom Scoring Model — Blinkit Bistro Success Score

A weighted composite metric built to surface **high-demand, low-saturation** locations:

```python
from sklearn.preprocessing import MinMaxScaler

# Component scores (normalized 0–1)
Demand Score       = Aggregate Rating × Votes
Feasibility Score  = (Votes × Rating) / (Total Restaurants + 1)
Saturation Index   = Total Restaurants / Avg Restaurants in City
Saturation Inverse = 1 - Saturation Index (Norm)   # rewards less crowded areas

# Final composite score
Blinkit Bistro Success Score = (
    0.3 × Delivery Availability Ratio   +  # online delivery readiness
    0.3 × Feasibility Score (Norm)      +  # demand vs. competition
    0.3 × Demand Score (Norm)           +  # raw popularity signal
    0.1 × Saturation Inverse               # market whitespace
)
```

| Component | Weight | What It Captures |
|---|---|---|
| Delivery Availability Ratio | 30% | % of restaurants already doing online delivery in that area |
| Feasibility Score | 30% | Demand relative to how many competitors already exist |
| Demand Score | 30% | Raw popularity — ratings × votes combined |
| Saturation Inverse | 10% | Rewards localities with fewer existing restaurants |

---

## 📊 Dashboard — 4 Pages (BLINKITBISTRO.pbix)

### Page 1 — Overview & Key Insights
| KPI | Value |
|---|---|
| Total Restaurants Analysed | **5,839** |
| Average Cost for Two | **₹897.87** |
| Average Rating | **3.49** |
| Avg Bistro Success Score | **0.22** |
| Average Demand Score | **2,003** |
| Average Feasibility Score | **576.74** |

### Page 2 — City & Locality Intelligence
- **India map** with all analysed city locations plotted
- **Top localities by Demand Score** — Park Street Area (44K), Connaught Place (30K), Indiranagar (24K)
- **Top cities by Bistro Success Score** — Chennai (0.37) → Bangalore (0.31) → Ahmedabad (0.28)
- Interactive **City filter** across all visuals

### Page 3 — Cuisine Analysis
- **Average Cost** — Sri Lankan (₹6K), South American (₹3.8K) lead premium segments
- **Average Rating** — German (4.7), Persian (4.6), Modern Indian (4.4) top rated
- **Average Votes** — North Indian (841), Continental (840) — highest mass demand
- **Treemap** — restaurant count distribution by cuisine type

### Page 4 — Recommended Store Locations
Top localities with **Success Score > 0.45**, recommended cuisines included:

| Locality | City | Success Score | Best Cuisines |
|---|---|---|---|
| Koramangala 5th Block | Bangalore | **0.70** | American, Burger, Cafe |
| Indiranagar | Bangalore | **0.52** | American, Italian, Pizza |
| Hauz Khas Village | Delhi | **0.50** | Asian |
| Hitech City | Hyderabad | **0.48** | American, Burger, Mexican |
| Indiranagar | Bangalore | **0.47** | Asian |
| Banjara Hills | Hyderabad | **0.46** | American, Burger, TexMex |
| Lower Parel | Mumbai | **0.45** | Bakery, Cafe, Desserts |
| Ballygunge | Kolkata | **0.45** | Bengali |

---

## ⚙️ Python Notebook Walkthrough (`ZOMATO_BISTRO.ipynb`)

### Step 1 — Load & Clean
```python
df = pd.read_csv('zomato.csv', encoding='latin-1')
df = df[df['Country Code'] == 1]          # India only
df = df.drop(['Switch to order menu'], axis=1)
```

### Step 2 — Feature Engineering
```python
# Explode multi-cuisine restaurants into individual rows
df['Cuisines'] = df['Cuisines'].str.split(', ')
df = df.explode('Cuisines')

# Demand Score & filter
df['Demand Score'] = df['Aggregate rating'] * df['Votes']
df = df[df['Demand Score'] != 0]          # drop unrated restaurants
```

### Step 3 — Reverse Geocoding
```python
from geopy.geocoders import Nominatim
geolocator = Nominatim(user_agent="geoapi")

def get_pincode(lat, lon):
    try:
        location = geolocator.reverse((lat, lon), timeout=10)
        return location.raw['address'].get('postcode', None)
    except:
        return None

df['Pincode'] = df.apply(lambda row: get_pincode(row['Latitude'], row['Longitude']), axis=1)
# Fill missing via City + Locality group forward-fill
df['Pincode'] = df.groupby(['City','Locality'])['Pincode'].transform(
    lambda x: x.fillna(method='ffill'))
```

### Step 4 — Group & Score
```python
grouped = df.groupby(['Pincode','Locality','City','Cuisines']).agg({
    'Votes': 'sum', 'Aggregate rating': 'mean',
    'Average Cost for two': 'mean', 'Restaurant ID': 'count',
    'Has Online delivery': lambda x: (x == "Yes").mean()
}).reset_index()

grouped['Feasibility Score'] = grouped['Votes'] * grouped['Aggregate rating'] / (grouped['Total Restaurants'] + 1)
grouped['Saturation Index']  = grouped['Total Restaurants'] / avg_per_city

# Normalize and compute final score
from sklearn.preprocessing import MinMaxScaler
scaler = MinMaxScaler()
# ... normalize components ...
grouped['Blinkit Bistro Success Score'] = (
    0.3 * grouped['Delivery Availability Ratio'] +
    0.3 * grouped['Feasibility Score (Norm)']    +
    0.3 * grouped['Demand Score (Norm)']         +
    0.1 * grouped['Saturation Inverse']
)
```

### Step 5 — Export
```python
grouped.to_csv('BLINKITBISTRO.CSV')

# Top locations with cuisines (Score > 0.45)
result = grouped[grouped['Blinkit Bistro Success Score'] > 0.45].groupby(
    ['Pincode','Locality','City']).agg({
    'Blinkit Bistro Success Score': 'max',
    'Cuisines': lambda x: ', '.join(sorted(set(x)))
})
result.to_csv('with_cuisine.csv')
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **Python** | Core analysis & scoring engine |
| **Pandas** | Data cleaning, groupby, feature engineering |
| **NumPy** | Numerical operations |
| **Matplotlib / Seaborn** | EDA visualizations |
| **Geopy (Nominatim)** | Reverse geocoding lat/lon → Pincode |
| **Scikit-learn (MinMaxScaler)** | Normalizing score components |
| **Power BI** | 4-page interactive Location Intelligence Dashboard |
| **Zomato Dataset (CSV)** | Source data — filtered to India (Country Code = 1) |

---

## 💡 Key Insights

- **Chennai** leads all cities in Bistro Success Score (0.37) — strong demand, less saturated than Delhi/Mumbai
- **Koramangala 5th Block, Bangalore** is the single best launch locality with a score of **0.70**
- **Park Street Area, Kolkata** has the highest raw Demand Score (44K votes × rating) of any locality
- **New Delhi** has the lowest Success Score (0.22) despite high volume — market is over-saturated
- **North Indian and Continental** cuisines offer the best volume play — highest average votes (840+)
- **German and Persian** cuisines rate highest (4.7, 4.6) but niche — better for premium micro-formats
- **Sri Lankan** commands the highest average cost (₹6K) — signals premium segment opportunity
- Areas with **high online delivery penetration** consistently score better — delivery-ready customer base

---

## 🚀 How to Run

```bash
# 1. Clone the repo
git clone https://github.com/pawan-111/Blinkit-Bistro-insight.git
cd Blinkit-Bistro-insight

# 2. Install dependencies
pip install pandas numpy matplotlib seaborn geopy scikit-learn jupyter

# 3. Add Zomato dataset
# Place zomato.csv in the project root directory

# 4. Run the notebook
jupyter notebook ZOMATO_BISTRO.ipynb

# 5. Open the dashboard
# → Power BI Desktop (free): open BLINKITBISTRO.pbix
# → No Power BI? View BLINKITBISTRO.pdf for static preview
```

---

## 📂 Files

| File | Description |
|---|---|
| `ZOMATO_BISTRO.ipynb` | Full Python pipeline — cleaning, scoring, geocoding, export |
| `BLINKITBISTRO.pbix` | Interactive Power BI dashboard (4 pages) |
| `BLINKITBISTRO.pdf` | Static PDF export of the full dashboard |
| `README.md` | Project documentation |

---

## 🔑 Concepts Demonstrated

`Location Intelligence` `Custom Composite Scoring` `Feature Engineering` `MinMax Normalization` `Market Saturation Analysis` `Reverse Geocoding` `Demand & Feasibility Modelling` `Cuisine Segmentation` `Power BI Dashboard` `Business Expansion Strategy via Data`

---

## 👤 Author


**Pawan** · Data Analyst / Data Engineer · [GitHub @pawan-111](https://github.com/pawan-111)

> ⭐ Star this repo if it helped you!
