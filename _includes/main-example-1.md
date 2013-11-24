
    import scala.collection.par._
    import Scheduler.Implicits.global

    (0 until 15000000).toPar.reduce(_ + _)

