# CST8921 – Cloud Industry Trends
## Lab 4 – Azure Databricks and Analyzing Files with Databricks

**Student Number:** 041196196 
**Date:** February 22, 2026  
 

---

## 1. Objective

The objective of this lab was to gain hands-on experience with the Azure data engineering pipeline by exploring raw data stored in a cloud data lake, transforming it using Apache Spark in a Databricks notebook, and loading the refined data into an analytics store for querying. The lab demonstrates a complete Extract-Transform-Load (ETL) workflow using Apache Spark and Parquet-format datasets representing a synthetic e-commerce platform. Due to permission restrictions on the MOC lab subscription, the lab was completed using Databricks Community Edition as an alternative to Azure Synapse Analytics, achieving the same Spark-based learning outcomes.

---

## 2. Dataset Overview

The dataset used in this lab was a synthetic e-commerce dataset provided in the weekly Lab 4 files, stored in Parquet format across three files:

| File | Columns | Rows | Notes |
|---|---|---|---|
| customers.parquet | customer_id, country, signup_date | 50,000 | signup_date range: 2022–2025 |
| orders.parquet | order_id, customer_id, order_date, order_amount, order_status | 202,000 | ~1% duplicates |
| order_events.parquet | event_id, order_id, event_time, event_type | 422,100 | ~0.5% duplicates |

---

## 3. Environment Setup

### Tool Used: Databricks Community Edition

