# Scala 2.13 new features

To play with a Scala 2.13 REPL:

    $ sbt
    > set resolvers += "pr" at "https://scala-ci.typesafe.com/artifactory/scala-pr-validation-snapshots/"
    > set scalaVersion := "2.13.0-pre-8f3a05a-SNAPSHOT"
    > console

---

```scala
// Getting field names from a case class:
Person(name = "hello", age = 29).productElementNames.toList // List("name", "age")
// Coupled with `productIterator` this provides a case class to Map converter:
(person.productElementNames zip person.productIterator).toMap
// Map[String, Any] = Map("name" -> "hello", "age" -> 29)

// A shorter version of the classic groupBy/mapValues:
List("a" -> 1, "c" -> 2, "a" -> 3).groupMap(_._1)(_._2)          // Map("a" -> List(1, 3), "c" -> List(2))
Map(1 -> "a", 2 -> "b", 4 -> "b").groupMap(_._2)(_._1)           // Map("b" -> List(2, 4), "a" -> List(1))
Array("a", "c", "c", "z", "c").zipWithIndex.groupMap(_._1)(_._2) // Map("a" -> Array(0), "c" -> Array(1, 2, 4), "z" -> Array(3))

// A shorter version of the classic groupBy/mapValues/reduce:
// val map1 = Map(1 -> 9, 2 -> 20)
// val map2 = Map(1 -> 100, 3 -> 300)
(map1.toSeq ++ map2).groupMapReduce(_._1)(_._2)(_ + _)   // Map[Int, Int] = Map(2 -> 20, 1 -> 109, 3 -> 300)
(map1.toSeq ++ map2).groupMapReduce(_._1)(_._2)(_ max _) // Map(1 -> 100, 2 -> 20, 3 -> 300)
"hello".groupMapReduce(identity)(_ => 1)(_ + _)          // Map('e' -> 1, 'h' -> 1, 'l' -> 2, 'o' -> 1)
// Counting words in a file:
Source.fromFile("file.txt")
  .getLines.to(LazyList)
  .flatMap(_.split("\\W+"))
  .groupMapReduce(identity)(_ => 1)(_ + _)

// val x = 45
Option.when(x > 0)(100) // Some(100)
// val x = -5
Option.when(x > 0)(100) // None
// def when[A](cond: Boolean)(a: => A): Option[A]
Option.unless(false)(3) // Some(3)
Option.unless(true)(3)  // None
Option.unless(list contains None)(list.flatten)
// val list = List(Some(1), Some(2))          =>    Some(List(1, 2))
// val list = List(Some(1), None, Some(2))    =>    None

// Where we used to do `Try("abc".toInt).toOption`:
"5".toIntOption                 // Option[Int] = Some(5)
"abc".toIntOption               // Option[Int] = None
"abc".toIntOption.getOrElse(-1) // Int = -1
// And its siblings for other types:
"5.7".toDoubleOption                    // Option[Double] = Some(5.7)
"true".toBooleanOption.getOrElse(false) // true
"oups".toBooleanOption.getOrElse(false) // false

// Where `List().min` or `List().max` throws an exception (by lack of elements),
// `List().minOption` returns None:
List(3, 2, 5).minOption // Some(2)
List().minOption        // None
// Same thing goes for minBy which has a minByOption sibling: 
List((3, 'a'), (1, 'b'), (5, 'c')).minByOption(_._1) // Some((1, 'b'))
List().minByOption(_._1)                             // None

List(("a", 2), ("b", 2), ("a", 5)).distinctBy(_._1) // List(("a", 2), ("b", 2))

// Where we used to do `array(array.lastIndexWhere(_._1 == 20))` or `list.reverse.find(_._1 == 20)`:
Array((10, 'a'), (20, 'b'), (30, 'c'), (20, 'd')).findLast(_._1 == 20)
// Option[(Int, Char)] = Some(20, 'd')

// We can now partition elements based on a function which returns either Left or Right.
val (lefts, rights) = List(Right(2), Left("a"), Left("b")).partitionMap(identity)
// lefts: List[String] = List("a", "b")
// rights: List[Int] = List(2)
// It's particularly interesting as it can be used as a type-based partitioner:
val (strings, ints) =
  List("a", 1, 2, "b", 19).partitionMap {
    case s: String => Left(s)
    case x: Int    => Right(x)
  }
// strings: List[String] = List("a", "b")
// ints: List[Int] = List(1, 2, 19)

// `def tap[U](f: (A) => U): A` applies a side effect, while returning the value it's applied on:
// Very handy for debugging or logging, this replaces things such
// as `val b = { val a = getA(); log(a); a }` by `val b = getA().tap(log)`
import util.chaining._
val x = 42.tap(println)
// 42
// x: Int = 42
val ls = List(1, 2, 3).map(_ * 2).tap(println).map(_ * 2).tap(println)
// List(2, 4, 6)
// List(4, 8, 12)
// ls: List[Int] = List(4, 8, 12)

// Applying a function on whatever object:
import scala.util.chaining._
42.pipe(_ * 2) // 84
def isPalindrome(i: Int) = i.toString.pipe(s => s == s.reverse)
x pipe h pipe g pipe f // f(g(h(x)))

// Where we used to do `20 + Random.nextInt(11)`:
Random.between(20, 30) // in [20, 30[

// `scala.collection.JavaConverters` becomes `scala.jdk.CollectionConverters`:
import scala.jdk.CollectionConverters.Ops._
java.util.Arrays.asList(1, 2, 3).asScala.toList // List[Int] = List(1, 2, 3)
List(1, 2, 3).asJava                            // java.util.List[Int] = [1, 2, 3]

// Future, Duration, Option, Stream Scala/Java converters:
import scala.jdk.FutureConverters.Ops._
Future.successful(42).toJava
import scala.jdk.DurationConverters.Ops._
java.time.Duration.ofNanos(123456).toScala // 123456 nanoseconds
import scala.jdk.OptionConverters.Ops._
Some(42).toJava // java.util.Optional[Int] = Optional[42]

// Dedicated resource management utility (loan pattern)
// Similar to Java's try with resource statement (and it works with Java objects implementing `Closable`)
import scala.util.Using
// val lines = List("hello", "world")
Using(new PrintWriter(new File("file.txt"))) {
  writer => lines.foreach(writer.println)
}
// scala.util.Try[Unit] = Success(())
// And we can create our own resources with `Releasable`:
import scala.util.Using
import scala.util.Using.Releasable
case class Resource(field: String)
implicit val releasable: Releasable[Resource] = resource => println(s"closing $resource")
Using(Resource("hello world")) { resource => resource.field.toInt }
// closing Resource(hello world)
// res0: scala.util.Try[Int] = Failure(java.lang.NumberFormatException: For input string: "hello world")

Right(1) orElse Left(2)                // Right(1)
Left(1) orElse Left(2)                 // Left(2)
Left(1) orElse Left(2) orElse Right(3) // Right(3)

// Underscore is accepted as a numeric literal separator:
val x = 1_000_000  // x: Int = 1000000
val pi = 3_14e-0_2 // pi: Double = 3.14

Some(1, "hello").unzip // (Some(1), Some("hello"))
None.unzip             // (None, None)

Some(2) zip Some('b') // Some((2, 'b'))
Some(2) zip None      // None
None zip Some('b')    // None
None zip None         // None

List.unfold("aaaabbaaacdeeffg") {
  case ""   => None
  case rest => Some(rest.span(_ == rest.head))
}
// List[String] = List("aaaa", "bb", "aaa", "c", "d", "ee", "ff", "g")

// Unicode arrows (→, ⇒, ←) are deprecated
```
