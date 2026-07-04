# Tableau Dashboard Build Guide
**PharmaBI: Pharmaceutical Sales Intelligence & Forecasting Platform**

**Tool:** Tableau Public (free)
**Data source:** Pharmacy_data.xlsx (star schema — 4 tables)
**Output:** 7-view interactive dashboard published to Tableau Public

---

## Overview

You need two data connections total:

| Connection | Source | Used for |
|---|---|---|
| Main | Pharmacy_data.xlsx | All 6 analysis views |
| Forecast | q1_2026_forecast.csv | Forecast view only |

That's it. Everything else comes from the Excel file directly.

---

## Part 1 — Connect the Excel File

### 1a — Open Tableau Public and connect

1. Open Tableau Public
2. On the start screen under **Connect** → click **Microsoft Excel**
3. Navigate to your project folder and select `Pharmacy_data.xlsx`
4. Tableau opens the **Data Source** tab

### 1b — Build the star schema

You will see your 4 sheets listed in the **Files / Sheets** panel on the left under the filename.

Drag them into the canvas in this order:

**First — drag `FactSales` into the canvas**

The canvas shows one table — FactSales is your fact table and goes in first.

**Second — drag `DimProduct` into the canvas**

Drop it next to FactSales. Tableau will automatically detect the relationship on `ProductID` and draw a line between them. If it doesn't auto-detect:
- Click the relationship line → Edit Relationship
- Left field: `ProductID` (FactSales)
- Right field: `ProductID` (DimProduct)
- Click OK

**Third — drag `DimPharmacy` into the canvas**

Same process — Tableau should auto-relate on `PharmacyID`. If not, set it manually.

**Fourth — drag `DimDate` into the canvas**

Auto-relates on `DateKey`. If not, set manually.

Your canvas should look like this:

```
         DimDate
            |
            | DateKey
            |
DimPharmacy ─── FactSales ─── DimProduct
 PharmacyID         PharmacyID   ProductID
```

### 1c — Verify the preview

At the bottom of the Data Source tab you'll see a data preview. Check:
- Rows are showing (should be 62,139)
- You can see columns from multiple tables

### 1d — Check data types

Click each column header in the preview and confirm these types:

| Column | Should be | Fix if wrong |
|---|---|---|
| `Date` (DimDate) | Date | Click column icon → Date |
| `RevenueEUR`, `MarginEUR`, `CostEUR` | Number (decimal) | Click column icon → Number (decimal) |
| `UnitsSold`, `Transactions` | Number (whole) | Click column icon → Number (whole) |
| `Latitude` | Geographic → Latitude | Click icon → Geographic Role → Latitude |
| `Longitude` | Geographic → Longitude | Click icon → Geographic Role → Longitude |
| `Category`, `Country`, `PharmacyType` | String (Abc) | Should be automatic |

### 1e — Name this data source

At the top left of the Data Source tab you'll see the connection name — click it and rename to `PharmaBI Main`

---

## Part 2 — Add the Forecast Data Source

This is the only separate CSV you need.

1. In the Data Source tab, click **Add** next to Connections (top left)
2. Select **Text File**
3. Navigate to `data/processed/q1_2026_forecast.csv`
4. Drag the file into the canvas
5. Rename this connection to `PharmaBI Forecast`

Check columns:
- `date` → Date type
- `projected_revenue`, `lower_bound_95`, `upper_bound_95` → Number (decimal)
- `category` → String

---

## Part 3 — Calculated Fields

Create these before building any views. They are reused across multiple sheets.

To create: go to any sheet → right-click in the Data panel → **Create Calculated Field**

### Field 1 — Margin %
**Name:** `Margin %`
```
SUM([MarginEUR]) / SUM([RevenueEUR])
```
Format as %: right-click the field in the Data panel → Default Properties → Number Format → Percentage → 1 decimal place

### Field 2 — Promo Label
**Name:** `Promo Label`
```
IF [PromoFlag] = "Yes" THEN "Promotional" ELSE "Non-Promotional" END
```

