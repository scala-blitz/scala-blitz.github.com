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

Using optimize rewrites data-parallel operations from scala collections to operations from workstealing collections, providing you all advantages: eliminated boxing, better inlining, data-structure specialization. This commonly gives 10-20x speedup for operations having small amount of work per-element.


## Quick example
Imagine you have some code, written previously that solver [Problem 1 from project Euler](https://projecteuler.net/problem=1).
    
	def problem1 = {
      (1 to 1000).filter(x => (x % 3 == 0)|| (x % 5 == 0)).reduce(_ + _)
    }
	
And you want it to run faster :-). 
You can potentially rewrite this code using our high-performance operations, but it'll take time.
Thus we provide you an optimize block that does this automaticaly during compilation.
All you need to do is cover your code in optimize{}:


    import scala.collection.optimizer._
	def problem1 = optimize {
      (1 to 1000).filter(x => (x % 3 == 0)|| (x % 5 == 0)).reduce(_ + _)
    }

optimize will rewrite function body to this:

    def setDisjoint(s1:HashSet[Int], s2:HashSet[Int]) = {
     import scala.collection.par._;
     import Scheduler.Implicits.sequential;
     !s1.toPar.exists(elem: Int => s2.contains(elem)) &&!s2.toPar.exists(elem: Int => s1.contains(elem))
    }

And you can clearly see speedup [on benchmarks](TODO).



