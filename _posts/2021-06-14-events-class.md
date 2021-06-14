---
layout: post
title:  "Crowd sourced classifier to identify air pollution sources"
date:   2021-06-14 00:00:00
categories: "data_science air_quality classifier machine_learning"
comments: false
author: Antoine Galataud, Inouk Bourgon
---

<style type="text/css">
.center {
    display:block;
    margin: 0 auto;
}
</style>

## Abstract

Foobot designs and manufactures indoor air quality monitors since 2013. These monitors are connected to a cloud platform that collects and process sensor readings from around the world; customers can visualize their data from mobile or web interfaces. Foobots have been used in homes, offices and research context.

As a sensor device manufacturer, our goal isn't only to provide a _thermometer_ for air quality, the end goal is to reduce exposure to air pollution for our customers. For that we have been involved in the engineering of several air purifier lines over the years. We also noticed it is key for users understand the actions that leaded to poorer air indoor. When one understands where pollution comes from, (s)he can perform remediation actions, or set an automated way to deal with it.

## Identify pollution sources

Air quality is complex and it is always challenging to identify a root cause when air pollution rises. It can be due to outdoor events, like wild fires or traffic pollution, but often, the pollution source is indoor. Within dwellings, there are a number of air quality events triggered by inhabitants. Our actions and behaviours indoor impact the air quality we live in; for the better, when we switch on the cooking hood, or for the worth when we are cleaning the bathroom with hash chemicals.

Foobot home sensors measures four environmental parameters: fine particulates (PM), volatile organic compounds (VOC), temperature and relative humidity. The challenge in creating a feature that can show user a root cause for a pollution event is to identify a unsique _signature_ over these 4 measured environmental parameters.

This article describes the process we followed to collect a dataset and to setup an automated air quality event classification process.

### Crowd collected dataset

We created a feature for our users to label their data. It allowed enrolled users to receive a push notification each time an air quality event was detected. The push notification opens Foobot mobile app where users were able to choose the action that was carried on, or to enter free text.

![sources]({{ site.baseurl }}/assets/viz_aqe/iphone_classify.png){: .center }
<center><i>Push notification for classification</i></center>
<br/>

The detection of an event, meaning the kind sensor deviation that constitue it, and the timing to trigger a push notification to a user is a topic on its own. It took us several iterations to maximize the number of responses and to avoid bothering users for non-event, or for events that happened too long ago and they weren't able to identify anymore.

Thanks this feature and to our user base we were able to collects tens of thousands of labeled events.

## Data analisys

Now lets introduce some basic techniques for data segmentation and classification, that we used to visualize labeled _air quality events_.

### Dataset description

The dataset that is used for the purpose of this article is a subset of Foobot full dataset. It contains _extracted features_ from sensor data time series and other continuous and categorical features, as well as associated tags given by users. These labels will be, used as classes to train a classifier. Selected samples are from devices and users located in the US and Europe.

For the sake of simplicity and readability we'll only detail 3 specific labels, though we have proven our method to work with quite a few others:
- _cooking_ consequence of a cooking action. This is one of the most common source of indoor air pollution, and it is quite challenging to detect accurately because of the variety of event types it amalgamates: frying, oven cooking, steaming, for example.
- _air renewal_ opening a door, a window, switching On some kind of ventilation. It appeared important to be able to detect remediation events. Either to reward users or to just to be able to prevent an automated system to ask user to perform an action already carried. We could also derive an airing score out of this metric.
- _presence_ people coming and going in the space. Wether someone is around or not is important for an automated system to decides what to communicate to the end user. It was also an interesting challenge since the Foobot home sensor doesn't feature a CO2 sensor.

### Tools