### Field 3 — Product Type
**Name:** `Product Type`
```
IF [IsGeneric] = "Yes" THEN "Generic" ELSE "Branded" END
```

### Field 4 — Store Size Label
**Name:** `Store Size`
```
IF [StoreSizeBand] = "L" THEN "Large"
ELSEIF [StoreSizeBand] = "M" THEN "Medium"
ELSE "Small"
END
```

---

## Part 4 — Colour Palette

Apply these hex codes consistently across every chart. Category colours must match your Python notebook charts.

To set custom hex colours in Tableau:
1. Click **Color** in the Marks card
2. Click **Edit Colors**
3. Click a dimension member → **More Colors** → enter the hex code

| Item | Hex |
|---|---|
| Prescription | `#2C6FAC` |
| OTC | `#3A9E6F` |
| Wellness | `#E07B3A` |
| Personal Care | `#9B5EA0` |
| Medical Devices | `#C0392B` |
| Branded | `#2C6FAC` |
| Generic | `#E07B3A` |
| Promotional | `#E07B3A` |
| Non-Promotional | `#2C6FAC` |
| Urban | `#2C6FAC` |
| Suburban | `#3A9E6F` |
| Rural | `#E07B3A` |

---

## Part 5 — Building the 7 Views

Create a new sheet for each view. Right-click the sheet tab at the bottom → **Rename Sheet** after building each one.

All sheets use **PharmaBI Main** as the data source except the Forecast view which uses **PharmaBI Forecast**.

To switch data source on a sheet:
- Top menu → **Data** → click the source you want → tick appears next to it

---

### View 1 — KPI Summary

**Sheet name:** `KPI Summary`
**Data source:** PharmaBI Main

This view shows 4 headline numbers: Total Revenue, Total Margin, Margin %, Total Transactions.

**Build steps:**

1. Double-click `RevenueEUR` in the Data panel — a single number appears
2. In the Marks card, change the mark type dropdown from Automatic to **Text**
3. You now have a single large number — this is your Total Revenue KPI
4. Format it: right-click the number on canvas → Format → Numbers → Currency, 0 decimal, € prefix

Repeat for 3 more sheets:
- `KPI Margin` — using `MarginEUR`
- `KPI Margin Pct` — using the `Margin %` calculated field
- `KPI Transactions` — using `SalesID` with aggregation set to COUNT

> These 4 sheets will be placed as tiles in the dashboard later.

---

### View 2 — Revenue by Category

**Sheet name:** `Revenue by Category`
**Data source:** PharmaBI Main

**Build steps:**

1. Drag `Category` (from DimProduct) → **Rows** shelf
2. Drag `RevenueEUR` → **Columns** shelf
3. Tableau creates a horizontal bar chart automatically
4. Click the **sort descending** button in the toolbar — Prescription moves to top

**Add margin % as colour:**

5. Drag `Margin %` (your calculated field) → **Color** in Marks card
6. Click Color → Edit Colors → select **Blue-Teal** sequential palette
7. Click **Reversed** so higher margin = darker colour

**Add revenue labels:**

8. Drag `RevenueEUR` → **Label** in Marks card
9. Click Label → tick **Show Mark Labels**
10. Format the label: click the label pill → Format → Currency, 0 decimal, €

**Add reference line for overall margin:**

11. Right-click the colour legend → **Add Reference Line**
12. Set to constant value: `0.28` (28% overall margin)
13. Label it: `Overall 28.0%`

**Format:**

14. Right-click x-axis → Format → Numbers → Currency, 0 decimal, €
15. Format → Lines → set Row Divider and Column Divider to None
16. Right-click the sheet title → Edit Title → type `Revenue & Margin % by Category`

---

### View 3 — Geographic Map

**Sheet name:** `Geographic Map`
**Data source:** PharmaBI Main

**Build steps:**

