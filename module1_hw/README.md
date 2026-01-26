# Module 1 Homework: Docker & SQL

This repository contains my solution for **Module 1 Homework: Docker & SQL**.

The homework focuses on:

* Running Docker containers
* Understanding Docker networking
* Loading data into PostgreSQL
* Querying data using SQL
* Basic Terraform workflow understanding

All commands and SQL queries used to answer the questions are documented below, as required.

---

## Question 1: Understanding Docker Images

**Command used:**

```bash
docker run -it --entrypoint bash python:3.13
```

Inside the container:

```bash
pip --version
```

**Result:**

```
pip 25.3
```
![alt text](image.png)
---

## Question 2: Docker Networking & docker-compose

Given the provided `docker-compose.yaml`, pgAdmin should connect to Postgres using:

* **Hostname:** `db`
* **Port:** `5432`

This is because containers in the same Docker network communicate using the service name and the internal container port.

---

## Data Preparation

### Download datasets

```bash
curl -L -o green_tripdata_2025-11.parquet https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet
curl -L -o taxi_zone_lookup.csv https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```
![alt text](image-1.png)
---

## PostgreSQL Setup (Docker)

```bash
docker run -it -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=ny_taxi -p 5433:5432 postgres:17-alpine
```

Connect to Postgres:

```bash
docker exec -it <container_id> psql -U postgres -d ny_taxi
```
I connected it using my container_id
---

## Table Creation

```sql
CREATE TABLE green_trips (
    lpep_pickup_datetime TIMESTAMP,
    trip_distance FLOAT,
    pulocationid INT,
    dolocationid INT,
    total_amount FLOAT,
    tip_amount FLOAT
);
```

```sql
CREATE TABLE zones (
    locationid INT,
    borough TEXT,
    zone TEXT,
    service_zone TEXT
);
```

---

## Data Loading (Parquet → Postgres)

Python container used for loading Parquet data:

```bash
docker run -it --rm -v $PWD:/data --entrypoint bash python:3.13
```

```bash
pip install pandas pyarrow psycopg2-binary sqlalchemy
```

```python
import pandas as pd
from sqlalchemy import create_engine

df = pd.read_parquet("/data/green_tripdata_2025-11.parquet")

engine = create_engine("postgresql://postgres:postgres@host.docker.internal:5433/ny_taxi")

df = df[[
    "lpep_pickup_datetime",
    "trip_distance",
    "PULocationID",
    "DOLocationID",
    "total_amount",
    "tip_amount"
]]

df = df.rename(columns={
    "PULocationID": "pulocationid",
    "DOLocationID": "dolocationid"
})

df.to_sql("green_trips", engine, if_exists="append", index=False)
Print("DONE")
```

CSV data (`taxi_zone_lookup.csv`) was imported into the `zones` table using pgAdmin.

---

## Question 3: Counting Short Trips

```sql
SELECT COUNT(*)
FROM green_trips
WHERE lpep_pickup_datetime >= '2025-11-01'
  AND lpep_pickup_datetime < '2025-12-01'
  AND trip_distance <= 1;
```

**Answer:** 8,007


![alt text](image-2.png)
---

## Question 4: Longest Trip Day

```sql
SELECT DATE(lpep_pickup_datetime) AS pickup_day,
       MAX(trip_distance) AS max_distance
FROM green_trips
WHERE trip_distance < 100
GROUP BY pickup_day
ORDER BY max_distance DESC
LIMIT 1;
```

**Answer:** 2025-11-14

![alt text](image-3.png)
---

## Question 5: Biggest Pickup Zone (Total Amount)

```sql
SELECT z.zone, SUM(g.total_amount) AS total_amount
FROM green_trips g
JOIN zones z ON g.pulocationid = z.locationid
WHERE DATE(lpep_pickup_datetime) = '2025-11-18'
GROUP BY z.zone
ORDER BY total_amount DESC
LIMIT 1;
```

**Answer:** East Harlem North

![alt text](image-4.png)
---

## Question 6: Largest Tip (East Harlem North Pickups)

```sql
SELECT z2.zone, SUM(g.tip_amount) AS total_tip
FROM green_trips g
JOIN zones z1 ON g.pulocationid = z1.locationid
JOIN zones z2 ON g.dolocationid = z2.locationid
WHERE z1.zone = 'East Harlem North'
  AND lpep_pickup_datetime >= '2025-11-01'
  AND lpep_pickup_datetime < '2025-12-01'
  AND z2.zone IN (
      'JFK Airport',
      'Yorkville West',
      'East Harlem North',
      'LaGuardia Airport'
  )
GROUP BY z2.zone
ORDER BY total_tip DESC;
```

**Answer:** JFK Airport

![alt text](image-5.png)
---

## Question 7: Terraform Workflow

Correct workflow sequence:

```text
terraform init
terraform apply -auto-approve
terraform destroy
```

---

## Notes

* All commands were executed using **GitHub Codespaces / Docker**
* SQL and shell commands are included directly in this README as required
