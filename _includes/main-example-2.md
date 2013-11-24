
    def triangles(n: Int) = for {
      x <- (0 until n).toPar
      y <- 0 until x
    } yield { y }: @unchecked