1. Double-click `Latitude` in the Data panel — a map appears automatically
2. Double-click `Longitude` — dots appear on the map

If the map doesn't appear automatically:
- Drag `Longitude` → **Columns**
- Drag `Latitude` → **Rows**
- Change mark type to **Map**

**Add one dot per pharmacy:**

3. Drag `PharmacyName` (from DimPharmacy) → **Detail** in Marks card
4. You now have 120 dots — one per pharmacy

**Colour by revenue:**

5. Drag `RevenueEUR` → **Color** in Marks card
6. Click Color → Edit Colors → select **Blue** sequential palette
7. Higher revenue pharmacies will be darker blue

**Size by revenue:**

8. Drag `RevenueEUR` → **Size** in Marks card
9. Bigger dots = higher revenue pharmacies

**Add tooltip:**

10. Drag these fields to **Tooltip** in Marks card:
    - `PharmacyName`
    - `Country`
    - `PharmacyType`
    - `StoreSizeBand`
    - `RevenueEUR`
    - `Margin %`

**Add filters:**

11. Drag `Country` → **Filters** shelf → Show Filter
12. Drag `PharmacyType` → **Filters** shelf → Show Filter

**Format the map:**

13. Top menu → **Map** → **Map Layers**
14. Tick: Country/Region borders, Coastline
15. Untick everything else for a clean look
16. Top menu → **Map** → **Background** → Normal

---

### View 4 — Generic vs Branded

**Sheet name:** `Generic vs Branded`
**Data source:** PharmaBI Main

**Build steps:**

1. Drag `Product Type` (your calculated field) → **Columns**
2. Drag `RevenueEUR` → **Rows**
3. You now have two bars — Branded and Generic total revenue

**Add multiple metrics side by side:**

4. Hold Ctrl and also drag to Rows: `MarginEUR`
5. Change the aggregation on MarginEUR: right-click the pill → Measure → Average
6. Change RevenueEUR aggregation to Average as well
7. You now have two charts side by side showing avg revenue and avg margin

**Add Margin % as a third measure:**

8. Drag `Margin %` → Rows as well
9. Now you have three charts showing the comparison

**Colour by Product Type:**

10. Drag `Product Type` → **Color** in Marks card
11. Set Branded = `#2C6FAC`, Generic = `#E07B3A`

**Add value labels:**

12. Drag `Measure Values` → **Label** in Marks card

**Format:**

13. Title: `Generic vs Branded — Revenue, Margin & Profitability`

---

### View 5 — Promotional Impact

**Sheet name:** `Promo Impact`
**Data source:** PharmaBI Main

**Build steps:**

1. Drag `Promo Label` (calculated field) → **Columns**
2. Drag `RevenueEUR` → **Rows** — change aggregation to Average
3. Drag `MarginEUR` → **Rows** — change aggregation to Average
4. Drag `UnitsSold` → **Rows** — change aggregation to Average
5. You now have three charts: avg revenue, avg margin, avg units — for promo vs non-promo

**Colour by Promo Label:**

6. Drag `Promo Label` → **Color**
7. Promotional = `#E07B3A`, Non-Promotional = `#2C6FAC`

**Add value labels:**

8. Drag `Measure Values` → **Label**

**Add annotation for the margin drop:**

9. Right-click on the Avg Margin chart → **Annotate** → **Area**
10. Type: `Promo margin drops 40.3% vs non-promo`
11. Position it between the two bars

**Second chart — promo rate by category:**

12. Create a new sheet: `Promo Rate by Category`
13. Drag `Category` → **Rows**
14. Drag `PromoFlag` → **Filters** — keep both Yes and No
15. Drag `SalesID` → **Columns** — aggregation: COUNT
16. Drag `PromoFlag` → **Color**
17. Change to a stacked bar: Show Me → Stacked bars
18. This shows transaction count split by promo/non-promo per category

---

### View 6 — Seasonal Trends

**Sheet name:** `Seasonal Trends`
**Data source:** PharmaBI Main

