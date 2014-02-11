---
layout: post
category : coding
title: "Simple chat on Akka Actors"
tagline: "Having fun at the Hack The Tower in London"
tags : [Scala, Akka, Distributed systems, Hackatons]
---
{% include JB/setup %}

The past Saturday I've been attending the Hack The Tower in London Salesforce office. The format is basically the one of an hackaton where people coming from different technology user groups (namely Scala, Java and Clojure) meet and have fun for one day in the nice setting of the Heron Tower in the centre of the City of London. It was the second time for me to attend this event, I will probably speak about the first one in another post, and I joined the group of people that was there to work on Scala.

### The idea

I arrived a bit late and I found that the team was already formed and the project to work on was already decided: start from a draft from one of the participant and build a simple IRC-like chat that leverages the Akka actor system for connecting the clients to a centralized server. The intention was then to put the server on heroku, but that's still a work in progress as it wasn't so easy to make Akka remoting work on it.

### Networking actors

The draft we started from was just allowing two JVMs to talk to each other on the same PC. As a matter of principle for Akka there is no difference between actors talking in this way or over the network; practically anyway there were some issues with addressing that we had to solve to get it working.

What we learned is that to correctly deliver and receive messages our server needs to be configured with the public address of itself as it won't correctly bind to 0.0.0.0. On the other hand a client which is connecting to such a server via ActorSelection needs to know the server public address *and* to be configured with its own public address in the Netty configuration.

### Show me some code!

Once networking issues were solved we had fun with some lighter things like adding private messaging capabilities, tidying up the code, avoiding the use of vars and so on. The result of the day is in [ this repository ]( https://github.com/mildlyskilled/actor-chat/tree/akka-actors ) hosted on Github.

#### Starting up

To start up an ActorSystem Akka requires very little ceremony:

    {% highlight scala %}
    val system = ActorSystem("AkkaChat", ConfigFactory.load.getConfig("chatclient"))
    {% endhighlight %}

### Some ideas

It often happens that while hacking on something some more ideas pop out of the blue and this is the case for some of our "unsolved issues" of the day.
