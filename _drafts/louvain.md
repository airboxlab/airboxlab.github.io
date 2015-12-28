---
layout: post
title:  "Detecting and visualizing pollution communities with Louvain method, Spark GraphX and D3.js"
categories: spark machine-learning D3 louvain
author: Antoine Galataud
comments: true
---

## Context

Here at Airboxlab, we collect and analyze indoor pollution to make people life safer and easier.
 
We're analyzing Foobot sensors data individually so we can inform and alert our customers when pollution levels are higher than recommended, but we're also trying to analyze global trends with various technics.

Lately, we've been interested in using community detection algorithms. Our hopes are to analyze communities size and evolution over time, find trends, and correlate their appearance with external factors.

These algorithms are known to be used by major social networks companies to detect communities in their networks. There are also lot of academic researches in this field; analyzing various types of data sets with community detection algorithms can help find how people are connected. For instance, analyzing phone call logs can highlight where money can be invested in new infrastructure, what are the main communities in a region or country and how they are composed (among criterion like language, age, ...) 

There are different methods for community detection, like *Minimum-cut*, *Hierarchical clustering*, *Girvan-Newman algorithm* or *Modularity maximization*.
Introductions for these methods can be found on [Wikipedia](https://en.wikipedia.org/wiki/Community_structure)

Discovering communities can also be seen as a clustering problem within graph structures.

## Louvain algorithm

Louvain method was originally described[^1] by Vincent D. Blondel, Jean-Loup Guillaume, Renaud Lambiotte and Etienne Lefebvre. The algorithm took the name of the *Université catholique de Louvain* in Belgium, where the 4 authors are researchers.

It aims at maximizing modularity using heuristic. 

#### Definition 

Modularity value to be optimized is defined as:
![mod](https://upload.wikimedia.org/math/c/f/6/cf68353cacdbbaf0c5d53c251083af4b.png "Modularity formula")

where 

- ![Aij](https://upload.wikimedia.org/math/f/8/9/f896be3d8636bddc74beebe184293aff.png) is the weight of the edge between *i* and *j*
- ![ki](https://upload.wikimedia.org/math/2/6/e/26e634477c7a1285bb21c5df84371894.png) is the sum of the weights of the edges attached to *i*
- ![m](https://upload.wikimedia.org/math/6/f/8/6f8f57715090da2632453988d9a1501b.png) is half the sum of all edge weights in the graph. 
- ![ci](https://upload.wikimedia.org/math/d/9/8/d9899588b2b28a768a63ade0f3523596.png) is the community to which *i* is attached
- ![delta](https://upload.wikimedia.org/math/f/1/0/f10f03c9836c36537d2539196058bfa2.png "delta") is 1 if ![ci](https://upload.wikimedia.org/math/d/9/8/d9899588b2b28a768a63ade0f3523596.png) = ![cj](https://upload.wikimedia.org/math/e/b/e/ebeeb713d38e86da62ba72e61376a622.png) (same community), 0 otherwise.

The algorithm has a 2 steps approach that are repeated iteratively until modularity value is optimized. 

First, the method looks for small communities by optimizing modularity locally. Second, it aggregates nodes belonging to the same community and builds a new network. Nodes of this network are the aggregated ones built during 2nd phase. It iterates until modularity value is optimized.

A graphical representation of the iterations can be found in original paper[^1] as below

![iterations]({{ site.url }}/assets/pol.jpg) 

#### Advantages

- scalability (performs faster on huge graphs than other methods)
- simple to code

#### Pitfalls

- Iterative process can hide small communities found during intermediate phases. The result may be a coarse-grained high level representation of communities, which may not have the granularity needed for analysis. Hopefully, the nature of the algorithm makes it simple to save intermediate phases' results so we can analyze different communities structures at different levels
- Heuristic used to  initialize phases and find local maximums can lead to not reproducible and not always optimized results. But this is the same with all data algorithms relying on heuristic (K-Means for instance)   
   
#### Applications

The original study was based on phone calls logs originated from Belgian telecommunication operators.

Some public examples of applications can be found on [Vincent Blondel's personal blog](https://perso.uclouvain.be/vincent.blondel/research/louvain.html).

Also, here is a detailed study made on [mobile phone data sets](http://arxiv.org/pdf/1502.03406.pdf).  

An implementation for Apache Spark is available on [github](https://github.com/Sotera/spark-distributed-louvain-modularity).

## Finding communities in Foobots

First of all, you may be asking: but how are Foobots connected to each other? Simple answer: they aren't. There is no direct relation or link between 2 devices, except when they both have the same owner.

As a consequence, we will have to draw artificial links (or edges) between them, based on following criteria:

- geographical distance[^2] between devices
- euclidean distance[^3] between average pollutant values of devices

Same thing for edges weights: as stated in modularity definition, weight between edges is taken into account and is playing a role in nodes (or vertices) grouping. 
Setting a meaningful weight on each edge is a tricky problem, and there is probably no single solution for it; it will highly depend on what we analyze, what we want to find or what we want to correlate to. 
For the purpose of this article, we choose to define weight by using the distance between pollutant values of devices: the closer values are, the bigger weight will be.

 

## Plotting results with D3


[^1]: [Louvain method original paper](http://arxiv.org/abs/0803.0476)
[^2]: [Haversine formula for geographical distance calculation](https://en.wikipedia.org/wiki/Haversine_formula)
[^3]: [Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance)
[4]: http://arxiv.org/pdf/1502.03406.pdf