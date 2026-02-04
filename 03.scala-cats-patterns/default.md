# Scala cats patterns
#### date: 24.01.2021 16:25

Here I have a list of patterns that can be simplified with Scala's [Cats](https://github.com/typelevel/cats)
library.

```scala
import cats.syntax.all._
```

Examples that are given here are purely for demonstration.

## Use `.collectFold` instead of `.collect` + `.sum`

Example:

```scala
val list: List[Int] = List(1, 2, 3, 4, 5)

val evenSum = list.collect {
  case i if i % 2 == 0 => i
}.sum

val evenSum1 = list.collectFold {
  case i if i % 2 == 0 => i
}
```

## Use `.foldMap` instead of `.map` + `.sum`

Example:

```scala
val listOfStrings = List("In", "cats", "we", "trust")

listOfStrings.map(_.length).sum

listOfStrings.foldMap(_.length)
```

## Use `.combineAll` instead of `.foldLeft` + `.combine`

Example:

```scala
val listOfTuples = List((1, 0), (2, 2), (3, 4))

listOfTuples.foldLeft((0, 0))(_ |+| _)

listOfTuples.combineAll
```

## Use `.collectFoldSome` instead of `.flatMap` + `.sum`

Example:

```scala
val listInts = List(1, 2, 3, 5)

listInts.flatMap(x => if (x % 2 == 0) Some(x + 1) else None).sum

listInts.collectFoldSome(x => if (x % 2 == 0) Some(x + 1) else None)
```

## Use `Either.fromOption` instead of `pattern matching` on `Option`

Example:

```scala
Option(1) match {
  case Some(v) => Right(v)
  case None    => Left("error")
}

Either.fromOption(Option(1), "error")
```

## Use `Either.fromTry` instead of `pattern matching` on `Try`

```scala
Try("12".toInt) match {
  case Failure(exception) => Left(exception.getMessage)
  case Success(v)         => Right(v)
}

Either.fromTry(Try("12".toInt)).leftMap(_.getMessage)
```


