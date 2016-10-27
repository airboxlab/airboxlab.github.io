---
layout: post
title:  ""
date:   2016-10-27 12:00:00
categories: "machine learning"
comments: true
author: Antoine Galataud
---

<style type="text/css">
.callout {
    float: right;
    margin-left: 5px;
}
</style>

![intro pic]({{ site.url }}assets/airqual_events_class/some.jpg)

## Intro

Series of 2 articles on how we're trying to solve indoor air quality events classification
What does it bring to the user: specific insights and advice, patterns of indoor air quality at home (e.g. cooking is the major source of pollution), custom integrations (trigger kitchen ventilation if cooking detected)
What are the main problems to solve

## 1st approach: experiments and lessons learned

2 weeks experiments with a few beta testers:

pros:

- good accuracy in measurements and users input
- controled environment where we put other sensors

cons:

- imbalanced training dataset: some classes have no data
- too short and not enough users to get enough data
- conducting experiment with different sensors isn't be portable

Conclusions from attempts to train with collected data

- single data point classification isn't the right approach
- need to detect air quality event first

## A definition of air quality event

Graph for single sensor: what is relevant to characterize an event (absolute or diff, variance, ...), what period of time, mutiple events at a time

Graph for several sensors: direction and trends, slow vs rapid increase/decrease in one/multiple sensor values

## Event detection

### Formalisation of the problem

Anomaly or outlier detection in a multi-dimensional dataset
Detection in a sliding time window

### Machine learning to the rescue

K-Means presentation + 2 dimensions sample clustering graph
Training: k-means training + centroids selection
Prediction: predict then compute distance
Production: K-Means prediction is fast to execute, no online learning/training as patterns don't evolve much

### Accuracy measurement
 
Sampling for some beta units
Visualization

## Conclusion

Summary
What to improve in detection
Next article about detected events classification




