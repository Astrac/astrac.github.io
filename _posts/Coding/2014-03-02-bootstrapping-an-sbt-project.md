---
layout: post
category : Coding
title: "Bootstrapping an sbt project"
tagline: "How to create a project and run some basic commands with the Scala Build Tool"
tags : [Scala, sbt]
---
{% include JB/setup %}

I usually find the initial frustrations that I experience when I try to learn a new language to depend quite often on getting acquainted with the tools. When I approached Scala [sbt](http://www.scala-sbt.org/) has been a major source of headaches and this often distracted me from learning the language. It is quite a complex tool, maybe a bit too complex sometimes, but it is also very powerful when you understand it: the fact that the build file is a Scala class itself allow you to achieve great levels of reuse and modularity in your build files. In this small tutorial we won't talk about any advanced feature of sbt, but we will see hot to get out of the way some common roadblocks and how to get quickly started with a new project.

[//]: <> (@@@@@@@@@@@)

### Installation

To install sbt you can use any version that comes pre-packaged for OS and it should just work as expected; it has installers available for Windows, Fedora and Debian-based systems with third party support for Mac and Gentoo. Personally though I prefer resorting to a manual installation as this approach tends to be more manageable when you want to upgrade sbt or use a specific version of it. Just download the zip or tgz in from [the website](http://www.scala-sbt.org/release/docs/Getting-Started/Setup.html) and unpack it in any folder on your system. Then you should create a symlink or add the extraction path to your environment PATH variable so that you can run it from anywhere easily. When you are done just create an `hello-world` folder and run:


```
sbt
```

It should do some steps and then present you the sbt prompt. To exit from sbt just press `CTRL+D` or use the `exit` command; `help` will give you a list of available commands.

### Folder layot

A complex Scala project could have the following folder structure:

```
project
 | - src
 | | - main
 | | | scala
 | | | | ...
 | | | java
 | | | | ...
 | | | resources
 | | | | ...
 | | - test
 | | | scala
 | | | | ...
 | | | java
 | | | | ...
 | | | resources
 | | | | ...
 | - project
 | | - ProjectBuild.scala
 | | ...
 | ...
```

In general you don't need to have all these folders, but to configure your build and add library dependencies or sbt plugins you should create the `project` folder and have a build file in it; it is possible to write one in two different ways: [by using an sbt file](http://www.scala-sbt.org/release/docs/Getting-Started/Basic-Def.html) or [by using a scala file](http://www.scala-sbt.org/release/docs/Getting-Started/Full-Def.html). I will use the latter as I like it more, use the documentation link above to find how to map the following instructions if you want to use an sbt build.

Let's create the following structure for our basic project:

```
hello-world
 | - src
 | | - main
 | | | - scala
 | | | | - helloworld
 | | - test
 | | | - scala
 | | | | - helloworld
 | - project
 | | - HelloWorldBuild.scala
```

### Build definition

As said above, a build definition in sbt could be done in different ways; the most commons are as sbt file and as scala build. I prefer the latter as it fits more naturally in my way of thinking and it helps creating more modular and reusable build files. This is how a basic build definition for the hello-world project looks like:

{% highlight scala %}
import sbt._

object HelloWorldBuild extends Build {
  val helloWorldProject = Project(id = "sbt-hello-world", base = file("."))
}
{% endhighlight %}

If you save this file and run sbt from the project root it should start the prompt and you should a line similar to this one:

```
[info] Set current project to sbt-hello-world (in build file:...)
>
```

As you can see the `id` variable we passed to the `Project` apply function is here displayed. The `base` parameter is instead useful when writing multi-project builds and specifies in which path to expect the `src` folder relatively to the project root.

Let's create now an application main class by adding the following file in src/main/scala/helloworld/HelloWorld.scala:

{% highlight scala %}
package helloworld

object HelloWorld {
  def main(args: Array[String]) {
    println("Hello world!")
  }
}
{% endhighlight %}

We can run this from the sbt shell by executing the command `run`:

```
> run
[info] Compiling 1 Scala source to /home/aldo/Documents/Projects/hello-world/target/scala-2.10/classes...
[info] Running helloworld.HelloWorld
Hello world!
[success] Total time: 0 s, completed 02-Mar-2014 13:14:56
```

If you define multiple objects with a `main` method in your project sbt will prompt you to choose one. Another thing that is worth knowing as it is quite useful is that if you prefix any sbt command with a `~` character sbt will re-run it automatically as soon as you change anything in the project sources.

### Adding dependencies

We have a very simple build and a classic hello world program running. Let's try to get something a nicer by adding a dependency on the [jFiglet library](http://lalyos.github.io/jfiglet/) and getting an ascii banner. In the library's page there is explained how to add import the library using maven xml configuration; this maps quite easily to settings in an sbt build as follows:

{% highlight scala %}
import sbt._
import Keys._

object HelloWorldBuild extends Build {

  lazy val jfiglet = "com.github.lalyos" % "jfiglet" % "0.0.2"

  lazy val helloWorldSettings = Project.defaultSettings ++ Seq(
    libraryDependencies ++= Seq(jfiglet)
  )

  lazy val helloWorldProject = Project(id = "sbt-hello-world", base = file("."),
    settings = helloWorldSettings)
}
{% endhighlight %}

Or using a .sbt build:

```
libraryDependencies += "com.github.lalyos" % "jfiglet" % "0.0.2"
```

As you can see a dependency is usually expressed using a simple DSL embedded in sbt that represents groupId, artifactId and version number of an artifact hosted in a maven repository. There are also other options and methods to specify for example whether or not the dependency should be valid for test or production environment, if dependencies must be downloaded transitively or not and so on. The bottom-line is that you can import any JVM artifact and use sites like [Maven Central](http://search.maven.org/) to find libraries to use.

Let's use jfiglet in our main function:

{% highlight scala %}
package helloworld

object HelloWorld {
  def main(args: Array[String]) {
    println(FigletFont.convertOneLine("Hello world!"))
  }
}
{% endhighlight %}

The ouput now is:

```
> run
[info] Compiling 1 Scala source to /home/aldo/Documents/Projects/hello-world/target/scala-2.10/classes...
[info] Running helloworld.HelloWorld
 _   _        _  _                                 _      _  _
| | | |  ___ | || |  ___   __      __  ___   _ __ | |  __| || |
| |_| | / _ \| || | / _ \  \ \ /\ / / / _ \ | '__|| | / _` || |
|  _  ||  __/| || || (_) |  \ V  V / | (_) || |   | || (_| ||_|
|_| |_| \___||_||_| \___/    \_/\_/   \___/ |_|   |_| \__,_|(_)


[success] Total time: 0 s, completed 02-Mar-2014 18:43:33
```

### Writing and running tests

A project of any size needs testing and sbt supports many test libraries; in the scala ecosystem quite popular libraries are [ScalaTest](http://www.scalatest.org/) and [specs2](http://etorreborre.github.io/specs2/). They both work well with sbt and provide a very good set of tools to write tests from the unit-level to more BDD-like specifications. I'm going to show how to add ScalaTest and how to run a simple suite, the same applies for specs2 if you want to give it a try.

Using the information provided on the [ScalaTest download page](http://www.scalatest.org/download) we add this to our build file:

{% highlight scala %}
import sbt._
import Keys._

object HelloWorldBuild extends Build {

  lazy val jfiglet = "com.github.lalyos" % "jfiglet" % "0.0.1"

  lazy val scalaTest = "org.scalatest" %% "scalatest" % "2.0" % "test"

  lazy val helloWorldSettings = Project.defaultSettings ++ Seq(
    libraryDependencies ++= Seq(jfiglet, scalaTest)
  )

  lazy val helloWorldProject = Project(id = "sbt-hello-world", base = file("."),
    settings = helloWorldSettings)
}
{% endhighlight %}

To write a dummy test we put a file named `HelloWorldSpec.scala` in `src/test/scala/helloworld` with this content:

{% highlight scala %}
import org.scalatest._

class HelloWorldSpec extends FlatSpec with Matchers {

  "A dummy test" should "just pass" in {
    true should be(true)
  }
}
{% endhighlight %}

The example is just an adaptation of what you can find on the ScalaTest website and is meant just to show how to write and run tests in an sbt project. To run it we just use the `test` command as follows:

```
> test
[info] Compiling 1 Scala source to /home/aldo/Documents/Projects/hello-world/target/scala-2.10/test-classes...
[info] ExampleSpec:
[info] A dummy test
[info] - should just pass
[info] Run completed in 244 milliseconds.
[info] Total number of tests run: 1
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 1, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[success] Total time: 2 s, completed 02-Mar-2014 19:09:14
```

### Conclusion

This article just scratches the surface on what the Scala Build Tool is able to do; there is much more in it and there are several complexities that have also caused some criticism on it. I hope anyway that this will be a useful resource to get started quickly when creating a new project, adding some libraries and beginning some TDD.
