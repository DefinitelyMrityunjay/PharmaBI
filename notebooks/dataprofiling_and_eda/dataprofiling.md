# 01 — Data Profiling
**PharmaBI: Pharmaceutical Sales Intelligence & Forecasting Platform**

**Purpose:** Systematically validate all four tables before any analysis begins. Every check here is a question answered — so that no analysis notebook has to stop and wonder about data quality.

**What this notebook covers:**
1. Connection check
2. Row counts and schema
3. Null checks
4. Duplicate checks
5. Referential integrity
6. Date range validation
7. Key column distributions
8. Business logic checks
9. Profiling summary

**Output:** A written summary at the end stating what is clean, what to watch out for, and any decisions made before analysis.

---

## 0. Setup & Connection

```python
from dotenv import load_dotenv
import os
import boto3
import awswrangler as wr
import pandas as pd

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

# ── Sanity check ────────────────────────────────────────────────────
df_check = run_query("SELECT COUNT(*) AS n FROM fact_sales")
print(f"Connection OK — {df_check['n'].iloc[0]:,} rows in FactSales")
```

---

## 1. Row Counts

First thing to confirm — each table has the expected number of rows. If any count is off, something went wrong in the S3 upload or Glue crawler.

```python
query = """
SELECT 'fact_sales'   AS table_name, COUNT(*) AS row_count FROM fact_sales   UNION ALL
SELECT 'dim_date',                   COUNT(*)               FROM dim_date     UNION ALL
SELECT 'dim_pharmacy',               COUNT(*)               FROM dim_pharmacy UNION ALL
SELECT 'dim_product',                COUNT(*)               FROM dim_product
"""

df_counts = run_query(query)

# Expected counts for validation
EXPECTED = {
    'fact_sales':   62139,
    'dim_date':     731,
    'dim_pharmacy': 120,
    'dim_product':  220
}

df_counts['expected'] = df_counts['table_name'].map(EXPECTED)
df_counts['match']    = df_counts['row_count'] == df_counts['expected']

print(df_counts.to_string(index=False))
print()

if df_counts['match'].all():
    print("All row counts match expected values.")
else:
    print("WARNING: Row count mismatch detected. Check S3 upload.")
```

Expected output:

```
   table_name  row_count  expected  match
   fact_sales      62139     62139   True
     dim_date        731       731   True
 dim_pharmacy        120       120   True
  dim_product        220       220   True

All row counts match expected values.
```

---

## 2. Schema Review

Pull the column names and data types Athena assigned to each table. Glue infers types automatically from CSV — worth confirming they are correct before building analysis on top.

```python
tables = ['fact_sales', 'dim_date', 'dim_pharmacy', 'dim_product']

for table in tables:
    df_schema = run_query(f"DESCRIBE {table}")
    print(f"\n── {table} ──")
    print(df_schema.to_string(index=False))
```

**What to check here:**

| Column | Expected type |
|---|---|
| `DateKey` | `bigint` or `int` — not `string` |
| `RevenueEUR`, `CostEUR`, `MarginEUR` | `double` or `float` |
| `UnitsSold` | `int` |
| `Latitude`, `Longitude` | `double` |
| `ListPriceEUR`, `StandardCostEUR` | `double` |

> If any numeric column was inferred as `string`, it needs to be cast in every query that uses it — e.g. `CAST(unitssold AS INT)`.

---

## 3. Null Checks

A null in a primary key or foreign key breaks joins. A null in a metric column breaks aggregations. Check every column across all tables.

### FactSales

```python
query_fs_nulls = """
SELECT
    COUNT(*) - COUNT(salesid)     AS salesid_nulls,
    COUNT(*) - COUNT(datekey)     AS datekey_nulls,
    COUNT(*) - COUNT(pharmacyid)  AS pharmacyid_nulls,
    COUNT(*) - COUNT(productid)   AS productid_nulls,
    COUNT(*) - COUNT(unitssold)   AS unitssold_nulls,
    COUNT(*) - COUNT(revenueeur)  AS revenueeur_nulls,
    COUNT(*) - COUNT(costeur)     AS costeur_nulls,
    COUNT(*) - COUNT(margineur)   AS margineur_nulls,
    COUNT(*) - COUNT(promoflag)   AS promoflag_nulls
FROM fact_sales
"""

print("── FactSales null counts ──")
df_fs_nulls = run_query(query_fs_nulls)
print(df_fs_nulls.T.rename(columns={0: 'null_count'}).to_string())
```

### DimPharmacy

