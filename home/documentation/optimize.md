---
layout: docsdefault
title: Sequential optimizations
permalink: /optimize.html

partof: documentation
num: 4
outof: 9
---



This section describes the `optimize` block,
used to speed up bulk operations on sequential Scala collections.

The `optimize` rewrites Scala collection operations to operations from workstealing collections,
providing the following advantages:
- it eliminates boxing
- it performs better inlining
- it specializes the operation to a particular data-structure

These transformations commonly give a 2-3x speedup for operations having a small amount of work per-element,
but in some cases involving boxing, the speedup can be as big as 50x.


## Quick example

Assume we have the following code that solves the [Problem 1 from Project Euler](https://projecteuler.net/problem=1).
    
    def ProjectEuler1(x: Range) = {
      x.filter(x => (x % 3 == 0) || (x % 5 == 0)).reduce(_ + _)
    }
	
And you want it to run faster :-).

You could potentially rewrite this code manually to `while` loops and low-level constructs to speed it up,
but this takes time and results in hard-to-read code.
For this reason, ScalaBlitz provides an `optimize` block that does this automaticaly during compilation.
All you need to do is enclose your code in an `optimize { }` statement:

    import scala.collection.optimizer._
    def ProjectEuler1(x: Range) = optimize {
      x.filter(x => (x % 3 == 0) || (x % 5 == 0)).reduce(_ + _)
    }

and the function body will be rewritten to something similar to this:

    def ProjectEuler1(x: Range) = {
      import scala.collection.par._
      implicit val scheduler = Scheduler.Implicits.sequential
      x.toPar.filter(x => (x % 3 == 0)|| (x % 5 == 0)).reduce(_ + _)
    }

If you measure the running time you will see a 3x speedup [on benchmarks](TODO).

Currently, the speedup is limited to only 3x due to array allocation inside the `filter`.
This allocation can be removed entirely by advanced analysis.
We are currently working on optimization that will rewrite same code to a single operation
that fuses filtering and reduction into a single step.


