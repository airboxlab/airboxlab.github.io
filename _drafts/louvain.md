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
 
## Louvain algorithm

Some well known examples are [Twitter and LinkedIn][1].     

### Principles
### Implementation in Spark

## Finding communities in Foobots

## Plotting results with D3

[1]: https://perso.uclouvain.be/vincent.blondel/research/louvain.html