```python
query_dp_nulls = """
SELECT
    COUNT(*) - COUNT(pharmacyid)    AS pharmacyid_nulls,
    COUNT(*) - COUNT(pharmacyname)  AS pharmacyname_nulls,
    COUNT(*) - COUNT(country)       AS country_nulls,
    COUNT(*) - COUNT(region)        AS region_nulls,
    COUNT(*) - COUNT(city)          AS city_nulls,
    COUNT(*) - COUNT(pharmacytype)  AS pharmacytype_nulls,
    COUNT(*) - COUNT(storesizeband) AS storesizeband_nulls,
    COUNT(*) - COUNT(latitude)      AS latitude_nulls,
    COUNT(*) - COUNT(longitude)     AS longitude_nulls
FROM dim_pharmacy
"""

print("── DimPharmacy null counts ──")
df_dp_nulls = run_query(query_dp_nulls)
print(df_dp_nulls.T.rename(columns={0: 'null_count'}).to_string())
```

### DimProduct

```python
# Note: DiscontinuedDate is expected to be null for active products (185 nulls expected)
query_dpr_nulls = """
SELECT
    COUNT(*) - COUNT(productid)        AS productid_nulls,
    COUNT(*) - COUNT(productname)      AS productname_nulls,
    COUNT(*) - COUNT(category)         AS category_nulls,
    COUNT(*) - COUNT(brand)            AS brand_nulls,
    COUNT(*) - COUNT(isgeneric)        AS isgeneric_nulls,
    COUNT(*) - COUNT(listpriceeur)     AS listpriceeur_nulls,
    COUNT(*) - COUNT(standardcosteur)  AS standardcosteur_nulls,
    COUNT(*) - COUNT(isdiscontinued)   AS isdiscontinued_nulls,
    COUNT(*) - COUNT(discontinueddate) AS discontinueddate_nulls
FROM dim_product
"""

print("── DimProduct null counts ──")
df_dpr_nulls = run_query(query_dpr_nulls)
print(df_dpr_nulls.T.rename(columns={0: 'null_count'}).to_string())
print()
print("Note: discontinueddate_nulls = 185 is expected (nulls = active products).")
```

---

## 4. Duplicate Checks

Primary keys must be unique. A duplicate SalesID means the same transaction was loaded twice and would inflate every revenue metric.

```python
query_dupes = """
SELECT
    'fact_sales'   AS table_name,
    'salesid'      AS primary_key,
    COUNT(*)       AS total_rows,
    COUNT(DISTINCT salesid) AS unique_keys,
    COUNT(*) - COUNT(DISTINCT salesid) AS duplicates
FROM fact_sales

UNION ALL

SELECT
    'dim_date', 'datekey',
    COUNT(*), COUNT(DISTINCT datekey),
    COUNT(*) - COUNT(DISTINCT datekey)
FROM dim_date

UNION ALL

SELECT
    'dim_pharmacy', 'pharmacyid',
    COUNT(*), COUNT(DISTINCT pharmacyid),
    COUNT(*) - COUNT(DISTINCT pharmacyid)
FROM dim_pharmacy

UNION ALL

SELECT
    'dim_product', 'productid',
    COUNT(*), COUNT(DISTINCT productid),
    COUNT(*) - COUNT(DISTINCT productid)
FROM dim_product
"""

df_dupes = run_query(query_dupes)
print(df_dupes.to_string(index=False))
print()

if (df_dupes['duplicates'] == 0).all():
    print("All primary keys are unique. No duplicates found.")
else:
    print("WARNING: Duplicate primary keys detected. Investigate before analysis.")
```

---

## 5. Referential Integrity

Every foreign key in FactSales must exist in its dimension table. If a PharmacyID in FactSales has no match in DimPharmacy, that transaction becomes an orphan — it cannot be joined and will silently drop from any analysis.

```python
# DateKey integrity
query_ri_date = """
SELECT COUNT(*) AS orphaned_datekeys
FROM fact_sales fs
LEFT JOIN dim_date dd ON fs.datekey = dd.datekey
WHERE dd.datekey IS NULL
"""

# PharmacyID integrity
query_ri_pharmacy = """
SELECT COUNT(*) AS orphaned_pharmacyids
FROM fact_sales fs
LEFT JOIN dim_pharmacy dp ON fs.pharmacyid = dp.pharmacyid
WHERE dp.pharmacyid IS NULL
"""

# ProductID integrity
query_ri_product = """
SELECT COUNT(*) AS orphaned_productids
FROM fact_sales fs
LEFT JOIN dim_product dpr ON fs.productid = dpr.productid
WHERE dpr.productid IS NULL
"""

ri_date     = run_query(query_ri_date).iloc[0, 0]
ri_pharmacy = run_query(query_ri_pharmacy).iloc[0, 0]
ri_product  = run_query(query_ri_product).iloc[0, 0]

print(f"Orphaned DateKeys    (no match in DimDate):     {ri_date}")
print(f"Orphaned PharmacyIDs (no match in DimPharmacy): {ri_pharmacy}")
print(f"Orphaned ProductIDs  (no match in DimProduct):  {ri_product}")
print()

if ri_date == 0 and ri_pharmacy == 0 and ri_product == 0:
    print("Referential integrity confirmed. All foreign keys have matching dimension records.")
else:
    print("WARNING: Orphaned foreign keys found. These rows will be lost in any JOIN.")
```

