---
layout: post
title: Transforming raw data to use BigQuery GEOGRAPHY functions - MiWaitWay Part 4
tags: [data-engineering, airflow, bigquery, google-cloud, miwaitway, gtfs]
comments: false
readtime: true
slug: transform-raw-data-to-use-bigquery-geography-functions-miwaitway
---

In this part of the MiWaitWay series, I transform raw data loaded in the previous article so that it can be used with BigQuery's powerful GEOGRAPHY functions. 

To learn more about the MiWaitWay project, check [the introduction article](/miwaitway-average-wait-time-on-stop/).
Check [the previous article](/ingest-gtfs-data-to-bigquery-miwaitway/), where I talk about loading raw data into BigQuery.

## What are BigQuery GEOGRAPHY functions, and why do I want to use them?

Geospatial analysis is complex.
I need to match vehicle locations to stop locations within a certain distance radius.
I do not want to roll out my distance calculation.

BigQuery provides [a suite of functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/geography_functions) to efficiently perform geospatial analysis.
The functions operate on GEOGRAPHY type, but I only have raw data containing latitude and longitude.
I aim to write a DAG to transform lat and lon columns into GEOGRAPHY columns for raw `stops` and `vehicle_positions` tables.

## Transforming raw data to work with GEOGRAPHY functions

The first step is to create the `prod_miway_data` dataset in BigQuery, as well as the `vehicle_position` and `stops` prod tables.
Some notable changes:
- Latitude and longitude columns were replaced with not null `location_point` column of GEOGRAPHY type for both tables;
- In the prod `vehicle_position` table, `vehicle_id`, `timestamp` and `trip_id` columns became not null;
- In the prod `stops` table, the `stop_id` column became not null.

