---
layout: docsdefault
title: Scheduler configuration
permalink: /scheduler.html

partof: documentation
num: 6
outof: 9
---



This section shows how to use schedulers -- import existing ones,
configure and instantiate an existing one,
even extend an existing one or implement your own scheduler.

2 most commonly used schedulers are already provided:
## Default parallel scheduler

This scheduler uses per-application threadpool to execute computations.
The simplest way to use this scheduler is to import a global default.
This is done as follows:

    import scala.collection.par.Scheduler.Implicits.global

The above is the simplest way to allow calling parallel operations on `Par[R]` collections
or reducables and zippables.

## Default sequential scheduler

We also have a scheduler that uses only invoker thread to perform computation.
The simplest way to use this scheduler is to add an import:

    import scala.collection.par.Scheduler.Implicits.sequential

Due to low-overheads in our operations implementations it can be up to 50 times faster to use our operation implementations instead of scala standard library operations. This can be automatized by using [optimize block]({{ homedir }}/home/documentation/optimize.html).

## Instantiating an existing scheduler

You can instantiate one of the existing scheduler templates if the default one
does not suit your needs.
Examples of template schedulers include `Scheduler.ExecutionContext` and `Scheduler.ForkJoin`:

    implicit val forkJoinScheduler =
      new Scheduler.ForkJoin(new Scheduler.Config.Default(4))

The above instantiates a work-stealing tree scheduler that uses fork/join pools.
The `Scheduler.Config` argument can be used to set the parallelism level and tune other details.

The execution context scheduler reuses some `scala.concurrent.ExecutionContext`
to do work-stealing tree scheduling:

    implicit val executionContextScheduler = {
      val ctx = scala.concurrent.ExecutionContext.Implicits.global
      val config = new Scheduler.Config.FromExecutionContext(4, ctx)
      new Scheduler.ExecutionContext(config)
    }

Always make the scheduler `implicit` in the scope in which you will be calling the parallel operations.


## Scheduler configurations

A scheduler configuration is a trait that looks as follows:

    trait Config {
      def parallelismLevel: Int
      def incrementStepFrequency: Int
      def incrementStepFactor: Int
      def maximumStep: Int
      def stealingStrategy: Strategy
    }

The `parallelismLevel` is a hint on how many CPUs your scheduler will attempt to use
to parallelize operations.
The other values listed here tie in deeply into how work-stealing tree scheduling works.
If your work units are very coarse, you might look into decreasing the `maximumStep`.
If, on the other hand, your workload is uniform and work units very fine-grained,
you should increase `maximumStep` as much as possible.

The `Config` that you will be usually interested in is `Config.Default`,
that takes a single parameter denoting the parallelism level.


## Extending existing schedulers

This and the following sections are meant for power users!
To extend an existing scheduler, simply subclass it -- see the source code
to learn more about the existing hierarchy.

For example, to enable work-stealing tree scheduling on your custom thread pool,
you can subclass the `Scheduler.WorkstealingTree`:

    class MyScheduler(val config: Scheduler.Config)
    extends Scheduler.WorkstealingTree {

      class MyTask[T, R](
        val root: Ref[T, R],
        val kernel: Kernel[T, R],
        val index: Int,
        val total: Int
      ) extends Scheduler.WorkstealingTreeTask[T, R] {
        def myTaskStartMethod() = workstealingTreeScheduling()
      }

      def dispatchWork[T, R](root: Ref[T, R], kernel: Kernel[T, R]) {
        for (idx <- 0 until config.parallelismLevel) {
          myDispatchMethod(new MyTask(root, kernel, idx, parallelismLevel))
        }
      }
    }


## Implementing your own scheduler

In most cases, however, the default scheduler should be sufficient,
but if you really feel that is necessary to implement a custom scheduler,
go ahead -- we challenge you to beat the performance of the default one!

In this case all you need to do is implement a single method that,
given a stealer and an operation kernel, executes a data-parallel operation:

    def invokeParallelOperation[T, R](s: Stealer[T], k: Kernel[T, R]): R

This is the most general way to obtain a scheduler and you should
only do this if you really know what you are doing or are embarking
on a research endeavour of finding a more efficient scheduler!

