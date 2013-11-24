---
layout: docsdefault
title: Design overview
permalink: /design.html

partof: documentation
num: 3
outof: 8
---



This section investigates the design of the new parallel collections in more detail.

As already mentioned, programmers need to simply call the `toPar` method to obtain
a parallel version of the collection.
This call returns a thin parallel wrapper of type `Par[C]` around the collection of type `C` -- in fact,
the `toPar` method can be called on any kind of object whatsoever.
This is an important difference with respect to existing Scala Parallel Collections,
where only specific collections had their parallel counterparts and other collections
had to be converted to the most similar parallel collection,
the computation cost of which was not always apparent in the code.

We said that `toPar` returns a thin parallel wrapper `Par[C]` around the collection of type `C`.
The `Par[C]` object has almost no methods -- the only interesting one is `seq` that converts
it back to the normal collection.
Question is -- how can we then call parallel operations on that object?
Parallel operations are added to `Par[C]` through an implicit conversion.
This design offers several advantages compared to hardcoding the operations into a strict hierarchy
of classes:

- collection operations become available through an `import` statement --
the clients can use the `import` to choose between different implementations based on
either Scala Macros, specialization or something completely different
- the signature no longer has to exactly correspond to the sequential counterparts
of the parallel methods -- in particular, most parallel operations now take an
additional implicit `Scheduler` parameter
- additional operations that are not part of this collections framework can
easily be added by clients to different collections

Specific collection (or object) types like `Array[T]`, `Range` or `mutable.HashSet[T]`
have their own implicit conversions that add methods to, say, `Par[Array[T]]`:

    implicit class Ops[T](a: Par[Array[T]]) extends AnyVal {
      def aggregate[S](z: S)(combop: (S, S) => S)(seqop: (S, T) => S): S
      def reduce[U >: T](op: (U, U) => U): U
    }

Such a parallel collections hierarchy is much more flexible.


## Reducables and zippables

There are some important disadvantages too -- the main problem with this design
is that clients can no longer use parallel collections generically.
With sequential collections you could write a function that takes
a `Seq[Float]` and invokes operations on it:

    def mean(a: Seq[Float]): Float = {
      val sum = a.reduce(_ + _)
      sum / a.length
    }

    mean(Vector(0.0f, 1.0f, 2.0f))
    mean(List(1.0f, 2.0f, 4.0f))

With `Par[Seq[Float]]` this is no longer possible:

    def mean(a: Par[Seq[Float]]): Float = {
      val sum = a.reduce(_ + _)
      sum / a.length
    }

The reason is that there are no extension methods defined for
instances of `Par[Seq[Float]]` -- these exist only for concrete parallel collection classes
such as `Par[Array[Float]]` or `Par[Range]`.
Why is this so?
Well, there is no efficient way to parallelize the general sequence abstraction.
The sequence might actually be implemented as a list, which is not efficiently parallelizable.

To cope with the problem that method signatures cannot be generic like this,
parallel collections define two special traits called `Reducable[T]` and `Zippable[T]`.
These traits allow writing generic collection code, since they are equipped with extension methods
for standard collection operations.
The `Zippable[T]` trait is a subtype of `Reducable[T]` since it offers more operations and is only
applicable to specific kinds of collections.

Using `Reducable[T]`, our method `mean` becomes:

    def mean(a: Reducable[Float]): Float = {
      val sum = a.reduce(_ + _)
      sum / a.length
    }

We can call `mean` like this:

    mean(Array(1.0f, 4.0f, 9.0f).toPar)
    mean(mutable.HashSet(1.0f, 2.0f, 3.0f).toPar)

But aren't the arguments to `mean` now supposed to have type `Par[Array[Float]]` and `Par[mutable.HashSet[Float]]`?

The answer is that they are implicitly converted to instances of `Reducable[Float]`.
Every collection type `R` with elements `T` that has an `Ops[T]` implicit wrapper defined for `Par[R]`
may additionally have an instance of the type-class `IsReducable[R, T]`:

    trait IsReducable[R, T] {
      def apply(pr: Par[R]): Reducable[T]
    }

