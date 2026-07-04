# PharmaBI: Pharmaceutical Sales Intelligence & Forecasting Platform

> A business intelligence project built for pharmaceutical retail analytics — covering data engineering, sales performance analysis, and time-series demand forecasting across a pan-European pharmacy network.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Business Objectives](#2-business-objectives)
3. [Architecture](#3-architecture)
4. [Technology Stack](#4-technology-stack)
5. [Dataset Overview](#5-dataset-overview)
6. [Data Dictionary](#6-data-dictionary)
7. [Analysis Modules](#7-analysis-modules)
8. [Key Findings](#8-key-findings)
9. [Tableau Dashboard](#9-tableau-dashboard)
10. [Forecasting Model](#10-forecasting-model)
11. [Repository Structure](#11-repository-structure)
12. [How to Run](#12-how-to-run)

---

## 1. Project Overview

PharmaBI is a data analytics project that simulates the kind of sales intelligence platform used by commercial analytics teams in pharmaceutical retail organisations. The platform takes transactional pharmacy sales data, processes it through a cloud-based pipeline, and surfaces business insights through an interactive Tableau dashboard and a Prophet-based sales forecast.

**Scope:**
- 62,139 sales transactions across 120 pharmacies in 8 European countries
- 731 days of data spanning January 2024 – December 2025
- 220 products across 5 categories: OTC, Prescription, Personal Care, Wellness, and Medical Devices

---

## 2. Business Objectives

This platform is built around six practical business questions a sales or commercial analytics team would actually ask:

| # | Business Question | Analysis Module |
|---|---|---|
| 1 | Which product categories and brands drive the most revenue and margin? | Revenue & Margin Analysis |
| 2 | How do generic products compare to branded equivalents in volume and profitability? | Generic vs. Branded Analysis |
| 3 | What is the measurable impact of promotional activity on units sold and margin? | Promo Impact Analysis |
| 4 | How does sales performance vary by country, region, and pharmacy type? | Geographic Segmentation |
| 5 | Are there seasonal demand patterns that could inform stock planning? | Seasonality Analysis |
| 6 | What is the projected revenue for the next quarter, by product category? | Demand Forecasting (Prophet) |

---

## 3. Architecture

```
Raw Data (CSV / Excel)
        │
        ▼
┌─────────────┐
│   AWS S3    │  ← Raw data storage (one prefix per table)
└─────────────┘
        │
        ▼
┌─────────────┐
│  AWS Glue   │  ← ETL: type casting, null checks, dimensional joins
└─────────────┘
        │
        ▼
┌─────────────┐
│ AWS Athena  │  ← SQL queries on transformed data
└─────────────┘
        │
        ▼
┌──────────────────────┐
│  Python              │  ← Analysis, aggregation, forecasting
│  Pandas / Prophet    │
└──────────────────────┘
        │
        ├──────────────────────────▶ Tableau Dashboard
        └──────────────────────────▶ Prophet Forecast Output
```

**How the pipeline works:**
- Raw files are uploaded to **AWS S3**, organised by table (`fact/`, `dim_date/`, `dim_pharmacy/`, `dim_product/`)
- **AWS Glue** handles schema validation, type enforcement, and joining dimensions to the fact table where needed
- **AWS Athena** provides serverless SQL querying directly on S3 — used for aggregations before pulling into Python
- Cleaned data is loaded into **Jupyter Notebooks** for analysis and forecasting
- Outputs feed into **Tableau** for dashboards and **Prophet** for the forward forecast

---

## 4. Technology Stack

| Layer | Tool | Purpose |
|---|---|---|
| Cloud Storage | AWS S3 | Raw and processed data storage |
| ETL | AWS Glue | Schema validation, transformation |
| Query Engine | AWS Athena | SQL on S3 |
| Analysis | Python 3.x | Data manipulation and analysis |
| Libraries | Pandas, NumPy | Aggregation and wrangling |
| Forecasting | Prophet (Meta) | Time-series demand forecasting |
| Visualisation | Tableau | Interactive dashboards |
| Environment | Jupyter Notebooks | Analysis and documentation |

---

## 5. Dataset Overview

The dataset uses a **star schema** — one central fact table linked to three dimension tables. This is a standard structure for retail analytics data warehouses.

| Table | Rows | Description |
|---|---|---|
| `FactSales` | 62,139 | Transaction-level records with revenue, cost, margin, and promo flag |
| `DimDate` | 731 | Date dimension covering 1 Jan 2024 – 31 Dec 2025 |
| `DimPharmacy` | 120 | Pharmacy attributes across 8 European countries |
| `DimProduct` | 220 | Product attributes across 5 categories |

**Geographic coverage:**

| Country | Pharmacies |
|---|---|
| Germany | 22 |
| France | 20 |
| Italy | 18 |
| Belgium | 14 |
| Poland | 13 |
| Spain | 12 |
| Netherlands | 11 |
| Austria | 10 |

**Pharmacy breakdown:**
- By Type: Urban (50), Suburban (47), Rural (23)
- By Size: Large (25), Medium (53), Small (42)

**Product breakdown:**
- By Category: OTC (61), Personal Care (47), Prescription (47), Wellness (41), Medical Devices (24)
- By Type: Branded (187), Generic (33)
- Active: 185 | Discontinued: 35

**Promotional activity:**
- Non-promo transactions: 54,708 (88%)
- Promo transactions: 7,431 (12%)

---

## 6. Data Dictionary

### FactSales

| Column | Type | Description |
|---|---|---|
| SalesID | VARCHAR | Unique transaction identifier (e.g. S0000001) |
| DateKey | INT | Foreign key to DimDate |
| PharmacyID | VARCHAR | Foreign key to DimPharmacy |
| ProductID | VARCHAR | Foreign key to DimProduct |
| UnitsSold | INT | Units sold per transaction (range: 1–25) |
| RevenueEUR | FLOAT | Revenue generated in EUR |
| CostEUR | FLOAT | Cost of goods sold in EUR |
| MarginEUR | FLOAT | Gross margin in EUR (Revenue − Cost) |
| PromoFlag | VARCHAR | Whether a promotional discount was applied (Yes / No) |

### DimDate

| Column | Type | Description |
|---|---|---|
| DateKey | INT | Primary key (format: YYYYMMDD) |
| Date | DATE | Calendar date |
| Year | INT | Calendar year (2024 or 2025) |
| Quarter | INT | Quarter number (1–4) |
| MonthNumber | INT | Month number (1–12) |
| MonthName | VARCHAR | Month name (e.g. January) |
| YearMonth | VARCHAR | Year-month label (e.g. 2024-01) |

### DimPharmacy

| Column | Type | Description |
|---|---|---|
| PharmacyID | VARCHAR | Primary key (e.g. PH0001) |
| PharmacyName | VARCHAR | Trading name of the pharmacy |
| Country | VARCHAR | Country of operation (8 countries) |
| Region | VARCHAR | Sub-national region |
| City | VARCHAR | City of operation |
| PharmacyType | VARCHAR | Urban / Suburban / Rural |
| OpenDate | DATE | Date the pharmacy opened |
| StoreSizeBand | VARCHAR | Store size: S / M / L |
| Latitude | FLOAT | Latitude coordinate |
| Longitude | FLOAT | Longitude coordinate |

### DimProduct

| Column | Type | Description |
|---|---|---|
| ProductID | VARCHAR | Primary key (e.g. PR0001) |
| ProductName | VARCHAR | Full product name including brand and pack details |
| Category | VARCHAR | OTC / Prescription / Personal Care / Wellness / Medical Devices |
| Brand | VARCHAR | Brand name |
| IsGeneric | VARCHAR | Whether the product is generic (Yes / No) |
| PackSize | VARCHAR | Pack size description (e.g. 30 tablets, 200 ml) |
| ListPriceEUR | FLOAT | Standard list price in EUR |
| StandardCostEUR | FLOAT | Standard cost of goods in EUR |
| LaunchDate | DATE | Date the product was launched |
| IsDiscontinued | VARCHAR | Whether the product is discontinued (Yes / No) |
| DiscontinuedDate | DATE | Date of discontinuation (null if still active) |

---

## 7. Analysis Modules

### Module 1 — Revenue & Margin Analysis
**Business Question:** Which categories, brands, and geographies generate the most revenue and deliver the highest margin?

**Approach:**
- Join `FactSales` with `DimProduct` and `DimPharmacy`
- Aggregate `RevenueEUR` and `MarginEUR` by Category, Country, and PharmacyType
- Calculate margin percentage: `(MarginEUR / RevenueEUR) × 100`
- Rank by revenue and by margin % to identify where revenue and profitability align — and where they diverge

**Key Metrics:** Total Revenue (EUR), Total Margin (EUR), Margin %, Revenue by Category, Revenue by Country

**Status:** `[TBD — complete after analysis]`

---

### Module 2 — Generic vs. Branded Performance
**Business Question:** Do generic products underperform branded products on revenue, or do they compensate through volume?

**Approach:**
- Filter `DimProduct` by `IsGeneric` and join to `FactSales`
- Compare average `UnitsSold`, `RevenueEUR`, and `MarginEUR` per transaction between generic and branded
- Break down by Category to see where generics are concentrated (note: 33 generics vs 187 branded products)

**Key Metrics:** Avg Units Sold per transaction (Generic vs Branded), Revenue per Unit, Margin per Unit

**Status:** `[TBD — complete after analysis]`

---

### Module 3 — Promotional Impact Analysis
**Business Question:** Does promotional activity drive meaningful uplift in units sold, or does it primarily erode margin?

**Approach:**
- Split `FactSales` by `PromoFlag` (Yes / No)
- Compare average `UnitsSold`, `RevenueEUR`, and `MarginEUR` between promo and non-promo transactions
- Note: promos account for ~12% of all transactions (7,431 of 62,139) — a realistic, non-trivial sample

**Key Metrics:** Avg Units Sold (Promo vs Non-Promo), Avg Revenue per Transaction, Avg Margin per Transaction, Margin difference

**Status:** `[TBD — complete after analysis]`

---

### Module 4 — Geographic & Pharmacy Segmentation
**Business Question:** How does sales performance vary across countries and pharmacy types?

**Approach:**
- Aggregate revenue and units by `Country`, `PharmacyType`, and `StoreSizeBand`
- Identify top-performing markets and store formats
- Use `Latitude` / `Longitude` in Tableau for a geographic map view

**Key Metrics:** Revenue by Country, Revenue by PharmacyType, Revenue by StoreSizeBand, Top 10 Pharmacies by Revenue

**Status:** `[TBD — complete after analysis]`

---

### Module 5 — Seasonality & Trend Analysis
**Business Question:** Are there seasonal patterns in sales that could inform stock planning decisions?

**Approach:**
- Join `FactSales` with `DimDate` and aggregate revenue by `MonthName`, `Quarter`, and `Year`
- Plot monthly revenue trend across the full 2024–2025 period
- Compare 2024 vs 2025 same-period performance
- Look at category-level seasonality — OTC in particular may show cold/flu seasonal peaks

**Key Metrics:** Monthly Revenue Trend, Quarter-on-Quarter Change, YoY Comparison, Category Seasonality Pattern

**Status:** `[TBD — complete after analysis]`

---

### Module 6 — Demand Forecasting (Prophet)
**Business Question:** What is the projected revenue for the next quarter, broken down by product category?

**Approach:**
- Aggregate `FactSales` to a daily revenue time series per category
- Train one Prophet model per category on 2024–2025 actuals
- Hold out the last 3 months (Oct–Dec 2025) for validation
- Forecast 90 days ahead (Q1 2026) with confidence intervals
- Evaluate using MAE and MAPE on the holdout period

**Key Metrics:** MAE, MAPE, Forecast vs Actuals chart, 90-Day Revenue Projection by Category

**Status:** `[TBD — complete after analysis]`

---

## 8. Key Findings

> ⚠️ **Reminder to self:** Fill this section after all modules are complete. These are the most important lines a recruiter reads — make every finding specific and metric-driven. No vague statements.

- **Revenue split by category:** `[TBD]` — which category leads, what % of total
- **Margin leaders:** `[TBD]` — which category or store type has the best margin %
- **Generic vs Branded:** `[TBD]` — how generics compare on revenue per unit and volume
- **Promo impact:** `[TBD]` — does promo drive units uplift, and at what margin cost
- **Top market:** `[TBD]` — leading country by revenue and any notable regional pattern
- **Seasonality:** `[TBD]` — which month or quarter peaks, and for which category
- **Forecast accuracy:** `[TBD]` — MAPE achieved on holdout, Q1 2026 projection headline

---

## 9. Tableau Dashboard

> 🔗 **Dashboard Link:** `[TBD — Tableau Public URL once published]`

| Dashboard View | What It Shows |
|---|---|
| Executive Summary | KPI tiles: Total Revenue, Total Margin, Margin %, Units Sold |
| Revenue by Category | Ranked bar chart with margin % overlay |
| Geographic Map | Pharmacy locations coloured by revenue band |
| Generic vs Branded | Side-by-side comparison of volume, revenue, and margin |
| Promo Impact | Promo vs non-promo performance comparison |
| Seasonal Trends | Monthly revenue line by category (2024–2025) |
| Forecast View | Prophet output: actuals, forecast line, confidence interval |

---

## 10. Forecasting Model

**Model:** Facebook Prophet

**Why Prophet:** Handles daily retail sales data well — decomposes trend and seasonality automatically, and is robust to missing days and outliers.

**Data format:** Daily `RevenueEUR` aggregated per category, structured as `ds` (date) and `y` (revenue) as required by Prophet.

**Training period:** January 2024 – September 2025

**Holdout period:** October – December 2025 (used for model validation)

**Forecast horizon:** 90 days (Q1 2026)

**Evaluation metrics:**

| Metric | Result |
|---|---|
| MAE | `[TBD]` |
| MAPE | `[TBD]` |
| RMSE | `[TBD]` |

---

## 11. Repository Structure

```
pharma-bi/
│
├── data/
│   ├── raw/                        # Source files (FactSales, DimDate, etc.)
│   └── processed/                  # Cleaned outputs after Glue / Athena
│
├── notebooks/
│   ├── 01_eda.ipynb                # Exploratory Data Analysis
│   ├── 02_revenue_margin.ipynb     # Module 1: Revenue & Margin
│   ├── 03_generic_vs_branded.ipynb # Module 2: Generic vs Branded
│   ├── 04_promo_impact.ipynb       # Module 3: Promotional Analysis
│   ├── 05_geo_segmentation.ipynb   # Module 4: Geographic Segmentation
│   ├── 06_seasonality.ipynb        # Module 5: Seasonality & Trends
│   └── 07_forecasting.ipynb        # Module 6: Prophet Forecasting
│
├── src/
│   ├── etl/
│   │   └── glue_job.py             # AWS Glue ETL script
│   ├── queries/
│   │   └── athena_queries.sql      # Reusable Athena SQL
│   └── forecast/
│       └── prophet_model.py        # Prophet training and forecast pipeline
│
├── tableau/
│   └── pharma_bi.twbx              # Packaged Tableau workbook
│
├── requirements.txt
└── README.md
```

---

## 12. How to Run

### Prerequisites
- Python 3.9+
- AWS credentials configured (S3, Glue, Athena access)
- Tableau Desktop or Tableau Public

```bash
# Clone the repository
git clone https://github.com/[your-username]/pharma-bi.git
cd pharma-bi

# Install dependencies
pip install -r requirements.txt

# Configure AWS
aws configure

# Launch notebooks
jupyter notebook
# Run in order: 01 → 02 → ... → 07
```

---

## Author

**[Your Name]**
Final Year Project · Data Analytics
`[University Name]` · `[Year]`

📧 `[email]`
🔗 LinkedIn: `[TBD]`
🔗 Tableau Public: `[TBD]`

---

> **📌 Before you submit or share this repo:** Complete Section 8 (Key Findings) and Section 10 (model metrics) with real numbers from your analysis. These two sections are what differentiate a strong project from a template.