**Build steps:**

1. Drag `Date` (from DimDate) → **Columns**
2. Click the `+` that appears on the Date pill to expand it — drill down to **Month**
3. You should see MONTH(Date) on Columns
4. Right-click the date pill → select **Exact Date** then change to **Month / Year** format
5. Drag `RevenueEUR` → **Rows**
6. Drag `Category` (from DimProduct) → **Color**

You now have a multi-line chart showing monthly revenue per category.

**Set category colours:**

7. Click Color → Edit Colors → set each category to its hex code from Part 4

**Add year separator:**

8. Drag `Year(Date)` → **Detail** in Marks card — this separates 2024 and 2025 lines
9. Right-click the x-axis → **Add Reference Line** → at `Date` = `01/01/2025`
10. Label: `2025`

**Format:**

11. Change mark type to **Line**
12. Rotate x-axis labels: right-click x-axis → Format → Alignment → 45 degrees
13. Title: `Monthly Revenue by Category — 2024 vs 2025`

---

### View 7 — Forecast View

**Sheet name:** `Forecast Q1 2026`
**Data source:** PharmaBI Forecast

> This is the only view using the separate CSV. Switch data source: top menu → Data → PharmaBI Forecast

**Build steps:**

1. Drag `date` → **Columns**
2. Drag `projected_revenue` → **Rows**
3. Drag `category` → **Color**
4. Change mark type to **Line**
5. Set category colours matching the rest of the dashboard

**Add confidence interval band:**

6. Drag `lower_bound_95` → **Rows** — a second axis appears
7. Right-click the right axis → **Dual Axis**
8. Right-click again → **Synchronise Axis**
9. In the Marks card, find the `lower_bound_95` mark → change type to **Area**
10. Click Color on that mark → reduce opacity to 15%
11. Repeat steps 6-10 for `upper_bound_95`

The shaded area between lower and upper bounds shows the 95% confidence interval.

**Add forecast start line:**

12. Right-click the x-axis → **Add Reference Line**
13. Constant value: `2026-01-01`
14. Label: `Forecast Start (Q1 2026)`

**Format:**

15. Title: `Q1 2026 Revenue Forecast by Category — Prophet Model`
16. Add annotation: right-click canvas → Annotate → Area → type `95% confidence interval shown`

---

## Part 6 — Building the Dashboard

### 6a — Create the dashboard

1. Click the **New Dashboard** icon at the bottom (looks like a grid with a +)
2. Name it `PharmaBI Overview`
3. Set size: top left of dashboard pane → Size → Fixed Size → **1400 x 900**

### 6b — Layout

Drag sheets from the left Sheets panel onto the dashboard canvas in this order:

```
┌──────────────────────────────────────────────────────┐
│  Title: PharmaBI — Pharmaceutical Sales Intelligence  │
├──────────┬──────────┬─────────────┬──────────────────┤
│  KPI     │  KPI     │  KPI        │  KPI             │
│ Revenue  │  Margin  │  Margin %   │  Transactions    │
├──────────┴──────────┴─────────────┴──────────────────┤
│                                                        │
│           Revenue by Category (full width)             │
│                                                        │
├────────────────────────┬───────────────────────────────┤
│                        │                               │
│   Geographic Map       │   Generic vs Branded          │
│                        │                               │
├────────────────────────┴───────────────────────────────┤
│                                                        │
│           Promo Impact (full width)                    │
│                                                        │
└──────────────────────────────────────────────────────┘
```

### 6c — Add the title

1. Drag a **Text** object from the Objects panel (bottom left of dashboard pane) to the top
2. Double-click it → type:
   `PharmaBI — Pharmaceutical Sales Intelligence Platform`
3. Format: bold, size 18, centre aligned
4. Add subtitle below: `120 Pharmacies · 8 Countries · 62,139 Transactions · 2024–2025`

### 6d — Arrange KPI tiles

