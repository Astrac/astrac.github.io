---
layout: post
category : Coding
title: "Retrying the ask pattern"
tagline: "A quick snippet to get actors retrying the ask pattern when it times out"
tags : [Scala, Akka, Distributed systems]
---
{% include JB/setup %}

In the last months I participated to the course about [Reactive Programming](https://www.coursera.org/course/reactive) on [Coursera](http://coursera.org). It was a great course, and quite hard too in several parts. I quite liked learning and playing with the *Akka Actor System* architecture: I appreciated the focus on resilience via supervision strategies, I liked the Actor abstraction and how strongly it resembles OOP in many ways (I would say that is more object oriented than classes in many object oriented languages) and I enjoyed the possibility to use routers to factor out big processes to many parallel (and possibly distributed) actors.

During one of the exercises, while trying to use the *ask pattern* in a distributed system with faulty communication I ended up writing a small reusable snippet of code that allows retrying the same communication a specified number of times before failing in case of timeout in receiving the response as follows:

{% highlight scala %}
val f: Future[Any] = target.askretry(message, nOfAttempts, attemptsRate)
{% endhighlight %}

But let's start with a brief introduction to the ask pattern...

[//]: <> (@@@@@@@@@@@)

### About actor communication

Actor *communication* is usually *asynchronous* and thus *unconfirmed*; a message is usually sent by an actor as follows:

{% highlight scala %}
// Using the operator !
target ! message
// Not using the ! operator
target.tell(message)
{% endhighlight %}

At this point the running actor doesn't have any clue on whether or not the message will be delivered or processed. The good thing is that it doesn't have to block either and it can proceed to doing other things (e.g. serving the next message). This is all fine until you consider some real-life use cases where you want an answer to be actually delivered to the running actor.

An example scenario could be one where you have a `CalculatorActor` which does some long computation and a `ClientActor` who is asking for the computation and interested in its result. It could be possible to model this scenario by storing in the client actor the identifiers of all messages that need an answer and have the calculator send results for the calculations along with the message identifiers. This is not too hard to implement but there is a lot of ceremony involved for a quite common scenario, so Akka designers provided an implementation of this pattern in their library.

## Enter the ask pattern

Let's imagine we have the scenario above and let's write an incredibly powerful calculator actor:

{% highlight scala %}
case class Calculate(id: Int, x: Int, y: Int)
case class Result(id: Int, res: Int)

class CalculatorActor(target: ActorRef) extends Actor {
  def receive: Actor.Receive = {
    case Calculate(id: Int, x: Int, y: Int) => sender ! Result(id, x + y)
  }
}
{% endhighlight %}

Surprisingly enough this will accept messages encapsulating two integers and return at a certain point in the future a result that contains their sum. A client of this will at some point ask for some data by doing something like:

{% highlight scala %}
// As an actor attribute
var pendingCalcs = Map.empty[Int, Calculate]

// In the receive
val id = ... // generate an unique id for the calculation
val x, y = ... // some values we need to sum

val msg = Calculate(id, x, y)
pendingCalcs = pendingCalcs + msg // Store the pending request somewhere
calculatorRef ! msg
{% endhighlight %}

Since there is no guarantee of when the response is going to come from the `CalculatorActor` we are forced here in storing the message in a structure (e.g. a map) and update it when receiving the result:

{% highlight scala %}
// ...
case Result(id, res) =>
  pendingCalcs.get(id) match {
    case Some(_, x, y) => println(s"$x + $y = $res")
    case None => ... // Some error reporting to notify the unexpected message
  }
{% endhighlight %}

This could work, but the machinery is tedious and we are forced into using mutable state; the solution provided by the creators of Akka is the *ask pattern*. It is enabled by importing `akka.pattern.ask` into scope and it works in the client code as follows:

{% highlight scala %}
// We need an implicit timeout for the response to come back
implcit val to = Timeout(1.seconds)
// We also need an execution context, we can use the actor system dispatcher
implicit val ec = context.system.dispatcher

val id = ... // generate an unique id for the calculation
val x, y = ... // some values we need to sum

val f = (calculatorRef ? Calculate(id, x, y)).mapTo[Result]
// same as val f = calculatorRef.ask(Calculate(id, x, y)).mapTo[Result]

// f is a Future[Result] so we can just handle it as usual
f.onComplete {
  case Success(Result(_, res)) = > println(s"$x + $y = $res")
  case Failure(ex) => ... // Report the exception
}
{% endhighlight %}

The response is received in the same part of the actor code that sends the message and it makes everything more clear about what is going on. There is a very important catch in dealing with futures within an actor: *never mutate any internal state of the actor from within a future*. This deserves the first prize as "the most common source of headaches when debugging actor software". The reason for this is that when you are in the body of a future you are in a different thread than the one that is running the actor itself and access to actor state has all the usual problems of concurrent access has. Accessing local variables is fine instead, and the future's content is the response message coming from the ask request. It is also worth noting that the `CalculatorActor` is not aware of this and we do not need to make any change on it.

One common thing that actors using the ask pattern want to do is sending the received response to another actor. For this Akka provides another pattern that is *pipe*. It is not really relevant to what I've done but I think it is quite interesting to know it and it is definitely useful. To send the result of a future to an actor the proper way of doing it is:

{% highlight scala %}
    import akka.pattern.pipe
    // ...
    // f is a Future of some value that needs to be sent to targetActor
    f.pipeTo(targetActor)
{% endhighlight %}

Going back to where we started from, all of this is really nice but what can we do to use this pattern in a scenario where we have faulty communication between actors? In a distributed system it could be quite easy the case that actors may not be able to reach each other over the network and a resilient system should implement some kind of strategy to mitigate transient issues. What I found useful when solving such a problem during the Reactive Programming course on Coursera was to make an extension to `ActorRef` to provide one more method which I named `askretry`.

## AskRetry

The purpose of this method is to specify a number and a rate for attempts that have to be done to deliver a message and get a response. The method signature is as follows:

{% highlight scala %}
def askretry[T](
  msg: T, maxAttempts: Int, rate: FiniteDuration)(implicit context: ActorContext): Future[Any]
{% endhighlight %}

The usage is as follows:

{% highlight scala %}
    val f = calculatorRef.askretry(Calculate(id, x, y), 10, 200.millis).mapTo[Result]
{% endhighlight %}

The code for this snippet is available on [this repository](https://github.com/Astrac/akka-askretry). If after 10 tries performed every 200 milliseconds if no answer is received the future returned will fail with a `AskTimeoutException` as in the case of a normal `ask` call. This is achieved by spawning a very small actor that will perform the requests and forward the response to the asking actor. This actor reads as follows:

{% highlight scala %}
/**
 * An actor that retries sending the same message until it receive a response
 * or until it reaches a limit of attempts.
 */
private class RetryingActor extends Actor
{
  // Import message classes
  import AskRetry._
  // Make an execution context available as the scheduler needs it
  implicit val ec = context.system.dispatcher
  // Keep count of attempts
  var attempts = 0

  /**
   * Wait for an Ask message containing the instructions for the actor task
   * @return Unit
   */
  def receive: Receive = {
    // In case we received an Ask invocation
    case Ask(target, message, rate, maxAttempts) =>
      // Create a cancellable subscription by scheduling the dispatch of a Retry
      // message to this actor accordingly to the parameters of the Ask message
      val subscription = context.system.scheduler.schedule(0.second, rate, self, Retry)
      // Switch behaviour passing along useful information to complete the task
      context.become(retrying(subscription, sender, target, message, maxAttempts))
  }

  /**
   * Keep retrying until the response is received or the maximum number of
   * attempts has been performed
   *
   * @param subscription Cancellable Subscription that can cancel the scheduled message
   * @param requester ActorRef The actor that originated the request being served
   * @param target ActorRef The actor that should respond to the ask request
   * @param message T The message to be delivered in the ask request
   * @param maxAttempts Int The maximum number that the request should be tried
   * @return Unit
   */
  def retrying[T](subscription: Cancellable, requester: ActorRef, target: ActorRef, message: T, maxAttempts: Int): Receive = {
    // In case the actor receives a scheduled message...
    case Retry =>
      // and if there are still attempts left
      if (attempts < maxAttempts) {
        // increase the attempts counter
        attempts = attempts + 1
        // try sending a message to the target
        target ! message
      } else {
        // if we can't do any other attempt let the requester know of the failure
        requester ! Status.Failure(RetryException(attempts))
        // cancel the scheduled message
        subscription.cancel()
        // stop the actor
        context.stop(self)
      }
      // In case we receive any other message, since this actor is hidden, it
      // can only be the response
    case response =>
      // Send it back to the original requester
      requester ! response
      // cancel the scheduled message
      subscription.cancel()
      // stop the actor
      context.stop(self)
  }
}
{% endhighlight %}

This actor has two behaviours and switches between them to perform the task. At the beginning messages will be handled in the standard `receive` function; the actor will wait for an `Ask` message and when received it will use the actor context's `scheduler` to deliver the `Retry` message the specified number of times at the specified rate; the scheduler has several methods to interact with actor at specified times or periods and it is quite an useful tool in Akka. Then the actor use the `context.become` method to switch its behaviour (i.e. the function that is used to handle incoming messages) to the `retrying` function. This will respond to `Retry` messages by sending again the message to the target actor and it will respond to any other message returning it to the requester actor and shutting itself off; after the specified number of attempts, if no answer is received, a `Failure` message encapsulating a `RetryException` is returned to the requester.

The public APIs (i.e. the `askretry` extension method and the `RetryException` class) are exposed in the following way:

{% highlight scala %}
object AskRetry {
  case class RetryException(attempts: Int) extends Exception(s"Cannot retry after $attempts attempts")

  private def retry[T](actor: ActorRef, msg: T, maxAttempts: Int, rate: FiniteDuration)(implicit context: ActorContext): Future[Any] = {
    implicit val to = Timeout.durationToTimeout(rate * (maxAttempts + 1))
    context.actorOf(RetryingActor.props) ? Ask(actor, msg, rate, maxAttempts)
  }

  implicit class RetryingActorRef(val ref: ActorRef) extends AnyVal {
    def askretry[T](
      msg: T, maxAttempts: Int, rate: FiniteDuration)(implicit context: ActorContext): Future[Any] =
        retry(ref, msg, maxAttempts, rate)
  }
}
{% endhighlight %}

To use it just do:

{% highlight scala %}
import astrac.akka.askretry
val f = target.askretry(msg, 5, 500.millis)
{% endhighlight %}

[The repository](https://github.com/Astrac/akka-askretry) of this small project contains the full annotated source code as well as some tests; in a further article I hope to talk about testing actor systems as it is quite an interesting challenge and Akka provides some powerful tools to get it done.
