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

The only implemented property is `neighbourOffsets`, which is
