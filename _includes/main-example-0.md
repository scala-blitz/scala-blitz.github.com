{% highlight scala %}
import scala.collection.optimizer._

def average(x: List[Double]) = optimize {
  x.sum / x.size
}
{% endhighlight %}
