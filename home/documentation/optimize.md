---
layout: docsdefault
title: Optimize block
permalink: /optimize.html

partof: documentation
num: 4
outof: 9
---



This section shows optimize block,
that is supposed to be used to speedup data-parallel operations 
on normal scala collections.

Using optimize rewrites data-parallel operations from scala collections to operations from workstealing collections, providing you all advantages: eliminated boxing, better inlining, data-structure specialization. This commonly gives 2-3x speedup for operations having small amount of work per-element, but for some tests speedup can be as big as 50x.


## Quick example
Imagine you have some code, written previously that solves [Problem 1 from project Euler](https://projecteuler.net/problem=1).
    
	def ProjectEuler1(x: Range) = {
      x.filter(x => (x % 3 == 0)|| (x % 5 == 0)).reduce(_ + _)
    }
	
And you want it to run faster :-). 
You can potentially rewrite this code using our high-performance operations, but it'll take time.
Thus we provide you an 'optimize' block that does this automaticaly during compilation.
All you need to do is cover your code in optimize{}:


    import scala.collection.optimizer._
	def ProjectEuler1(x: Range) = optimize {
      x.filter(x => (x % 3 == 0)|| (x % 5 == 0)).reduce(_ + _)
    }

optimize will rewrite function body to something similar to this:

    def ProjectEuler1(x: Range) = {
        import scala.collection.par._;
		implicit val scheduler = Scheduler.Implicits.sequential;
		x.toPar.filter(x => (x % 3 == 0)|| (x % 5 == 0)).reduce(_ + _)
    }

And you can clearly see 3 times speedup [on benchmarks](TODO).
You don't get a better speedup becouse the bottleneck here is array allocation inside filter.
This allocation can be removed entirely by advanced analisys. We're currently working on optimization that will rewrite same code to single operation that applies filtering and reduction as one step.


