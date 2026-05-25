#  Databricks Data Profiling — `customers_raw` Table

> Deep-dive statistics and trend tracking for your table — column by column, over time.

---

##  What Is Data Profiling?

While **Anomaly Detection** watches for *is the data late or missing?*, **Data Profiling** goes deeper — it answers *what does the data actually look like inside?*

Data Profiling computes **summary statistics** on your `customers_raw` table over time so you can track trends like:

- Are more rows having NULL values than before?
- Has a column's distribution changed unexpectedly?
- Are values drifting away from what's considered "normal"?

Think of it like a **regular health checkup for your data** — not just "did it arrive?" but "is it in good shape when it gets here?"

---

##  Anomaly Detection vs Data Profiling — What's the Difference?

| Feature | Anomaly Detection | Data Profiling |
|---|---|---|
| **What it checks** | Freshness & row count | Column-level statistics & distributions |
| **Setup** | Schema-level, auto | Table-level, manual setup |
| **Scope** | All tables in a schema | Selected tables only |
| **Use case** | "Did data arrive on time?" | "Is the data quality good inside?" |
| **Output** | Health badge + incidents | Metric tables + dashboard |
| **ML model tracking** |  No |  Yes |

> **In short:** Use both together for full data quality coverage on `customers_raw`.

---

##  What Does Data Profiling Monitor?

### 1.  Statistical Distribution
Tracks per-column statistics over time:
- Min, max, mean, median
- Percentiles (e.g. 90th percentile of a numeric column)
- Number and fraction of NULL values
- Number and fraction of zero values
- Distribution of values in categorical columns (e.g. `customer_country`)

**Example:** If `customer_age` normally averages 35 but suddenly drops to 12 — profiling captures that shift.

---

### 2.  Data Drift
Compares the **current data distribution** against:
- A **previous time window** (e.g. today vs. last week)
- A **baseline table** you define (e.g. a known-good snapshot of `customers_raw`)

**Example:** If 5% of `customer_email` values were null last month but it's now 40%, drift detection flags this.

---

### 3.  ML Model Performance *(if applicable)*
If `customers_raw` feeds a machine learning model, profiling can track:
- How model inputs are shifting over time
- How predictions are changing
- Whether model version A performs better than version B

---

##  Profile Types — Which One Fits `customers_raw`?

| Profile Type | When to Use | How It Works |
|---|---|---|
| **Snapshot** | Tables without a timestamp column | Scans the **entire table** every refresh. Max size: 4TB. |
| **Time Series** | Tables with a timestamp column | Computes metrics per **time window** (e.g. daily/weekly). Only last 30 days. |
| **Inference** | Tables storing ML model inputs & predictions | Tracks model inputs, predictions, and ground-truth labels over time. |

> **For `customers_raw`:** If your table has a timestamp column (like `created_at` or `updated_at`), use **Time Series**. Otherwise, use **Snapshot**.

---

##  How Data Profiling Works (Behind the Scenes)

```
customers_raw (primary table)
        │
        ├─── Optional: Baseline table (known-good snapshot for drift comparison)
        │
        ▼
Databricks creates a Profile attached to the table
        │
        ▼
Profiling job runs on Serverless compute
        │
        ├─── Computes PROFILE METRICS TABLE  ──► Summary statistics per column
        │
        └─── Computes DRIFT METRICS TABLE    ──► Change vs. previous window / baseline
                │
                ▼
        Auto-generated Dashboard (fully customizable)
        +
        Queryable metric tables in Unity Catalog
```

- **Does NOT modify** `customers_raw` in any way.
- **Does NOT slow down** any pipeline that writes to `customers_raw`.
- Metric tables are stored as **Delta tables** in a Unity Catalog schema you choose.

---

##  What Gets Created When You Enable Profiling?

Databricks creates **two metric tables** and **one dashboard** automatically:

### 1. Profile Metrics Table
Stores summary statistics per column per time window.

```sql
-- Example query
SELECT *
FROM <your_schema>.<table_name>_profile_metrics
ORDER BY window_start_time DESC
LIMIT 50;
```

Common columns you'll see: `column_name`, `data_type`, `num_nulls`, `percent_null`, `min`, `max`, `mean`, `stddev`, `distinct_count`

---
<img width="1919" height="1002" alt="image" src="https://github.com/user-attachments/assets/17c65905-e845-4322-9638-7008054254fc" />

### 2. Drift Metrics Table
Stores how much each column has drifted compared to the previous window or baseline.

```sql
-- Example query
SELECT *
FROM <your_schema>.<table_name>_drift_metrics
ORDER BY window_start_time DESC
LIMIT 50;
```

Common columns: `column_name`, `drift_type`, `drift_value`, `threshold`

---
<img width="1916" height="1011" alt="image" src="https://github.com/user-attachments/assets/9537e19b-7c01-4734-be4c-aa4fdb5eaed3" />


### 3. Auto-Generated Dashboard
A Lakeview dashboard is created automatically showing:
- Column-level statistics over time (line graphs)
- Drift scores per column
- NULL % trends
- Distribution histograms

You can fully customize this dashboard in the Databricks Dashboards UI.