This short analysis uses [Spark](https://spark.apache.org) (mostly for data extraction) as well as [scikit-learn](http://scikit-learn.org/). Visualizations are performed with [matplotlib](https://matplotlib.org/).

### Overview

Intuitively, air quality events will have different "signatures", that is they will be expressed differently in data. For instance, cooking could be a source of particulate matters, while air renewal can translate into change in temperature, humidity and/or volatile organic compounds (VOC). That are just a few examples to build our intuition, there are many others.

To get a first feeling of how they show up in data, let's see how _difference between maximum and minimum value around event time_* is characterized

<div style='font-size:12px;margin-bottom:15px'>*feature extraction in time windows isn't discussed in this article</div>

![sources]({{ site.baseurl }}/assets/viz_aqe/pm_diff_max_min_box.png){: .center }
<center><i>Boxplot of difference between PM max and min values, by tag</i></center>
<br/>

A first interpretation of this [box plot](https://en.wikipedia.org/wiki/Box_plot) is that cooking is statistically the event that makes PM readings vary most: difference in 25%-75% range is wider than for other events, and maximum is much higher. Note also the shape of the plot for air renewal, indicating that variance is also important: could be due to PM being cleaned out when air is renewed.

Let's see some other plots for VOCs and humidity.

![sources]({{ site.baseurl }}/assets/viz_aqe/voc_diff_max_min_box.png){: .center }
<center><i>Boxplot of difference between VOC max and min values, by tag</i></center>
<br/>

![sources]({{ site.baseurl }}/assets/viz_aqe/hum_diff_max_min_box.png){: .center }
<center><i>Boxplot of difference between humidity max and min values, by tag</i></center>
<br/>

This time, we can deduce that VOC and humidity levels seem to be impacted most by air renewal (first) and presence.

We now have an intuition about how air quality changes reflects on some of Foobot's sensors. We can go a bit deeper to better exploit variance in data in order to separate event types.

### Exploiting variance in data with Principal Components Analysis

[Principal Components Analysis (PCA)](https://en.wikipedia.org/wiki/Principal_component_analysis) is a very popular technique that is mostly used for

- dimensionality reduction: if variance in data can be explained by few _principal components_ (where number of principal components &lt; number of features) then one can use these components for further analysis, mining or model training
- visualisation: 2 or 3 principal components are easy to plot, so PCA is convenient for visualizing how variance is explained in data, especially if you associate labels with data points.

PCA also has its limits: it really works well when problem is linear, that is when data you want to split is linearly separable. Let's not forget also that PCA really just provides a rotation (or projection onto a different space) that maximizes the variance (principle: diagonalization of the covariance/correlation matrix) for the given data, using fewer dimensions. It can't go beyond (it's not a classifier).

Using PySpark and scikit-learn, it's a straightforward task to setup a toy pipeline that pre-processes our dataset (like standardizing features), then applies PCA.
Following plots are demonstrating PCA with 2 and 3 components.

![sources]({{ site.baseurl }}/assets/viz_aqe/pca_k_2.png){: .center }
<center><i>PCA of air quality events (k=2)</i></center>
<br/>

![sources]({{ site.baseurl }}/assets/viz_aqe/pca_k_3.png){: .center }
<center><i>PCA of air quality events (k=3)</i></center>
<br/>

Here we achieve a visualization that projects all the features of our dataset onto the PCA features space (2 or 3 dimensions). We can see a trend in groups of points that seem to belong to each label. However, a simple interpretation can state that linear separability across tags isn't achieved: although data belonging to air renewal and cooking groups seem to distinguish from each other well, presence (_many people_) have data points mixed up with the 2 previous classes. A simple intuition could explain this: presence can reflect in many ways on sensor readings, and probably the same way as for other classes.

### Going non-linear with Polynomial Expansion

Since we weren't able to linearly separate data in a clear way, let's try a non-linear method to see if we can better manage this. A first option is [polynomial expansion](https://en.wikipedia.org/wiki/Polynomial_expansion) (a particular method of basis expansion) which consists in projecting data onto a different space, here formed by polynomial combinations of all existing features. For instance, if you have features `[a b]`, a 2nd degree expansion would result in `[ a b ab a² b²]`. We can then use the result as an input for a linear transformer or classifier, which is good also to preserve the low complexity of a linear algorithm to make projections or decisions in a non-linear space.

Though you can guess it: it will require huge computational power if polynomial degree is high and number of features is big. Also, the higher the degree, the more models using this new feature space will tend to overfit.

For the purpose of this article, we'll only try 2nd degree polynomial.

![sources]({{ site.baseurl }}/assets/viz_aqe/pca_pe_k_2.png){: .center }
<center><i>Polynomial expansion & PCA (k=2)</i></center>
<br/>

![sources]({{ site.baseurl }}/assets/viz_aqe/pca_pe_k_3.png){: .center }
<center><i>Polynomial expansion & PCA (k=3)</i></center>
<br/>

A clear change in projection, and a (visually) better separation of cooking and air renewal data. Presence (many people) is still not properly separated.

### Going non-linear with Kernel PCA

Another method uses the [kernel trick](https://en.wikipedia.org/wiki/Kernel_method) applied to PCA. It's basically also a change of space, as it projects data onto a higher dimensional space. A _kernel_ function is applied to each sample before PCA is applied.

Multiple "standard" kernel functions exist (like _gaussian radial basis function_ (RBF)). Choosing the right one is a difficult task when it's meant to be used in a model training pipeline. Here we're using the _cosine_ kernel function, only because it offers good visualization results.

Note: as of writing, kernels for feature engineering aren't available in Spark, but it's fairly simple to create your own.

![sources]({{ site.baseurl }}/assets/viz_aqe/kpca_k_2.png){: .center }
<center><i>Kernel PCA (k=2)</i></center>
<br/>

![sources]({{ site.baseurl }}/assets/viz_aqe/kpca_k_3.png){: .center }
<center><i>Kernel PCA (k=3)</i></center>
<br/>

Again, separation between cooking and air renewal data is good. This time 'presence' (many people) seems to overlap with cooking only, which makes sense: people are generating VOCs (breath, sweat, cosmetics, ...) and particles (like dust moved by human movements). Cooking can also have impact on the same sensors: frying or using the oven create particles, and VOCs can be emitted by gaz stove or other combustion/heating by-products.

### A supervised learning technique: Linear Discriminant Analysis

We didn't manage to separate data between classes well with PCA in the previous section. In order to go further, we're now going to use a supervised learning technique that takes into account classes (tags). Actually, it's a well known classifier that has several advantages: efficient computing, no hyper parameter tuning, and possibility to reduce dimensionality.

This is what [Linear Discriminant Analysis](https://en.wikipedia.org/wiki/Linear_discriminant_analysis) is capable of. Like PCA, it's a linear transformation, and the goal is to find a projection that maximizes variance, but this time between classes. It runs well under the assumptions that class densities are Gaussian and that each class covariance matrix is similar, assumptions that are not often met in real world problems.

We're going to use it here only for its dimensionality reduction capability, and we'll project our data onto a subspace made only of the number of dimensions we're interested in: LDA lets us chose number of components `k`, but it can only be such as `k < n-1` (`n` = number of classes).

LDA remains a linear method (linear decision boundaries) and tends to overfit the data as number of features (dimensions) grows.

![sources]({{ site.baseurl }}/assets/viz_aqe/lda_k_2.png){: .center }
<center><i>LDA (k=2)</i></center>
<br/>

![sources]({{ site.baseurl }}/assets/viz_aqe/lda_k_3.png){: .center }
<center><i>LDA (k=3)</i></center>
<br/>

This time, even if it's not a strict visual separation of data, classes seem to form distinct groups.

### Non-linear LDA

In order to obtain quadratic (no more linear) decision boundaries and improve our classification, we're now going to project our data onto a higher dimensional space formed by using polynomial expansion before fitting LDA.

It's also worth noting that _Quadratic Discriminant Analysis_ exists, and in practice gives very similar results.

![sources]({{ site.baseurl }}/assets/viz_aqe/qlda_k_2.png){: .center }
<center><i>Polynomial LDA (k=2)</i></center>
<br/>

![sources]({{ site.baseurl }}/assets/viz_aqe/qlda_k_3.png){: .center }
<center><i>Polynomial LDA (k=3)</i></center>
<br/>

This time, a clear separation is obtained.

## Accuracy verification

As we discovered, separating or classifying air quality events classes is a non-trivial task. Pure linear methods don't give satisfactory results - classes still overlap - and we had to use the technique of basis expansion (polynomials) along with LDA in order to get a proper visual separation (and classification). Although results are promising to build a classifier, there are some limitations to the algorithms and methods used in this article: PCA isn't a classification algorithm and can only be, when conditions are met, used to reduce dimensionality of a dataset. On the other hand, LDA is a classifier that is known to perform well, but it had to be combined with basis expansion. We also have to considered:

- generalization capability of our classifier, which should be verified with (cross-)validation and testing.
- users live in different indoor and outdoor conditions: from one region of the world to another, habits are different, outdoor air quality is different, temperature and humidity vary greatly.
- air quality events are difficult to track by nature: there can be several happening at the same time, there can be a significant delay between start of event and impact on sensor readings, same type of event will have a different expression from one place to another (e.g. air renewal in a polluted city vs countryside).

### Crowd sourced classification check

In the same way that we uses a subset of our users to help build a labeled dataset we proposed them to verify the accuracy of our classifier. We introduce a new feature that is this time not asking the user to label an event, but rather to check if the label to assigned to an event is correct:

![sources]({{ site.baseurl }}/assets/viz_aqe/iphone_valid.png){: .center }
<center><i>Event validation push notification</i></center>
<br/>

### Results

With user being able to enter sub-categories or reclassify wrongly labeled events, we were able to iterate and improve. Today our accuracy in detecting the different events are the following for the main ones:

- _cooking_ 88%
- _air renewal_ 98%
- _presence_ 96%
- _cleaning_ 100%

Worth to be noticed is that we are able to label these events in a streaming fashion, meaning that data points are classified live while they are being collected thanks to Spark streaming.
