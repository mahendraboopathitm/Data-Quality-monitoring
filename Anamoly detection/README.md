#  Databricks Anomaly Detection — `customers_raw` Table

> Automatically monitors your data for quality issues — no manual checks needed.


##  What Is Anomaly Detection?

Anomaly Detection is a built-in Databricks feature (part of **Unity Catalog Data Quality Monitoring**) that automatically watches your tables and alerts you when something looks wrong with your data.

Think of it like a **smoke alarm for your data** — it learns what "normal" looks like over time, and raises a flag when things go off-pattern.

It is currently enabled on our **`customers_raw`** table.

---

##  What Does It Monitor?

Anomaly detection tracks two key things:

### 1.  Freshness
> *"Is the data being updated on time?"*

- Databricks studies the **historical pattern** of when the `customers_raw` table gets new commits/updates.
- It builds a model to predict **when the next update should arrive**.
- If an update is **unusually late**, the table is flagged as **Stale**.

**Example:** If `customers_raw` normally gets new data every hour, and 3 hours pass with no update — Databricks raises a freshness alert.

---

### 2.  Completeness
> *"Did enough rows arrive?"*

- Databricks studies the **historical row counts** written to `customers_raw` in the last 24 hours.
- It predicts a **normal range** of expected rows.
- If the number of rows written is **below the lower bound** of that range, the table is flagged as **Incomplete**.

**Example:** If `customers_raw` typically receives 10,000–15,000 new rows daily and only 200 rows arrive — Databricks raises a completeness alert.

---

### 3.  Percent Null *(Beta feature)*
> *"Are more columns showing NULL values than usual?"*

- For each column in `customers_raw`, Databricks tracks what percentage of newly written rows have NULL values.
- If any column's NULL rate jumps **higher than its historical norm**, the table is flagged as Incomplete.

**Example:** If the `customer_email` column is normally 2% null, and suddenly 40% of new rows have a null email — that's an anomaly.

---

##  Health Status Indicators

After each scan, every table in the schema gets one of these statuses (visible in Catalog Explorer):

| Status | Meaning |
|---|---|
|  **Healthy** | All checks passed. Data looks normal. |
|  **Unhealthy** | A freshness or completeness anomaly was detected. |
|  **Training** | Still learning the baseline. Shown for newly monitored tables while building the model. |
|  **Error** | Anomaly detection ran into a problem scanning the table. |
|  **Excluded** | Table is manually excluded from monitoring. |
|  **Not Enabled** | The schema does not have anomaly detection turned on. |

> **Note:** Our `customers_raw` table will show **Training** initially — this is normal. It needs enough historical data to establish a baseline before it can flag anomalies.

---

##  How It Works (Behind the Scenes)

```
customers_raw table
        │
        ▼
Databricks background job runs automatically
        │
        ├─── Checks FRESHNESS  ──► Compares actual commit time vs. predicted commit time
        │
        ├─── Checks COMPLETENESS ──► Compares actual row count vs. predicted range
        │
        └─── Checks % NULL (Beta) ──► Compares null rate per column vs. historical norm
                │
                ▼
        Results stored in:
        system.data_quality_monitoring.table_results
                │
                ▼
        Health indicators shown in Catalog Explorer
```

- **Databricks uses intelligent scanning** — it prioritizes frequently used or high-impact tables and scans them more often.
- **It does NOT modify the `customers_raw` table** in any way.
- **It does NOT add overhead** to any pipeline or job that writes to `customers_raw`.

---

##  How to View Results

### Option 1 — Catalog Explorer (Quickest)
1. Open **Databricks Catalog Explorer**.
2. Navigate to the schema containing `customers_raw`.
3. Look at the **health indicator icon** next to the table name.

### Option 2 — Data Quality Monitoring UI
1. Open the schema page in Catalog Explorer.
2. Click the **Details** tab.
3. Click **View Results**.
4. You'll see:
   - % of healthy tables
   - Active incidents (Unhealthy tables)
   - Reason (Freshness or Completeness)
   - Root cause (which upstream job caused the issue)
   - Impact level (High / Medium / Low)
   - 
<img width="1918" height="981" alt="image" src="https://github.com/user-attachments/assets/3fddb3d9-ee88-4ba1-b65a-e598dddccf54" />

### Option 3 — Query the System Table Directly
```sql
SELECT *
FROM system.data_quality_monitoring.table_results
WHERE table_name = 'customers_raw'
ORDER BY event_time DESC
LIMIT 50;
```

---

##  Setting Up Alerts

You can configure a **Databricks SQL Alert** to notify your team (via email, Slack, etc.) when `customers_raw` becomes Unhealthy.

1. Go to **Databricks SQL → Alerts**.
2. Create a new alert on the `system.data_quality_monitoring.table_results` table.
3. Set your condition (e.g., `status = 'UNHEALTHY'` and `table_name = 'customers_raw'`).
4. Configure your notification channel (email, Slack webhook, etc.).

---

##  Requirements Checklist

| Requirement | Status |
|---|---|
| Unity Catalog enabled workspace |  Required |
| Serverless compute available |  Required (enabled by default with UC) |
| MANAGE SCHEMA or MANAGE CATALOG privilege (to enable) |  Required for setup |
| SELECT or BROWSE privilege (to view results) |  Required for team members |

---

##  Known Limitations

- **Views and foreign tables** are not supported — only Delta tables.
- **Completeness** is based purely on row counts, not on specific column-level metrics like fraction of zeros or NaN values (Percent Null is separate, in Beta).
- **Smart scanning** may delay health indicators by up to 2 weeks for lower-priority tables on the first scan.
- **Event freshness** (based on event timestamp columns) is not supported in the current version.

---

## How to Disable (If Ever Needed)

> **Warning:** Disabling anomaly detection **permanently deletes** all monitoring history and results for the schema. This cannot be undone.

1. In Catalog Explorer, go to the schema → **Details** tab.
2. Click the pencil (edit) icon.
3. Toggle off **Anomaly Detection** in the Data Quality Monitoring dialog.
4. Click **Save**.

---

##  Quick Reference

| What | Where |
|---|---|
| Enable/Disable | Catalog Explorer → Schema → Details tab |
| View health status | Catalog Explorer → Table list (icon next to table name) |
| View incidents & trends | Schema Details → View Results |
| Raw results data | `system.data_quality_monitoring.table_results` |
| Set up alerts | Databricks SQL → Alerts |
| Metastore-level dashboard | Import `metastore-quality-dashboard.lvdash.json` template |

---

## Further Reading

- [Official Databricks Docs — Anomaly Detection](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/anomaly-detection/)
- [Alerts for Anomaly Detection](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/anomaly-detection/alerts)
- [Review Logged Results](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/anomaly-detection/results)

---


