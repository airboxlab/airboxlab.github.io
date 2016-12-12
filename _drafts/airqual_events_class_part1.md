---
layout: post
title:  "Applying Machine Learning to Air Quality Events Analysis: Part I"
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

<!-- http://wide-wallpapers.net/download/orion-nebula-in-the-infrared-wide-wallpaper-1920x1200.jpg -->
![orion galaxy?](http://localhost:4000/assets/airqual_events_class/orion_nebula.jpg)

<br/>

Why was Dark Vador wearing a mask? When you hear him breathing, it's obvious he has serious lung problems, at least severe asthma. Maybe he didn't pay attention to Death Star indoor air quality enough. 
But who could blame? This was a huge space to monitor, with troopers coming and going all the time, cosmic dust from all galaxies, chemicals and exhaust gazes from spaceships... 

Indoor air quality can harm your internals, we know that. But if we aren't able to know pollution sources, it gets tricky to avoid them; beyond taking basic actions when air quality degrades, it'd much more interesting to remove the source, change ones habits, automatically trigger the appropriate solution. It's useful to trigger your ventilation every time VOC level is above a threshold, but what if you become aware that it comes from your cleaning habbits?

This is what we're working on at Airboxlab, and this article will be the first of a series of 2 describing how we decided to tackle the problem of pollution source awareness and the benefits it could bring. It also relates the mistakes we did along the way, and how we learned from them.

## First approach: experiments and lessons learned

In the early days we were very enthusiastic: we'd have to monitor 1 or 2 houses for a while, ask users to note what they were doing (cooking, vacuum cleaning, ...) while we were monitoring. Even better, we could get super accurate insight by installing extra sensors and monitors. 

When the experiment ends, we collected data and worked with the [LIST](http://list.lu) to build our models. After a few weeks of work, conclusions came like a cold shower: no way to extract any significant pattern from collected data. Although graphs showed correlation between user activity and sensors response, no model could be built. Attempts to build classification models showed disappointing accuracy.

However, this wasn't a show stopper. We learned a lot from this first failed attempt. 

- although accuracy was good and users carefully reported time there were doing things that could impact air quality, experiment was too short in time and data was scarse
- dataset was imbalanced: some classes we wanted to learn about had no data at all
- conducting experiment with different sensors, in a controlled environment, isn't a portable approach. Our users wouldn't be equipped the same way, and they obviously wouldn't be that disciplined in reporting their actions. The resulting model wouldn't match our users habits

More importantly, we realized from last point that our experimented users were doing what we asked them. But our real users, the ones who bought our product and installed the app, would they try that hard to help us collect data? At that time, we weren't able to ask them about causes because we weren't telling them when *something* happened. Solution at that time was to open the app, select a time range and select a catagory. Actually, some users were already doing it. But without any trigger, it was very hard to have a clue about what happened, when exactly, for how long, etc. Existing tags dataset collected from real users input was inaccurate and improper to learning. 

It became clear initial problem could be divided into 2 distinct ones:

- how can we detect air quality events?
- how can we classify them?

And that would mean not a single model to build, but several. First things first, we started to work on air quality events detection so we could notify users and ask to tag the event.

## Air Quality Events: an (approximate) definition 

This first unsuccessful attempt to give meaning to Foobot readings lead us to fundamental questions we eluded from the beginning: what do we want to detect? Air quality events. Ok, but what is it?

Air quality event definition isn't as simple as it sounds. One may think of all pollution events that can happen at home, like when you burn your cooking, that will generally translates into a sudden increase of particulate matters (PMs).

But events aren't only about pollution, and we wanted to be able to detect more to get. What about opening a window? At first sight, it should be revealed by a change in temperature and/or humidity. But in what direction (increase or decrease)? In what proportion? 
This kind of event is also depending on external factors like season and location: in winter in New York, opening the window will have a different event signature than during summer, or in a different location like countryside. 

Also, events can be defined by different features, and not only by a sensor value at a specific moment of time: volatility of the value, difference between the minimum and maximum value over a period of time, necessary time to revert to initial value...

More over, let's think about sensor value context: a small change in one sensor value on a short period of time may not be significant when seen alone. But coupled with another small change from another sensor, this could get interesting. Even more if all sensors are impacted. And direction also has its role to play (increase vs decrease): a combination of increases and/or decreases may be frequent and not seen as an event, but another combination could be more rare, and worth paying attention to it.

Let's have a look at below graph: it's 24 hours in the day of a Foobot. Lot of things happen and it illustrates well the complexity of detecting events, without generating too much noise for the user (not every variation is an event). Some events are brutal changes of values in a short period of time, some last longer and there is a bigger period of time between start and end phases. Some involves all sensors but the 3 last are only reflecting on temperature and humidity.

![sensors graph](http://localhost:4000/assets/airqual_events_class/sensors_values_events.png)

Finally, air quality event detection is a non trivial problem. It requires more than simple algorithm or heuristics: this is a very good application of using machine learning algorithms.

## Air Quality Events Detection

### Detecting anomalies in the air

Air quality events we want to detect are actually abnormal and most probably infrequent changes in the values sent by Foobot sensors. For instance, when you open the oven, important amount of PMs can be released (if something have burned), or when you open your window, temperature and humidity may change, as well as VOCs, CO2 and PMs.

We took the following approach: we know we don't know all combinations of the events we want to detect, and we don't even know the events we want to detect. Actually, we want to detect and classify much more than a few categories, and the best way is to let users tell us what they do.

We also want to let users know with no delay that something happen and ask them to tag the event, providing a list of possible categories, but also a free text field so they can create new categories. 2 benefits to this approach:

- users want to be able to react immediately when air quality degrades 
- reducing time between detection and notification gives users more chances to remember what they were doing at that time. Sending notifications or emails every day in batch wouldn't be efficient.
     
To achieve this, we needed to design and implement 2 technical parts:

- something able to detect anomalies by comparing a vector of values to an existing and already organized set of vectors
- a streaming layer to continuously analyze incoming data  

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




