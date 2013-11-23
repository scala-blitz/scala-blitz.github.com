---
layout: docsdefault
title: Performance evaluation
permalink: /performance.html

partof: documentation
num: 8
outof: 8
---



A considerable effort was invested into making parallel collections efficient.
To validate this, proper benchmarks are required that show the performance across
different collections, operations and workloads,
comparing them to other data-parallel frameworks,
standard Scala collections
and hand-written, optimal sequential programs.

An extensive suite of benchmarks and microbenchmarks is included in
We have used the [ScalaMeter](http://axel22.github.com/scalameter/) performance measurement framework to implement it.
These benchmarking suite is run regularly during development.
You can see and compare the results for a range of different architectures:

- [Intel 2.66 GHz Quad Core Xeon X5355 with hyperthreading](http://chara.epfl.ch/~prokopec/npc-chara/report/)
- [Intel 3.40 GHz Quad Core i7-2600 with hyperthreading](http://chara.epfl.ch/~prokopec/npc-i7/report/)

