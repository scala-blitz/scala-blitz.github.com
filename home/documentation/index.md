---
layout: docsdefault
title: Documentation
permalink: /index.html

partof: documentation
num: 1
outof: 9
---



Welcome to the Documentation section!
Here you can find examples of how to use parallel collections,
information about how this framework works, as well as larger
application examples.
If you would like to start from the beginning, go to the 
Getting Started section, otherwise, pick one of the sections below.


### [Getting started]({{ homedir }}/home/documentation/gettingstarted.html)

The definite place to get started with using parallel collections.
Includes a basic introduction, some simple examples and tips on what to watch out for when using parallel collections.


### [Design overview]({{ homedir }}/home/documentation/design.html)

This section describes the overall design of parallel collections.
It introduces zippable and reducables, stealers and mergers, and kernels and schedulers.


### [Parallel collections classes]({{ homedir }}/home/documentation/classes.html)

The choice of a particular parallel collection can have a noticeable impact on performance.
This section contains a comprehensive overview of different parallel collections classes describing their performance characteristics and internals.


### [Scheduler configuration]({{ homedir }}/home/documentation/scheduler.html)

While the parallel collection schedulers have good performance irregardless of the workload, they are highly configurable.
This section describes how you can set parallelism levels, granularities, scheduling policies, parallel execution thresholds
and much more to tune the scheduler to the specifics of your application, if necessary.


### [Extending the framework]({{ homedir }}/home/documentation/extending.html)

This section contains detailed information on extending parallel collections with custom collections,
adding custom parallel operations
and using the ScalaBlitz framework as a backend for your own data-parallel API or framework.


### [Example applications]({{ homedir }}/home/documentation/example-apps.html)

This section contains several larger benchmark examples, a detailed walkthrough on how to parallelize
these applications using parallel collections, and a performance evaluation for each example.


### [API reference](http://lampwww.epfl.ch/~prokopec{{ homedir }}/index.html#package)

The API reference is a definitive source of information on the specific data-parallel operations
and collections, as well as the internals of the framework.


### [Performance evaluation]({{ homedir }}/home/documentation/performance.html)

ScalaBlitz are being evaluated on a per-commit basis on a wide of different microbenchmarks
and several larger benchmark applications.
The evaluation is performed on several platforms with different performance characteristics.
This section contains quantitative insight on when parallelizing operations is beneficial and to what extent.




















