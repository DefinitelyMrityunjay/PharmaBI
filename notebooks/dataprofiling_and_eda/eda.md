# 02 — Exploratory Data Analysis
**PharmaBI: Pharmaceutical Sales Intelligence & Forecasting Platform**

**Purpose:** Join the four tables together and explore patterns across every business dimension. SQL handles all aggregations via Athena. Python handles formatting and visualisation. This is the workflow used in real analytics teams — compute in the database, visualise in Python.

**What this notebook covers:**
1. Setup & connection
2. Revenue and margin by product category
3. Revenue by country and pharmacy attributes
4. Generic vs branded performance
5. Promotional activity patterns
6. Monthly and quarterly trends
7. Top products and top pharmacies
8. EDA summary and hypotheses

---

## 0. Setup & Connection

```python
from dotenv import load_dotenv
import os
import boto3
import awswrangler as wr
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import seaborn as sns

load_dotenv()

# ── Config ──────────────────────────────────────────────────────────
S3_OUTPUT = "s3://pharma-bi-raw/athena-results/"
DATABASE  = "pharma_bi_db"

session = boto3.Session(
    aws_access_key_id=os.getenv("AWS_ACCESS_KEY_ID"),
    aws_secret_access_key=os.getenv("AWS_SECRET_ACCESS_KEY"),
    region_name=os.getenv("AWS_DEFAULT_REGION")
)

def run_query(sql: str) -> pd.DataFrame:
    """Run an Athena SQL query and return a pandas DataFrame."""
    return wr.athena.read_sql_query(
        sql=sql,
        database=DATABASE,
        s3_output=S3_OUTPUT,
        boto3_session=session
    )

# ── Plot style ───────────────────────────────────────────────────────
plt.rcParams.update({
    'figure.dpi':        120,
    'axes.spines.top':   False,
    'axes.spines.right': False,
    'axes.titlesize':    13,
    'axes.titleweight':  'bold',
    'axes.labelsize':    11,
    'xtick.labelsize':   10,
    'ytick.labelsize':   10,
    'font.family':       'sans-serif',
})
PALETTE = ['#2C6FAC', '#3A9E6F', '#E07B3A', '#9B5EA0', '#C0392B']

# ── Sanity check ────────────────────────────────────────────────────
df_check = run_query("SELECT COUNT(*) AS n FROM fact_sales")
print(f"Connection OK — {df_check['n'].iloc[0]:,} rows in FactSales")
```

---

## 1. Revenue and Margin by Product Category

```python
query_category = """
SELECT
    dpr.category,
    COUNT(fs.salesid)                                                        AS transactions,
    SUM(fs.unitssold)                                                        AS total_units,
    ROUND(SUM(fs.revenueeur), 2)                                             AS total_revenue,
    ROUND(SUM(fs.margineur), 2)                                              AS total_margin,
    ROUND(AVG(fs.revenueeur), 2)                                             AS avg_revenue_per_tx,
    ROUND(AVG(CAST(fs.unitssold AS DOUBLE)), 2)                              AS avg_units_per_tx,
    ROUND(SUM(fs.margineur) / SUM(fs.revenueeur) * 100, 1)                  AS margin_pct,
    ROUND(SUM(fs.revenueeur) / SUM(SUM(fs.revenueeur)) OVER () * 100, 1)    AS revenue_share_pct
FROM fact_sales fs
JOIN dim_product dpr ON fs.productid = dpr.productid
GROUP BY dpr.category
ORDER BY total_revenue DESC
"""

df_cat = run_query(query_category)
print(df_cat.to_string(index=False))
```

