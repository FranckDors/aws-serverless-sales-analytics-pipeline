# Serverless Sales Analytics Pipeline on AWS

An end-to-end, serverless data pipeline on AWS that ingests incremental daily orders data into S3, catalogs it with Glue, queries it with Athena, and visualizes it in QuickSight.

> Inspired by a data engineering tutorial on YouTube. I followed the build and documented my own implementation and results here.

## Architecture

<img width="2800" height="1040" alt="9a28e468-75ab-45f2-b218-b77523bfbcd3" src="https://github.com/user-attachments/assets/824f2573-8692-427f-8a4e-3c0ee81090a7" />

```
Orders data (CSV) → S3 → Glue Crawler → Glue Data Catalog → Athena (SQL) → QuickSight (Dashboards)
```

## Tools Used

- **Amazon S3**: storage for raw orders data, organized by date partition
- **AWS IAM**: user and role-based access (instead of using the root account)
- **AWS Glue**: crawler and Data Catalog for automatic schema discovery
- **Amazon Athena**: SQL queries directly on S3 data, no database needed
- **Amazon QuickSight**: charts and dashboards (SPICE import)

## Dataset

Superstore orders dataset from Kaggle. Only one day of data was loaded at a time to simulate incremental daily loads.

## How It Works

1. **IAM**: Created an IAM user instead of working as root, plus a service role for Glue with S3 access.
2. **S3**: Created a bucket with an `orders/` folder. Each day's data goes in its own partition folder, e.g. `snapshot_day=20170101/`.
3. **Glue Crawler**: Crawls the S3 path, infers the schema (columns and data types), and creates a table in the Data Catalog. The crawler is set to scan only new subfolders on each run.
4. **Incremental load**: Added a second day of data (`snapshot_day=20170104/`) and re-ran the crawler. A new partition was registered automatically.
5. **Athena**: Queried the cataloged table with standard SQL. Query results are stored in a separate S3 folder.
6. **QuickSight**: Connected to Athena, imported data into SPICE, built a bar chart and pie chart, and published a dashboard.

## Key Result: Partition Pruning

Filtering on the partition column reduced the data Athena scanned:

| Query | Data Scanned |
|-------|--------------|
| `SELECT *` (all partitions) | 4.46 KB |
| `WHERE snapshot_day = '20170101'` | 2.52 KB |

That is roughly a **54% reduction** in data scanned, which means lower cost and faster queries. The dataset is small, but the same principle scales to large data.

## Queries I Ran

```sql
-- 1. Total sales by category (the query behind the bar chart)
SELECT category, SUM(sales) FROM "db_superstoreswl1"."superstore_da01"
GROUP BY category;

-- 2. Full table scan across all partitions
SELECT * FROM orders;

-- 3. Partition-pruned query, one day only
SELECT * FROM orders
WHERE snapshot_day = '20170101';

-- 4. Profit by city (the data behind the pie chart)
SELECT city, SUM(profit) AS total_profit
FROM orders
GROUP BY city
ORDER BY total_profit DESC;
```
## Screenshots

Screenshots of each stage (S3 partitions, Glue table schema, Athena queries, and the QuickSight dashboard) are in the [`screenshots/`](screenshots/) folder.

| | |
|---|---|
| S3 partition folders | Glue table schema |
| Athena query and data scanned | QuickSight dashboard |

## What I Learned

- How a data catalog stores metadata while the data stays in S3
- How partitioning reduces Athena scan cost and query time
- SPICE vs. direct query tradeoffs in QuickSight
- Why to use IAM users and roles instead of the root account

## Future Improvements

- Schedule the crawler (e.g. daily) instead of running on demand
- Convert CSV to Parquet to cut scan costs further
- Automate ingestion with Lambda or Glue ETL jobs
- Provision the infrastructure with Terraform or CloudFormation

## Cost

Completed within the AWS free tier, for under $1.
