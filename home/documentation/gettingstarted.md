---
layout: docsdefault
title: Getting started
permalink: /gettingstarted.html

partof: documentation
num: 2
outof: 8
---



Parallel collections strive to facilitate parallel programming by hiding low-level parallelization details
from the user, while in the same time giving them a familiar high-level collection operations.
As long as the programmer can identify sufficient data-parallelism in his program and expose
it through a data collection, he is able to benefit from increased performance.

Parallel Collections were originally introduced into Scala in release 2.9.
Why another data-parallel collections framework?
While they provided programmers with seamless data-parallelism and an easy way
to parallelize their computations, they had several downsides.
First, the generic library-based approach in Scala Parallel Collections
had some abstraction overheads that made them unsuitable for certain types
of computations involving number crunching or linear algebra.
To make efficient use of parallelism, overheads like boxing or use of iterators
have to be eliminated.
Second, pure task-based preemptive scheduling used in Scala Parallel Collections
does not handle certain kinds of irregular data-parallel operations well.
The data-parallel operations in this framework are provided for a wide range of
collections, and they greatly reduce both of these overheads.


## Invoking parallel operations

Lets try out a simple, minimal example of a data-parallel operation.
Assume you are given a large array of real numbers and would like to compute their mean value.
The first thing to do is to make a scheduler available.
You may either instantiate your own or use the default one, available through an import:

    import scala.collection.par._
    import scala.collection.par.Scheduler.Implicits.global

This will make sure that the scheduler is using the same thread pool like the
rest of the concurrent flock (futures and actors) -- the default `ExecutionContext`
in the `scala.concurrent` package.

To invoke data-parallel operations on the array `a` you just need to call `toPar`
on the collection that you want to parallelize.
If that particular collection supports data-parallel operations, you will be able to call them.
Arrays are easy to parallelize, so we will use them to call the `reduce` operation
and sum all the numbers of together:

    def mean(a: Array[Float]): Float = {
      val sum = a.toPar.reduce(_ + _)
      sum / a.length
    }

The `reduce` operation above takes consecutive numbers and adds them together in parallel.
Its signature for arrays of `Float`s is as follows:

    def reduce(op: (Float, Float) => Float): Float

This operation is much like the `reduceLeft` operation that you know from standard Scala collections.
However, it can group consecutive integers in an arbitrary way when summing them,
although it will never invert their relative order.
For example, given an array of integers `1, 2, 3`, it may reduce them as `(1 + 2) + 3` or `1 + (2 + 3)`,
but not as `(2 + 1) + 3`.
This means that applying the reduction operator `op` to any grouping of consecutive elements must give the same final result.
Formally, we say that the `reduce` needs `op` to be associative, but not commutative.

Which kind of data structures or collections are viable for parallelization?
Generally speaking, these are data-structures any part of which can be accessed with equal amount of work.
This allows different processors to access different parts of the data structure with equal effort and process it in parallel.
For example, it takes an equal amount of computation to index an array at the first or the last index, ignoring any memory effects related to caching or paging.
Hash tables are typically implemented with arrays, so it should be no problem to parallelize them too.
Any kind of collection for which elements are not actually stored in memory, but computed procedurally, like `Range`s can also be efficiently parallelized.
For trees this is not generally the case -- a linked list is a special kind of a tree that is very imbalanced.
To process a linked list in parallel would require one processor to start at its head, but another to traverse potentially
thousands of elements before starting to do any useful work.
For this reason, the trees that can be reasonably parallelized have to be balanced.
Many collections are balanced trees -- balanced binary search trees, hash tries, various heaps.

In summary, the collections from the Scala standard library that this framework allows you to call parallel operations on are:

- `Array`s -- any type of an array can be efficiently parallelized with negligible overhead
- `Range`s -- ranges are collections used as a replacement for `for` loops
- `mutable.HashMap` and `mutable.HashSet` -- mutable hash tables containing elements or key-value pairs
- `immutable.HashMap` and `immutable.HashSet` -- immutable hash tries are tree-like structures that act as persistent hash tables
- `immutable.TreeSet` -- immutable tree sets are balanced binary search trees storing an ordered set of elements
- `immutable.Conc` -- conc trees are a parallel counterpart to functional cons `List`s you know from Scala -- `Conc`s are sequences with a efficient concatenation operation


## Other operations

Operations on collections can be divided into two major groups.
First, there are operations that return single values as results.
In Scala collections terminology, we call these the *accessor* methods.
One example of an accessor method is the `reduce` we've seen above.
But, there are others -- to start with, a more general version of the `reduce` is the `aggregate` method.
Lets assume we have an array of words (`Array[String]`) representing a text, and we want to count the total length of that text.
With aggregate, we need to specify a zero length (of type `Int`) and a way to aggregate words (of type `String`) into lengths (of type `Int`), just like with `foldLeft` from the standard Scala collections.
Additionally, we need to specify how two lengths (type `Int`) are combined together:

    def totalLength(text: Array[String]): Int = text.toPar.aggregate[Int](0) {
      (len1: Int, len2: Int) => len1 + len2
    } {
      (len: Int, word: String) => len + word.length
    }

Or shorter:

    def totalLength(text: Array[String]): Int = text.toPar.aggregate(0)(_ + _) {
      _ + _.length
    }

With this example in mind, lets describe the `aggregate` method a bit more generally.
To use the `aggregate` the client must choose the type of the values to aggregate,
like `Int` above.
Then, an associative operation with a neutral element must be defined.
Above, that's `+` with the integer zero.
Then, the user must specify how to combine elements of the collection, in this case `String`s,
with the aggregation elements, in this case `Int`s.