```python
fig, axes = plt.subplots(1, 2, figsize=(13, 5))

# ── Chart 1: Total revenue by category ──────────────────────────────
bars = axes[0].barh(
    df_cat['category'][::-1],
    df_cat['total_revenue'][::-1] / 1e6,
    color=PALETTE,
    height=0.55
)
axes[0].set_xlabel('Total Revenue (€M)')
axes[0].set_title('Total Revenue by Category')
axes[0].xaxis.set_major_formatter(mticker.FormatStrFormatter('€%.1fM'))

# Annotate with revenue share %
for bar, share in zip(bars, df_cat['revenue_share_pct'][::-1]):
    axes[0].text(
        bar.get_width() + 0.01,
        bar.get_y() + bar.get_height() / 2,
        f'{share}%',
        va='center', fontsize=9, color='#555'
    )

# ── Chart 2: Margin % by category ───────────────────────────────────
bars2 = axes[1].barh(
    df_cat['category'][::-1],
    df_cat['margin_pct'][::-1],
    color=PALETTE,
    height=0.55
)
axes[1].set_xlabel('Gross Margin %')
axes[1].set_title('Margin % by Category')
axes[1].xaxis.set_major_formatter(mticker.FormatStrFormatter('%.0f%%'))
axes[1].set_xlim(0, 42)

# Annotate margin %
for bar, pct in zip(bars2, df_cat['margin_pct'][::-1]):
    axes[1].text(
        bar.get_width() + 0.3,
        bar.get_y() + bar.get_height() / 2,
        f'{pct}%',
        va='center', fontsize=9, color='#555'
    )

plt.suptitle('Category Performance — Revenue vs Margin %', fontsize=14, fontweight='bold', y=1.01)
plt.tight_layout()
plt.savefig('outputs/01_category_revenue_margin.png', bbox_inches='tight')
plt.show()
```

**What this tells you:**

- Prescription leads on total revenue (32.4% share) but has the lowest margin % (21.9%)
- Wellness and Personal Care have the best margin % (33.6% and 33.5%) — most profitable per euro sold
- OTC has the highest transaction count and highest avg units per transaction — volume-driven category
- These two charts together show that revenue rank ≠ profitability rank — a key business insight

---

## 2. Revenue by Country

```python
query_country = """
SELECT
    dp.country,
    COUNT(DISTINCT dp.pharmacyid)                                            AS pharmacy_count,
    COUNT(fs.salesid)                                                        AS transactions,
    ROUND(SUM(fs.revenueeur), 2)                                             AS total_revenue,
    ROUND(SUM(fs.margineur), 2)                                              AS total_margin,
    ROUND(SUM(fs.margineur) / SUM(fs.revenueeur) * 100, 1)                  AS margin_pct,
    ROUND(SUM(fs.revenueeur) / COUNT(DISTINCT dp.pharmacyid), 2)             AS avg_revenue_per_pharmacy,
    ROUND(SUM(fs.revenueeur) / SUM(SUM(fs.revenueeur)) OVER () * 100, 1)    AS revenue_share_pct
FROM fact_sales fs
JOIN dim_pharmacy dp ON fs.pharmacyid = dp.pharmacyid
GROUP BY dp.country
ORDER BY total_revenue DESC
"""

df_country = run_query(query_country)
print(df_country.to_string(index=False))
```

```python
fig, axes = plt.subplots(1, 2, figsize=(13, 5))

# ── Chart 1: Total revenue by country ───────────────────────────────
axes[0].barh(
    df_country['country'][::-1],
    df_country['total_revenue'][::-1] / 1e6,
    color='#2C6FAC',
    height=0.55
)
axes[0].set_xlabel('Total Revenue (€M)')
axes[0].set_title('Total Revenue by Country')
axes[0].xaxis.set_major_formatter(mticker.FormatStrFormatter('€%.1fM'))

# ── Chart 2: Revenue per pharmacy — normalised view ──────────────────
# This is the more meaningful comparison — removes pharmacy count bias
colors = ['#2C6FAC' if v == df_country['avg_revenue_per_pharmacy'].max()
          else '#95b8d1' for v in df_country['avg_revenue_per_pharmacy']]

axes[1].barh(
    df_country.sort_values('avg_revenue_per_pharmacy')['country'],
    df_country.sort_values('avg_revenue_per_pharmacy')['avg_revenue_per_pharmacy'] / 1e3,
    color=colors,
    height=0.55
)
axes[1].set_xlabel('Avg Revenue per Pharmacy (€k)')
axes[1].set_title('Revenue per Pharmacy by Country\n(normalised for pharmacy count)')
axes[1].xaxis.set_major_formatter(mticker.FormatStrFormatter('€%.0fk'))

plt.suptitle('Country Performance — Total vs Normalised Revenue', fontsize=14, fontweight='bold', y=1.01)
plt.tight_layout()
plt.savefig('outputs/02_country_revenue.png', bbox_inches='tight')
plt.show()
```

**What this tells you:**

- Germany leads total revenue (18.2%) proportional to its pharmacy count (22)
- The normalised chart (revenue per pharmacy) reveals which markets are genuinely more productive per location
- Margin % is consistent across all 8 countries (27.8–28.2%) — no country has a pricing advantage