---
<img width="1910" height="1016" alt="image" src="https://github.com/user-attachments/assets/3c22c73a-ebdb-4a79-8c2a-b7288d7bc2f1" />

##  How to Enable Data Profiling on `customers_raw`

### Via Databricks UI (Easiest)

1. Open **Catalog Explorer**.
2. Navigate to your `customers_raw` table.
3. Click the **Quality** tab.
4. Click **Add Monitor / Create Profile**.
5. Choose the profile type:
   - **Snapshot** — if no timestamp column
   - **Time Series** — if there's a `created_at` / `updated_at` column
6. Set the output schema (where metric tables will be stored).
7. Optionally provide a **baseline table** for drift comparison.
8. Click **Save**.

### Via API (Python)

```python
from databricks.sdk import WorkspaceClient
import databricks.sdk.service.catalog as catalog

w = WorkspaceClient()

w.quality_monitors.create(
    table_name="your_catalog.your_schema.customers_raw",
    assets_dir="/Shared/data-profiling/customers_raw",
    output_schema_name="your_catalog.your_monitoring_schema",
    snapshot=catalog.MonitorSnapshot()  # or use time_series / inference_log
)
```

---

##  How to View Profiling Results

### Option 1 — Catalog Explorer
1. Go to **Catalog Explorer** → `customers_raw` table.
2. Click the **Quality** tab.
3. View the latest statistics and drift scores directly.

### Option 2 — Auto-Generated Dashboard
1. In the sidebar, click **Dashboards**.
2. Find the dashboard named after your table (auto-created by profiling).
3. Explore column trends, drift, and NULL rates visually.

### Option 3 — Query the Metric Tables Directly

```sql
-- View profile statistics for customers_raw
SELECT
    window_start_time,
    column_name,
    percent_null,
    mean,
    min,
    max,
    distinct_count
FROM <your_schema>.customers_raw_profile_metrics
ORDER BY window_start_time DESC;
```

```sql
-- View drift for customers_raw
SELECT
    window_start_time,
    column_name,
    drift_type,
    drift_value
FROM <your_schema>.customers_raw_drift_metrics
WHERE drift_value IS NOT NULL
ORDER BY window_start_time DESC;
```

> Replace `<your_schema>` with the output schema you chose during setup.

---

##  Setting Up Alerts

You can alert your team when a profiling metric goes out of range.

**Example — Alert when NULL % spikes in `customer_email`:**

```sql
SELECT
    window_start_time,
    column_name,
    percent_null
FROM <your_schema>.customers_raw_profile_metrics
WHERE column_name = 'customer_email'
  AND percent_null > 0.10   -- alert if more than 10% are null
ORDER BY window_start_time DESC
LIMIT 1;
```

1. Go to **Databricks SQL → Alerts**.
2. Create an alert on the above query.
3. Set trigger: when result row count > 0.
4. Set notification: email / Slack / webhook.

---

##  Baseline Table (Optional but Recommended)

A baseline table is a **snapshot of what "good data" looks like** for `customers_raw`. Profiling uses it to measure drift against a known standard.

**Good choices for baseline:**
- A known-clean historical export of `customers_raw`
- The version of the table from a date when data quality was confirmed good

**Requirements:**
- Must match the **schema** of `customers_raw`
- Should represent the expected distribution of values

---

## 📋 Requirements Checklist

| Requirement | Detail |
|---|---|
| Unity Catalog enabled workspace |  Required |
| Databricks SQL access |  Required |
| `USE CATALOG` + `USE SCHEMA` privilege | Required |
| `SELECT` on `customers_raw` |  Required |
| `MANAGE` on catalog, schema, or table |  Required to create profile |
| Serverless compute |  Used automatically (no setup needed) |

---

##  Known Limitations

- Only **Delta tables** are supported (managed, external, views, materialized views, streaming tables).
- **Time Series and Inference** profiles only compute metrics over the **last 30 days**. Contact Databricks to adjust.
- **Snapshot** profile maximum table size is **4TB**. For larger tables, switch to Time Series.
- Materialized views do **not** support incremental processing in profiling.
- Not all Databricks regions support data profiling — check your region's feature availability.

---

##  Quick Reference

| What | Where |
|---|---|
| Enable profiling | Catalog Explorer → `customers_raw` → Quality tab |
| View stats & drift | Catalog Explorer → Quality tab OR auto-generated Dashboard |
| Profile metrics table | `<output_schema>.customers_raw_profile_metrics` |
| Drift metrics table | `<output_schema>.customers_raw_drift_metrics` |
| Dashboard | Databricks Dashboards → auto-created on first run |
| Set up alerts | Databricks SQL → Alerts → query metric tables |
| API reference | [Data Profiling Python SDK](https://api-docs.databricks.com/python/lakehouse-monitoring/latest/index.html) |

---

##  Further Reading

- [Official Databricks Docs — Data Profiling](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/)
- [Create a Profile via UI](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/create-monitor-ui)
- [Create a Profile via API](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/create-monitor-api)
- [Profile Metric Tables Schema](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/monitor-output)
- [Profile Alerts](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/monitor-alerts)
- [Custom Metrics](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/custom-metrics)

---

