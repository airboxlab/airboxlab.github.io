---
layout: post
title:  "A streaming pipeline for the IoT with Apache Spark & microservices"
date:   2016-08-22 08:00:00
categories: streaming microservices iot spark real-time
comments: true
author: Antoine Galataud
---

<style type="text/css">
.callout {
    float: right;
    margin-left: 5px;
}
</style>

![oil pipeline](http://localhost:4000/assets/data_pipeline/oilpipeline.jpg)

## What is Airboxlab doing with Foobot data?

We build Foobot, which sends indoor air quality data on a regular schedule. This data represents our users most valuable information, and is why they are buying a Foobot for. 
This data itself is the core of our business, and as engineers we have to secure the process of receiving and storing it. 

However, data alone isn't enough to give users a smart and complete experience so they can learn and act to improve their environment: an important part of the value is in the processing of this data, how we react to it, and at what speed. 
Indoor air quality is a matter of comfort and health, hence being alerted too late that the air you breath is polluted reduces interest and impact of the product a lot. In a similar manner, if you used a toxic cleanser but your ventilation system kicks in too late you just bought a mute doctor: he knows what you're suffering from but can't help!
 
That's why engineers at Airboxlab invest a lot in building a reliable, scalable and extensible streaming data pipeline. We want to make sure data is safe, that we react appriopriately and in a timely fashion to it, and that the next useful external system integration (be it a thermostat, HVAC system, IFTTT-like platform, customized stream...) will be straightforward.

## Anatomy of our streaming pipeline

### Sources, Sinks & Jobs

**Sources** in our pipelines are various. Most data (in volume) is emitted by Foobot sensors, but there are also events sent by applications (mobile, web), internal sources of static data (for instance device or user definitions that can serve to enrich context), and intermediate results in the pipeline that can be served as sources in other steps.

**Sinks** are also of various nature. We can categorize them as following:

- persistent storage, like SQL and NoSQL databases, HDFS and S3
- transient like message brokers to which analysis from processed input are sent on structured topics. Downstream pipeline clients subscribe to the topics they are interested in, process incoming messages, and send results to another sink 
- external like external API offered by partners

**Jobs** are the processes that transform input from a source to a desired result and output it to a sink. We have 2 types: _Spark Streaming_ jobs and _microservices_. 

**Communication** between sources, sinks & jobs depends on physical location of them (e.g. Foobot devices are living in users networks, data is crossing WAN), performance requirements (in memory vs over network), and flexibility (need to start/stop a source/sink dynamically).

Finally, a streaming pipeline is all about sources, sinks and how data goes from one to another. With (you guess so) complex interactions between them. That's what I'm going to detail in the next section.

### Sensor data streaming pipeline

This is our main data pipeline, the richer and more complex one (there are also other peripheral pipelines that fulfill specific requirements). We can summarize its functions as following:

1. **Ingestion**

   ![ingestion](http://localhost:4000/assets/data_pipeline/data_pipeline_ingestion.png)

   - **Storage**
   
      Between sensor data emission and storage the path must be as short as possible. The shorter it is, the less we expose ourself to failure and eventually lost data. Hence storage happens right away after reception. Job responsible for receiving and storing is extremely simple, load-balanced, and tolerant to failure.
   
      Also, we don't throw anything, even "junk" data that can sometimes be sent during temporary or permanent failure of a sensor: this won't be taken into account by downstream consumers of the main pipeline but is used for failure detection.

   - **Transformation**

      Sensor data is sent in a specific, raw, compressed format in order to save bandwidth. This step transforms this data into a json document. It also performs input validation in the meantime.

   - **Contextual enrichment**
    
      Raw sensor data is stored but not used as is downstream: we need to calibrate them in order to have meaningful values. Each device has its own calibration (each sensor is slightly different). Device-specific static data is also added to the context, and the whole forms the first rich json document that is sent to multiple downstream consumers.
      

2. **Aggregations & statistics computing**

   **Stats** are continuously computed for each device connected to our backend. Most of them are quite basic, like cumulative moving averages for each sensor, some others are based on regressions, like averages of top minimum or maximum values. 

   Most statistics computations never imply loading historical values: this would be a scaling problem. Whenever possible, we use a rolling/cumulative version of the statistic. 

   Statistics are used by several consumers:
   
   - calibration adapter that is used to adapt sensors baseline
   - other jobs in the pipeline that can trigger events based on statictics values
   - reporting tools, for instance one that email a periodic email to users telling him how was air quality in the past days or weeks<br/><br/>

   We also perform **aggregations** that are used downstream or served to users (values available by API, used by mobile and web apps). It reduces throughput 

3. **Alerting**

   ![ingestion](http://localhost:4000/assets/data_pipeline/data_pipeline_alerting.png)

   Alerting is the process of raising an alert when a particular event occurs. That is often related to a sensor data value crossing a standard or user-defined threshold, what we call instant threshold crossing, but can be more complex when it comes to "air quality event" detection. A simple type of event would be a pollution event, one that starts when a pollutant crosses a threshold and ends when it comes back to normal. This implies using mechanisms like windowing and stateful operations to preserve event state, or applying hysteresis for values constantly crossing thresholds up and down.

   There are 2 main types of consumers to these alerts:

   - Direct user notifiers, mostly using mobile push notifications
   - External systems notifications: for instance we can trigger the ventilation system linked to an Ecobee thermostat if particulate matter level is higher than what the user defined.<br/><br/>

4. **Machine learning**

   We apply machine learning at various levels, from calibration baseline improvement to air quality events detection. We use algorithm implementations provided by Spark ML that can be integrated in Spark Streaming jobs: here Spark reveals all its power, as all these bricks feet well together.
   Downstream consumers are somehow the same as for alerting.

   There are things to say here, but we'll cover this in a future blog post.

5. **Backup**

   ![ingestion](http://localhost:4000/assets/data_pipeline/data_pipeline_backup.png){: .callout}

   Even backup can be done in a near real-time manner: instead of fetching and storing data in an external backup store on a daily basis, why not benefit from streaming pipeline to do it at event processing time. This is how we implemented our incremental and asynchronous backup system. This has several advantages:

   - 2 seat belts: data is stored on arrival, and almost immediately backed up.
   - we can develop alternative pipelines to handle temporary or non mission critical tools: we store backup data in HDFS, that can be a source for [Spark Streaming](http://spark.apache.org/docs/latest/streaming-programming-guide.html#basic-sources).
   - we have an up-to-date offline version of our data on which we can perform heavy data mining and model training.


## Keeping the pipeline flexible

The backbone of our pipeline relies on Spark Streaming, which allows us to dedicate more or less resources (CPU & RAM) to streaming jobs, making it able to scale well. Dividing each functional process into separate jobs allows us to re-deploy or scale them independently.

An other important thing in our day-to-day job is to be able to "plug" a new job at the "periphery" of the pipeline quickly. For instance, we often decide to interact with a new external vendor API, and we don't want to think about anything else than:

- what will the service subscribe to?
- how will it interact with the API?
  
Most importantly, we want to avoid modifying other jobs in the pipeline as much as possible. This allows us to develop and deploy the new service without restarting anything else.

On another hand, we want subscriptions to be dynamic too. For instance, a new user connects on IFTTT and wants its [Foobot to turn on his WeMo switch](https://ifttt.com/connect/foobot/wemo_switch)  whenever global pollution level is too bad. To do that, we will record user configuration but also make the service responsible for interacting with IFTTT listen to relevant user's device events. 
Similarly, if user disconnects from Foobot channel, we don't want the service keep listening to his thresholds alerts.

In order to fulfill these requirements, we opted for a **microservices** architecture very early. This brings the following benefits:

- _easier to develop a new service_: we invested a bit in simplifying services creation, so creating a new one is a matter of minutes.
- _easier to extends a service functionality_: let's say you want to add a trigger to your IFTTT channel: you'll only need to change this service code, and retest it. Nothing else. This greatly reduces the risk of regression in other functionalities, and allows us to deploy faster and more often
- _efficient communication_: each service subscribes and reacts to the events it's supposed to listen to. We structured messaging topics so that services receive only what they need.
- _easier to test_: well, easier than having to deploy the whole pipeline and re-testing everything. Testing a pipeline is still harder than a classic piece of software though, so we invested in a internal testing microframework that helps writing integration tests.
- _easier to scale_ a particular integration: one of our integration becomes popular? we can spin up new services only for this type, making the whole architecture cost-efficient.

![subscribers](http://localhost:4000/assets/data_pipeline/data_pipeline_subscribers.png)

## Conclusion

This article gave an overview of one of the critical parts of our system, and explains why we choose Spark Streaming and microservices to implement it. 
A future series of articles will dive into more specific parts of the pipeline.


