---
layout: post
category : Coding
title: "Test driven game of life - Part 1"
tagline: "How to build the Conway's game of life functionally with TDD"
tags : [Scala, Akka, Distributed systems, Hackatons]
---
{% include JB/setup %}

In this series of articles I want to talk about how to build the popular [Conway's game of life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life) in Scala with a *functional* and *test-driven* approach. The game of life is an example of cellular automata in which the universe is a board divided in cells that can be alive or dead; at each generation of the simulation the board evolves to a new status, some cells dye, other cells live on and other get born following four simple rules:

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

### First iteration of tests

