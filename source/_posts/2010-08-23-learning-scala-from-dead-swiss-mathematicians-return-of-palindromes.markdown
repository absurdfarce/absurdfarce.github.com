---
layout: post
title: "Learning Scala from Dead Swiss Mathematicians : Return of Palindromes"
date: 2010-08-23 12:00
comments: true
categories: [scala euler palindrome]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2010/08/learning-scala-from-dead-swiss.html)

In a [recent post](blog/2010/07/26/learning-scala-from-dead-swiss-mathematicians-palindromes/) we implemented a predicate to identify integers which were also palindromes. Our original implementation converted the input integer to a String in order to more easily access the leading digit, something that can't easily be done with an input integer. But is this really the case?

If we could easily convert Integers into a List[Integer] [1] we could then easily access the "head" and "tail" of this list for purposes of comparison. Ideally this conversion could be automated so that we don't have to explicitly track these conversions. Fortunately Scala's implicit conversions provide exactly these features. A simple implementation looks something like the following:

{% codeblock lang:scala %}
implicit def int2list(arg:Int):List[Int] = 
  if (arg <= 9)
    List(arg)
  else
    int2list(arg / 10) ::: List(arg % 10)
{% endcodeblock %}

It's worth taking a moment to point out a few things about this function:

* We're assuming base 10 integers here. We could make the function more flexible by adding a parameter to specify the integers base if necessary
* A better implementation would be purely tail recursive, but this turns out to be a bit trickier than expected; more on that in a future post.

With this conversion in place we can now define a properly tail recursive predicate to check for palindromes:

{% codeblock lang:scala %}
def byInt(arg:List[Int]):Boolean = {

  if (arg.length == 0 || arg.length == 1)
    return true
  if (arg.head != arg.last)
    false
  else
    byInt(arg.slice(1,arg.length - 1))
}
{% endcodeblock %}

Very nice, but Scala's pattern matching allows for an even better (by which we mean "simpler") implementation that makes use of pattern guards:

{% codeblock lang:scala %}
def byIntMatch(arg:List[Int]):Boolean = {

  // Note that we don't need to check for lists of length 
  // 0 or length 1 as we do in byInt above.  The first 
  // two cases of our match operation below handle these 
  // cases.
  arg match {
    case List() => true
    case List(_) => true
    case arghead :: rest if arg.last == arghead => byIntMatch(rest.slice(0,rest.length - 1))
    case _ => false
  }
}
{% endcodeblock %}

The full implementation can be found at [github](https://github.com/heuristicfencepost/euler). Most of the code discussed here can be found in the org.fencepost.palindrome package; a full solution to Problem 4 using this code is also included.

[1] We use integers here only as a matter of convenience; of course we really only need a Byte as long as we're talking about base 256 or less. We can always optimize for space later if needed.

UPDATE - Shortly after completing this post I began another chapter in the "stairway" book. That chapter (covering details of the implementation of the List class in Scala) promptly pointed out that the list concatenation operator executes in time proportional to the size of the operand on the left. In order to avoid the performance hit we shift to an approach based on iteration and mutation.

{% codeblock lang:scala %}
implicit def int2list(arg:Int):List[Int] = {

  val buff = new ListBuffer[Int]
  var counter = arg
  while (counter > 9) {
    buff += (counter % 10)
    counter = (counter / 10)
  }
  buff += counter
  buff toList
}
{% endcodeblock %}

The process for choosing when to use an iterative approach over a functional approach is still somewhat opaque. Scala has an explicit bias in favor of functional methods yet in many cases (including this one) an iterative implementation is the correct one to use. Presumably identifying when to use which approach is largely a function of experience with the language.