---

## 6. Date Range Validation

The dataset should cover exactly 1 Jan 2024 to 31 Dec 2025. Confirm this and check that FactSales transactions fall within that window.

```python
query_dates = """
SELECT
    MIN(date)               AS earliest_date,
    MAX(date)               AS latest_date,
    COUNT(DISTINCT year)    AS years_covered,
    COUNT(DISTINCT quarter) AS quarters_covered,
    COUNT(*)                AS total_days
FROM dim_date
"""

print("── DimDate coverage ──")
print(run_query(query_dates).to_string(index=False))
```

```python
# Confirm FactSales transactions fall within the date range
query_fs_dates = """
SELECT
    MIN(dd.date)                    AS earliest_transaction,
    MAX(dd.date)                    AS latest_transaction,
    COUNT(DISTINCT fs.datekey)      AS distinct_transaction_days
FROM fact_sales fs
JOIN dim_date dd ON fs.datekey = dd.datekey
"""

print("── FactSales transaction date range ──")
print(run_query(query_fs_dates).to_string(index=False))
```

```python
# Transactions per year — confirm both years are represented
query_yearly = """
SELECT
    dd.year,
    COUNT(*)                         AS transactions,
    ROUND(SUM(fs.revenueeur), 2)     AS total_revenue_eur
FROM fact_sales fs
JOIN dim_date dd ON fs.datekey = dd.datekey
GROUP BY dd.year
ORDER BY dd.year
"""

print("── Transactions and revenue by year ──")
print(run_query(query_yearly).to_string(index=False))
```

---

## 7. Key Column Distributions

Understand the shape of the core metric columns before running any analysis. This catches skew, outliers, and unexpected ranges.

```python
# Revenue, Cost, Margin, Units distributions
query_dist = """
SELECT
    ROUND(MIN(revenueeur), 2)                      AS revenue_min,
    ROUND(MAX(revenueeur), 2)                      AS revenue_max,
    ROUND(AVG(revenueeur), 2)                      AS revenue_avg,
    ROUND(APPROX_PERCENTILE(revenueeur, 0.5), 2)  AS revenue_median,

    ROUND(MIN(margineur), 2)                       AS margin_min,
    ROUND(MAX(margineur), 2)                       AS margin_max,
    ROUND(AVG(margineur), 2)                       AS margin_avg,

    MIN(unitssold)                                 AS units_min,
    MAX(unitssold)                                 AS units_max,
    ROUND(AVG(CAST(unitssold AS DOUBLE)), 2)       AS units_avg
FROM fact_sales
"""

print("── FactSales metric distributions ──")
print(run_query(query_dist).T.rename(columns={0: 'value'}).to_string())
```

```python
# PromoFlag split
query_promo = """
SELECT
    promoflag,
    COUNT(*)                                               AS transactions,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1)   AS pct_of_total
FROM fact_sales
GROUP BY promoflag
ORDER BY transactions DESC
"""

print("── PromoFlag distribution ──")
print(run_query(query_promo).to_string(index=False))
```

```python
# Product category breakdown
query_cat = """
SELECT
    category,
    COUNT(*)                                              AS product_count,
    SUM(CASE WHEN isgeneric = 'Yes'        THEN 1 ELSE 0 END) AS generic_count,
    SUM(CASE WHEN isgeneric = 'No'         THEN 1 ELSE 0 END) AS branded_count,
    SUM(CASE WHEN isdiscontinued = 'Yes'   THEN 1 ELSE 0 END) AS discontinued_count
FROM dim_product
GROUP BY category
ORDER BY product_count DESC
"""

print("── Product breakdown by category ──")
print(run_query(query_cat).to_string(index=False))
```

```python
# Pharmacy breakdown by country
query_pharm = """
SELECT
    country,
    COUNT(*)                      AS pharmacy_count,
    COUNT(DISTINCT pharmacytype)  AS types_present,
    COUNT(DISTINCT storesizeband) AS sizes_present
FROM dim_pharmacy
GROUP BY country
ORDER BY pharmacy_count DESC
"""

print("── Pharmacy count by country ──")
print(run_query(query_pharm).to_string(index=False))
```

---

## 8. Business Logic Checks

These checks confirm the data makes sense from a business perspective — not just technically clean, but logically consistent.

### Check 1 — Negative margins

Margin should always be positive (Revenue − Cost > 0).

