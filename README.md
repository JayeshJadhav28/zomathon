# 🍽️ Zomathon 2026 — PS1: KPT Prediction Improvement
### Improving Kitchen Prep Time (KPT) Prediction Through Signal Quality & System Design

<div align="center">

[![Team](https://img.shields.io/badge/Team-ByteWise-1A237E?style=for-the-badge)](https://github.com/JayeshJadhav28/zomathon)
[![Competition](https://img.shields.io/badge/Zomathon-2026-E53935?style=for-the-badge)](https://github.com/JayeshJadhav28/zomathon)
[![Track](https://img.shields.io/badge/Track-Data%20Science%20%26%20System%20Design-2E7D32?style=for-the-badge)](https://github.com/JayeshJadhav28/zomathon)
[![Problem](https://img.shields.io/badge/Problem%20Statement-1-E65100?style=for-the-badge)](https://github.com/JayeshJadhav28/zomathon)

</div>

---

<div align="center">

| 💰 Rs 620 Cr | ⏱️ 55% | 🚨 83% |
|:---:|:---:|:---:|
| **Annual Cost Saving** | **Rider Wait Reduced** | **Severe Cases Eliminated** |

</div>

---

## 🚀 Open in Google Colab

| Notebook | Description | Open |
|----------|-------------|------|
| `Day1_Problem_Understanding.ipynb` | Root cause analysis of KPT failures | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://tinyurl.com/288r35wa) |
| `Day2_Generate_Data.ipynb` | Synthetic dataset generation (500 restaurants, 50K orders) | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://tinyurl.com/4s576d5w) |
| `Day3_Analysis_and_Charts.ipynb` | Data analysis + 5 evidence charts | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://tinyurl.com/4dryjrzt) |
| `Day4_Simulation.ipynb` | Baseline vs improved system simulation | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://tinyurl.com/3z5z5hkz) |
| `Day5_PDF_Numbers_Reference.ipynb` | All final numbers & calculations reference | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://tinyurl.com/yuehnhhs) |

---

## 📎 Quick Links

| Resource | Link |
|----------|------|
| 📄 Full 25-Page Report (PDF) | [View Report](https://tinyurl.com/6ma484zk) |
| 📦 Dataset (Google Drive) | [merchants.csv + orders.csv](https://tinyurl.com/3xcjx3kw) |
| 🐙 GitHub Repository | [github.com/JayeshJadhav28/zomathon](https://github.com/JayeshJadhav28/zomathon) |

> ⚠️ **Data Disclaimer:** All data in this repository is **synthetically generated** using `numpy` and `pandas`. No proprietary Zomato data is included or referenced.

---

## 📋 Table of Contents

- [Problem Statement](#-problem-statement)
- [The Core Problem](#-the-core-problem)
- [Our Solution — 4 Layer Architecture](#-our-solution--4-layer-architecture)
- [Key Results](#-key-results-simulation-10000-orders)
- [Repository Structure](#-repository-structure)
- [Dataset Schema](#-dataset-schema)
- [How to Run Locally](#-how-to-run-locally)
- [Simulation Methodology](#-simulation-methodology)
- [Scalability Strategy](#-scalability-strategy)
- [Implementation Roadmap](#-implementation-roadmap)
- [Tech Stack](#-tech-stack)
- [Team](#-team)

---

## 🎯 Problem Statement

**Kitchen Prep Time (KPT)** is the elapsed time between order confirmation and food being genuinely ready for pickup. Zomato must predict this **before the food is even made** so riders are dispatched at exactly the right moment.

The current system relies on a single signal — the **Food Order Ready (FOR) button** pressed by restaurant merchants in the Mx app. Our analysis of 50,000 orders reveals this signal is **corrupted in 27% of cases**, causing systematic prediction failures across the entire delivery pipeline.

```
ORDER PLACED → KITCHEN STARTS → FOOD READY ✗ → RIDER ARRIVES → CUSTOMER RECEIVES
                    ◄──────────── KPT ────────────►
                                      ↑
                         FOR button pressed here
                         (often WRONG timing — 58% of restaurants)
```

---

## 🔍 The Core Problem

We identified **3 root causes** behind KPT prediction failure:

| # | Problem | Scale |
|---|---------|-------|
| 01 | **Rider-Influenced FOR Signal** — Merchants press FOR when rider *arrives*, not when food is *ready*, corrupting the KPT label by the full rider travel time | **58% of restaurants** |
| 02 | **No Kitchen-Wide Visibility** — Zomato sees only its own orders; Swiggy, dine-in, and takeaway orders are completely invisible yet share the same kitchen | **59% of kitchen load hidden** |
| 03 | **Human Inconsistency in Labeling** — 300,000+ restaurants press FOR at different times, creating a training dataset with inconsistently generated labels | **27% of orders with >5 min error** |

### Downstream Impact

```
KPT too HIGH → Rider dispatched LATE → food gets cold → bad ratings
KPT too LOW  → Rider dispatched EARLY → waits 7.7 min avg → Rs 620 Cr wasted/year
```

---

## 🏗️ Our Solution — 4 Layer Architecture

Our solution **does not change the KPT model**. It gives the existing model better, cleaner, more complete inputs.

```
┌─────────────────────────────────────────────────────────┐
│                  KPT PREDICTION MODEL                    │
│              (existing model, unchanged)                 │
└───────────────────────┬─────────────────────────────────┘
                        │ Better inputs
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼               ▼
  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ LAYER 1  │   │ LAYER 2  │   │ LAYER 3  │   │ LAYER 4  │
  │ Fix FOR  │   │New Signals│  │ Kitchen  │   │Merchant  │
  │ Signal   │   │          │   │Load Index│   │Workflow  │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘
  ₹0 cost        App-first      Zero hardware   Mx App update
  Deploy now     300K+ merchants Covers 59%      Optional stages
```

---

### Layer 1 — De-Noising the FOR Signal (`FREE TO DEPLOY`)

**Idea 1.1 — FOR Trustworthiness Score**

A per-merchant reliability metric computed from existing data at zero infrastructure cost:

```python
FOR_Reliability_Score = (
    orders_where_FOR_pressed_BEFORE_rider_arrived_by_gte_2min
    / total_orders_past_30_days
)
```

| Score | Tier | % of Restaurants | Action |
|-------|------|-----------------|--------|
| < 0.3 | UNRELIABLE | 58% | Discount FOR; use rider pickup + historical median |
| 0.3–0.7 | AVERAGE | 39% | Moderate trust; cross-check against rider pickup |
| > 0.7 | RELIABLE | 3% | Trust FOR fully as primary label |

**Idea 1.2 — Statistical Outlier Detection**

Per-merchant IQR-based outlier filter applied before any label reaches the training pipeline:

```python
lower_bound = median_kpt - 1.5 * IQR
upper_bound = median_kpt + 1.5 * IQR

if reported_kpt < lower_bound or reported_kpt > upper_bound:
    replace_with = historical_median_kpt  # clean label
```

**Idea 1.3 — Multi-Signal Fusion**

Instead of one corrupted signal, fuse four signals with adaptive weights:

```
True_KPT = w₁×FOR_KPT + w₂×Rider_Pickup_KPT + w₃×Historical_KPT + w₄×App_Activity_KPT
```

| Signal | Unreliable (<0.3) | Average (0.3–0.7) | Reliable (>0.7) |
|--------|:-----------------:|:-----------------:|:---------------:|
| FOR Timestamp (w₁) | 0.10 | 0.35 | 0.60 |
| Rider Pickup (w₂) | 0.40 | 0.30 | 0.20 |
| Historical Median (w₃) | 0.30 | 0.20 | 0.10 |
| App Activity (w₄) | 0.20 | 0.15 | 0.10 |

---

### Layer 2 — Introducing New Signals (`APP-FIRST`)

**Idea 2.1 — App-Based Kitchen Sensing (Zero Hardware)**

Uses the existing Mx app device as a passive kitchen sensor. All signals captured without any new hardware:

| Signal | Source | Indicates |
|--------|--------|-----------|
| Ambient Noise Level | Microphone (on-device ML) | Kitchen activity level |
| Screen Interaction Patterns | App telemetry | Merchant attentiveness |
| Order Acknowledgment Delay | Notification → accept timestamp | Kitchen busyness |
| App Background Switches | App focus telemetry | Swiggy/competitor usage → hidden load |

> 🔒 **Privacy**: Audio processed entirely on-device. Only a numerical score (0–10) is transmitted. No audio is recorded or stored.

**Idea 2.2 — IoT Instrumentation (Top 10% Merchants)**

For ~30,000 high-volume merchants handling 45% of all orders:

```
Sensor → Edge Device → Zomato API → KPT Model Input
```

| Sensor | Cost | Signal |
|--------|------|--------|
| Thermal/IR Sensor | ₹800/unit | Cooking activity detection |
| Weight Sensor | ₹800/unit | Food packaging detection |
| BLE Beacon | ₹100/unit | True rider pickup timestamp |
| Smart Label Printer | Already exists | Packaging start signal |

**ROI:** ₹39 Cr investment → ₹620 Cr annual saving = **15× ROI in Year 1**

**Idea 2.3 — Computer Vision on Existing CCTV**

```
CCTV Feed → Edge Device (Jetson Nano ₹12K) → Kitchen Metrics API → KPT Model
```

4 CV models run on-device: People counter · Activity classifier · Packaging detector · Queue counter

> 🔒 **Privacy**: Raw video never transmitted. Zomato receives only: `{staff_count: 3, activity: "high", bags_waiting: 2}`

---

### Layer 3 — Kitchen Load Index (`ZERO HARDWARE`)

**The Visibility Gap:**

```
What Zomato Sees:        Zomato orders + history + FOR button  =  41% of kitchen load
What Actually Exists:    + Swiggy + Dine-in + Takeaway         = 100% of kitchen load
```

**KLI Formula:**

```python
KLI = f(
    zomato_active_orders,           # 100% known, free, exact
    estimated_competitor_orders,    # pattern anomaly detection
    google_maps_busyness_score,     # free API, foot traffic proxy
    time_of_day_multiplier,         # 1.3× during 12-2pm, 7-10pm
    day_of_week_multiplier,         # 1.2× weekends
    special_event_flag,             # IPL/rain/festivals → 2-3× spike
    merchant_self_reported_busyness # 1-5 slider in Mx app
)
# Updated every 5 minutes per restaurant
# KLI is NOT a model change — it is a new input feature
```

**Merchant Incentive System** to improve self-reporting accuracy:

| Tier | Threshold | Reward |
|------|-----------|--------|
| Accuracy Bonus | >80% accuracy on >80% of orders | ₹500–₹2,000/month |
| Visibility Boost | >80% for 2 months | Better Zomato search ranking |
| Gold Kitchen Badge | >90% for 3 months | Badge displayed on listing |
| Warning | <30% accuracy consistently | Reduced order allocation |

---

### Layer 4 — Merchant Workflow Redesign (`MX APP UPDATE`)

**Idea 4.1 — Granular Kitchen Progress Tracking**

Replaces the single FOR button with an optional 5-stage tracker:

```
BEFORE:  [Order Received] → [Order Accepted] → [Food Ready ✗]
                                                 (single button, often wrong)

AFTER:   [Order Received] → [Prep Started] → [Cooking] → [Packaging] → [Ready ✓]
          AUTO              OPTIONAL          OPTIONAL     OPTIONAL      REQUIRED
```

| Stage Pressed | Signal to Model | Benefit |
|--------------|-----------------|---------|
| Prep Started | Start timer from real zero | Accurate KPT baseline |
| Cooking | Kitchen is active | Confirm not idle |
| Packaging | Food 1–3 min from ready | Pre-dispatch rider earlier |
| Ready ✓ | Ground truth KPT label | Clean training data |

**Idea 4.2 — Item-Level Prep Time Configuration**

Bottom-up KPT calculation from menu item config:

```python
Order_KPT = max(prep_time for each item in order) \
           + packaging_time_fixed   # 1.5 min
           + concurrent_order_penalty  # KLI × 0.3

# Example: Butter Chicken (20) + Naan (5) + Raita (2)
# = max(20, 5, 2) + 1.5 + KLI_penalty = 21.5 min + load adjustment
```

Solves the **cold-start problem** for new restaurants with zero historical data.

---

## 📊 Key Results (Simulation — 10,000 Orders)

| Metric | Baseline | Improved | Reduction |
|--------|:--------:|:--------:|:---------:|
| Avg Rider Wait | 7.7 min | 3.4 min | **▼ 55%** |
| Rider Wait P50 | 6.2 min | 2.6 min | **▼ 58%** |
| Rider Wait P90 | 17.9 min | 8.2 min | **▼ 54%** |
| ETA Error P50 | 7.0 min | 3.1 min | **▼ 56%** |
| ETA Error P90 | 17.9 min | 8.2 min | **▼ 54%** |
| Orders with wait > 5 min | 56.0% | 27.9% | **▼ 50%** |
| Orders with wait > 10 min | 32.0% | 5.6% | **▼ 83%** |

### Annual Cost Saving Calculation

```
Daily Zomato orders:     2,000,000
Rider wait reduced by:   × 4.3 min/order
Daily minutes saved:     = 8,600,000 min/day
Rider cost per minute:   × ₹2
Daily cost saving:       = ₹1.72 Cr/day
Annual cost saving:      × 365 days = ₹620 Cr/year
```

---

## 📁 Repository Structure

```
zomathon/
│
├── 📓 Day1_Problem_Understanding.ipynb    # Root cause analysis & pipeline mapping
├── 📓 Day2_Generate_Data.ipynb            # Synthetic dataset generation
├── 📓 Day3_Analysis_and_Charts.ipynb      # EDA + 5 evidence charts
├── 📓 Day4_Simulation.ipynb               # Discrete event simulation
├── 📓 Day5_PDF_Numbers_Reference.ipynb    # Final numbers & calculations
│
├── 📊 merchants.csv                       # 500 synthetic restaurants
├── 📊 orders.csv                          # 50,000 synthetic orders
│
├── 🖼️ chart1_kpt_distribution.png         # Actual vs FOR-based KPT histogram
├── 🖼️ chart2_error_by_reliability.png     # Label error by reliability tier
├── 🖼️ chart3_rider_wait_heatmap.png       # Rider wait by restaurant size & time
├── 🖼️ chart4_reliability_distribution.png # FOR reliability score distribution
├── 🖼️ chart5_kitchen_visibility.png       # Kitchen load visibility pie chart
├── 🖼️ simulation_results.png              # 4-chart simulation comparison grid
│
└── .gitignore
```

---

## 🗃️ Dataset Schema

### `merchants.csv` — 500 Synthetic Restaurants

| Column | Type | Description |
|--------|------|-------------|
| `merchant_id` | string | Unique restaurant identifier (M0001–M0500) |
| `restaurant_type` | string | `small` / `medium` / `large` |
| `true_reliability_score` | float | Ground truth FOR reliability (0–1) |
| `avg_actual_kpt` | float | True average kitchen prep time (minutes) |
| `has_swiggy` | bool | Whether restaurant is on Swiggy |
| `has_dine_in` | bool | Whether restaurant has dine-in |
| `zomato_visibility_fraction` | float | What fraction of kitchen load Zomato sees |

### `orders.csv` — 50,000 Synthetic Orders

| Column | Type | Description |
|--------|------|-------------|
| `order_id` | string | Unique order identifier |
| `merchant_id` | string | Foreign key to merchants.csv |
| `actual_kpt` | float | Ground truth prep time (minutes) — never observable in reality |
| `for_kpt` | float | FOR-button-reported KPT (noisy label used in training) |
| `label_error` | float | `for_kpt - actual_kpt` — measures corruption |
| `is_peak_hour` | bool | Whether order was during 12–2pm or 7–10pm |
| `rider_travel_time` | float | Time for rider to reach restaurant (minutes) |
| `is_label_corrupted` | bool | Whether `|label_error| > 5 minutes` |

---

## ⚙️ How to Run Locally

### Prerequisites

```bash
Python 3.8+
pip install pandas numpy matplotlib seaborn scikit-learn lightgbm jupyter
```

### Clone & Run

```bash
git clone https://github.com/JayeshJadhav28/zomathon.git
cd zomathon
jupyter notebook
```

### Run in Order

```
1. Day2_Generate_Data.ipynb          # Creates merchants.csv and orders.csv
2. Day3_Analysis_and_Charts.ipynb    # Requires merchants.csv + orders.csv
3. Day4_Simulation.ipynb             # Requires merchants.csv + orders.csv
4. Day5_PDF_Numbers_Reference.ipynb  # Aggregates all final numbers
```

> 💡 **Running in Colab?** Each notebook has a setup cell at the top that downloads the CSVs directly from this GitHub repo — no manual uploads needed.

---

## 🔬 Simulation Methodology

We built a **discrete event simulation** that runs both systems (baseline and improved) across 10,000 orders under identical conditions.

### Simulation Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Total orders | 10,000 | Statistically significant sample |
| Restaurant types | Small / Medium / Large | Reflects real Zomato distribution |
| Peak hour ratio | 40% of orders | Matches actual Zomato peak data |
| FOR reliability distribution | Beta(2, 5) | Fitted to our 500-restaurant dataset |
| Rider travel time | 4–14 minutes | Urban delivery range estimate |

### Baseline vs Improved System

```
BASELINE SYSTEM                    IMPROVED SYSTEM (OUR SOLUTION)
─────────────────────────          ──────────────────────────────────
✗ FOR timestamp as only label  →   ✓ FOR Trustworthiness Score applied
✗ No reliability filtering     →   ✓ Statistical outlier filtering
✗ No outlier detection         →   ✓ Multi-signal fusion (4 signals)
✗ No Kitchen Load Index        →   ✓ Kitchen Load Index integrated
✗ No peak hour adjustment      →   ✓ Peak hour multiplier factored in
✗ No multi-signal fusion       →   ✓ Item-level prep time baseline
= High noise, high rider wait  =   Low noise, low rider wait
```

---

## 📈 Scalability Strategy

A 3-tier deployment strategy for 300,000+ merchants:

| Tier | Target | Solutions | Cost | Timeline | Coverage |
|------|--------|-----------|------|----------|----------|
| **Tier 1** | ALL 300,000+ merchants | FOR reliability score, outlier detection, multi-signal fusion, KLI v1, Google Maps API | **₹0** | 4–6 weeks | 100% merchants |
| **Tier 2** | Top 30% by volume (~90,000) | Mx app tracker, item-level config, accuracy bonus, app sensing | **~₹15 Cr** | 3–6 months | ~65% of orders |
| **Tier 3** | Top 10% by volume (~30,000) | IoT sensors, edge CV on CCTV, full multi-signal fusion | **₹39 Cr** | 6–12 months | ~45% of orders |

**Total: ₹54 Cr investment → ₹620 Cr annual saving = 11× ROI in Year 1**

---

## 🗓️ Implementation Roadmap

```
M1──────M3──────────────M6──────────────────────────M12
│  Phase 1  │      Phase 2      │          Phase 3          │
│  Software │  App + Incentives │   IoT + AI Vision         │
│   ₹0      │     ~₹15 Cr      │        ₹39 Cr             │
│  100% cvg │    65% orders     │       45% orders          │
│ 30% label↑│   45% wait↓       │      55% wait↓            │
```

### Phase 1 — Months 0–3 (Software Foundation, Zero Cost)
- ✅ FOR Reliability Score for all 300,000+ merchants
- ✅ Statistical outlier detection pipeline
- ✅ Kitchen Load Index v1 (Zomato orders + time factors)
- ✅ Google Maps busyness API as KPT feature

### Phase 2 — Months 3–6 (App Redesign & Incentives)
- ✅ Mx app progress tracker (optional stages)
- ✅ Item-level prep time config in menu onboarding
- ✅ Accuracy bonus programme for Tier 2 merchants
- ✅ App-based kitchen sensing rollout (opt-in)

### Phase 3 — Months 6–12 (IoT + Full Multi-Signal Fusion)
- ✅ IoT sensor kit shipped to top 30,000 merchants
- ✅ Edge CV pilot at 100 volunteer restaurants
- ✅ Full multi-signal fusion model in production
- ✅ Self-calibrating item prep times across all menus

---

## 🛠️ Tech Stack

| Category | Tools |
|----------|-------|
| **Language** | Python 3.8+ |
| **Data Generation** | `numpy`, `pandas`, `scipy` |
| **Analysis & Visualization** | `matplotlib`, `seaborn` |
| **Machine Learning** | `scikit-learn`, `lightgbm` |
| **Simulation** | Custom discrete event simulation (pure Python) |
| **Notebooks** | Jupyter Notebook / Google Colab |
| **Distribution Fitting** | `scipy.stats.beta` (Beta(2,5) for FOR reliability) |

---

## 👥 Team

**Team ByteWise** — Dnyanshree Institute of Engineering & Technology

| Name | Role |
|------|------|
| Jayesh Jadhav | Data Science & System Design |
| Omkar Khade | Data Science & Analysis |
| Vinayak Kharade | System Design & Simulation |

---

## ⚠️ Limitations

| Limitation | Mitigation |
|-----------|------------|
| Synthetic dataset — real distributions may differ | Methodology validated on real data before production |
| Competitor order estimation is inferred, not direct | Used as 1 of 6 KLI inputs, not primary signal |
| Merchant behaviour change requires habit adoption | All intermediate stages optional; Tier 1 needs zero merchant action |
| IoT sensors need field maintenance | Tier 3 targets only highest-ROI merchants |

---

## 🔮 Future Work

- **LLM Anomaly Detection** — Fine-tuned model to detect unusual FOR patterns from merchant chat logs
- **Federated Learning** — Train KPT models locally on restaurant edge devices, privacy-preserving
- **Real-Time ETA Correction** — Update customer ETA dynamically mid-cook using live kitchen signals

---

## 📜 License & Data Notice

This repository was created for **Zomathon 2025**, organized by Coding Ninjas in collaboration with Eternal Limited (Zomato). All ideas, designs, and materials created during the competition are subject to the competition's Terms & Conditions regarding confidentiality and intellectual property.

**All dataset files (`merchants.csv`, `orders.csv`) are 100% synthetically generated. No proprietary or real Zomato data is used, stored, or referenced anywhere in this repository.**

---

<div align="center">

**Built in 5 days · 500 restaurants · 50,000 orders · 4 Python notebooks · 25-page report**

[![GitHub](https://img.shields.io/badge/GitHub-JayeshJadhav28-181717?style=flat-square&logo=github)](https://github.com/JayeshJadhav28/zomathon)
[![Report](https://img.shields.io/badge/Full%20Report-25%20Pages-E53935?style=flat-square)](https://tinyurl.com/6ma484zk)
[![Dataset](https://img.shields.io/badge/Dataset-Google%20Drive-4285F4?style=flat-square&logo=googledrive)](https://tinyurl.com/3xcjx3kw)

</div>