1. First drag a **Horizontal** layout container to the top area
2. Then drag each KPI sheet into that container one by one
3. They will sit side by side automatically
4. Right-click each KPI sheet inside the dashboard → **Hide Title** (removes the sheet name label)

### 6e — Make filters global

1. On the Geographic Map in the dashboard, click the Country filter dropdown → **Apply to All Worksheets**
2. Repeat for PharmacyType filter
3. Now when someone filters by country on the map, every view updates

### 6f — Second dashboard — Trends & Forecast

1. Create a second dashboard: click + again at the bottom
2. Name it `Trends & Forecast`
3. Size: same 1400 x 900
4. Add:
   - Seasonal Trends — top half, full width
   - Forecast Q1 2026 — bottom half, full width

### 6g — Navigation between dashboards

On the first dashboard:
1. Drag a **Button** object from the Objects panel
2. Edit Button → Navigate to Sheet → select `Trends & Forecast`
3. Label: `→ Trends & Forecast`

On the second dashboard:
1. Same process — navigate back to `PharmaBI Overview`
2. Label: `← Overview`

---

## Part 7 — Formatting Rules

Apply these to every sheet before publishing:

| Element | Setting |
|---|---|
| Background | White |
| Gridlines | None — Format → Lines → set all to None |
| Borders | None on sheets inside dashboard |
| Font | Tableau Book or Arial |
| Title size | 13pt bold |
| Label size | 9pt |
| Padding | 8px on each sheet in dashboard |

To remove gridlines on a sheet:
- Top menu → **Format** → **Lines** → set Row Divider, Column Divider, Grid Lines all to **None**

---

## Part 8 — Publishing to Tableau Public

### 8a — Save and publish

1. Top menu → **File** → **Save to Tableau Public As**
2. Sign in with your Tableau Public account
3. Name: `PharmaBI - Pharmaceutical Sales Intelligence`
4. Click **Save**
5. Your browser opens automatically showing the live dashboard

### 8b — Set description on Tableau Public

On your Tableau Public profile:
1. Click the workbook → pencil icon → Edit Details
2. Paste this description:
```
Pharmaceutical Sales Intelligence Platform — pan-European retail pharmacy 
analytics covering 62,139 transactions across 120 pharmacies in 8 countries 
(2024–2025). Revenue & margin analysis, promotional impact, geographic 
segmentation, generic vs branded comparison, and Q1 2026 Prophet forecasting. 
Built with Python, AWS (S3, Glue, Athena), and Tableau.
```
3. Tags: `data-analytics`, `pharma`, `python`, `aws`, `tableau`, `prophet`
4. Visibility: **Public**

### 8c — Copy your URL

Copy the URL — it will look like:
`https://public.tableau.com/views/PharmaBI-PharmaceuticalSalesIntelligence/PharmaBI`

Paste it into:
- `README.md` Section 9 — Dashboard Link
- Your LinkedIn project description
- Your resume under the project entry

---

## Part 9 — Post-Build Checklist

```
□ Star schema connected — FactSales + 3 dimensions from Excel
□ Forecast CSV connected separately
□ All 4 calculated fields created
□ Category colours consistent across all views (matching notebook charts)
□ KPI Summary — 4 tiles showing correct numbers
□ Revenue by Category — sorted, coloured by margin %, labelled
□ Geographic Map — 120 dots, coloured and sized by revenue, filters working
□ Generic vs Branded — branded vs generic comparison across 3 metrics
□ Promo Impact — promo vs non-promo with margin drop annotation
□ Seasonal Trends — monthly lines per category, 2024 and 2025
□ Forecast Q1 2026 — Prophet line with confidence band
□ Two dashboards with navigation buttons between them
□ Global filters applied across all views
□ Published to Tableau Public
□ URL added to README Section 9
□ URL added to LinkedIn
□ README Section 8 Key Findings filled in with real numbers
□ README Section 10 Forecast metrics filled in with MAE and MAPE
```

---

*Once the checklist is complete, the project is ready to show to interviewers.*