# data-warehouse

BigQuery data warehouse for yellow taxi trip data analysis.

## Setup and Data Structures

### Create external table from yellow taxi data

```sql
CREATE OR REPLACE EXTERNAL TABLE `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_external`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://taxi-data-bucket-484819-a6/*.parquet']
);
```

### Create regular table in BQ

```sql
CREATE OR REPLACE TABLE `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`
AS
SELECT *
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_external`;
```

## Key Findings and Analysis

### 1. Total Record Count

**20,332,093** total records in yellow taxi trips dataset.

### 2. External Table vs Materialized Table - Query Performance

Data read estimate for distinct pickup locations query:
- **External table**: 0 MB (stored in GCP bucket)
- **Materialized table**: 155.12 MB

**External table query:**
```sql
SELECT COUNT(DISTINCT PULocationID) as distinct_pickup_locations
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_external`;
```

**Materialized table query:**
```sql
SELECT COUNT(DISTINCT PULocationID) as distinct_pickup_locations
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;
```

### 3. Columnar Database - Column Selection Impact

BigQuery is a columnar database that only scans the specific columns requested in the query. Querying two columns requires reading more data than querying one column.

**One column query (155 MB):**
```sql
SELECT PULocationID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;
```

**Two column query (310 MB):**
```sql
SELECT PULocationID, DOLocationID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;
```

### 4. WHERE Clause Column Scanning

The WHERE clause forces BigQuery to read specific columns regardless of the SELECT clause. This query returned **8,333** records where fare_amount equals zero.

```sql
SELECT count(*)
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`
WHERE fare_amount = 0;
```

### 5. Partitioning and Clustering Strategy

For queries that filter by `tpep_dropoff_datetime` and order by `VendorID`, the optimal approach is to partition by date and cluster by VendorID:

```sql
CREATE OR REPLACE TABLE `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_optimized`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID
AS
SELECT * 
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;
```

### 6. Performance Comparison - Optimized vs Non-Optimized

**Without partitioning/clustering (310.24 MB):**
```sql
SELECT DISTINCT VendorID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`
WHERE tpep_dropoff_datetime >= '2024-03-01' 
  AND tpep_dropoff_datetime < '2024-03-16'
ORDER BY VendorID;
```

**With partitioning/clustering (26.84 MB):**
```sql
SELECT DISTINCT VendorID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_optimized`
WHERE tpep_dropoff_datetime >= '2024-03-01' 
  AND tpep_dropoff_datetime < '2024-03-16'
ORDER BY VendorID;
```

**Result: ~91% reduction in data scanned with proper optimization.**

### 7. External Table Storage

External table data is stored in the GCP bucket, which is why BigQuery shows zero estimate bytes. The storage costs are incurred at the bucket level, not BigQuery.

### 8. When to Use Clustering

Clustering is NOT always best practice. It helps when:
- You frequently filter or aggregate on specific columns
- Queries use specific column patterns repeatedly
- Working with high-cardinality columns
- Table size exceeds 1 GB

### 9. COUNT(*) Query Optimization

SELECT COUNT(*) often shows 0 MB processed because BigQuery can answer it using table metadata, without scanning the actual data.

```sql
SELECT count(*) from `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;
```
