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

The GTFS is very popular and is de facto a standard for transportation agencies around the world. For the MiWaitWay project, I will use information from both GTFS Schedule (a.k.a. static) and GTFS Realtime (real-time) feeds to calculate the average wait time at each bus line's stops.

You can find the GTFS data provided by MiWay transit agency, my chosen agency for this project, [here](https://www.mississauga.ca/miway-transit/developer-download/).

## Settings up service account for Airflow

Before I can do anything, I need to provide Airflow with permission to access BigQuery and Google Cloud Storage (GCS).

I created a service account via GCP UI and granted it the role of BigQuery Admin. The scope is too large, but it is okay for my small project, and I can revisit it later.

The next step was to create a GCS bucket and grant the service account access to it. I went to the bucket's permissions tab and added the service account as the storage bucket owner.

Following [this guide](https://docs.astronomer.io/learn/connections/bigquery), I created a connection ID in Airflow using the service account JSON file I downloaded from the GCP Console.

## Ingesting GTFS Schedule data to BigQuery

Ingesting GTFS Schedule data is relatively simple. I built an Airflow pipeline that downloads a single zip file that contains all the static feeds.
After unpacking the feeds, each one is uploaded to a Google Cloud Storage (GCS) bucket and then to the `raw_miway_data.[feed_name]` table in BigQuery. The pipeline runs once a day. It is probably still overkill, as I do not expect static data to change that often.

I did not encounter any big issues while developing the pipeline.
You can find the final source code [here](https://github.com/VMois/miwaitway/blob/9e6400dabf2829944e987f7e72b291f5a46c4f92/dags/extract_static_miway_data.py).
Let's take a look at GTFS Realtime data. This is where I had some problems.


## Ingesting GTFS Realtime data to BigQuery

If GTFS Schedule data has a single URL that contains a zip file with all feeds included, GTFS Realtime data works differently.
Each real-time feed has a unique URL that points to a Protobuf file.
If it is a single file, how does it provide real-time data? The Protobuf file gets updated with the most recent data every 10-15 seconds (at least, this is what I measured with MiWay). As a consequence, there is no historical real-time data available.

For my project, I currently only require real-time vehicle positions. I omit trip and service alert updates. You can find the details about the vehicle position feed [here](https://gtfs.org/realtime/feed-entities/vehicle-positions/).

### Fetching vehicle positions

My first idea to fetch the vehicle positions was to try and re-use Airflow. This would require DAG to run 24/7.
This is not exactly what Airflow was designed for and it created a few issues.

First is a scheduling overhead, Airflow will need to deal with rescheduling a task and it needs to make sure that this DAG runs all the time.
As soon as current task is finished, a new one needs to be scheduled within 10-15 seconds window otherwise update Protobuf file might be missed.
Even though it is not a tragedy to loss some data, I would not want that. I could make this task run indefenitely but it will not eliminate our next problem - occupying LocalExecutor slot on Airflow.

In the Airflow setup post, I opted to use LocalExecutor to fit The DAG that runs 24/7 would constantly occupy Airflow LocalExecutor. It means I would need to increase concurrency and allow for 2 or more DAG run at the same time pushing my minimal and barely running (on less than 2GB RAM) Airflow instance to not-so-fun-time.

Nevertheles, the first version of the real-time ingesting was built as an Airflow DAG.
It was collecting the vehicle positions from MiWay website and saving them in the bucket as CSV files.
I wanted another pipeline to run once an hour or so and load CSV files into the BigQuery.

In the DAG that collects real-time positions, I added an additional logic to collect positions in batches and correctly set the interval.
I did not wanted to re-schedule task every 10-15 seconds.
In additiona, I increased the DAG concurrency to 3 and used a `priority_weight` parameter to make sure that task responsbile for real-time ingestion gets scheduled over any other task in the future.
It definetely reduced the overhead on scheduler but did not eliminate all the pain points.

To detect when Protobuf file on MiWay website is updated, I calculate a hash of the file content and compared it with the next content.
If it is different, it means the file contains a new data.
I needed to use XCom values to properly pass the previous hashes between task runs which added more complexity.
To read the Protobuf files I used the [gtfs-realtime-bindings](https://github.com/MobilityData/gtfs-realtime-bindings/blob/master/python/README.md).
All in all, using Airflow worked for ingestion but felt off.
For interested, you can find the code for DAG [here](https://github.com/VMois/miwaitway/commit/213e086d3351abdb51a215f6f56c55eb12369c3c).

After some thinking, I decided to let Airflow do what it does the best - orchestrate, and move the real-time ingestion into a separate lighweight container.
This allowed me to control when and how exactly the code will be executed.
The container runs essentially the same Python code I used in the DAG in an infinite loop, collects vehicle locations in batches and saves them as CSV file in GCS bucket. 
You can find the initial commit [here](https://github.com/VMois/miwaitway/commit/a5508452511865e406f879386fbbbb29a2bf1b57).

I did make a few stupid mistakes by not properly clearing the data ([commit](https://github.com/VMois/miwaitway/commit/597536f92e09dc7fbf228c7378ca00d854269543)) and incorrectly deduplicating the rows ([commit](https://github.com/VMois/miwaitway/commit/9e6400dabf2829944e987f7e72b291f5a46c4f92)).
At the end, for each batch, the ingestor would de-duplicate vehicle positions based on vehicle_id and timestamp keys before uploading a file to the bucket. 
It was a good lesson to learn.

The same Google Cloud service account file I used for Airflow is mounted into the container to allow it access to the bucket. Not the best security but good enough for my use case.
The final code for `gtfs_realtime_ingestor` can be found [here](https://github.com/VMois/miwaitway/tree/9e6400dabf2829944e987f7e72b291f5a46c4f92/gtfs_realtime_ingestor).

### Loading vehicle positions CSV files from bucket to BigQuery

After I moved ingestion of vehicle positions to a separate container, I needed one last piece - loading CSV files to BigQuery.
I wrote a DAG called `load_realtime_to_bq` to do that.
It runs once every 90 minutes, loads all files into a staging table called `stage_vehicle_position` and then using MERGE statement merges stage table into the main `vehicle_position` table.

`vehicle_position` table is partitioned based on day in the timestamp field, and partition expiry set to 30 days. As discussed in the previous post, partitioning is needed to ensure fast query execution and partition expiration will ensure only the latest data is preserved saving on storage space.

The whole setup seemed good, until a merge query in the `load_realtime_to_bq` DAG failed because the stage table had two or more rows that matched.
Being confident that ingestor container eliminates all duplicates based `vehicle_id` and `timestamp` keys (read previous chapter), it means I have duplicated rows between the files.
An investigration confirmed that. Because I load all CSV files to stage table in one batch, it allowed same rows to end up in the same table and cause MERGE query to fail.

I decided to load each file one-by-one. In that case, MERGE query will not have issues with mutltiple rows matching.
And even if later files contain rows that were already loaded in the main table, MERGE query will simply match and update the row instead of ingesting a duplicate.
You can find the fix in [this commit](https://github.com/VMois/miwaitway/commit/f1232eaa2bd1e2de2d489920243d1fd1c51b6830).

I do suspect that duplicated rows have certain differences. Despite having the same vehicle_id and timestamp the later rows might have a better occupancy and other miscelenaous data.
I do not need it to calculate average wait time on a stop, but I might need to come back to it and ensure, for example, that files are loaded in the order of their creation instead of randomly as it is done now.

On a side note, while switching the DAG to file-by-file loading, I run into a problem of SQL template not rendering when I call BigQueryInsertJobOperator manually.
As it was [nicely explained to me](https://stackoverflow.com/questions/78427683/airflow-bigqueryinsertjoboperator-does-not-render-jinja2-template-file-when-call) it is an anti-pattern in Airflow.
When directly calling operator inside PythonOperator (a.k.a running Python), Airflow has no ability to perform additional steps to setup an operator (like render SQL template). Interestingly, many users expect operators behave like normal Python classes/functions and this creates a bit of a problem.
