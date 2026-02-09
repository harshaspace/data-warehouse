# data-warehouse
BigQuery data warehouse


### Create external table from yellow taxi data

"""

CREATE OR REPLACE EXTERNAL TABLE `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_external`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://taxi-data-bucket-484819-a6/*.parquet']
);

"""

### Create regular table in BQ

"""
CREATE OR REPLACE TABLE `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`
AS
SELECT *
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_external`;

"""

1. 20332093

2. Data read estimate for following queries in External table 0 MB vs Materialised table 155.12 MB

"""
SELECT COUNT(DISTINCT PULocationID) as distinct_pickup_locations
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_external`;

"""

"""
SELECT COUNT(DISTINCT PULocationID) as distinct_pickup_locations
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;

"""

3. BigQuery is a columnar database, and it only scans the specific columns requested in the query. Querying two columns (PULocationID, DOLocationID) requires reading more data than querying one column (PULocationID), leading to a higher estimated number of bytes processed.

Estimate bytes : 155 MB
"""
SELECT PULocationID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;

"""

Estimate bytes : 310 MB
"""
SELECT PULocationID, DOLocationID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;
"""

4. In folowing case, The WHERE fare_amount = 0 clause forces BigQuery to read the fare_amount column regardless of whether you use COUNT(*) or COUNT(fare_amount).

Count 8333
"""
SELECT count(*)
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`
WHERE fare_amount = 0;

"""

5. For queries that filter by tpep_dropoff_datetime and order by VendorID, the best strategy is:
Partition by tpep_dropoff_datetime and Cluster by VendorID

"""
CREATE OR REPLACE TABLE `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_optimized`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID
AS
SELECT * 
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`;

"""

6. 

Estimate bytes: 310.24 MB
"""

SELECT DISTINCT VendorID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`
WHERE tpep_dropoff_datetime >= '2024-03-01' 
  AND tpep_dropoff_datetime < '2024-03-16'
ORDER BY VendorID;

"""

Estimate bytes: 26.84 MB
"""
SELECT DISTINCT VendorID
FROM `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips_optimized`
WHERE tpep_dropoff_datetime >= '2024-03-01' 
  AND tpep_dropoff_datetime < '2024-03-16'
ORDER BY VendorID;

"""

7. The data storage of external table is GCP bucket, that is why it shows zero estimate in BQ.

8. It is NOT always best practice to cluster your data in BigQuery, clustering helps when
    - You frequently filter or aggregate on specific columns
    - Queries use specific column patterns repeatedly
    - High-cardinality columns
    - Table size > 1 GB

9. SELECT COUNT(*) often shows 0 MB processed because BigQuery can answer it using table metadata, without scanning the actual data.

"""
SELECT count(*) from `curious-helix-484819-a6.taxi_dataset.yellow_taxi_trips`

"""
