---
layout: default
title: Rendering the Mandelbrot Set in Parallel
permalink: /mandelbrot.html

num: 10
outof: 50
partof: examples
description: This example shows how to render a Mandelbrot set in parallel and how to use the schedulers to achieve proper load-balancing.
image: {{ homedir }}/resources/images/mandel-icon.png
---



Mandelbrot set is a set of complex numbers that do not diverge when a certain mathematical operation is applied to them.
A complex number `zc = xc + i * yc` is a part of a Mandelbrot set iff the sequence:

    z(0) = 0
    z(n + 1) = z(n) * z(n) + zc

is bounded, i.e. does not grow beyond a certain absolute value.
Mandelbrot sets can be visualized by mapping each pixel `(x, y)` of an image to a complex number `xc + i * yc`.
Since in practice we cannot exactly determine whether the sequence `z(n)` is bounded,
we compute its elements until either the absolute value of `z(n)` becomes larger than `sqrt(2)`
(meaning that the complex number will certainly diverge)
or we compute `z(threshold)`, where `threshold` is the maximum number of iterations we allow
(meaning that the complex number will probably not diverge).

If we color pixels that diverged this way white and those that did not black, we 
obtain the following image:


<div class="imageframe-deep">
  <img src="{{ homedir }}/resources/images/mandel-bw.jpg"/>
</div>

This, however, is not very interesting -- Mandelbrot set visualizations typically
color pixels depending on their convergence rate, i.e. how many iterations were
necessary to decide if the corresponding complex number converges or not.

<div class="imageframe-deep">
<img src="{{ homedir }}/resources/images/mandel-color-2.png"/>
<img src="{{ homedir }}/resources/images/mandel-color-1.png"/>
<img src="{{ homedir }}/resources/images/mandel-color-3.jpg"/>
</div>

Notice that rendering a Mandelbrot set image is trivially data-parallel -- we can compute each
pixel independently of the other pixels.
Here's how we can compute the number of iterations before deciding on convergence of a complex
number `(xc, yc)`:

    def compute(xc: Double, yc: Double, threshold: Int): Int = {
      var i = 0
      var x = 0.0
      var y = 0.0
      while (x * x + y * y < 2 && i < threshold) {
        val xt = x * x - y * y + xc
        val yt = 2 * x * y + yc
      
        x = xt
        y = yt
      
        i += 1
      }
      i
    }

The image is a discretized representation of the continuous Mandelbrot set, so we have to map
each pixel to a certain complex number.
Assume that the image represents the part of the real plane bounded with `(xlo, xhi)` and
`(ylo, yhi)` on x- and y-axes, respectively,
and that image width and height are `wdt` and `hgt`, respectively.
We transform each pixel `(x, y)` to `(xc, yc)` as follows:

      val xc = xlo + (xhi - xlo) * x / wdt
      val yc = ylo + (yhi - ylo) * y / hgt

If our image is represented in memory as an array of 32-bit values called `pixels` (of size `wdt * hgt`),
we can assign each pixel `(x, y)` to array entry `y * wdt + x`
and compute all the pixel values as follows:

    for (idx <- 0 until (wdt * hgt)) {
      val x = idx % wdt
      val y = idx / wdt
      val xc = xlo + (xhi - xlo) * x / wdt
      val yc = ylo + (yhi - ylo) * y / hgt

      val iters = compute(xc, yc, threshold)
      val color = if (iters < threshold) 0xff000000 else 0xffffffff
      pixels(idx) = color
    }

Above we iterate through the range of all the pixel indices `idx` and convert
them to `(x, y)` coordinates.
This will render the pixels in the set black, and others white.
We can tweak the colors to reflect how many iterations were needed
to determine divergence, for example like this:

    val a = 0xff << 24
    val r = math.min(255, 1.0 * iters / threshold * 255).toInt << 16
    val g = math.min(255, 2.0 * iters / threshold * 255).toInt << 8
    val b = math.min(255, 3.0 * iters / threshold * 255).toInt << 0
    val color = a | r | g | b

The Mandelbrot image is so far computed sequentially.
What if we now want to speed this up using `p` processors?
First, we need to instantiate a `Scheduler` that will execute
our operation in parallel.
This is done by first creating a (default) scheduler configuration
and then instantiating the scheduler itself.
We will be using the workstealing tree scheduler for this example:

    import scala.collection.par._

    val config = new workstealing.Scheduler.Config.Default(p)
    implicit val s = new workstealing.Scheduler.ForkJoin(config)

Notice that the `Scheduler` must be `implicit`ly available.
Second, we need to convert our range into its parallel correspondant,
by calling `toPar` on it.
The entire example now becomes (TODO - after renaming, change `Scheduler` to `WorkstealingTreeScheduler`):

    val config = new workstealing.Scheduler.Config.Default(p)
    implicit val s = new workstealing.Scheduler.ForkJoin(config)

    for (idx <- (0 until (wdt * hgt)).toPar) {
      val x = idx % wdt
      val y = idx / wdt
      val xc = xlo + (xhi - xlo) * x / wdt
      val yc = ylo + (yhi - ylo) * y / hgt

      val iters = compute(xc, yc, threshold)
      val a = 0xff << 24
      val r = math.min(255, 1.0 * iters / threshold * 255).toInt << 16
      val g = math.min(255, 2.0 * iters / threshold * 255).toInt << 8
      val b = math.min(255, 3.0 * iters / threshold * 255).toInt << 0
      val color = a | r | g | b
      pixels(idx) = color
    }

You can see the complete code of [this example here](TODO).

Rendering the Mandelbrot set is a particularly good example of how much
better the workstealing tree scheduler is at irregular workloads.
In particular, compared to existing Parallel Collections in Scala 2.9 and 2.10,
the new scheduler can be up to 50% more efficient on an 8-core machine.
If you would like to see the Mandelbrot computation in action,
just play with the applet below, which allows switching between the
classic parallel collections and the new scheduler.

TODO add applet

