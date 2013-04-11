---
layout: post
title: "Default arguments and implicit conversions in Scala"
date: 2010-10-10 12:00
comments: true
categories: [scala]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2010/10/default-arguments-and-implicit.html)

In a [previous post](blog/2010/08/23/learning-scala-from-dead-swiss-mathematicians-return-of-palindromes/) we went in search of an implicit conversion from Int to List[Int] such that each member of the list corresponds to the value at an equivalent position in the input Int (i.e. 987 = List(9,8,7)). At the time we mentioned that a properly tail recursive implementation proved to be a bit more complicated than one might expect. In this post we'll examine these problems in some detail.

A properly tail recursive implementation of this conversion function makes use of an accumulator array to store state as we recurse. 

{% codeblock lang:scala %}
package org.fencepost.defaults

import org.scalatest.Suite

class ImplicitDefaultTest1 extends Suite {

  def int2list(arg:Int, acc:List[Int]):List[Int] = {

    val newmember = arg % 10
    if (arg <= 9)
      List(newmember) ::: acc
    else
      int2list(arg / 10,List(newmember) ::: acc)
  }
  
  implicit def toList(arg:Int) = int2list(arg,List())

  def testImplicit() = {

    assert(0.length == 1)
    assert(0(0) == 0)
    assert(5.length == 1)
    assert(5(0) == 5)
    assert(12345.length == 5)
    assert(12345(0) == 1)
    assert(12345(2) == 3)
    assert(12345(4) == 5)
    assert(98765432.length == 8)
    assert(98765432(4) == 5)
  }
}
{% endcodeblock %}

The test above passes so everything looks good so far. On a second look, however, we note that the wrapper function toList() is less than ideal. The accumulator needs to be initialized to the empty list in order for the function to work correctly but defining a second function just to pass in an extra arg looks like unnecessary cruft. Scala 2.8 introduced default arguments to address situations such as this; perhaps we can deploy default arguments here to clean up our test:

{% codeblock lang:scala %}
package org.fencepost.defaults

import org.scalatest.Suite

class ImplicitDefaultTest2 extends Suite {

  implicit def int2list(arg:Int, acc:List[Int] = List()):List[Int] = {

    val newmember = arg % 10
    if (arg <= 9)
      List(newmember) ::: acc
    else
      int2list(arg / 10,List(newmember) ::: acc)
  }

  def testImplicit() = {

    assert(0.length == 1)
    assert(0(0) == 0)
    assert(5.length == 1)
    assert(5(0) == 5)
    assert(12345.length == 5)
    assert(12345(0) == 1)
    assert(12345(2) == 3)
    assert(12345(4) == 5)
    assert(98765432.length == 8)
    assert(98765432(4) == 5)
  }
}
{% endcodeblock %}

When we attempt to execute the test above we're greeted rather rudely by sbt:

    [info] Compiling test sources...
    [error] .../src/test/scala/org/fencepost/defaults/ImplicitDefaultTest2.scala:21:
     value length is not a member of Int
    [error]     assert(0.length == 1)
    [error]              ^
    [error] .../src/test/scala/org/fencepost/defaults/ImplicitDefaultTest2.scala:22:
     0 of type Int(0) does not take parameters
    [error]     assert(0(0) == 0)
    ...

Clearly the implicit conversion of Int to List[Int] wasn't in play when this test was executed. But why not? Logically int2list(arg:Int, acc:List[Int] = List()) will convert Ints to List[Int] everywhere int2list(arg:Int, acc:List[Int]) does. We can demonstrate the validity of this claim by fooling the compiler using a variation on the front-end function we used before:

{% codeblock lang:scala %}
package org.fencepost.defaults

import org.scalatest.Suite

class ImplicitDefaultTest3 extends Suite {

  def int2list(arg:Int, acc:List[Int] = List()):List[Int] = {

    val newmember = arg % 10
    if (arg <= 9)
      List(newmember) ::: acc
    else
      int2list(arg / 10,List(newmember) ::: acc)
  }

  implicit def toList(arg:Int) = int2list(arg)

  def testImplicit() = {

    assert(0.length == 1)
    assert(0(0) == 0)
    assert(5.length == 1)
    assert(5(0) == 5)
    assert(12345.length == 5)
    assert(12345(0) == 1)
    assert(12345(2) == 3)
    assert(12345(4) == 5)
    assert(98765432.length == 8)
    assert(98765432(4) == 5)
  }
}
{% endcodeblock %}

As expected this test passes without issue.

My suspicion is that this issue is a side effect of the fact that default arguments apparently aren't represented in the type system. It's not surprising that int2list(arg:Int, acc:List[Int]) isn't available as an implicit conversion; there's no way for the runtime to supply the required "acc" argument for an input Int instance. This is not true for int2list(arg:Int, acc:List[Int] = List()); in that case the default value of "acc" could be used to perform the translation. Note, however, that these two functions are represented by the same type in the Scala runtime:

    $ scala
    Welcome to Scala version 2.8.0.final (OpenJDK Client VM, Java 1.6.0_18).
    Type in expressions to have them evaluated.
    Type :help for more information.
    
    scala>   def int2list(arg:Int, acc:List[Int]):List[Int] = {
    ...
    int2list: (arg: Int,acc: List[Int])List[Int]
    
    scala>   def int2list2(arg:Int, acc:List[Int] = List()):List[Int] = {
    ...
    int2list2: (arg: Int,acc: List[Int])List[Int]

If the type system is unaware that default arguments are available for all arguments other than the type to convert from then it's not at all surprising that a function would be excluded from the set of valid implicit conversion functions.

Results tested and verified on both Scala 2.8.0 and 2.8.1 RC2.