---

## 3. Revenue by Pharmacy Type and Store Size

```python
query_phtype = """
SELECT
    dp.pharmacytype,
    COUNT(DISTINCT dp.pharmacyid)                                            AS pharmacy_count,
    COUNT(fs.salesid)                                                        AS transactions,
    ROUND(SUM(fs.revenueeur), 2)                                             AS total_revenue,
    ROUND(SUM(fs.revenueeur) / COUNT(DISTINCT dp.pharmacyid), 2)             AS avg_revenue_per_pharmacy,
    ROUND(SUM(fs.margineur) / SUM(fs.revenueeur) * 100, 1)                  AS margin_pct
FROM fact_sales fs
JOIN dim_pharmacy dp ON fs.pharmacyid = dp.pharmacyid
GROUP BY dp.pharmacytype
ORDER BY total_revenue DESC
"""

query_size = """
SELECT
    dp.storesizeband,
    COUNT(DISTINCT dp.pharmacyid)                                            AS pharmacy_count,
    COUNT(fs.salesid)                                                        AS transactions,
    ROUND(SUM(fs.revenueeur), 2)                                             AS total_revenue,
    ROUND(SUM(fs.revenueeur) / COUNT(DISTINCT dp.pharmacyid), 2)             AS avg_revenue_per_pharmacy,
    ROUND(SUM(fs.margineur) / SUM(fs.revenueeur) * 100, 1)                  AS margin_pct
FROM fact_sales fs
JOIN dim_pharmacy dp ON fs.pharmacyid = dp.pharmacyid
GROUP BY dp.storesizeband
ORDER BY total_revenue DESC
"""

df_phtype = run_query(query_phtype)
df_size   = run_query(query_size)

# Map S/M/L to readable labels
df_size['storesizeband'] = df_size['storesizeband'].map({'S': 'Small', 'M': 'Medium', 'L': 'Large'})

print("── By pharmacy type ──")
print(df_phtype.to_string(index=False))
print()
print("── By store size ──")
print(df_size.to_string(index=False))
```

```python
fig, axes = plt.subplots(1, 2, figsize=(13, 5))

# ── Chart 1: Avg revenue per pharmacy by type ────────────────────────
axes[0].bar(
    df_phtype['pharmacytype'],
    df_phtype['avg_revenue_per_pharmacy'] / 1e3,
    color=['#2C6FAC', '#3A9E6F', '#E07B3A'],
    width=0.5
)
axes[0].set_ylabel('Avg Revenue per Pharmacy (€k)')
axes[0].set_title('Revenue per Pharmacy by Type')
axes[0].yaxis.set_major_formatter(mticker.FormatStrFormatter('€%.0fk'))

for i, (val, count) in enumerate(zip(df_phtype['avg_revenue_per_pharmacy'], df_phtype['pharmacy_count'])):
    axes[0].text(i, val / 1e3 + 1, f'n={count}', ha='center', fontsize=9, color='#555')

# ── Chart 2: Avg revenue per pharmacy by store size ──────────────────
size_order = ['Small', 'Medium', 'Large']
df_size_sorted = df_size.set_index('storesizeband').reindex(size_order).reset_index()

axes[1].bar(
    df_size_sorted['storesizeband'],
    df_size_sorted['avg_revenue_per_pharmacy'] / 1e3,
    color=['#95b8d1', '#2C6FAC', '#1a4a7a'],
    width=0.5
)
axes[1].set_ylabel('Avg Revenue per Pharmacy (€k)')
axes[1].set_title('Revenue per Pharmacy by Store Size')
axes[1].yaxis.set_major_formatter(mticker.FormatStrFormatter('€%.0fk'))

for i, (val, count) in enumerate(zip(df_size_sorted['avg_revenue_per_pharmacy'], df_size_sorted['pharmacy_count'])):
    axes[1].text(i, val / 1e3 + 1, f'n={count}', ha='center', fontsize=9, color='#555')

plt.suptitle('Pharmacy Attributes — Revenue per Location (normalised)', fontsize=14, fontweight='bold', y=1.01)
plt.tight_layout()
plt.savefig('outputs/03_pharmacy_type_size.png', bbox_inches='tight')
plt.show()
```

> The `n=` labels show how many pharmacies are in each group — important context for comparing bars of different sizes.

---

