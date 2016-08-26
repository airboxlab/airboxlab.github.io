---
layout: post
title:  "A streaming pipeline for the IoT with Apache Spark & microservices"
date:   2016-08-22 08:00:00
categories: streaming microservices iot spark realtime
comments: true
author: Antoine Galataud
---

## IoT at Airboxlab

We build Foobot, which sends indoor air quality data on a regular schedule. This data represents our users most valuable information, and is why they are buying a Foobot for. 
This data itself is the core of our business, and as engineers we have to secure the process of receiving and storing it. 

However, data alone isn't enough to give users a smart and complete experience so they can learn and act to improve their environment: an important part of the value is in the processing of this data, how we react to it, and at what speed. 
Indoor air quality is a matter of comfort and health, hence being alerted too late that the air you breath is polluted reduces interest and impact of the product a lot. In a similar manner, if you used a toxic cleanser but your ventilation system kicks in too late you just bought a mute doctor: he knows what you're suffering from but can't help!
 
That's why engineers at Airboxlab invest a lot in building a reliable, scalable and extensible streaming pipeline. We want to make sure data is safe, that we react appriopriately and in a timely fashion to it, and that the next useful external system integration (be it a thermostat, HVAC system, IFTTT-like platform, customized stream...) will be straightforward.

## Anatomy of a streaming pipeline

Sources & Sinks

Streaming pipeline
  Ingestion
    - storage
    - transformation
    - contextual enrichment
  Stats computing
  Alerting
  Machine learning
  Backup

Offline data pipeline

## Pipeline subscribers

Peripheral services that consume events created in the pipeline
Fine-grained configuration based on topic subscription, by user or device
Comes and goes, as features are implemented (push notifications, external devices triggers, IFTTT, ...)
Microservices help
 - easy to setup a new service (service creation is simplified)
 - easy to subscribe and react to an event (topics are structured and documented)
 - easy to test (micro framework that simplifies integration testing)
 - easy to scale a particular feature (e.g. IFTTT more used? spin up new services)