Due to permission restrictions on the MOC DS-10295 Azure subscription **(detailed in Section 7 – Issues Encountered)**, this lab was completed using **Databricks Community Edition** ([community.cloud.databricks.com](https://community.cloud.databricks.com)) instead of Azure Synapse Analytics or a full Azure Databricks workspace. Databricks Community Edition provides a free Apache Spark environment with full notebook support, covering all core Spark transformation and analysis objectives of this lab.

**Screenshot 1 – Databricks Community Edition Home**
![alt text](<Screenshot 2026-02-22 at 18.46.06.png>)
> *Caption: Databricks Community Edition workspace home screen confirming successful login and environment ready for use.*

---

## 4. Part 1 – Data Upload and Exploration

### Step 1 – Upload Data to Unity Catalog Volume

A Unity Catalog Volume named `lab4` was created under `workspace → default` in the Databricks Catalog. All three parquet files were uploaded directly to the volume at the path `/Volumes/workspace/default/lab4/`, replacing the ADLS Gen2 container upload that would be performed in a full Azure environment.

**Screenshot 2 – Files Uploaded to Unity Catalog Volume**

![alt text](<Screenshot 2026-02-22 at 18.47.49.png>)
> *Caption: Databricks Catalog showing the lab4 volume with all three parquet files successfully uploaded (customers.parquet, order_events.parquet, and orders.parquet) with the upload summary confirming 3 files uploaded.*

---

### Step 2 – Load and Inspect Data in Spark Notebook

A new Python notebook was created and connected to Serverless compute. Due to the parquet files containing nanosecond-precision timestamps (INT64 TIMESTAMP(NANOS)), which are not natively supported by the Serverless compute engine, pandas was used as an intermediate layer before converting to Spark DataFrames:

```python
import pandas as pd

df_customers = spark.createDataFrame(pd.read_parquet("/Volumes/workspace/default/lab4/customers.parquet"))
df_orders = spark.createDataFrame(pd.read_parquet("/Volumes/workspace/default/lab4/orders.parquet"))
df_events = spark.createDataFrame(pd.read_parquet("/Volumes/workspace/default/lab4/order_events.parquet"))

df_orders.printSchema()
df_orders.show(5)
```

**Screenshot 3 – Spark Notebook Schema and Data Preview**

![alt text](<Screenshot 2026-02-22 at 18.20.06.png>)
> *Caption: Databricks notebook output showing printSchema() and show(5) results for the orders dataset, confirming all three files loaded successfully into Spark DataFrames.*

---

## 5. Part 2 – Data Transformation

### Step 3 – Remove Duplicates

Duplicate records were identified and removed from the orders dataset. Record counts before and after deduplication were verified:

```python
df_dedup = df_orders.dropDuplicates()
print("Before:", df_orders.count())
print("After:", df_dedup.count())
```

**Screenshot 4 – Deduplication Row Count Comparison**

![alt text](<Screenshot 2026-02-22 at 18.23.33.png>)
> *Caption: Notebook output showing before and after row counts, confirming duplicate records were successfully removed from the orders dataset.*

---

### Step 4 – Fix Data Types and Create Derived Columns

Timestamp columns were converted to proper datetime format, and Year and Month columns were derived to support time-based analytics:

```python
from pyspark.sql.functions import to_timestamp, year, month

df_clean = df_dedup.withColumn("order_date", to_timestamp("order_date"))
df_transformed = (
    df_clean
    .withColumn("Year", year("order_date"))
    .withColumn("Month", month("order_date"))
)
df_transformed.show(5)
```

**Screenshot 5 – Transformed DataFrame with Year and Month Columns**

![alt text](<Screenshot 2026-02-22 at 18.24.25.png>)
> *Caption: Notebook output showing df_transformed.show(5) with the new Year and Month columns visible alongside the original order data.*

---

### Step 5 – Write Transformed Data to Refined Zone

The transformed DataFrame was written back to the Unity Catalog Volume in a `refined/` subdirectory, simulating the refined zone write that would normally target an ADLS Gen2 container:

```python
df_transformed.write.mode("overwrite").parquet("/Volumes/workspace/default/lab4/refined/")
print("Refined data written successfully!")
```

**Screenshot 6 – Refined Data Written Successfully**

![alt text](<Screenshot 2026-02-22 at 18.26.41.png>)
> *Caption: Notebook output confirming the transformed DataFrame was successfully written to the refined subdirectory in the Unity Catalog Volume.*

---

## 6. Part 3 – Load & Analyze Data

### Step 6 – SQL Analysis and Visualization

The transformed data was registered as a temporary view and queried using Spark SQL to produce revenue and order count aggregations by year:

```python
df_transformed.createOrReplaceTempView("refined_orders")

result = spark.sql("""
    SELECT Year, COUNT(*) AS total_orders, ROUND(SUM(order_amount), 2) AS total_revenue
    FROM refined_orders
    GROUP BY Year
    ORDER BY Year
""")
result.show()
```

A grouped count analysis was also run directly on the DataFrame:

```python
df_transformed.groupBy("Year").count().orderBy("Year").show()
```

**Screenshot 7 – SQL Aggregation Results by Year**

![alt text](<Screenshot 2026-02-22 at 18.28.01.png>)
> *Caption: Notebook output showing total orders and revenue grouped and ordered by year from the refined orders dataset.*

**Screenshot 8 – Grouped Year Count Analysis**
![alt text](<Screenshot 2026-02-22 at 18.25.10.png>)
> *Caption: Notebook output showing order counts grouped by Year using the Spark DataFrame groupBy operation.*

---

## 7. Issues Encountered

| # | Issue | Resolution |
|---|---|---|
| 1 | **No permission to create Azure Synapse workspace** — MOC DS-10295 subscription returned an authorization failure when attempting to create a Synapse workspace in the `cst8921` resource group | Attempted Azure Databricks workspace as the lab permits either tool |
| 2 | **No permission to create Azure Databricks workspace** — Same subscription returned `Microsoft.Databricks/workspaces/write` authorization error on deployment | Switched to Databricks Community Edition (free, no Azure subscription required) |
| 3 | **PARQUET_TYPE_ILLEGAL on table upload** — When uploading orders.parquet via the "Create or modify table" UI, the SQL warehouse preview failed with `Illegal Parquet type: INT64 (TIMESTAMP(NANOS,false))` | Uploaded files to a Unity Catalog Volume instead, bypassing the SQL warehouse preview entirely |
| 4 | **DBFS_DISABLED error in notebook** — Attempting to read files via `dbfs:/FileStore/` path in the notebook failed because Public DBFS root is disabled in the Serverless environment | Used Unity Catalog Volume paths (`/Volumes/workspace/default/lab4/`) instead of DBFS |
| 5 | **Nanosecond timestamp incompatibility with spark.read.parquet()** — Reading parquet files directly via Spark failed due to nanosecond-precision timestamps not supported by Serverless compute | Used `pandas.read_parquet()` as an intermediate reader then converted to Spark DataFrame via `spark.createDataFrame()` |

---

**Screenshot A – Synapse Workspace Permission Error**

![alt text](<Screenshot 2026-02-22 at 19.03.40.png>)
> *Caption: Azure Portal error showing authorization failure when attempting to create an Azure Synapse Analytics workspace on the MOC subscription.*

**Screenshot B – Databricks Workspace Permission Error**

![alt text](<Screenshot 2026-02-22 at 17.57.44.png>)
> *Caption: Azure Portal deployment error showing Microsoft.Databricks/workspaces/write permission denied on the MOC DS-10295 subscription.*

**Screenshot C – Parquet Type Illegal Error (SQL Warehouse)**

![alt text](<Screenshot 2026-02-22 at 19.05.25.png>)
> *Caption: Databricks UI error showing PARQUET_TYPE_ILLEGAL when attempting to preview orders.parquet via the SQL warehouse table creation flow.*

**Screenshot D – DBFS Disabled Error**

![alt text](<Screenshot 2026-02-22 at 19.09.03.png>)
> *Caption: Notebook error showing DBFS_DISABLED when attempting to read files from the /FileStore/ path in the Serverless compute environment.*

**Screenshot E – Nanosecond Timestamp Error**

![alt text](<Screenshot 2026-02-22 at 19.10.15.png>)
> *Caption: Notebook error showing PARQUET_TYPE_ILLEGAL nanosecond timestamp incompatibility when reading parquet directly via spark.read.parquet() on the Volume path.*

---

## 8. Findings and Analysis

### Data Quality Observations
- The `orders.parquet` file contained approximately 1% duplicate records (~2,000 rows), which were successfully removed using `dropDuplicates()`.
- The `order_events.parquet` file contained approximately 0.5% duplicates (~2,100 rows).
- The parquet files used nanosecond-precision INT64 timestamps, which required a pandas intermediate read step to handle compatibility with Databricks Serverless compute...A real-world data engineering issue that arises when parquet files are generated by systems with different timestamp precision standards.

### Analytics Results
- Order activity was distributed across 2023–2025 based on the order_date range defined in the dataset.
- Adding Year and Month partition columns improves query performance on large datasets by enabling partition pruning in downstream analytics.

### Key Architectural Insights
- **Unity Catalog Volumes** in Databricks serve a similar role to ADLS Gen2 containers, they provide structured, governed file storage that is directly accessible from Spark notebooks.
- **Apache Spark** provides distributed processing power for large-scale transformations including deduplication, type casting, and derived column generation.
- **Parquet format** is ideal for analytics pipelines due to its columnar storage and compression, though nanosecond timestamp precision can cause compatibility issues across different compute engines.
- **Databricks Community Edition** successfully replicates the core Spark notebook experience of a full Azure Databricks or Synapse Spark pool, making it a viable fallback when Azure subscription permissions are restricted.

---

## 9. Final Observations and Learning Outcomes

- Successfully uploaded raw Parquet data to a Databricks Unity Catalog Volume and explored it using Apache Spark in a Python notebook.
- Applied data quality transformations including deduplication, type casting, and derived column creation using PySpark.
- Wrote transformed data to a refined zone and queried it using Spark SQL aggregations to produce revenue and order count analytics by year.
- Understood the ETL pattern common in modern cloud data platforms and how raw and refined data zones are structured in a lakehouse architecture.
- Gained practical troubleshooting experience dealing with real cloud permission restrictions, compute compatibility issues, and parquet timestamp precision. These are all common real-world data engineering challenges.
- Demonstrated adaptability by successfully completing all lab objectives using an alternative tool (Databricks Community Edition) when the primary Azure environment was unavailable due to MOC subscription restrictions.