## 4. Generic vs Branded Performance

```python
query_generic = """
SELECT
    dpr.isgeneric,
    COUNT(fs.salesid)                                       AS transactions,
    SUM(fs.unitssold)                                       AS total_units,
    ROUND(SUM(fs.revenueeur), 2)                            AS total_revenue,
    ROUND(AVG(fs.revenueeur), 2)                            AS avg_revenue_per_tx,
    ROUND(AVG(fs.margineur), 2)                             AS avg_margin_per_tx,
    ROUND(SUM(fs.margineur) / SUM(fs.revenueeur) * 100, 1) AS margin_pct,
    ROUND(SUM(fs.revenueeur) / SUM(fs.unitssold), 2)       AS revenue_per_unit
FROM fact_sales fs
JOIN dim_product dpr ON fs.productid = dpr.productid
GROUP BY dpr.isgeneric
"""

df_generic = run_query(query_generic)

# Clean up label for display
df_generic['label'] = df_generic['isgeneric'].map({'Yes': 'Generic', 'No': 'Branded'})
print(df_generic.to_string(index=False))
```

```python
metrics   = ['avg_revenue_per_tx', 'avg_margin_per_tx', 'margin_pct', 'revenue_per_unit']
labels    = ['Avg Revenue\nper Tx (€)', 'Avg Margin\nper Tx (€)', 'Margin %', 'Revenue\nper Unit (€)']

fig, axes = plt.subplots(1, 4, figsize=(15, 5))

branded = df_generic[df_generic['isgeneric'] == 'No'].iloc[0]
generic = df_generic[df_generic['isgeneric'] == 'Yes'].iloc[0]

for ax, metric, label in zip(axes, metrics, labels):
    vals   = [branded[metric], generic[metric]]
    colors = ['#2C6FAC', '#E07B3A']
    bars   = ax.bar(['Branded', 'Generic'], vals, color=colors, width=0.5)
    ax.set_title(label, fontsize=10)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

    for bar, v in zip(bars, vals):
        ax.text(
            bar.get_x() + bar.get_width() / 2,
            bar.get_height() + max(vals) * 0.02,
            f'{v:.1f}',
            ha='center', fontsize=9, fontweight='bold'
        )
    ax.set_ylim(0, max(vals) * 1.2)

plt.suptitle('Generic vs Branded — Key Metrics Comparison', fontsize=14, fontweight='bold', y=1.02)
plt.tight_layout()
plt.savefig('outputs/04_generic_vs_branded.png', bbox_inches='tight')
plt.show()
```

**What this tells you:**

- Branded products outperform generics on every metric shown
- The margin % gap (28.5% branded vs 25.5% generic) shows generics are not just cheaper — they are structurally less profitable
- Revenue per unit difference (branded ~€19.87 vs generic ~€16.87) confirms the price gap

---

## 5. Promotional Activity

```python
query_promo = """
SELECT
    fs.promoflag,
    COUNT(fs.salesid)                                       AS transactions,
    ROUND(AVG(CAST(fs.unitssold AS DOUBLE)), 2)             AS avg_units,
    ROUND(AVG(fs.revenueeur), 2)                            AS avg_revenue,
    ROUND(AVG(fs.margineur), 2)                             AS avg_margin,
    ROUND(SUM(fs.revenueeur), 2)                            AS total_revenue,
    ROUND(SUM(fs.margineur) / SUM(fs.revenueeur) * 100, 1) AS margin_pct
FROM fact_sales fs
GROUP BY fs.promoflag
ORDER BY fs.promoflag
"""

df_promo = run_query(query_promo)
df_promo['label'] = df_promo['promoflag'].map({'No': 'Non-Promo', 'Yes': 'Promo'})
print(df_promo.to_string(index=False))
```