There are other variants of this method such as `mapReduce`, `fold`, `foreach`, `min`, `max`, `sum`, `forall`, `find`, etc.

Second, there are operations that return other collections as results.
We call these the *transformer* methods.
Transformer methods construct collections in parallel by using an abstraction called a `Merger`.
`Merger`s are just like `Builder`s in sequential collections, but they can also be efficiently combined.
This allows multiple processors to add elements to independent parts of the collection,
and then merge those parts together.

Here is an example of a simple `filter` operation on an array that returns only even integers:

    def even(array: Array[Int]): Array[Int] = array.toPar.filter(_ % 2 == 0).seq

Same as with accessors, we must first call `toPar` to obtain an instance of the array on which
we can call parallel methods.
After calling `filter` we again obtain a parallel array, so we must call `seq` to turn it back into a
normal array of integers.


## Important considerations

There are several important caveats to consider when using parallel collections.
First of all, you should be very careful about performing **side-effects** while invoking parallel operations.
The following example of calculating the sum of the elements in the array will not produce the perhaps-desired results:

    def sum(a: Array[Int]): Int = {
      var sum = 0
      for (x <- a.toPar) sum += x
      sum
    }

This example attempts to update the value `sum` in parallel while traversing the elements of the array.
This will almost certainly produce an invalid sum.
One reason is that the operation `+=` is not atomic -- it consists of reading the `sum` value from memory
and writing a new value back to it.
Another reason is that reads and writes to `sum` are not properly synchronized -- `sum` should at least
be annotated as `@volatile`.

In the above example, it's a much better idea to use `reduce` as shown earlier.
Often, you will need to step away from the sequential mindset of using a `for`-loop and update
a value, and try to express your computation as a data-parallel operation.

You might be tempted to instantiate a `java.util.concurrent.atomic.AtomicInteger` above,
and use atomic add operations to update it.
Good luck!
Your implementation will not only be 20-30 times slower compared to optimal sequential version,
it will also not scale beyond a 2x speedup on most modern desktop CPUs.
The solution using `reduce`, in this framework, is not only far more efficient, it is also optimal.

Next, most operations will require providing custom binary operators, and in almost all cases these operators have to be **associative**.
We've already seen this with `reduce` and `aggregate`.
An example of an operator that is not associative is `-`.
Trying to reduce a list of numbers in parallel like this:

    def sum(a: Array[Int]) = a.toPar.reduce(_ - _)

has absolutely no meaning, as the result depends how integers are associated to processors -- it will produce
bogus and non-deterministic results.
For example, given an array `1, 2, 3` it may be computed as `1 - (2 - 3) = 2` or `(1 - 2) - 3 = -4`.
Luckily, most meaningful operators that you will use are associative.

A more subtle property that some operators have to have is that of **idempotence**.
When you define an associative operator with the neutral element for `aggregate`,
the operator might be called more than once by the scheduler to reduce the values.
For example, you might want to use `aggregate` create a frequency map of character occurences
in a text:

    def frequency(text: Array[Char]) = {
      text.toPar.aggregate(mutable.Map[Char, Int]() withDefaultValue 0)(_ ++= _) {
        (fmap, c) => fmap(c) += 1
      }
    }

This is a bad idea -- the associative operator `++=` defined on mutable maps here
copies all the elements of the right hand frequency map to the left hand
frequency map by mutating it and returning it.
Calling this operator more than once on the same pair of maps is not the same
as calling it exactly once -- the operator is thus not idempotent.

There are other considerations related to performance.
Your operators should not be inherently prone to **memory bottlenecks** or **false sharing**.
You should not expect a parallel loop that continually modifies the same location
in memory to achieve significant speedup.

    val cache = mutable.Set[String]()
    def add(a: Array[String]) {
      for (word <- a.toPar) cache.synchronized {
        cache += word
      }
    }

Remember -- on most modern hardware, reading concurrently from the same memory location is a ok,
but writing to the same memory location is not scalable even if you manage to do it semantically correct.
That is not to say that you should never write to the same memory location concurrently -- this is perfectly
fine as long as the bulk of your work amortizes the cost of writing to a shared location.
While the example with `cache` above is likely to lead to scalability problems,
the following code should be perfectly fine:

    val palindromes = mutable.Set[String]()
    def add(a: Array[String]) = for (word <- a.toPar) {
      if (isPalindrome(word)) cache.synchronized {
        palindromes += word
      }
    }

Above it is unlikely that most words in your text are palindromes, so all the checks that you do
amortize the cost of occasionally writing to `palindromes`.
In general, if in doubt -- measure!

Code that inherently creates a lot of garbage and triggers GC cycles is another recipe for killing scalability.
For example, you should avoid using operators that do **boxing**.
Although parallel collections are highly optimized and specialized for primitive types,
if you use existing non-specialized classes, there is nothing parallel collections can do about it.
Using the standard library collections in Scala 2.11 and doing operations on boxed primitives is not a good idea:

    def negate(ints: immutable.HashSet[Int]) = for (x <- ints.toPar) yield -x

Maintaining these integers in a primitive integer array is a much better idea:

    def negate(ints: Array[Int]) = for (x <- ints.toPar) yield -x

These are some of the things you should watch out for when using parallel collections to ensure correct semantics
and superior performance they can provide you with.

