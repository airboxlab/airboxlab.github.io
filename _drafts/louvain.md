---
layout: post
title:  "Detecting and visualizing pollution communities with Louvain algorithm, Spark and D3"
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

Discovering communities can also be seen as a clustering problem within graph structures.

## Louvain algorithm

Louvain method was originally described by Vincent D. Blondel, Jean-Loup Guillaume, Renaud Lambiotte and Etienne Lefebvre. The algorithm took the name of the *Université catholique de Louvain* in Belgium, where the 4 authors are researchers.

It aims at maximizing modularity using heuristic. 

### Definition 

Modularity value to be optimized is defined as:
![mod](https://upload.wikimedia.org/math/c/f/6/cf68353cacdbbaf0c5d53c251083af4b.png "Modularity formula")

The algorithm has a 2 steps approach that are repeated iteratively until modularity value is optimized. 

First, the method looks for small communities by optimizing modularity locally. Second, it aggregates nodes belonging to the same community and builds a new network. Nodes of this network are the aggregated ones built during 2nd phase. It iterates until modularity value is optimized.  

### Advantages

- scalability (performs faster on huge graphs than other methods)
- simple to code

### Pitfalls

- Iterative process can hide small communities found during intermediate phases. The result may be a coarse-grained high level representation of communities, which may not have the granularity needed for analysis. Hopefully, the nature of the algorithm makes it simple to save intermediate phases' results so we can analyze different communities structures at different levels
- Heuristic used to  initialize phases and find local maximums can lead to not reproductible and not always optimized results. But this is the same with all data algorithms based on heuristic (see K-Means for instance)   
   
Some well known usages in social networks are [Twitter and LinkedIn][2]. The original study was based on phone calls logs originated from Belgian telecommunication operations

### Implementation in Spark

## Finding communities in Foobots

## Plotting results with D3

## References

[Community structure](https://en.wikipedia.org/wiki/Community_structure)
[Louvain method original paper](http://arxiv.org/abs/0803.0476)

[2]: https://perso.uclouvain.be/vincent.blondel/research/louvain.html
[4]: http://arxiv.org/pdf/1502.03406.pdf