```python
fig, axes = plt.subplots(1, 3, figsize=(13, 5))

metrics = ['avg_units', 'avg_revenue', 'avg_margin']
titles  = ['Avg Units Sold', 'Avg Revenue per Tx (€)', 'Avg Margin per Tx (€)']
colors  = ['#2C6FAC', '#E07B3A']

non_promo = df_promo[df_promo['promoflag'] == 'No'].iloc[0]
promo     = df_promo[df_promo['promoflag'] == 'Yes'].iloc[0]

for ax, metric, title in zip(axes, metrics, titles):
    vals = [non_promo[metric], promo[metric]]
    bars = ax.bar(['Non-Promo', 'Promo'], vals, color=colors, width=0.5)
    ax.set_title(title, fontsize=10)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

    for bar, v in zip(bars, vals):
        ax.text(
            bar.get_x() + bar.get_width() / 2,
            bar.get_height() + max(vals) * 0.02,
            f'{v:.2f}',
            ha='center', fontsize=9, fontweight='bold'
        )
    ax.set_ylim(0, max(vals) * 1.2)

# Annotate the margin drop on the third chart
margin_drop = round((non_promo['avg_margin'] - promo['avg_margin']) / non_promo['avg_margin'] * 100, 1)
axes[2].annotate(
    f'−{margin_drop}% margin\non promo',
    xy=(1, promo['avg_margin']),
    xytext=(0.5, promo['avg_margin'] + 8),
    fontsize=9, color='#C0392B',
    arrowprops=dict(arrowstyle='->', color='#C0392B', lw=1.2)
)

plt.suptitle('Promotional Impact — Units, Revenue and Margin', fontsize=14, fontweight='bold', y=1.02)
plt.tight_layout()
plt.savefig('outputs/05_promo_impact.png', bbox_inches='tight')
plt.show()
```

**What this tells you:**

- Promotions produce almost no uplift in units sold (7.02 promo vs 7.19 non-promo)
- Average margin collapses 40.3% on promo transactions (€40.94 → €24.43)
- Promotions are eroding margin with no compensating volume benefit — this is a key finding for Module 3

```python
# Promo rate by category — which categories use promotions most
query_promo_cat = """
SELECT
    dpr.category,
    COUNT(CASE WHEN fs.promoflag = 'Yes' THEN 1 END)   AS promo_transactions,
    COUNT(fs.salesid)                                   AS total_transactions,
    ROUND(
        COUNT(CASE WHEN fs.promoflag = 'Yes' THEN 1 END) * 100.0 / COUNT(fs.salesid),
    1) AS promo_rate_pct
FROM fact_sales fs
JOIN dim_product dpr ON fs.productid = dpr.productid
GROUP BY dpr.category
ORDER BY promo_rate_pct DESC
"""

df_promo_cat = run_query(query_promo_cat)

fig, ax = plt.subplots(figsize=(8, 4))
bars = ax.barh(
    df_promo_cat['category'][::-1],
    df_promo_cat['promo_rate_pct'][::-1],
    color='#E07B3A',
    height=0.5
)
ax.set_xlabel('Promo Rate (%)')
ax.set_title('Promotional Rate by Category', fontweight='bold')
ax.xaxis.set_major_formatter(mticker.FormatStrFormatter('%.0f%%'))

for bar, val in zip(bars, df_promo_cat['promo_rate_pct'][::-1]):
    ax.text(bar.get_width() + 0.1, bar.get_y() + bar.get_height() / 2,
            f'{val}%', va='center', fontsize=9)

plt.tight_layout()
plt.savefig('outputs/06_promo_rate_by_category.png', bbox_inches='tight')
plt.show()
```

---

## 6. Monthly and Quarterly Trends

```python
query_monthly = """
SELECT
    dd.year,
    dd.monthnumber,
    dd.monthname,
    COUNT(fs.salesid)                    AS transactions,
    ROUND(SUM(fs.revenueeur), 2)         AS total_revenue,
    ROUND(SUM(fs.margineur), 2)          AS total_margin
FROM fact_sales fs
JOIN dim_date dd ON fs.datekey = dd.datekey
GROUP BY dd.year, dd.monthnumber, dd.monthname
ORDER BY dd.year, dd.monthnumber
"""

df_monthly = run_query(query_monthly)
df_2024    = df_monthly[df_monthly['year'] == 2024].reset_index(drop=True)
df_2025    = df_monthly[df_monthly['year'] == 2025].reset_index(drop=True)
months     = df_2024['monthname'].str[:3].tolist()  # Jan, Feb, etc.
```