```python
query_neg_margin = """
SELECT COUNT(*) AS negative_margin_rows
FROM fact_sales
WHERE margineur < 0
"""

neg = run_query(query_neg_margin).iloc[0, 0]
print(f"Transactions with negative margin: {neg}")
print("Expected: 0")
```

### Check 2 — Margin arithmetic consistency

`MarginEUR` should equal `RevenueEUR - CostEUR`. Allowing a tolerance of 0.01 for floating point rounding.

```python
query_margin_check = """
SELECT COUNT(*) AS margin_mismatch_rows
FROM fact_sales
WHERE ABS(margineur - (revenueeur - costeur)) > 0.01
"""

mismatch = run_query(query_margin_check).iloc[0, 0]
print(f"Rows where MarginEUR != RevenueEUR - CostEUR: {mismatch}")
print("Expected: 0")
```

### Check 3 — Zero or negative revenue

```python
query_zero_rev = """
SELECT COUNT(*) AS zero_or_negative_revenue
FROM fact_sales
WHERE revenueeur <= 0
"""

zero_rev = run_query(query_zero_rev).iloc[0, 0]
print(f"Transactions with zero or negative revenue: {zero_rev}")
print("Expected: 0")
```

### Check 4 — Discontinued products still being sold

Flag any transactions where the product is marked as discontinued. May be legitimate old stock clearance — worth knowing before analysis.

```python
query_discontinued = """
SELECT
    COUNT(*)                        AS transactions_with_discontinued,
    COUNT(DISTINCT fs.productid)    AS distinct_discontinued_products,
    ROUND(SUM(fs.revenueeur), 2)    AS revenue_from_discontinued
FROM fact_sales fs
JOIN dim_product dpr ON fs.productid = dpr.productid
WHERE dpr.isdiscontinued = 'Yes'
"""

print("── Sales from discontinued products ──")
print(run_query(query_discontinued).to_string(index=False))
print()
print("Note: Sales from discontinued products may represent old stock clearance.")
print("These will be included in analysis unless a decision is made to exclude them.")
```

### Check 5 — Daily transaction volume

Checks for any days with suspiciously low activity that might indicate missing data.

```python
query_daily = """
SELECT
    MIN(daily_count)               AS min_transactions_per_day,
    MAX(daily_count)               AS max_transactions_per_day,
    ROUND(AVG(daily_count), 1)     AS avg_transactions_per_day
FROM (
    SELECT datekey, COUNT(*) AS daily_count
    FROM fact_sales
    GROUP BY datekey
) daily
"""

print("── Daily transaction volume ──")
print(run_query(query_daily).to_string(index=False))
```

---

## 9. Profiling Summary

Update this after running all checks above.

```python
summary = """
DATA PROFILING SUMMARY
======================

Dataset: PharmaBI Retail Pharmacy Sales
Profiled: [fill in date]

ROW COUNTS
----------
fact_sales:   62,139  ✓
dim_date:        731  ✓
dim_pharmacy:    120  ✓
dim_product:     220  ✓

NULLS
-----
FactSales:    No nulls in any column. Clean.
DimPharmacy:  No nulls in any column. Clean.
DimProduct:   185 nulls in DiscontinuedDate — expected (185 active products have no discontinued date).
DimDate:      No nulls in any column. Clean.

DUPLICATES
----------
All primary keys are unique across all four tables. No duplicates.

REFERENTIAL INTEGRITY
---------------------
All foreign keys in FactSales have matching records in their dimension tables.
No orphaned transactions.

DATE RANGE
----------
DimDate covers 1 Jan 2024 to 31 Dec 2025 (731 days including leap day).
All FactSales transactions fall within this range.

BUSINESS LOGIC
--------------
- No negative margins found.
- MarginEUR = RevenueEUR - CostEUR confirmed for all rows (within 0.01 rounding tolerance).
- No zero or negative revenue transactions.
- [TBD] Discontinued product sales: fill in count after running Check 4.
- [TBD] Daily transaction range: fill in min/max/avg after running Check 5.

DECISIONS FOR ANALYSIS
----------------------
1. DiscontinuedDate nulls: expected — do not treat as a data quality issue.
2. StoreSizeBand values are S/M/L — map to Small/Medium/Large in analysis notebooks for display.
3. Discontinued products: [TBD — decide whether to include or exclude after reviewing Check 4 output].
4. PromoFlag is a Yes/No string — filter using WHERE promoflag = 'Yes' in analysis queries.
5. IsGeneric and IsDiscontinued are Yes/No strings — same pattern applies.

OVERALL STATUS
--------------
Data is clean. No blocking issues found. Safe to proceed to EDA.
"""

print(summary)
```

---

*Next: `02_eda.ipynb` — Exploratory Data Analysis*