This typeclass allows parallel collections of type `Par[R]` to be converted to `Reducable[T]`.
A similar mechanism exists for `Zippable[T]`.
Parallel collections implemented in this framework can be easily converted to reducables or zippables.

The rest of this section goes deeper into the architectural details of the framework.
Unless you're seeking knowledge to extend the framework with new collections or operations,
you may skip it and go directly to the [Parallel Collections classes section]({{ homedir }}/home/documentation/classes).


## Stealers

To enable efficient splitting of work among processors, the parallel collections API
defines an abstraction called a `Stealer`.
These are essentially iterators on steroids.
They allow work to be split at any point, or even stolen from the iterator!
All parallel collections in this framework have their own stealer implementations.
But stealers are never visible to the programmer developing applications using parallel collections.
Rather, they exist only to power users to extend this framework with additional collections.

The `Stealer` trait looks roughly as follows:

    trait Stealer[T] extends Iterator[T] {
      def advance(step: Int): Int
      def markStolen(): Boolean
      def split: (Stealer[T], Stealer[T])
    }

At any point, a stealer is owned by a single processor.
Before calling `next` and `hasNext`, the processor calls the `advance` method,
specifying the approximate size of the batch of elements he would like to traverse using `next` and `hasNext`.
It can only then call `next` and `hasNext`.
Between processing these batches, the processor checks if there have been any attempts
to steal it using the `markStolen` method.
Processors looking for work will call the `markStolen` method to notify that this stealer
should no longer be advanced.
Both `advance` and `markStolen` should atomically update the state of the stealer.
After the stealer becomes stolen
its state becomes stale, and in can no longer be `advance`d,
but its `split` method can be called to divide the remaining elements
into two substealers.
This method is *idempotent* and may be called multiple times,
but only after the stealer was stolen.

For a more detailed description of the interaction between these methods,
see the [`Stealer` source code](TODO).
To see a simple example implementation, consider studying [indexed stealers](TODO)
for ranges and arrays.


## Mergers

While stealers were iterators on steroids, mergers are builders on steroids.
They define an additional method `merge` that can be used to fuse two mergers together:

    trait Merger[T, Repr] extends Builder[T, Repr] {
      def +=(elem: T): Merger[T, Repr]
      def result: Repr
      def merge(that: Merger[T, Repr]): Merger[T, Repr]
    }

After the `merge` method is called, method `+=` should no longer be called
on either `this` or `that` merger -- those become invalid.
This method returns a new merger that contains elements of both mergers.

Again, mergers are never visible to application programmers -- they exist
only to support *transformer* operations mentioned earlier.


## Kernels

For every parallel operation call a `Kernel` is generated.
Kernels are objects that abstract over how batches of elements are processed
and how the results are combined together.
Its interface closely resembles the following:

    trait Kernel[T, R] {
      def zero: R
      def combine(a: R, b: R): R
      def apply(s: Stealer[T], approximateChunkSize: Int): R
    }

In essence, kernels define how to use a stealer by calling its
`hasNext` and `next` methods between `advance` calls in order
to compute a partial result for that batch.
An kernel for:

    Array("word1", "word2").toPar.aggregate(0)(_ + _)(_ + _.length)

might look like this:

    class AggregateKernel extends Kernel[String, Int] {
      def zero = 0
      def combine(a: Int, b: Int) = a + b
      def apply(s: Stealer[String], approximateChunkSize: Int) = {
        var len = 0
        while (s.hasNext) len += s.next().length
        len
      }
    }

In this framework, ScalaMacros are used to generate efficient and optimized kernels for every
parallel operation call.


## Schedulers

Schedulers abstract over how the parallel computation executes.
Their primary goal is to answer how the result is computed in parallel
given an operation `Kernel` and a collection `Stealer`.

    trait Scheduler {
      def invokeParallelOperation[T, R](s: Stealer[T], k: Kernel[T, R]): R
    }

The default scheduler in this framework is based on the work-stealing tree scheduling.
This is a highly-efficient technique that handles uniform, low-cost workloads
as well as highly irregular ones.
There are several variants of this scheduler that allow using different types
of underlying thread pools.