```python
fig, axes = plt.subplots(2, 1, figsize=(13, 9))

# ── Chart 1: Monthly revenue — 2024 vs 2025 overlay ─────────────────
x = range(len(months))
axes[0].plot(x, df_2024['total_revenue'] / 1e3, marker='o', linewidth=2,
             color='#2C6FAC', label='2024', markersize=5)
axes[0].plot(x, df_2025['total_revenue'] / 1e3, marker='o', linewidth=2,
             color='#E07B3A', label='2025', markersize=5, linestyle='--')
axes[0].set_xticks(x)
axes[0].set_xticklabels(months)
axes[0].set_ylabel('Revenue (€k)')
axes[0].set_title('Monthly Revenue — 2024 vs 2025', fontweight='bold')
axes[0].yaxis.set_major_formatter(mticker.FormatStrFormatter('€%.0fk'))
axes[0].legend()
axes[0].fill_between(x,
    df_2024['total_revenue'] / 1e3,
    df_2025['total_revenue'] / 1e3,
    alpha=0.08, color='#3A9E6F', label='YoY gap')

# Annotate Q3 peak
axes[0].annotate('Q3 peak\nboth years',
    xy=(6, df_2024['total_revenue'].iloc[6] / 1e3),
    xytext=(7.5, 395),
    fontsize=9, color='#2C6FAC',
    arrowprops=dict(arrowstyle='->', color='#2C6FAC', lw=1.2)
)

# ── Chart 2: Quarterly revenue comparison ────────────────────────────
query_quarterly = """
SELECT
    dd.year,
    dd.quarter,
    ROUND(SUM(fs.revenueeur), 2) AS total_revenue
FROM fact_sales fs
JOIN dim_date dd ON fs.datekey = dd.datekey
GROUP BY dd.year, dd.quarter
ORDER BY dd.year, dd.quarter
"""

df_q = run_query(query_quarterly)
df_q24 = df_q[df_q['year'] == 2024]['total_revenue'].values / 1e3
df_q25 = df_q[df_q['year'] == 2025]['total_revenue'].values / 1e3
q_labels = ['Q1', 'Q2', 'Q3', 'Q4']
xq = range(len(q_labels))
w  = 0.35

axes[1].bar([i - w/2 for i in xq], df_q24, width=w, color='#2C6FAC', label='2024')
axes[1].bar([i + w/2 for i in xq], df_q25, width=w, color='#E07B3A', label='2025')
axes[1].set_xticks(xq)
axes[1].set_xticklabels(q_labels)
axes[1].set_ylabel('Revenue (€k)')
axes[1].set_title('Quarterly Revenue — 2024 vs 2025', fontweight='bold')
axes[1].yaxis.set_major_formatter(mticker.FormatStrFormatter('€%.0fk'))
axes[1].legend()

# Annotate YoY change per quarter
for i, (v24, v25) in enumerate(zip(df_q24, df_q25)):
    change = (v25 - v24) / v24 * 100
    color  = '#3A9E6F' if change >= 0 else '#C0392B'
    axes[1].text(i + w/2 + 0.05, v25 + 5, f'{change:+.1f}%',
                 fontsize=8, color=color, va='bottom')

plt.tight_layout()
plt.savefig('outputs/07_monthly_quarterly_trend.png', bbox_inches='tight')
plt.show()
```

**What this tells you:**

- Q3 (July–August) is the strongest quarter in both years
- 2025 outperforms 2024 in every quarter — consistent positive growth trend
- The YoY change annotations on the quarterly chart show the growth rate per quarter at a glance

---

## 7. Top Products and Top Pharmacies

```python
query_top_products = """
SELECT
    dpr.productname,
    dpr.category,
    ROUND(SUM(fs.revenueeur), 2)  AS total_revenue,
    SUM(fs.unitssold)             AS total_units,
    COUNT(fs.salesid)             AS transactions
FROM fact_sales fs
JOIN dim_product dpr ON fs.productid = dpr.productid
GROUP BY dpr.productname, dpr.category
ORDER BY total_revenue DESC
LIMIT 10
"""

df_top_prod = run_query(query_top_products)

# Shorten product names for chart display
df_top_prod['short_name'] = df_top_prod['productname'].str[:30]

fig, ax = plt.subplots(figsize=(10, 6))

category_colors = {
    'Prescription':  '#2C6FAC',
    'OTC':           '#3A9E6F',
    'Wellness':      '#E07B3A',
    'Personal Care': '#9B5EA0',
    'Medical Devices': '#C0392B'
}
bar_colors = df_top_prod['category'].map(category_colors)

bars = ax.barh(
    df_top_prod['short_name'][::-1],
    df_top_prod['total_revenue'][::-1] / 1e3,
    color=bar_colors[::-1],
    height=0.6
)
ax.set_xlabel('Total Revenue (€k)')
ax.set_title('Top 10 Products by Revenue', fontweight='bold')
ax.xaxis.set_major_formatter(mticker.FormatStrFormatter('€%.0fk'))

# Legend for categories
from matplotlib.patches import Patch
legend_elements = [Patch(facecolor=v, label=k) for k, v in category_colors.items()
                   if k in df_top_prod['category'].values]
ax.legend(handles=legend_elements, loc='lower right', fontsize=8)

plt.tight_layout()
plt.savefig('outputs/08_top_products.png', bbox_inches='tight')
plt.show()
```

