{% highlight scala %}
import scala.collection.optimizer

def average(x: List[Double]) = optimize {
  x.sum / x.size
}
{% endhighlight %}
