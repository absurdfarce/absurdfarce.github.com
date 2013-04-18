---
layout: post
title: "Forks and Joins in Scala"
date: 2011-02-04 12:00
comments: true
categories: [scala concurrency fork_join clojure]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2011/02/forks-and-joins-in-scala.html)

A short while ago Alex Miller (aka puredanger) wrote up a [blog entry](http://tech.puredanger.com/2011/01/04/forkjoin-clojure) detailing how to use Clojure to interact with the new fork/join concurrency framework to be included with Java7 (and already available from Doug Lea's site). The meat of Alex's solution is a Java wrapper for Clojure's IFn type which integrates nicely with the fork/join framework. At the time there was some discussion about whether a Scala implementation would require an equivalent amount of scaffolding. It turns out that supporting this functionality in Scala requires a bit more effort than one might expect, although the problems one faces are quite different than what Alex worked through in Clojure. We review the steps to a working Scala implementation and see what we can learn from them.

Before we move on, a few notes about fork/join itself. A full overview is outside the scope of this post, however details can be found in the [jsr166y section of Doug Lea's site](http://gee.cs.oswego.edu/dl/concurrency-interest/index.html). A solid (although slightly out-of-date) review can be found in a [developerWorks article](http://www.ibm.com/developerworks/java/library/j-jtp11137/index.html) by Brian Goetz.

We begin as simply as possible; a straight port of Alex's code (in this case an implementation of Fibonacci using fork/join) to Scala. We create a named function fib that will be recursively called by our RecursiveTasks. We also use an implicit conversion to map no-arg anonymous functions onto RecursiveTask instances using the [technique for supporting SAM interfaces](http://absurdfarce.github.io/blog/2011/01/07/transmutation-of-scala-closures-into-sam-instances/) discussed in an earlier post. The resulting code looks pretty good:

{% codeblock lang:scala %}
package org.fencepost.forkjoin

import jsr166y._

import org.scalatest.Suite

class ForkJoinTestFailOne extends Suite {

  implicit def fn2wrapper(fn:()=>Int):RecursiveTask[Int] = {

    return new RecursiveTask[Int] { override def compute = fn.apply }
  }

  def testFibonacci() = {

    def fib(n:Int):Int = {

      if (n <= 1)
        return n
      val f1 = { ()=> fib(n - 1) }
      f1 fork
      val f2 = { ()=> fib(n - 2) }
      (f2 compute) + (f1 join)
    }

    val pool = new ForkJoinPool()
    assert ((pool invoke { ()=> fib(1) }) == 1)
    assert ((pool invoke { ()=> fib(2) }) == 1)
    assert ((pool invoke { ()=> fib(3) }) == 2)
    assert ((pool invoke { ()=> fib(4) }) == 3)
    assert ((pool invoke { ()=> fib(5) }) == 5)
    assert ((pool invoke { ()=> fib(6) }) == 8)
    assert ((pool invoke { ()=> fib(7) }) == 13)
  }
}
{% endcodeblock %}

Unfortunately an attempt to build with sbt gives an unexpected error:

    [info] == test-compile ==
    [info]   Source analysis: 2 new/modified, 0 indirectly invalidated, 0 removed.
    [info] Compiling test sources...
    [error] .../src/test/scala/org/fencepost/forkjoin/ForkJoinTestFailOne.scala:23: method compute cannot be accessed in jsr166y.RecursiveTask[Int]
    [error]       (f2 compute) + (f1 join)
    [error]           ^
    [error] one error found

A bit of investigation reveals the problem; RecursiveTask.compute() is a protected abstract method in the fork/join framework. The compute method in the class returned by our implicit conversion consists of nothing more than a call to fib.apply() but there's no way to know this when fib is defined. It follows that an attempt to access RecursiveTask.compute() from within the body of fib is (correctly) understood as an attempt to access a protected method.

How does Alex get around this problem in Clojure? He doesn't; the problem is actually resolved via the Java wrapper. Note that in his Java code compute() has been "elevated" to be a public method. He's thus able to call this method without issue from his "fib-task" function. Without this modification his code fails with similar errors [1].

Strangely, there's no obvious way to resolve this issue within Scala itself. The language provides protected and private keywords to restrict access to certain methods but there is no public keyword; methods are assumed to be public if not annotated otherwise. As a consequence there's no obvious way to "raise" the access level of a method in a subclass. We work around this constraint by way of an egregious hack; we define a new subclass of RecursiveTask containing a surrogate method that provides public access to compute. We then return this subclass from our implicit conversion.

Our new implementation looks like this:

{% codeblock lang:scala %}
package org.fencepost.forkjoin

import jsr166y._

import org.scalatest.Suite

class ForkJoinTestFailTwo extends Suite {

  class Wrapper[T](somefunc:()=>T) extends RecursiveTask[T] {

    override def compute = somefunc.apply

    // RecursiveTask.compute() is protected and there's no obvious way to
    // elevate it's access level within a Scala class definition.  The type
    // system (very correctly) doesn't allow a random function access to
    // protected methods of a class even when that method is invoked by
    // a method within the class itself.  Defining a new method with standard
    // access resolves the issue.
    def surrogate = compute
  }

  implicit def fn2wrapper(fn:()=>Int):Wrapper[Int] = {

    return new Wrapper[Int](fn)
  }

  def testFibonacci() = {

    def fib(n:Int):Int = {

      if (n <= 1)
        return n
      val f1 = { ()=> fib(n - 1) }
      f1 fork
      val f2 = { ()=> fib(n - 2) }
      (f2 compute) + (f1 join)
    }

    val pool = new ForkJoinPool()
    assert ((pool invoke { ()=> fib(1) }) == 1)
    assert ((pool invoke { ()=> fib(2) }) == 1)
    assert ((pool invoke { ()=> fib(3) }) == 2)
    assert ((pool invoke { ()=> fib(4) }) == 3)
    assert ((pool invoke { ()=> fib(5) }) == 5)
    assert ((pool invoke { ()=> fib(6) }) == 8)
    assert ((pool invoke { ()=> fib(7) }) == 13)
  }
}
{% endcodeblock %}

This code now compiles, but when we try to actually execute the test with sbt we see a curious error; the test starts up and then simply hangs:

    [info] == test-compile ==
    [info] 
    [info] == copy-test-resources ==
    [info] == copy-test-resources ==
    [info] 
    [info] == test-start ==
    [info] == test-start ==
    [info] 
    [info] == org.fencepost.forkjoin.ForkJoinTestFailTwo ==
    [info] Test Starting: testFibonacci
    *wait for a long, long time*

Well, this isn't good. What's going on here?

The key point to understand is that the implicit conversion is called every time an existing value needs to be converted to a different type. As written the calls to f1.fork() and f1.join() both require conversion, and the implicit conversion function in play here will return a new instance on each invocation. This means that even though the input to the conversion function is identical in both cases (f1) the object that invokes fork() will be a different object than that which invokes join(). For fork/join tasks this matters; join() is called on an object that was never forked(). Voila, an unterminated wait.

We can resolve this issue one of two ways:

* Caching previous values returned by the implicit conversion function and re-using them when appropriate
* Explicitly specifying the type of f1 and f2 as our wrapper subclass. This has the effect of forcing implicit conversion at the time of assignment rather than when we call fork() and join()

I chose the first approach, mainly because it seemed a bit cooler. Oh, and it's a bit cleaner logically as well. The final result runs without issue:

{% codeblock lang:scala %}
package org.fencepost.forkjoin

import scala.collection.mutable.LinkedList

import jsr166y._

import org.scalatest.Suite

class ForkJoinTest extends Suite {

  val cache = new LinkedList[(()=>Int,Wrapper[Int])]()

  class Wrapper[T](somefunc:()=>T) extends RecursiveTask[T] {

    override def compute = somefunc.apply

    // RecursiveTask.compute() is protected and there's no obvious way to
    // elevate it's access level within a Scala class definition.  The type
    // system (very correctly) doesn't allow a random function access to
    // protected methods of a class even when that method is invoked by
    // a method within the class itself.  Defining a new method with standard
    // access resolves the issue.
    def surrogate = compute
  }

  // Rationale for a caching implicit conversion can be found below.  We use
  // a list of tuples rather than a map since hashing buys us nothing here;
  // there is always one (and exactly one) object with a given object identity.
  implicit def fn2wrapper(fn:()=>Int):Wrapper[Int] = {

    val cacheval = cache find (t => (t != null) && (t._1 eq fn))
    cacheval match {
      case Some((_,cval)) => cval
      case None => {

          val newcval = new Wrapper[Int](fn)
          cache.next = cache.next :+ (fn,newcval)
          newcval
      }
    }    
  }

  def testFibonacci() = {

    def fib(n:Int):Int = {

      if (n <= 1)
        return n

      // The approach used here only works if our implicit conversion always
      // returns the same Wrapper instance for a given function.  For a given
      // RecursiveTask instance join() must be called on the same instance that
      // originally called fork().  It follows that an implicit conversion such
      // as:
      //
      // implicit def fn2wrapper(fn:()=>Int):Wrapper[Int] = {
      //    return new Wrapper(fn)
      // }
      //
      // won't be adequate; fork() and join() will be called on two separate
      // objects (since implicit conversion is performed on each method call)
      // and join() will never return.
      //
      // Alternately we can force conversion when f1 and f2 are assigned values
      // using something like the following;
      //
      // val f1:Wrapper[Int] = { ()=> fib(n - 1) }
      // val f2:Wrapper[Int] = { ()=> fib(n - 2) }
      //
      // Clearly in this case f1.fork() and f1.join() will be called on the
      // same instance of a RecursiveTask subclass.
      val f1 = { ()=> fib(n - 1) }
      f1 fork
      val f2 = { ()=> fib(n - 2) }
      (f2 surrogate) + (f1 join)
    }


    val pool = new ForkJoinPool()
    assert ((pool invoke { ()=> fib(1) }) == 1)
    assert ((pool invoke { ()=> fib(2) }) == 1)
    assert ((pool invoke { ()=> fib(3) }) == 2)
    assert ((pool invoke { ()=> fib(4) }) == 3)
    assert ((pool invoke { ()=> fib(5) }) == 5)
    assert ((pool invoke { ()=> fib(6) }) == 8)
    assert ((pool invoke { ()=> fib(7) }) == 13)
  }
}
{% endcodeblock %}

[1] Using Alex's code (as published on his blog):

    bartok ~/git/puredanger_forkjoin $ clojure process.clj 
    (1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987 1597 2584 4181)

After changing IFnTask.compute() to protected (with no other changes):

    bartok ~/git/puredanger_forkjoin $ clojure process.clj 
    (Exception in thread "main" java.lang.reflect.InvocationTargetException
     at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
     at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
     at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
     at java.lang.reflect.Method.invoke(Method.java:616)
     at jline.ConsoleRunner.main(Unknown Source)
    Caused by: java.lang.RuntimeException: java.lang.IllegalArgumentException: No matching field found: compute for class revelytix.federator.process.IFnTask (process.clj:0)
     at clojure.lang.Compiler.eval(Compiler.java:5440)
     at clojure.lang.Compiler.load(Compiler.java:5857)
     at clojure.lang.Compiler.loadFile(Compiler.java:5820)
     at clojure.main$load_script.invoke(main.clj:221)
     at clojure.main$script_opt.invoke(main.clj:273)