```python
query_top_pharmacies = """
SELECT
    dp.pharmacyname,
    dp.country,
    dp.pharmacytype,
    ROUND(SUM(fs.revenueeur), 2)  AS total_revenue,
    COUNT(fs.salesid)             AS transactions
FROM fact_sales fs
JOIN dim_pharmacy dp ON fs.pharmacyid = dp.pharmacyid
GROUP BY dp.pharmacyname, dp.country, dp.pharmacytype
ORDER BY total_revenue DESC
LIMIT 10
"""

df_top_ph = run_query(query_top_pharmacies)
df_top_ph['label'] = df_top_ph['pharmacyname'].str[:25] + ' (' + df_top_ph['country'].str[:2] + ')'

fig, ax = plt.subplots(figsize=(10, 6))

type_colors = {'Urban': '#2C6FAC', 'Suburban': '#3A9E6F', 'Rural': '#E07B3A'}
ph_colors   = df_top_ph['pharmacytype'].map(type_colors)

bars = ax.barh(
    df_top_ph['label'][::-1],
    df_top_ph['total_revenue'][::-1] / 1e3,
    color=ph_colors[::-1],
    height=0.6
)
ax.set_xlabel('Total Revenue (€k)')
ax.set_title('Top 10 Pharmacies by Revenue', fontweight='bold')
ax.xaxis.set_major_formatter(mticker.FormatStrFormatter('€%.0fk'))

legend_elements = [Patch(facecolor=v, label=k) for k, v in type_colors.items()]
ax.legend(handles=legend_elements, loc='lower right', fontsize=8)

plt.tight_layout()
plt.savefig('outputs/09_top_pharmacies.png', bbox_inches='tight')
plt.show()
```

---

## 8. EDA Summary and Hypotheses

```python
summary = """
EDA SUMMARY
===========

Dataset: PharmaBI Retail Pharmacy Sales (2024-2025)
Total Revenue:    ~€8.63M across 62,139 transactions
Overall Margin %: 28.1%

CATEGORY INSIGHTS
-----------------
- Prescription: highest revenue (32.4%) but lowest margin % (21.9%)
- Wellness + Personal Care: best margin % (33.5-33.6%) — most profitable per euro
- OTC: highest transaction count and highest avg units per tx (9.15)
- Revenue rank ≠ profitability rank — key insight for Module 1

GEOGRAPHIC INSIGHTS
-------------------
- Germany leads total revenue (18.2%) — proportional to pharmacy count
- Margin % consistent across all 8 countries (27.8-28.2%)
- Normalised revenue per pharmacy reveals true market productivity

GENERIC VS BRANDED
------------------
- Branded outperforms generic on every metric
- Margin % gap: 28.5% branded vs 25.5% generic
- Generics are structurally less profitable, not just cheaper

PROMOTIONAL ACTIVITY
--------------------
- Promos: 12% of transactions but margin collapses 40.3%
- No meaningful volume uplift from promotions
- Hypothesis: promotions are margin-destructive

SEASONALITY
-----------
- Q3 (July-August) peak consistent across both years
- 2025 outperforms 2024 in every quarter
- Stable margin % across seasons — no pricing pressure

HYPOTHESES FOR ANALYSIS MODULES
--------------------------------
Module 1: Revenue rank ≠ profitability rank — separate the two stories
Module 2: Generics structurally less profitable — investigate by category
Module 3: Promos destroy margin with no volume uplift — quantify the cost
Module 4: Normalise by pharmacy count before comparing countries/types
Module 5: Q3 summer peak + 2025 growth trend to confirm in detailed analysis
Module 6: Stable seasonality pattern should make Prophet forecasts reliable

Charts saved to outputs/ folder.
"""

print(summary)
```

---

*Next: `03_revenue_margin.md` — Module 1: Revenue & Margin Analysis*