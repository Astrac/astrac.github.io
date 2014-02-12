---
layout: post
category : coding
title: "Simple chat on Akka Actors"
tagline: "Having fun at the Hack The Tower in London"
tags : [Scala, Akka, Distributed systems, Hackatons]
---
{% include JB/setup %}

The past Saturday I've been attending the Hack The Tower in London Salesforce office. The format is basically the one of an hackaton where people coming from different technology user groups (namely Scala, Java and Clojure) meet and have fun for one day in the nice setting of the Heron Tower in the centre of the City of London. It was the second time for me to attend this event, I will probably speak about the first one in another post, and I joined the group of people that was there to work on Scala.

@@@@@@@@@@@

### The project

I arrived a bit late and I found that the team was already formed and the project to work on was already decided: start from a draft from one of the participant and build a simple IRC-like chat that leverages the Akka actor system for connecting the clients to a centralized server. The intention was then to put the server on heroku, but that's still a work in progress as it wasn't so easy to make Akka remoting work on it.

The Akka actor system is a parallel and distributed computing framework inspired by the Erlang systems. It features networking, routing, supervision, clustering and many other features. An actor is basically a unit of functionality that runs independently from other actors and that shares nothing of its internal state; all interactions with and between actors are meant to happen through opaque `ActorRef` instances and by the exchange of (immutable) messages. For this reason in a certain sense Actors are the most pure form of object orientation despite being a concept that found popularity in the functional world.

### Networking actors

The draft we started from was just allowing two JVMs to talk to each other on the same PC. As a matter of principle for Akka there is no difference between actors talking in this way or over the network; practically anyway there were some issues with addressing that we had to solve to get it working.

What we learned is that to correctly deliver and receive messages our server needs to be configured with the public address of itself as it won't correctly bind to 0.0.0.0. On the other hand a client which is connecting to such a server via ActorSelection needs to know the server public address *and* to be configured with its own public address in the netty configuration.

### The server

The server actor is indeed very simple and as one would expect no network details are polluting its implementation. This is a version of it stripped of some non-mandatory bits:

{% highlight scala %}
class ChatServerActor extends Actor {

  val connectedClients:collection.mutable.Set[ActorRef] = Set()

  def receive = {

    case m @ ChatMessage(x: String) =>
      connectedClients.filter(_ != sender).foreach(_.forward(m))

    case RegisterClientMessage(client: ActorRef) =>
        this.connectedClients += client

    case m @ PrivateMessage(target, _) =>
      connectedClients.filter(_.path.name.contains(target)).foreach(_.forward(m))

    case RegisteredClients =>
      sender ! RegisteredClientList(connectedClients)

    case Unregister =>
        this.connectedClients -= sender
        sender ! PoisonPill
  }
}
{% endhighlight %}

The server is using a mutable set to store the set of connected clients. The mutability is not an issue as the Actor is not sharing state with any other thread and it is guaranteed to process messages sequentially by Akka. I'm not a big fun of mutable collections in general, writing this again I would probably use a var with an immutable Set or avoid at managing any mutability on my own and use instead the `context.become` function to alter the behaviour of the server every time a client connects or leaves.

The receive function implements the behaviour of the actor by pattern-matching on the possible messages (defined elsewhere). The conciseness of the scala collection library allow to keep each handler really simple: finding the target for a private message or excluding the sender for broadcast is easy feels natural with higher order functions like `filter` and `foreach`.

It is also worth noting the difference of usage between `!` (aka `tell`) and `forward`: the former is making explicit that the sender of the message is the server actor, the latter is passing the message along transparently, so that for example a client can receive a message from another client without knowing that a server was in the middle.

To start the server actor the only thing that is needed is to create it from an ActorSystem instance as follows:

{% highlight scala %}
val system = ActorSystem("AkkaChat")
val server = system.actorOf(Props[ChatServerActor], name = "chatserver")
{% endhighlight %}

### The client

The client anatomy is just a bit more complex than the server one as it has to deal with user input and server lookup. This is a breakdown of how the main looks like:

{% highlight scala %}
def main(args:Array[String]) {
  // Gets a name for the actor
  val identity = readLine()
  val system = ActorSystem("AkkaChat")

  // These values are stored in the configuration
  val serverAddress = system.settings.config.getString("actor-chat.server.address")
  val serverPort = system.settings.config.getString("actor-chat.server.port")

  // Finds the server actor
  val server = system.actorSelection(s"akka.tcp://AkkaChat@$serverAddress:$serverPort/user/chatserver")

  // Starts the client actor passing the server and the identity
  val client = system.actorOf(Props(classOf[ChatClientActor], server, identity), name = identity)

  // Useful regex to match the private message command
  val privateMessageRegex = """^@([^\s]+) (.*)$""".r

  // Prompt loop
  Iterator.continually(readLine()).takeWhile(_ != "/exit").foreach { msg =>
    msg match {
      case "/list" =>
        server.tell(RegisteredClients, client)

      case "/join" =>
        server.tell(RegisterClientMessage(client), client)

      case privateMessageRegex(target, msg) =>
        server.tell(PrivateMessage(target, msg), client)

      case _ =>
        server.tell(ChatMessage(msg), client)
    }
  }

  println("Exiting...")
  server.tell(Unregister, client)
}
{% endhighlight %}

Everything is quite simple in here; the `Iterator.continually` function may seem a bit strange for people not used to functional programming but it is just creating a stream containing all the commands that a user will send and that will block until the next command when the program is waiting for user input.

It's also interesting to see how with `tell` we can specify a second parameter so that the main loop will send messages that can be replied to the client actor rather than to the main function itself (that, not being an actor, couldn't receive them).

To complete the outline we are just missing the client actor, which is responsible for handling server messages. Its code reads as follows:

{% highlight scala %}
class ChatClientActor(server: ActorSelection, id: String) extends Actor {
    def receive = {
      case ChatMessage(message) =>
        println(s"$sender: $message")

      case PrivateMessage(_, message) =>
        println(s"- ${sender.path.name}: $message")

      case RegisteredClientList(list) =>
        for (x <- list) println(x)
   }
}
{% endhighlight %}

This is again pretty straightforward: each message has a very simple handling routine that just prints some information for the user.

There are some naive assumptions in all this system (e.g. the client identity being the user name, the lacking of supervision and others), but it was quite useful to get the basic point about actors in Akka. Moreover we had a lot of fun during the day and we went home quite satisfied by this playing.

### Some ideas

It often happens that while hacking on something some more ideas pop out of the blue and this is the case for some of our "unsolved issues" of the day. This was the case as well and there are a couple of things that I have been thinking of during the last days and that may be part of some future hacking and exploration:

* Basic setup for an Akka cluster
* Using UPnP to set up an Akka cluster
* Retrieve the publicly available IP address that makes the host machine reachable by a remote server


### References

The result of the day is in [ this repository ]( https://github.com/mildlyskilled/actor-chat/tree/akka-actors ) hosted on Github.

Big thanks to [Kwabena Aning](https://github.com/mildlyskilled) for the idea and to all the people there for the fun, I look forward to the next opportunity to do some hacking together!
