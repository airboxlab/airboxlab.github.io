---
layout: post
title:  "Applying Machine Learning to Air Quality Events Detection"
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

This first unsuccessful attempt to give meaning to Foobot readings drove us to fundamental questions we eluded from the begining: what do we want to detect? Air quality events. Ok, but what is it?

Air quality event definition isn't as simple as it sounds. One may think of all pollution events that can happen at home, like when you burn your cooking, that will generally translates into a sudden increase of particulate matters.

But events aren't only about pollution, and we wanted to be able to detect more to get. What about opening a window? At first sight, it should be revealed by a change in temperature and/or humidity. But in what direction (increase or decrease)? In what proportion? 
This kind of event is also depending on external factors like season and location: in winter in New York, opening the window will have a event signature than during summer, or during winter in Sydney.

Also, events can be defined by different features, and not only by a sensor value at a specific moment of time: volatility of the value, difference between the minimum and maximum value over a period of time, necessary time to revert to initial value...

More over, let's think about sensor value context: a small change in one sensor value on a short period of time may not be significant when seen alone. But coupled with another small change from another sensor, this could get interesting. Even more if all sensors are impacted. And direction also has its role to play (increase vs decrease): a combination of increases and/or decreases may be frequent and not seen as an event, but another combination could be more rare, and worth paying attention to it.

Finally, air quality event detection is a non trivial problem. It requires more than simple algorithm or heuristics: this is a very good application of using machine learning algorithms.

![Graph for single sensor: what is relevant to characterize an event (absolute or diff, variance, ...), what period of time, mutiple events at a time]()

![Graph for several sensors: direction and trends, slow vs rapid increase/decrease in one/multiple sensor values]()

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




