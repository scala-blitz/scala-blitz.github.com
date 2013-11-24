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

An extensive suite of benchmarks and microbenchmarks is included in the <a href="https://github.com/scala-blitz/scala-blitz/tree/master/src/test/scala/org/scala/optimized/test/scalameter" target="_top">source code</a>.
The <a href="http://axel22.github.com/scalameter/" target="_top">ScalaMeter</a> performance measurement framework was used to implement it,
and this benchmarking suite is run regularly during development.
You can see and compare the results for a range of different architectures:


<ul>
{% for pg in site.pages %}
  {% if pg.partof == "benchmarks" %}
    <li><a href="{{homedir}}/home/documentation/benchmarks/{{ pg.permalink }}">{{ pg.title }}</a></li>
  {% endif %}
{% endfor %}
</ul>

