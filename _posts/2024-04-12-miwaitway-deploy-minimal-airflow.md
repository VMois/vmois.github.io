---
layout: post
title: Deploying Apache Airflow on a resource-constrained VM on Google Cloud - MiWaitWay Part 1
tags: [data-engineering, miwaitway, airflow, google-cloud]
comments: false
readtime: true
slug: deploy-airflow-google-cloud-miwaitway
---

In this part of the MiWaitWay series, I focus on configuring Apache Airflow to consume as few resources as possible and deploy it to a small VM on Google Cloud. In addition, I need to leave some resources for a web app that will host a dashboard/map showing the average wait time for each bus stop.

To learn more about the MiWaitWay project and its high-level architecture, check [the introduction article](/miwaitway-average-wait-time-on-stop/).

## Selecting VM on Google Cloud

I chose an e2-small instance with 2GB of memory and two vCPUs (shared). It is relatively cheap (around $15 per month) and should be able to host Airflow and a web app for users.

## Configuring "lightweight" Airflow

I took an official [Docker Compose file for Airflow](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html), and to reduce memory footprint, I removed services such as:

- *worker*, executes tasks given by the scheduler; the scheduler can run tasks locally, so there is no need for a worker;
- *triggerer*, runs an event loop for deferrable tasks; I do not need this feature;
- *redis and flower*, without workers, there is no need to have Redis

It leaves only three services:

- *scheduler*, required to run tasks 
- *webserver*, required for the Airflow UI
- *Postgres*stores metadata for Airflow; for extreme savings, it can be replaced with SQLite, but I do not want to risk it.

I set up Airflow to use `LocalExecutor` to run tasks locally on the scheduler. To allow up to 3 DAGs to run simultaneously, parallelism was set to 3.

Related commits:
- [LocalExecutor and parallelism](https://github.com/VMois/miwaitway/commit/658d911d365780e391d805e98a2583491cea94ab#diff-3fde9d1a396e140fefc7676e1bd237d67b6864552b6f45af1ebcc27bcd0bb6e9R11-R12)

In the `docker-compose.yaml` file, I set a limit for memory usage and CPU for each service. If not enough memory is available for the scheduler or webserver, Airflow will probably crash. The memory required also depends on the number of DAGs and their complexity.

After some testing, I set the memory limit for the webserver at 800MB, the scheduler at 700MB, and Postgres at 350 MB. **1.85GB** of memory in total is allocated for Airflow only. This number might go up as I have not run any heavy DAGs so far. 

After setting up Airflow, there is not much memory left for the web app. This means the web app must be memory-efficient to fit on the VM. Hosting web applications on a serverless platform is always possible, but I prefer keeping everything on a single machine. As of now, I will not worry about web app efficiency.

Related commits:
- [Set memory limits for Airflow components](https://github.com/VMois/miwaitway/commit/fb95beb4f88ced93821092183ce089103328817b)

## Deploying Airflow

Airflow services run as Docker containers on VM. In the future, the web app and any other components will be added to the `docker-compose.yaml` file and run alongside. 

Before deploying, I modified `docker-compose.yaml` to use environment variables from the `.env` file instead of hardcoding those values in the code. 

Related commits:
- [Use .env file for Postgres service](https://github.com/VMois/miwaitway/commit/446800c94de44bffec5c81ed589b176de270c49c) and [use .env file for Postgres config in Airflow](https://github.com/VMois/miwaitway/commit/3bb57af318b3627ac081dd80b55f9b6d299a3159);
- [Use .env file for Airflow UI credentials](https://github.com/VMois/miwaitway/commit/d7469a5b567947703419e403b2be11a103aa29fe)
- [Keep config/logs/plugins folders](https://github.com/VMois/miwaitway/commit/9b41e32905dbb62b7b99ae89824365d1ae221fb0) for Airflow service to map to.

With the `docker-compose.yaml` file set up, I can proceed with deployment. After the MiWaitWay repository is cloned on a local VM filesystem, the process is straightforward from the VM shell:

1. Pull the latest changes from the main branch
2. Rebuild images
3. Restart the containers; the downtime is acceptable

The example of a simple bash script I use:

```bash
#!/bin/bash

if [ ! -f .env ]; then
    echo ".env is not found! Aborting!"
    exit 1
fi

git pull origin main
docker compose build
docker compose down && docker compose up -d
```

The DAGs folder is mapped into the local file system, so if only DAG files are modified, there is no need to restart the containers. Pulling the latest changes is enough, and Airflow containers will pick them up.

The issue arises when rebuilding a container(s) is necessary. In this case, Airflow will need to be restarted, which risks losing running DAGs. I will need to test this behavior later when deploying the first DAGs, but I am okay with this issue for now.

Related commits:
- [Add simple deploy bash script](https://github.com/VMois/miwaitway/commit/5d9d8fb1dc3b10cbdf1f00ef191ade22335bd7cd)
- [Make deploy script check for .env file](https://github.com/VMois/miwaitway/commit/432ea17dfb1c8a1e6550f506cbc112cf6c686233)

## Accessing Airflow UI

Now that Airflow runs on the VM, the next issue to solve is how to access the Airflow UI. One approach would be to allow HTTPS traffic, but I will need to provision an SSL certificate and configure a proxy such as nginx. Although it will be later required for a web app, I would like to avoid exposing Airflow to the Internet.

Another way to connect to Airflow UI is via SSH tunneling. Essentially, you forward data from a selected port on your local machine to a selected port on the VM via SSH. I run gcloud CLI tool on my local machine to establish a tunnel like this:

```bash
gcloud compute ssh airflow-and-web \
    --project miwaitway \
    --zone us-central1-c \
    -- -NL 8080:localhost:8080  # options that are passed to ssh command
```

SSH options (can be found in `man ssh`):

- `N`, do not execute a remote command. This option is only helpful for forwarding ports;
- `L`, forward local port 8080 to remote port 8080. `localhost` means remote host in this context.

Now, I can access Airflow UI on the VM from the `localhost:8080` address in a browser on my laptop. When I no longer need a connection, I can close the tunnel.
