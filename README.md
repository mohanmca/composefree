#ComposeFree

ComposeFree is a small lib inspired by Runars
[Composable Application Architecture](http://functionaltalks.org/2014/11/23/runar-oli-bjarnason-free-monad/)
which minimizes the boilerplate required to build an application based on a Coproduct of
Free DSLs.

To use it, include in build.sbt

```scala
resolvers += Resolver.bintrayRepo("bondlink", "composefree")

libraryDependencies += "bondlink" %% "composefree" % "1.0.1"
```

Basic use is a pared down version of the manual process, with the following high level steps:

* Define DSLs as sealed families of case classes
* Define interpreters for DSLs as NaturalTransformations
* Define the application Coproduct type for your composed DSLs
* Create an instance of ComposeFree[YourApplicationType] and import it

For example, let's say we wanted to combine a simple Console DSL with the Pure DSL
provided in the ComposeFree lib.

First we define an ADT for our Console operations, and pattern match it
in a NaturalTransformation to an effectful monad.

```scala
import scalaz.Id.Id
import scalaz.~>

sealed trait ConsoleOps[A]
case class print(s: String) extends ConsoleOps[Unit]

object RunConsole extends (ConsoleOps ~> Id) {
  def apply[A](op: ConsoleOps[A]) = op match {
    case print(s) => println(s)
  }
}
```

Now we need a NaturalTransformation for the Pure dsl.

```scala
import composefree.puredsl._

object RunPure extends (PureOp ~> Id) {
  def apply[A](op: PureOp[A]) = op match {
    case pure(a) => a
  }
}
```

Then we can define the Coproduct type for our application, and obtain our ComposeFree
instance.

```scala
import composefree.ComposeFree
import scalaz.Coproduct

object Program {
  type Program[A] = Coproduct[ConsoleOps, PureOp, A]
}

object compose extends ComposeFree[Program.Program]
```

Last we will create an interpreter for our program type by combining our individual
interpreters.

```scala
import composefree.syntax._

val interp = RunConsole |: RunPure
```

And finally we can define a program and execute it.

```scala
val prog: compose.Composed[Unit] = {
  import compose._
  for {
    s <- pure("Hello world!").as[PureOp]
    // use of .as[T] helper to cast as super type is
    // required for operations with type parameters that cannot be
    // implicitly converted to the correct coproduct member type
    _ <- print(s)
  } yield ()
}
// prog: compose.Composed[Unit] = Gosub(Suspend(Coproduct(\/-(Coproduct(\/-(pure(Hello world!)))))),<function1>)

prog.runWith(interp)
// Hello world!
// res6: scalaz.Id.Id[Unit] = ()
```

Composite commands can be defined in individual DSLs and mixed into
larger programs as follows.

```scala
object PureComposite {
  import compose.lift._
  import composefree.syntax._
  import scalaz.Free

  def makeTuple(s1: String, s2: String): Free[PureOp, (String, String)] =
    for {
      a <- pure(s1).as[PureOp]
      b <- pure(s2).as[PureOp]
    } yield (a, b)
}
// defined object PureComposite

import compose._
// import compose._

import Program._
// import Program._

val prog = for {
  s <- PureComposite.makeTuple("Hello", "World!").as[Program].op
  _ <- print(s._1)
  _ <- print(s._2)
} yield ()
// prog: scalaz.Free[compose.RecNode,Unit] = Gosub(Gosub(Suspend(Coproduct(\/-(Coproduct(\/-(pure(Hello)))))),<function1>),<function1>)

prog.runWith(interp)
// Hello
// World!
// res7: scalaz.Id.Id[Unit] = ()
```
