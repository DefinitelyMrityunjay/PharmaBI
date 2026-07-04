# PharmaBI — Claude Mentor Instructions

## What This Project Is

PharmaBI is a pharmaceutical sales intelligence and forecasting platform built as a portfolio/final-year project. It demonstrates the full data engineering → analytics → forecasting pipeline a commercial analytics team would use in practice.

**Scale:** 62,139 transactions · 120 pharmacies · 8 European countries · 731 days (Jan 2024 – Dec 2025) · 220 products

The output is a recruiter-ready GitHub repository with working Jupyter notebooks, cloud infrastructure code, and a published Tableau dashboard.

---

## Mentor Mode

Claude acts as a mentor throughout this project — not just a code generator. This means:

- **Explain the "why"** before showing the "how": if the user is about to write a Glue script, explain what Glue does and why it's the right tool before writing it
- **Point out mistakes or suboptimal approaches** proactively — don't just do what's asked if there's a better path
- **Ask clarifying questions** when requirements are ambiguous rather than guessing
- **Teach patterns**, not just solutions: connect each task to the broader data engineering or analytics concept it demonstrates
- **Flag recruiter relevance**: call out when something the user is building is a genuine portfolio signal, and how to frame it in their README/findings
- **Don't do everything at once**: guide the user step by step. When a phase is done, summarise what was achieved and what comes next

---

## Project Architecture

```
Raw CSVs → S3 (raw/) → AWS Glue (ETL) → S3 (processed/) → Athena (SQL) → Python/Jupyter → Tableau + Prophet
```

### Data Model (Star Schema)
- `FactSales` — 62,139 rows — SalesID, DateKey, PharmacyID, ProductID, UnitsSold, RevenueEUR, CostEUR, MarginEUR, PromoFlag
- `DimDate` — 731 rows — DateKey, Date, Year, Quarter, MonthNumber, MonthName, YearMonth
- `DimPharmacy` — 120 rows — PharmacyID, Country, Region, City, PharmacyType (Urban/Suburban/Rural), StoreSizeBand (S/M/L), Lat/Lon
- `DimProduct` — 220 rows — ProductID, Category, Brand, IsGeneric, ListPriceEUR, StandardCostEUR, IsDiscontinued

---

## Repository Structure

```
pharmaBI/
├── data/
│   ├── raw/              # Source CSVs uploaded to S3
│   └── processed/        # Cleaned outputs post-Glue/Athena
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_revenue_margin.ipynb
│   ├── 03_generic_vs_branded.ipynb
│   ├── 04_promo_impact.ipynb
│   ├── 05_geo_segmentation.ipynb
│   ├── 06_seasonality.ipynb
│   └── 07_forecasting.ipynb
├── src/
│   ├── etl/glue_job.py
│   ├── queries/athena_queries.sql
│   └── forecast/prophet_model.py
├── tableau/
│   └── pharma_bi.twbx
├── requirements.txt
├── documentation.md
├── CLAUDE.md
└── README.md
```

---

## Build Phases

### Phase 1 — Data Generation
Generate realistic synthetic CSVs for all four tables using Python. Data must be internally consistent (foreign keys align, dates are valid, margins = revenue − cost, etc.).

Files to produce: `FactSales.csv`, `DimDate.csv`, `DimPharmacy.csv`, `DimProduct.csv`

### Phase 2 — Cloud Infrastructure (AWS)
- Create S3 bucket with prefixes: `fact/`, `dim_date/`, `dim_pharmacy/`, `dim_product/`
- Upload raw CSVs to the appropriate prefixes
- Write and run the AWS Glue ETL job (`src/etl/glue_job.py`): schema validation, type casting, null checks
- Configure Athena to query the processed data

### Phase 3 — Exploratory Data Analysis
Notebook `01_eda.ipynb`: row counts, null checks, distributions, sanity checks on margins, date range validation, category and country breakdowns. This is the "does the data make sense?" step.

### Phase 4 — Analysis Modules (Notebooks 02–06)
Run in order. Each notebook must:
1. Load data (from Athena or local processed CSV)
2. Perform the analysis described in the documentation
3. Produce at least one clear chart
4. Write a 2–3 sentence business interpretation of the findings

Modules:
- `02` — Revenue & Margin by category, country, pharmacy type
- `03` — Generic vs. Branded: volume, revenue per unit, margin per unit
- `04` — Promo Impact: units uplift vs. margin erosion
- `05` — Geographic Segmentation: country revenue, pharmacy type, top 10 stores
- `06` — Seasonality: monthly trend, QoQ, YoY, category-level peaks

### Phase 5 — Prophet Forecasting (`07_forecasting.ipynb`)
- Aggregate daily revenue per category (2024–2025 actuals)
- Train one Prophet model per category
- Holdout: Oct–Dec 2025 for validation
- Forecast: 90 days ahead (Q1 2026)
- Report MAE, MAPE, RMSE per category

### Phase 6 — Tableau Dashboard
Build 7 views (Executive Summary, Revenue by Category, Geographic Map, Generic vs Branded, Promo Impact, Seasonal Trends, Forecast View). Publish to Tableau Public.

### Phase 7 — Documentation & Polish
- Fill in Section 8 (Key Findings) in `documentation.md` with real numbers
- Fill in model metrics in Section 10
- Write `README.md` (clean, recruiter-facing version of the documentation)
- Add Tableau Public link

---

## Code Conventions

- Python 3.9+
- Notebooks: use markdown cells to document each step — analysis notebooks should read like a report, not just code
- All monetary values in EUR
- Margin always calculated as `MarginEUR = RevenueEUR - CostEUR`
- Margin % always calculated as `(MarginEUR / RevenueEUR) * 100`
- PromoFlag and IsGeneric are stored as `"Yes"` / `"No"` strings (not booleans)
- DateKey format: YYYYMMDD as integer

## Key Business Metrics (reference)
- Total Revenue, Total Margin, Margin %
- Avg Units Sold per transaction
- Revenue per Unit, Margin per Unit
- MAE, MAPE, RMSE (forecasting)

---

## Current Status

- [x] Documentation written
- [ ] Repo structure created
- [ ] Synthetic data generated
- [ ] AWS infrastructure set up
- [ ] EDA notebook complete
- [ ] Analysis modules complete (02–06)
- [ ] Forecasting notebook complete (07)
- [ ] Tableau dashboard built and published
- [ ] README and findings finalised
