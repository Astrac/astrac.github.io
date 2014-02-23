---
layout: post
category : Coding
title: "Test driven game of life - Part 1"
tagline: "Building the Conway's game of life functionally and TDD-style"
tags : [Scala, TDD]
---
{% include JB/setup %}

In this series of articles I want to talk about how to build the popular [Conway's game of life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life) in Scala with a *functional* and *test-driven* approach. We did this with the London Scala User Group during an Hack The Tower event and it was a really fun way to get people involved in Scala and to see some TDD in action.

The game of life is an example of cellular automata in which the universe is a board divided in cells that can be alive or dead; at each generation of the simulation the board evolves to a new status, some cells dye, other cells live on and other get born following four simple rules:

1. Any live cell with fewer than two live neighbours dies, as if caused by under-population.
2. Any live cell with two or three live neighbours lives on to the next generation.
3. Any live cell with more than three live neighbours dies, as if by overcrowding.
4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

It's amazing how, given an initial state with some living cells, these four simple rules can give birth to a variety of more or less chaotic behaviours. To start with this we are going to imagine that we have an infinite board and on this board we keep track only of living cells. We will avoid using classes and other OO-like constructs and we'll focus on functions and data types.

[//]: <> (@@@@@@@@@@@)

### Data types

All the simulation functions will be contained in a Scala `object` named `LifeSym`; this is the definition of our basic data types:

{% highlight scala %}
object LifeSym {
  type Cell = (Int, Int)
  type Board = Set[Cell]
  // ...
}
{% endhighlight %}

What this means is that `Cell` will be a type alias for a pair of `Int` and `Board` will be a `Set` of `Cell`. We also need some convenience functions to create and access our data structures as follows:

{% highlight scala %}
object LifeSym {
  // ...
  def cell(x: Int, y: Int): Cell = (x, y)
  def board(cs: Cell*): Board = cs.toSet
  def x(c: Cell) = c._1
  def y(c: Cell) = c._2
  // ...
}
{% endhighlight %}

The `board` function accepts a variable number of `Cell` parameters, which makes the creation of a board nicely resemble a DSL:

{% highlight scala %}
val b = board(cell(1, 2), cell(1, 3), cell(1, 4))
{% endhighlight %}

### Basic board/cell functions

The rules of the game depend very much on the neighbourhood of a cell, so there should be a function that tells us given a cell what are its living neighbours and if two cells are neighbours; to determine these two things is needed in turn a function to get the neighbouring coordinates of a cell, i.e. the points that have 1 of distance between any x and/or y. The signatures of this functions could be as follows:

{% highlight scala %}
object LifeSym {
  // ...
  val neighbourOffsets: Set[(Int, Int)] = (for {
    x <- -1 to 1
    y <- -1 to 1
  } yield (x, y)).toSet - ((0, 0))

  def neighbourCoordinates(c: Cell): Set[Cell] = ???
  def areNeighbours(c1: Cell, c2: Cell): Boolean = ???
  def livingNeighbours(b: Board, c: Cell): Set[Cell] = ???
  // ...
}
{% endhighlight %}

#### Missing implementations

The `???` is a special value defined in the standard Scala library; it is defined as follows:

{% highlight scala %}
def ???: Nothing = throw new NotImplementedError
{% endhighlight %}

This is a place-holder commonly used when stubbing out ideas. Its type is `Nothing` that in Scala is the type that is descendant of any other type (as `Any` is the ancestor of any other type) so it will type-check in almost any situation, it doesn't have any instance and is the type returned by throwing an exception. Hitting a `???` at runtime will result in an error saying that an implementation is missing.

#### About the neighbour offsets generation

The only implemented property in the above functions is `neighbourOffsets`, which is a `Set` containing all the pairs having x and y within the [[-1, 1]] range except when they are both equal to 0; it is constructed using for comprehension and removing the (0, 0) pair from the generated set. The need for double parenthesis may not be clear for people new to Scala so it probably deserves a bit of explanation: what happens is that in Scala all the operators are actually methods defined as a normal method would be defined; syntactic sugar makes it possible to use them (as well as normal methods) without the `.` and *without parenthesis*.

This means is that when you write somewhere:

{% highlight scala %}
val x = 2 + 2
{% endhighlight %}

Scala actually translates it to:

{% highlight scala %}
val x = 2.+(2)
{% endhighlight %}

If we used only one set of parenthesis as in:

{% highlight scala %}
val mySet: Set[(Int, Int)] = ...
mySet - (0, 0)
{% endhighlight %}

This would be translated to:

{% highlight scala %}
val mySet: Set[(Int, Int)] = ...
mySet.-(0, 0)
{% endhighlight %}

The parenthesis would be assumed to belong to the method invocation and not to a tuple constructor and the compiler would complain that it doesn't know how to apply the `-` function of `Set[(Int, Int)]` to two integer parameters as it expects a single pair of integers instead. With two set of parenthesis instead the compiler can correctly translate the invocation as follows:

{% highlight scala %}
val mySet: Set[(Int, Int)] = ...
mySet.-((0, 0))
{% endhighlight %}

Which type checks fine and is what we actually meant.

#### First iteration of tests

We drafted our functions, it's time to write some tests for how we expect them to work. Doing this we will be able to quickly implement and validate them as well as be sure that the functionality remain the same if later we (or someone else) will decide to change something and the tests will also work as a basic runnable documentation for the API. We are going to use [ScalaTest](http://www.scalatest.org/) version 2.0, to get started we'll have to define the test class as follows:

{% highlight scala %}
import org.scalatest._

class LifeSymTest extends FunSpec with Matchers {

  import LifeSym._

  // ...
}
{% endhighlight %}

ScalaTest is a test framework that [supports many styles](http://www.scalatest.org/user_guide/selecting_a_style), from unit testing to more BDD-like approach. In this case I'm going to use an approach that is in between Unit testing and BDD, so I extend the `FunSpec` class and mix in the `Matchers` trait to use the nice DSL that it provides for defining expectations. Let's start writing our unit tests into the test suite defined above:

{% highlight scala %}
describe("LifeSym") {
  it("should correctly identify neighbouring coordinates") {
    neighbourCoordinates(cell(1, 1)) should equal(
      Set(cell(0, 0), cell(0, 1), cell(0, 2), cell(1, 0),
        cell(1, 2), cell(2, 0), cell(2, 1), cell(2, 2)))
  }

  it("should correctly tell if two cells are neighbours or not") {
    areNeighbours(cell(2, 3), cell(4, 5)) should be (false)
    neighbourCoordinates(cell(4, 5)) foreach {
      c => areNeighbours(c, cell(4, 5)) should be (true)
    }
  }

  it("should produce a list of living neighbours of a cell") {
    val simpleBoard = board(cell(2, 3), cell(4, 5), cell(5, 6))
    livingNeighbours(simpleBoard, cell(2, 3)) should be (empty)
    livingNeighbours(simpleBoard, cell(4, 5)) should equal(Set(cell(5, 6)))
    livingNeighbours(simpleBoard, cell(5, 6)) should equal(Set(cell(4, 5)))
  }
  // ...
}
{% endhighlight %}

Looking at this code it is possible to appreciate how the Scala language puts some emphasis on being able to write nice domain specific languages within itself by using the richness of its syntax and abstractions. The `describe` and `it` nesting is provided by the `FunSpec` class while `should`, `be`, `equal` and `empty` are provided by the powerful `Matchers` trait.

In a real-world scenario you may probably want to write more test cases for each functions, but what we have here cover the basics of what these functions should provide us. If we run the tests now this is what we will see:

```
[info] LifeSymTest:
[info] LifeSym
[info] - should correctly identify neighbouring coordinates *** FAILED ***
[info]   scala.NotImplementedError: an implementation is missing
[info] - should correctly tell if two cells are neighbours or not *** FAILED ***
[info]   scala.NotImplementedError: an implementation is missing
[info] - should produce a list of living neighbours of a cell *** FAILED ***
[info]   scala.NotImplementedError: an implementation is missing
[info] ScalaTest
[info] Run completed in 1 second, 585 milliseconds.
[info] Total number of tests run: 3
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 0, failed 3, canceled 0, ignored 0, pending 0
[info] *** 3 TESTS FAILED ***
[error] Failed: Total 3, Failed 3, Errors 0, Passed 0
[error] Failed tests:
[error]   org.lsug.restlife.test.LifeSymTest
[error] (test:test) sbt.TestsFailedException: Tests unsuccessful
[error] Total time: 2 s, completed 23-Feb-2014 18:05:13
```

The actual output will contain also stack traces, I removed them as they are not really important at the moment. What the output tells us is which tests failed and a bit of useful information. We will now implement the functions and try to get our tests passing. It is a fun and useful exercise to *get the bar green* when playing with a pet project as we are doing now; in the real world, in a big software that needs to be maintained for a long time by many people, this approach has also many other advantages:

* It is way quicker to run a suite of tests rather than checking things manually
* The tests will be run in CI, ideally before merging topic branches, thus catching breaking changes before they make it into a release.
* Tests will help refactoring parts of the system when they become old as even not knowing all the details you can validate your work quickly by running them.
* If written in a good way tests will work as a *living documentation* of the code base.

#### Implementing the functions

The `neighbourCoordinates` function should get the neighbouring offsets that we defined above and apply each of them to the given cell to produce a new set of coordinates. In Scala, when we want to produce a new collection starting from one we have and applying some transformation we are most likely looking at a use case for the `map` function:

{% highlight scala %}
def neighbourCoordinates(c: Cell): Set[Cell] = neighbourOffsets.map { case (ofX, ofY) => cell(x(c) + ofX, y(c) + ofY) }
{% endhighlight %}

The `map` function is defined for many *container types* (e.g. collections, Option, Future) and for a fictional `Container` type is defined as follows:

{% highlight scala %}
trait Container[T] {
  def map[U](f: T => U): Container[U]
}
{% endhighlight %}

The Scala collection `map` actually has a slightly more complex signature, but for most practical purposes this is the one you should think about. The function applies a provided function from `T` (the type inside the container) to a type `U` and returns a new container of `U`. In our case both `T` and `U` are equal to `Cell` and the container type is `Set`. The `case` statement used as the parameters part of a closure is a way to de-construct the pair of offset coordinates into its two components. The function body creates a new cell whose coordinates are taken from the given cell plus the given offsets. In Java or any imperative language this could've been written as:

{% highlight java %}
public Set<Cell> neighbourCoordinates(Cell cell) {
  Set<Cell> newSet = new Set<Cell>();

  for (Cell c : neighbourOffsets) {
    newSet.add(new Cell(cell.getX() + c.getX(), cell.getY() + c.getY()));
  }

  return newSet;
}
{% endhighlight %}

Lots of ceremonies for a very simple task. Higher order functions as `map` are one of the most powerful tools in the toolbox of a functional developer. Let's give a run to tests to see if we did everything right:

```
[info] LifeSymTest:
[info] LifeSym
[info] - should correctly identify neighbouring coordinates
[info] - should correctly tell if two cells are neighbours or not *** FAILED ***
[info]   scala.NotImplementedError: an implementation is missing
[info]   at scala.Predef$.$qmark$qmark$qmark(Predef.scala:252)
[info] - should produce a list of living neighbours of a cell *** FAILED ***
[info]   scala.NotImplementedError: an implementation is missing
[info] ScalaTest
[info] Run completed in 1 second, 462 milliseconds.
[info] Total number of tests run: 3
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 1, failed 2, canceled 0, ignored 0, pending 0
[info] *** 2 TESTS FAILED ***
[error] Failed: Total 3, Failed 2, Errors 0, Passed 1
[error] Failed tests:
[error]   org.lsug.restlife.test.LifeSymTest
[error] (test:test) sbt.TestsFailedException: Tests unsuccessful
[error] Total time: 2 s, completed 23-Feb-2014 18:37:03
```

The test we designed for the first function is passing! We can then implement the other functions as follows:

{% highlight scala %}
def areNeighbours(c1: Cell, c2: Cell): Boolean = neighbourCoordinates(c1).contains(c2)

def livingNeighbours(b: Board, c: Cell): Set[Cell] = b.filter(other => areNeighbours(c, other))
{% endhighlight %}

So `areNeighbours` is using the neighbouring coordinates set of the first given cell to check if it contains the second cell; `livingNeighbours` is instead using the `areNeighbours` function to filter the board and retain only the cells that are neighbours of the provided one. The `filter` function is as `map` a powerful tool that allow to express in a very concise way things that in an imperative language would take explicitly iterating over a loop in a rather verbose way.

Running our tests now this is the output:

```
[info] LifeSymTest:
[info] LifeSym
[info] - should correctly identify neighbouring coordinates
[info] - should correctly tell if two cells are neighbours or not
[info] - should produce a list of living neighbours of a cell
[info] ScalaTest
[info] Run completed in 1 second, 276 milliseconds.
[info] Total number of tests run: 3
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 3, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[info] Passed: Total 3, Failed 0, Errors 0, Passed 3
[success] Total time: 1 s, completed 23-Feb-2014 18:42:24
```

Everything works as expected, our tests are passing, we can now re-iterate to build the functions that will actually do the simulation on top of our results.