You can find CREATE TABLE statements for the [vehicle_position](https://github.com/VMois/miwaitway/blob/b747a3ac49f65862100428a47fd9cec8cce3550c/dags/sql/setup/create_prod_vehicle_position_table.sql) and the [stops](https://github.com/VMois/miwaitway/blob/b747a3ac49f65862100428a47fd9cec8cce3550c/dags/sql/setup/create_prod_stops_table.sql) tables.
Both have 30-day partition expiry times, similar to raw tables.

The next step is to create DAG to transfer data from raw to prod with some transformations. 
Currently, timestamps in the `vehicle_position` table are in UTC.
To match the MiWay Agency's timezone, I decided to transform data according to their local time.
The DAG will start around 1 AM local time and process all data for the previous day.

The DAG contains two tasks:

1. Run the MERGE statement to transform data for `vehicle_position`, from raw table to prod. 
The statement can be found [here](https://github.com/VMois/miwaitway/blob/b747a3ac49f65862100428a47fd9cec8cce3550c/dags/sql/transform_raw_vehicle_positions_to_prod.sql).
2. Run the CREATE OR REPLACE statement to fully replace prod `stops` table each run.
I replace the table completely because the amount of data is small.
The statement can be found [here](https://github.com/VMois/miwaitway/blob/b747a3ac49f65862100428a47fd9cec8cce3550c/dags/sql/transform_raw_stops_to_prod.sql).

The final code for the DAG can be found [here](https://github.com/VMois/miwaitway/blob/b747a3ac49f65862100428a47fd9cec8cce3550c/dags/transform_raw_to_prod.py).

## Issues

I encountered several issues that I think are worth mentioning.

### Duplicated data in vehicle_positions

After running the first version of raw-to-prod DAG, I encountered an error with the MERGE statement indicating that two source rows matched for the same target row.
The MERGE statement used a `vehicle_id` and `timestamp` for matching.
The SQL query below confirmed that there are duplicates when only using those two columns.

```sql
SELECT vehicle_id, timestamp, COUNT(*)
FROM `miwaitway.raw_miway_data.vehicle_position`
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = '2024-05-08' 
GROUP BY vehicle_id, `timestamp`
HAVING COUNT(*) > 1
```

After debugging, I realized that vehicle_id and timestamp can be the same for some rows, but trip_id will differ.
It looks like this happens when the bus reaches the end of its route and switches to a new route (and direction), such as North to South.
Whether it is a MiWay issue or I did not read the GTFS specification, I added `trip_id` as a third column to the MERGE statement.

### Zombie tasks in Airflow

Sometimes, Airflow tasks were becoming zombies.
After monitoring the `docker stats` output, I realized that the scheduler container is getting close to its memory limit of 700MB and that the scheduler can run two tasks simultaneously.
This combination probably caused zombie task issues (usually out-of-memory errors).
I decided to increase the scheduler's memory by 50MB (remember, our whole instance only has 2GB) and set task parallelism to 1.
In addition, I saw that PostgreSQL is consuming only 50MB of RAM (hooray!), so I decreased its memory limit from 300MB to 100 MB.

After making those changes, it seems like the scheduler works fine, and there are no zombie tasks.
But it is a good reminder that resources are constrained, and I should not forget about it.

### Missing vehicle position data

When I got the first data in prod tables I was keen on doing some preliminary analysis.
A long time ago, I wrote the first version of the SQL script to match positions to stops.
You can find the query below.
I do not want to go into detail about how it works.
I plan to discuss the analysis later in the series.

```sql
DECLARE var_distance_from_stop INT64;
DECLARE var_today_est_limit TIMESTAMP;
DECLARE var_yesterday_est_limit TIMESTAMP;

SET var_today_est_limit = TIMESTAMP("2024-05-10 04:00:00 UTC");
SET var_yesterday_est_limit = TIMESTAMP("2024-05-09 04:00:00 UTC");
SET var_distance_from_stop = 30;  -- in meters


WITH selected_positions AS (
  SELECT * FROM `miwaitway.prod_miway_data.vehicle_position` WHERE timestamp BETWEEN var_yesterday_est_limit AND var_today_est_limit
)
, vehicles_near_stops AS (
  SELECT
      stop_id,
      stops.stop_name,
      vehicle.vehicle_id AS vehicle_id,
      vehicle.timestamp,
      vehicle.trip_id,
  FROM
      `miwaitway.prod_miway_data.stops` AS stops,
      selected_positions AS vehicle
  WHERE ST_DWithin(
          stops.location_point,
          vehicle.location_point,
          var_distance_from_stop)
) 
, enriched AS (
  SELECT 
      vns.stop_id,
      vns.vehicle_id,
      vns.timestamp as timestamp,
      t.route_id,
      t.direction_id,
      vns.trip_id,
      LEAD(vns.timestamp) OVER (PARTITION BY stop_id, t.route_id, t.direction_id ORDER BY vns.timestamp DESC) as previous_timestamp,
      LEAD(vns.vehicle_id) OVER (PARTITION BY stop_id, t.route_id, t.direction_id ORDER BY vns.timestamp DESC) as previous_vehicle_id,
      LEAD(vns.trip_id) OVER (PARTITION BY stop_id, t.route_id, t.direction_id ORDER BY vns.timestamp DESC) as previous_trip_id,
  FROM vehicles_near_stops AS vns
  JOIN `miwaitway.raw_miway_data.trips` AS t ON vns.trip_id = t.trip_id
)
SELECT stop_id, route_id, ROUND(AVG(TIMESTAMP_DIFF(timestamp, previous_timestamp, SECOND) / 60)) as avg_wait_time
FROM enriched
WHERE vehicle_id != previous_vehicle_id AND trip_id != previous_trip_id
GROUP BY stop_id, route_id
```

I ran the query and got data that initially looked okay, but the average wait time was too long in some instances.
I saw that buses did not arrive for several hours in the middle of the day for some stops. It felt off. Either MiWay data is off, or I am doing something wrong (disclaimer: it was me).

After extensive checking and comparing bus timestamps near stops with the official schedule, I concluded that some data was lost on my side.
The query to transform raw to prod data looked fine.
I went to check the logs for the vehicle position ingestor from [the previous article](/ingest-gtfs-data-to-bigquery-miwaitway/), and indeed, there were gaps because, for some reason, the ingestor crashed.

This was not a fun experience. In addition to trying to understand if my analysis is correct, I do not want to question if all the data is available.
Currently, the vehicle positions ingestor container keeps all 100 batches (around 15-20 minutes of data) in memory. In case of a crash, the data is lost.
There are other issues, such as the `vehicle_id` column being missed sometimes.
Overall, the ingestor container should become more reliable.

## Conclusion

When I started with this part, I expected to finish it quickly.
However, I discovered several important issues that need to be dealt with.
Before calculating an average wait time, I wanted to ensure the data would not be lost easily.

In the following article, I will improve vehicle position ingestor reliability.
You can find the planned work in [this issue](https://github.com/VMois/miwaitway/issues/28).
Thank you for your attention.

