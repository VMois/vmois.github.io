---
layout: post
title: Ingesting GTFS data into BigQuery - MiWaitWay Part 3
tags: [data-engineering, airflow, bigquery, google-cloud, miwaitway]
comments: false
readtime: true
slug: ingest-gtfs-data-to-bigquery-miwaitway
---

In this part of the MiWaitWay series, I will describe how I loaded data from the MiWay website into BigQuery.

To learn more about the MiWaitWay project, check [the introduction article](/miwaitway-average-wait-time-on-stop/).

## What is GTFS data?

In the previous articles, I mentioned GTFS data several times. But what is it?

General Transit Feed Specification (GTFS) is an open standard that transit agencies can use to publish relevant information about the transit system.
The standard consists of two main parts: GTFS Schedule and GTFS Realtime.
GTFS Schedule contains different feeds about stops, timetables, and more. The information is stored in CSV files and usually does not change often.
On the other hand, GTFS Realtime provides near real-time feeds on vehicle positions, service alerts, etc. It is powered by the Protobuf protocol. You can learn more about GTFS by visiting an [official website](https://gtfs.org/).

The GTFS is very popular and is de facto a standard for transportation agencies around the world.
For the MiWaitWay project, I will use information from both GTFS Schedule and GTFS Realtime feeds to calculate the average wait time at each bus line's stops.

You can find the GTFS data provided by MiWay transit agency, my chosen agency for this project, [here](https://www.mississauga.ca/miway-transit/developer-download/).

## Settings up service account for Airflow

Before I can do anything, I need to provide Airflow with permission to access BigQuery and Google Cloud Storage (GCS).

I created a service account via GCP UI and granted it the role of BigQuery Admin. 
The permission scope is large, but it is okay for my small project, and I can revisit it later.

The next step was to create a GCS bucket and grant the service account access to it. I went to the bucket's permissions tab and added the service account as the storage bucket owner.

Following [this guide](https://docs.astronomer.io/learn/connections/bigquery), I created a connection ID in Airflow using the service account JSON file I downloaded from the GCP Console.

## Ingesting GTFS Schedule data to BigQuery

Ingesting GTFS Schedule data is relatively simple. I built an Airflow pipeline that downloads a single zip file that contains all the feeds (one feed = one CSV file).
After unpacking the feeds, each one is uploaded to a GCS bucket and then to the `raw_miway_data.[feed_name]` table in BigQuery. The pipeline runs once a day. It is probably still overkill, as I do not expect static data to change that often.

I did not encounter any big issues while developing the pipeline.
You can find the final source code [here](https://github.com/VMois/miwaitway/blob/9e6400dabf2829944e987f7e72b291f5a46c4f92/dags/extract_static_miway_data.py).
Let's take a look at GTFS Realtime data. This is where I had some problems.


## Ingesting GTFS Realtime data to BigQuery

If GTFS Schedule data has a single URL that contains a zip file with all feeds included, GTFS Realtime data works differently.
Each real-time feed has a unique URL that points to a Protobuf file.
If it is a single file, how does it provide real-time data? The Protobuf file gets updated with the most recent data every 10-15 seconds (at least, this is what I measured with MiWay). As a consequence, there is no historical real-time data available.

For my project, I currently only require real-time vehicle positions. I omit trip and service alert updates. You can find the details about the vehicle position feed [here](https://gtfs.org/realtime/feed-entities/vehicle-positions/).

### Fetching vehicle positions

My first idea to fetch the vehicle positions was to use Airflow. This would require DAG to run 24/7.
This differs from what Airflow was designed for, creating a few issues.

First, there is a scheduling overhead. Airflow will need to deal with rescheduling a task and ensure that this DAG runs all the time.
As soon as the current task is finished, a new one needs to be scheduled within a 10- 15-second window; otherwise, the updated Protobuf file might be missed.
Even though losing some data is not a tragedy, I would not want that.
I could make this task run indefinitely, but it would not eliminate our next problem—occupying the LocalExecutor slot on Airflow.

In the [setup post](/deploy-airflow-google-cloud-miwaitway/), I used LocalExecutor as I had minimal resources to host the Airflow instance. The DAG that runs 24/7 would constantly occupy LocalExecutor. This means I would need to increase concurrency and allow for 2 or more DAG to run at the same time, pushing my minimal and barely running (on less than 2GB RAM) Airflow instance to not-so-fun time.

Nevertheless, the first version of the real-time ingesting was built as an Airflow DAG.
It collected the vehicle positions from the MiWay website and saved them in the bucket as CSV files.
I wanted another pipeline to run once an hour or so and load CSV files into the BigQuery.

In the DAG, I added logic to collect positions in batches to avoid rescheduling tasks every 10-15 seconds.
A single batch contains data from a hundred unique Protobuf files.
It definitely reduced the overhead on the scheduler but did not eliminate all the pain points.
In addition, I increased the DAG concurrency to 3 and used a `priority_weight` parameter in a task operator to make sure that the task responsible for real-time ingestion gets scheduled over any other task in the future.

To detect when the Protobuf file on the MiWay website is updated, I calculate a hash of the file content and compare it with the hash computed from a file downloaded 2 seconds later.
If it is different, it means the file contains new data.
I needed to use XCom values to correctly pass the previous hashes between task runs, which added more complexity.
To read the Protobuf files, I used the [gtfs-realtime-bindings](https://github.com/MobilityData/gtfs-realtime-bindings/blob/master/python/README.md).
All in all, using Airflow worked for ingestion but felt off.
If interested, you can find the code for DAG [here](https://github.com/VMois/miwaitway/commit/213e086d3351abdb51a215f6f56c55eb12369c3c).

After some thinking, I decided to let Airflow do what it does best - orchestrate and move the real-time ingestion into a separate lightweight container.
This allowed me to control when and how exactly the code will be executed.
The container operates by running the same Python code from the DAG in an infinite loop. It accumulates vehicle locations in batches and stores them as a CSV file in a GCS bucket. 
You can find the initial commit [here](https://github.com/VMois/miwaitway/commit/a5508452511865e406f879386fbbbb29a2bf1b57).

I did make a few stupid mistakes by not correctly clearing the data ([commit](https://github.com/VMois/miwaitway/commit/597536f92e09dc7fbf228c7378ca00d854269543)) and incorrectly deduplicating the rows ([commit](https://github.com/VMois/miwaitway/commit/9e6400dabf2829944e987f7e72b291f5a46c4f92)).
In the end, for each batch, the vehicle positions ingestor would deduplicate vehicle positions based on `vehicle_id` and `timestamp` columns from the feed before uploading. 
It was a good lesson to learn.

The same Google Cloud service account file I used for Airflow is mounted into the container to allow it access to the bucket. The final code for `gtfs_realtime_ingestor` can be found [here](https://github.com/VMois/miwaitway/tree/9e6400dabf2829944e987f7e72b291f5a46c4f92/gtfs_realtime_ingestor).

### Loading vehicle positions CSV files from bucket to BigQuery

After I moved the ingestion of vehicle positions to a separate container, I needed one last piece: loading CSV files to BigQuery.
I wrote a DAG called `load_realtime_to_bq` to do that.
It runs once every 60 minutes, loads all CSV files from the bucket into a staging table called `stage_vehicle_position`, and then, using the MERGE statement, merges the stage table into the main `vehicle_position` table.

The `vehicle_position` table is partitioned based on a day extracted from the timestamp column, and the partition expiry is set to 30 days. As discussed [in the data flow post](/data-flow-bigquery-miwaitway/), partitioning is needed to ensure fast query execution, and partition expiration will ensure only the latest data is preserved, saving on storage space.

The whole setup seemed good until a merge query in the `load_realtime_to_bq` DAG failed because the staging table had two or more matched rows ([BigQuery docs with details](https://cloud.google.com/bigquery/docs/reference/standard-sql/dml-syntax#example_7)).
Being confident that the ingestor container eliminates all duplicates based on the `vehicle_id` and `timestamp` columns meant I had duplicated rows between the files.
An investigation confirmed that. Because I loaded all CSV files to the staging table in one batch, the duplicate rows ended up in the same table, causing the MERGE query to fail.

I decided to load each file one by one. In that case, the MERGE query will not have issues with multiple rows matching.
And even if later files contain rows already loaded in the main table, the MERGE query will match and update the row instead of ingesting a duplicate.
You can find the fix in [this commit](https://github.com/VMois/miwaitway/commit/f1232eaa2bd1e2de2d489920243d1fd1c51b6830).

I do suspect that duplicated rows have specific differences. Despite having the same `vehicle_id` and `timestamp` values, the newer rows might have better occupancy and other miscellaneous data.
I do not need it to calculate the average wait time on a stop, but I might need to revisit it and ensure, for example, that files are loaded in the order in which they were created instead of randomly.

On a side note, while switching the DAG to file-by-file loading, I encountered a problem with the SQL template not rendering when I called `BigQueryInsertJobOperator` manually.
As it was [nicely explained to me](https://stackoverflow.com/questions/78427683/airflow-bigqueryinsertjoboperator-does-not-render-jinja2-template-file-when-call) it is an anti-pattern in Airflow.
When directly calling an operator inside PythonOperator, Airflow cannot perform additional steps to set up an operator (like rendering SQL template).

Later in the project, I will work on a monitoring system to ensure that the DAG and the container continuously ingest the vehicle positions.

## Conclusion

I am currently ingesting all the necessary raw data to calculate the wait time at bus stops.
But before I can work on wait times, I will need to transform the data into a form that allows me to use powerful [BigQuery GEOGRAPHY functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/geography_functions).
This is a topic for the next article. Thank you for your attention.
