---
layout: post
title: "Ruby and Concurrency: Design with Actors and Akka"
date: 2012-01-05 12:00
comments: true
categories: [ruby jruby akka concurrency actors]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2012/01/ruby-and-concurrency-design-with-actors.html)

In a [semi-recent post](http://absurdfarce.github.io/blog/2011/09/12/ruby-and-concurrency-the-mechanics-of-akka-and-jruby/) we looked at how we might define actors in JRuby using a combination of Akka and code blocks. That post was very focused on the process of creating actors; we didn't necessarily consider how to build systems with actors and/or whether Ruby might be helpful in doing so. There are plenty of issues to ponder here, but before we dig in let's take a step back and talk briefly about some of the characteristics of actors.

Actors implement a shared-nothing model for concurrency. Mutable system state should be contained by one or more actors in the system. This state is not exposed to external entities directly; it can only be accessed or modified via messages sent to the actor. Since the actor is the only player in this drama that can access the state it contains there is no need to lock or synchronize state access. It should be fairly clear that an actor consists of some amount of mutable system state and some logic for handling incoming messages... and not much more.

Okay, sounds great... but how does it work in practice? We'll consider this question by way of an alogrithm that by now should be pretty familiar: prime number generation using the [Sieve of Eratosthenes](http://en.wikipedia.org/wiki/Sieve_of_eratosthenes). We considered this chestnut (or perhaps you prefer "war horse" at this point) in a [previous post](http://absurdfarce.github.io/blog/2010/10/22/concurrent-prime-factorization-in-go) discussing an implementation of this algorithm in the [Go language](http://golang.org). Let's review that implementation and see what else we can do with it.

The support of lightweight goroutines in Go encouraged a pipeline implementation with one goroutine per discovered prime. We can think of the "state" of this system as the total set of discovered primes. Each goroutine knows only about the prime it contains and the channel where candidates should be sent if they "pass" (i.e. do not divide evenly into it's prime number). Once the goroutine is created it's state doesn't change. New state is added by creating a new goroutine for a newly-discovered prime. State is never deleted; once a prime is discovered removing it from consideration is non-sensical. As a consequence state is completely distributed; no entity in the system knows about all discovered primes.

We also note that the algorithm described here isn't very friendly to parallel decomposition [1]. Candidate values are compared to previously discovered primes one at a time: if the candidate passes the test it's allowed to move on, otherwise no further evaluation occurs. This technique is referred to as "fail-fast" behaviour; if a value fails a test it doesn't lead to any wasteful extra work. The concurrent approach is quite different: compare the candidate to all discovered primes at once and return success only if all tests pass. Comparisons are done independently, so even if the "first" [2] comparison fails all other comparisons still execute. We lose our fail-fast behaviour but gain the ability to decompose the entire process into smaller jobs that can execute in parallel. A trade-off in engineering... surprising, I know.

We'll set out to implement the Sieve of Eratosthenes in Ruby using JRuby and Akka. Our new implementation will have more of a bias towards parallelism; this time we're okay with throwing away some work. Clearly we'll need an actor to compare a candidate to one or more discovered primes; that is, after all, why we're here. We can think of these actors as maintaining the state of our system (just like the goroutines in the Go implementation) so we'll borrow a term from MVC and call these "model" actors. Concurrently evaluating candidates against these models implies an organizing actor that is aware of all models; it seems natural to call this the "controller" actor. To keep things simple we don't want to support an unlimited number of models so we create a fixed-size "pool" and distribute system state (i.e. the set of discovered primes) between these models. When a new prime is discovered the controller will be responsible for adding that prime to an existing model.

The implementation in Ruby looks like this:

{% codeblock lang:ruby %}
require 'java'
require 'lib/akka-actor-1.2.jar'
require 'lib/scala-library.jar'

java_import 'akka.actor.Actors'
java_import 'akka.actor.UntypedActor'
java_import 'akka.actor.UntypedActorFactory'

module Sieve

  # Basic Enumerable wrapper for a Controller actor... just a convenience thing really
  class Primes
    include Enumerable

    def initialize(controller)
      @controller = controller
    end

    def each
      loop do
        yield @controller.sendRequestReply(:next)
      end
    end
  end

  # Enumerable implementation representing the set of prime number candidates > 10.  Use of an
  # enumerable here allows us to isolate the state associated with candidate selection to this
  # class, freeing up the model and controller actors to focus on other parts of the computation
  class Candidates
    include Enumerable

    def initialize
      # Note that this initial value is never actually returned; we're only setting the stage for the
      # first increment to generate the first candidate
      @next = 9
    end

    # Primes must be a number ending in 1, 3, 7 or 9... a bit of reflection will make it clear why
    def each
      loop do
        @next += (@next % 10 == 3) ? 4 : 2
        yield @next
      end
    end
  end

  class Controller < UntypedActor

    def initialize
      @models = 0.upto(3).map do |idx|
        model = Actors.actorOf { Sieve::Model.new }
        model.start
        model
      end

      # Seed models with a few initial values... just to get things going
      @seeds = [2,3,5,7]
      0.upto(3).each { |idx| @models[idx].tell [:add,@seeds[idx]] }

      @candidates = Candidates.new
    end

    # Part of the lifecycle for an Akka actor.  When this actor is shut down
    # we'll want to shut down all the models we're aware of as well
    def postStop
      @models.each { |m| m.stop }
    end

    def onReceive(msg)

      case msg
      when :next

        # If we still have seeds to return do so up front
        seed = @seeds.shift
        if seed
          self.getContext.replySafe(seed)
          return
        end

        # If we're still here then we need to evaluate candidates against our models.  Each
        # candidate value is fed into the models in parallel.  The first value that all models
        # agree is prime is returned as the value
        val = @candidates.find do |candidate|
          @models.map { |m| m.sendRequestReplyFuture [:isprime,candidate] }.all? do |f|
            f.await
            return false if not f.result.isDefined
            f.result.get
          end
        end

        # Now that we have a prime value we need to update the state of one of our models to
        # include this new value.  For now we just choose a model at random
        @models[(rand @models.size)].tell [:add,val]

        # Finally, send a response back to the caller
        self.getContext.replySafe val
      end
    end
  end

  # A model represents a fragment of the state of our sieve, specifically some subset
  # of the primes discovered so far.
  class Model < UntypedActor

    def initialize
      @primes = []
    end

    def onReceive(msg)

      # It's times like this that one really does miss Scala's pattern matching
      # but case fills in nicely enough
      (type,data) = msg
      case type
      when :add
        @primes << data
      when :isprime

        # If we haven't been fed any primes yet we can't say much...
        if @primes.empty?
          self.getContext.replySafe nil
          return
        end

        # This model only considers a value prime if it doesn't divide evenly into any
        # prime it already knows about.  Of course we have to make an exception if we're
        # testing one of the primes we already know about
        resp = @primes.none? do |prime|
          data != prime and data % prime == 0
        end
        self.getContext.replySafe resp
      else
        puts "Unknown type #{type}"
      end
    end
  end
end
{% endcodeblock %}

Ruby offers a number of features which help us out here. As shown in this implementation messages can be as simple as a list with a leading symbol (used to indicate the message "type") and a payload contained in the remainder of the list. This follows a similar convention found in Erlang, although that language uses tuples rather than lists. Support for destructuring/multiple assignment makes working with these messages quite simple.

Our previous work building actors from code blocks doesn't apply here due to an implementation detail so we define both the controller and model actors as distinct classes. As it turns out this change isn't much of a problem; we're able to implement both classes, a helper Enumerable and a second Enumerable wrapping the controller in less than 140 lines with comments. Full code (including tests) can be found on [github](https://github.com/heuristicfencepost/sieve_actors).

Spend a little time with JRuby and Akka and it becomes clear that they work well together and form a good basis for building concurrent applications in Ruby.

[1] This is a bit of a loaded statement; you will get some parallelism as multiple candidates move through the "stages" (i.e. goroutines) of the pipeline. That said, this notion of parallelism applies to the system as a whole. The process of determining whether a single candidate is or is not a prime number is still very sequential.

[2] "First" here means nothing more than "whichever test happens to complete and return a value first"