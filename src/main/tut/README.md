##Dependent Types vs Exhaustive Match

As it turns out, some special care is required to maintain
enforcement of exhaustive pattern match when attempting to link dependent types in
each branch of the match.

While the solution is stupidly simple, it was surprising enough to me that it took some
flailing to arrive at, and if it saves one other person the trouble, this will have been worth
publishing.

Here's a contrived example to illustrate the problem.

Let's say we have some different hats of different sizes. We want to process orders for hats,
but we want the compiler to enforce that a given hat order is placed deliberately for a
specified size.

For our purposes, we model this as follows:

```tut:silent
sealed trait Hat[A]
case class BaseballCap[A]() extends Hat[A]
case class Fedora[A]() extends Hat[A]

sealed trait Size
case object M extends Size
case object L extends Size
case object XL extends Size

case class Order[S <: Size, H <: Hat[S]](hat: H, size: S)
```

Now the reason for this machinery, let's say, is that we have different paths to check
inventory for each style and size of hat, so we want to exhaustively handle each
style, but lock the order returned to the input received.

Naively:

```tut
object Order {
  def place[S <: Size, H <: Hat[S]](hat: H, size: S): Order[S, H] = hat match {
    case _: BaseballCap[_] => ??? // style specific inventory lookup
    case _: Fedora[_] => ??? // style specific inventory lookup
  }
}
```

This does what we want, insofar as we can handle each style differently, and the compiler will
enforce that the size and style we attach to the resulting order match the size and style that
were requested on input.

Of course we'd also like to make sure that the compiler forces us to address new styles as they
are added to inventory, so we do the right thing and add the following to build.sbt.

```scala
scalacOptions ++= Seq(
	"-Xfatal-warnings",
	"-Xlint")
```

And then we test it out.

```tut
object Order {
  def place[S <: Size, H <: Hat[S]](hat: H, size: S): Order[S, H] = hat match {
    case _: BaseballCap[_] => ??? // style specific inventory lookup
    //case _: Fedora[_] => ??? Are we blowing up on inexhaustive match?
  }
}
```

But it's still compiling! What gives?

It turns out the issue is fairly simple, but it was not so obvious to me.

In the code above, we are matching on a type of H, not Hat. Interestingly, scalac
cannot fail with a fruitless type test or scrutinee incompatible error because neither are true,
but it cannot enforce exhaustiveness because H is a subtype of our sealed trait Hat.

After some silly tug of war between taking an H vs taking a Hat as a parameter (in keeping with the
example) it occurred to me there is a stupid simple way to have both, cast up.

```tut:fail
object Order {
  def place[S <: Size, H <: Hat[S]](hat: H, size: S): Order[S, H] = (hat: Hat[_]) match {
    case _: BaseballCap[_] => ??? // style specific inventory lookup
    //case _: Fedora[_] => ??? Are we blowing up on inexhaustive match?
  }
}
```

And voila, we have our exhaustive enforcement, and our guarantee of the correct style and size
making it through to our eventual Order.

*There are surely much simpler ways to solve this particular problem, it was hastily manufactured to
illustrate this issue. There are certainly many legitimate use cases where you need an exhaustive
match paired with type dependence from input to output.
