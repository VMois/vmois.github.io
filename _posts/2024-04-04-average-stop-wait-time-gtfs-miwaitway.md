---
layout: post
title: Measuring average wait time at a stop using GTFS real-time data
tags: [data-engineering, miwaitway]
comments: false
readtime: true
slug: miwaitway-average-wait-time-on-stop
---

I like public transport, and I enjoy software engineering. For a long time, I wanted to build a project that worked with data end-to-end. From data ingestion throughout the analysis to the presentation to the external end user. The only thing that stopped me was not finding an analysis topic I would want to dive into (low-effort excuse, I know, but it is what it is). As I am growing an interest in public transportation and am a day-to-day user of it, I have recently found a topic I would like to explore - an average wait time at a stop.

## Why is average wait time important?

This is an oddly specific metric. But, the usability of public transport largely depends on its frequency. Ideally, in a densely populated area, you would want to see a bus/tram at a stop for the same route every 5-10 minutes. At such frequencies, there is no need to check an app; you walk to a stop, and there is a good chance your ride will be within minutes. There are other nonobvious consequences, such as safety. If you do not feel comfortable in a current vehicle, taking off at the next stop and waiting 5-10 minutes for the subsequent transport is not a big deal. Overall, reliable frequency is essential. Even if it is a less populated area, having a 15-minute wait time in 90+% of cases is already very impressive and will boost ridership.

## About the project

You cannot improve what you cannot measure, which is what this project is about. Luckily, many public transit agencies provide a real-time feed of their vehicles that can be used by services such as Google Maps and the [Transit app](https://transitapp.com/). Using the real-time vehicle location data, we can calculate how frequently vehicles visit each stop. Then, we can display the statistics to curious users via a web page. Below is a mockup of the UI.

<img src="/assets/posts/miwaitway/miwaitway_initial_ui_design.png" alt="Mockup of UI for the project" loading="lazy" />

For the data source, I chose the [GTFS-rt](https://developers.google.com/transit/gtfs-realtime) (GTFS real-time) feed from [Mississauga Transit Agency](https://www.mississauga.ca/miway-transit/) (MiWay). I use their services frequently and am personally interested in metrics. As a tribute to the data source, I named the project **MiWaitWay**.

I plan this project to be educational - learning new tools and solving new problems. The secondary goal is to help me understand a bit more about how public transport is run. Because I want to learn specific tools, part of the tech stack is already decided. As I want to learn more about Google BigQuery, I will build on Google Cloud. Using other Google services, such as cloud storage or compute engine (VMs), naturally makes sense. Apache Airflow - a widely known orchestration tool - will also be used. Other tech choices can be made later.

## High-level system design

To constrain costs and keep it simple (KISS), I will host the project on a single VM on Google Cloud. The high-level system design is below.

<img src="/assets/posts/miwaitway/miwaitway_initial_system_design.png" alt="Simple diagram of potential system design for the MiWaitWay project" loading="lazy" />

As a disclaimer, I have already started the project, but it is in the early stages. I will write about what has been done so far in the following articles. After that, I will continue developing and sharing insights about the project. Feel free to check the [MiWaitWay Github repository](https://github.com/VMois/miwaitway).
