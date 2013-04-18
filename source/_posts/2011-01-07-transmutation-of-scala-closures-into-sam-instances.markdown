---
layout: post
title: "Transmutation of Scala closures into SAM instances"
date: 2011-01-07 12:00
comments: true
categories: [scala ruby jruby SAM]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2011/01/transmutation-of-scala-closures-into.html)

Okay, it's not actually *alchemy*. But it is pretty cool nonetheless.

A brief refresher on terminology: a *single abstract method* (or SAM) interface or abstract method contains exactly one method that concrete implementations must define. Java includes a number of interfaces that satisfy this property, including Runnable, Callable and Comparable.

The basic idea at play here is that a closure or anonymous function (either will do here) can be used in place of a SAM implementation if the parameters and return type of the closure match up to those of the method. That assumes your language cares about such things, of course; duck typing makes this less of an issue (as we'll see).

JRuby automatically performs this operation via [closure conversion](https://github.com/jruby/jruby/wiki/CallingJavaFromJRuby) (scroll down to the bottom of the page). Stuart Sierra has recently [published](http://stuartsierra.com/2010/12/16/single-abstract-method-macro) a macro for doing something similar in Clojure. Even Java is considering including this feature in an eventual closure implementation (see Brian Goetz's [writeup](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-2.html) for details). Why should Scala miss out on all the fun?

Let's take a look at some code to make this discussion more tangible. A simple example of closure conversion in JRuby looks  something like the following:

{% codeblock lang:ruby %}
require 'java'

queue = java.util.concurrent.LinkedBlockingQueue.new()
exec = java.util.concurrent.ThreadPoolExecutor.new(2,2,1,java.util.concurrent.TimeUnit::SECONDS,queue)
exec.prestartAllCoreThreads

# Use a closure that does some kind of computation rather than just returning a value.                                                                                        
# As always last expression in the closure is the return value.                                                                                                               
future = exec.submit do
  arr = []
  1.upto(5) { |val| arr.push(2 * val) }
  arr
end
exec.shutdown
rv = future.get
if rv.length == 5 && (rv.include? 2) && (rv.include? 6)
  print "Good\n"
else
  print "Not good\n"
end
{% endcodeblock %}

This is where duck typing helps us out; the only real requirement on our closure is that we actually return something (in order to clearly distinguish between Runnable and Callable). Our objective is to implement a Scala unit test that does something similar. Any such approach will be built around Scala's support for implicit conversion of types, but in this case a bit of care and feeding is required to line up the parameters and return types of the closure with that of the contents of the SAM interface. The basic approach works as follows:

* The implicit conversion accepts a function with the same parameter list and return value as the lone method in the SAM interface
* The conversion then returns a new concrete instance of the SAM interface. The implementation of the method doesn't need to be anything other than invoking apply() on the input function

The resulting Scala implementation (implemented as a ScalaTest class) is:

{% codeblock lang:scala %}
package org.fencepost

import scala.collection.mutable._

import org.scalatest.Suite

import java.util.concurrent._

class SAMTest extends Suite {

  implicit def fn2runnable(fn:()=>Unit):Runnable = {
    new Runnable { def run = fn.apply }
  }

  implicit def fn2callable[A](fn:()=>A):Callable[A] = {
    new Callable[A] { def call = fn.apply }
  }

  // We can now use a closure as a replacement for a Runnable instance
  def testClosureAsRunnable() = {

    var result = ListBuffer(1,2,3)
    val t = new Thread({ () =>
        result += 4

        // Addition of new item to the list buffer returns the item added so
        // leaving it as the last expression would violate our ()=>Unit type.
        // A simple println solves this.
        println("Done")
      })
    t.start
    t.join
    assert(result.size == 4)
    assert(result.contains(4))
  }

  // Verify that parameterized types are supported as well while demonstrating
  // integration with java.util.concurrent.  We deliberately avoid references
  // to scala.concurrent in order to avoid confusing the issue.
  def testClosureAsParameterizedSAM() = {

    val exec = new ThreadPoolExecutor(2,2,1,TimeUnit.SECONDS,new LinkedBlockingQueue())
    exec.prestartAllCoreThreads

    val future = exec.submit({ ()=> List(1,2,3).map(_ * 2) })
    val result = future.get

    exec.shutdown

    assert(result.size == 3)
    assert(result.contains(2))
    assert(result.contains(4))
    assert(result.contains(6))
  }
}
{% endcodeblock %}

