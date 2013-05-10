---
layout: post
title: "Ruby and Concurrency: The Mechanics of Akka and JRuby"
date: 2011-09-12 12:00
comments: true
categories: [ruby jruby akka concurrency actors]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2011/09/ruby-and-concurrency-mechanics-of-akka.html)

I'm interested in concurrency. I'm also interested in Ruby. There doesn't seem to be much reason to keep these two guys apart anymore.

This post marks the beginning of an occasional series on the topic of using Ruby to write concurrent code. Ruby doesn't yet have a big reputation in the world of concurrent and/or parallel applications, but there is some interesting work being done in this space. And since problems in concurrency are notoriously difficult to reason about we could probably do a lot worse than attempt to address those problems in a language designed to make life easier for the developer.

We begin with [Akka](http://akka.io), an excellent library designed to bring actors, actor coordination and STM to Scala and, to a slightly lesser degree, the JVM generally. Our initial task is simple: we wish to be able to define an Akka actor by providing a message handler as a code block. We'll start by attempting to implement the actor as a standalone class and then abstract that solution into code which requires the input block and nothing else.

A simple initial implementation looks something like this:


{% codeblock lang:ruby %}
require 'java'
require 'akka-actor-1.2-RC6.jar'
require 'scala-library.jar'

java_import 'akka.actor.UntypedActor'
java_import 'akka.actor.Actors'

# Start with something simple.  Implement the actor as a distinct
# class and start it up within the Akka runtime.

# Look, I've implemented the actor API!
class SomeActor < UntypedActor

  def onReceive msg
    puts "Received message: #{msg.to_s}"
  end
end

# So we should be fine then, right?
ref = Actors.actorOf SomeActor.new
ref.start
ref.tell "foo"
{% endcodeblock %}

    [@varese ruby]$ ruby --version
    jruby 1.6.2 (ruby-1.8.7-p330) (2011-05-23 e2ea975) (OpenJDK Client VM 1.6.0_22) [linux-i386-java]
    [@varese ruby]$ ruby akka_sample1.rb 
    ArgumentError: Constructor invocation failed: ActorRef for instance of actor [org.jruby.proxy.akka.actor.UntypedActor$Proxy0] is not in scope.
     You can not create an instance of an actor explicitly using 'new MyActor'.
     You have to use one of the factory methods in the 'Actor' object to create a new actor.
     Either use:
      'val actor = Actor.actorOf[MyActor]', or
      'val actor = Actor.actorOf(new MyActor(..))'
      (root) at akka_sample1.rb:20

Well, that didn't work very well. Apparently the class we defined in JRuby is being exposed to the Java lib as a proxy object and that proxy's class is unknown to the Akka runtime. No problem; Akka supports a factory model for actor creation, and by using that approach the underlying class of our actor should become a non-issue. With a few simple changes we're ready to try again:

{% codeblock lang:ruby %}
require 'java'
require 'akka-actor-1.2-RC6.jar'
require 'scala-library.jar'

java_import 'akka.actor.UntypedActor'
java_import 'akka.actor.Actors'

# A second attempt.  Define our actor in a standlone class again
# but this time use an ActorFactory (via closure coercion) to
# interact with Akka.

# Define our actor in a class again...
class SomeActor < UntypedActor

  def onReceive msg
    puts "Received message: #{msg.to_s}"
  end
end

# ... and then provide an UntypedActorFactory to instantiate
# the actor.  The factory interface qualifies as a SAM so
# we only need to provide a closure to generate the actor and
# let JRuby's closure coercion handle the rest.
ref = Actors.actorOf do
  SomeActor.new
end

ref.start
ref.tell "foo"
{% endcodeblock %}

    [@varese ruby]$ ruby akka_sample2.rb
    Received message: foo

We now have a working actor, but we still have some work to do; remember, we want to be able to define arbitrary actors by supplying just a code block. We need a few additional pieces to make this work:

* A generic actor implementation whose state includes a block or Proc instance. The onReceive method of this actor could then simply call the contained block/Proc, passing the input message as an arg.
* An ActorFactory implementation which takes a code block as an arg, stores it in internal state and then uses that block to build an instance of the generic actor described above on demand.

A first cut at this concept might look something like this:

{% codeblock lang:ruby %}
require 'java'
require 'akka-actor-1.2-RC6.jar'
require 'scala-library.jar'

java_import 'akka.actor.UntypedActor'
java_import 'akka.actor.UntypedActorFactory'
java_import 'akka.actor.Actors'

# Shift to working with code blocks via a generic actor and a simple
# factory for creating them.

class SomeActor < UntypedActor

  # Look, I've got a constructor... but it'll get ignored!
  def initialize(proc)
    @proc = proc
  end

  def onReceive(msg)
    @proc.call msg
  end
end

class SomeActorFactory
  include UntypedActorFactory

  def initialize(&b)
    @proc = b
  end

  def create
    SomeActor.new @proc
  end
end

# Create a factory for an actor that uses the input block to pass incoming messages.
factory = SomeActorFactory.new do |msg|
  puts "Message recieved: #{msg.to_s}"
end
ref = Actors.actorOf factory

ref.start
ref.tell "foo"
{% endcodeblock %}

    [@varese ruby]$ ruby akka_sample3.rb 
    ArgumentError: wrong number of arguments for constructor
      create at akka_sample3.rb:29
      (root) at akka_sample3.rb:37

What went wrong here? UntypedActor is a concrete class with a defined no-arg constructor. That constructor is being called in favor of the one we've provided, and as a consequence our block never gets into the mix. There's almost certainly a cleaner way to solve this using JRuby, but for the moment we can get around the problem (in an admittedly ugly way) by providing a setter on our generic actor class:

{% codeblock lang:ruby %}
require 'java'
require 'akka-actor-1.2-RC6.jar'
require 'scala-library.jar'

java_import 'akka.actor.UntypedActor'
java_import 'akka.actor.UntypedActorFactory'
java_import 'akka.actor.Actors'

# Continue with the generic actor implementation, but shift to using a setter
# for passing in the Proc to which we'll delegate message handling.

class SomeActor < UntypedActor

  def proc=(b)
    @proc = b
  end

  def onReceive(msg)
    @proc.call msg
  end
end

class SomeActorFactory
  include UntypedActorFactory

  def initialize(&b)
    @proc = b
  end

  def create
    rv = SomeActor.new
    rv.proc = @proc
    rv
  end
end

# Create a factory for an actor that uses the input block to pass incoming messages.
factory = SomeActorFactory.new do |msg|
  puts "Message recieved: #{msg.to_s}"
end
ref = Actors.actorOf factory

ref.start
ref.tell "foo"
{% endcodeblock %}

    [@varese ruby]$ ruby akka_sample4.rb 
    Message recieved: foo

We now have what we wanted.

If you're interested in this topic note that Nick Sieger has covered similar ground (including the interaction between JRuby and Akka) [here](http://blog.engineyard.com/2011/concurrency-in-jruby). Nick's article draws on some [very good work](http://metaphysicaldeveloper.wordpress.com/2010/12/16/high-level-concurrency-with-jruby-and-akka-actors/) done by Daniel Ribeiro late last year. The code referenced in Daniel's article is [available](https://github.com/danielribeiro/RubyOnAkka) on Github. I didn't come across Daniel's post until my own was nearly done but there is quite a bit of overlap between his code and mine. That said, I recommend taking a look at both articles, if for no other reason than the fact that both authors are much better at writing Ruby code than I am.