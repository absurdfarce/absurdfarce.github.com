---
layout: post
title: "Learning Scala from Dead Swiss Mathematicians : Palindromes"
date: 2010-07-26 12:00
comments: true
categories: [scala euler palindrome]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2010/07/learning-scala-from-dead-swiss.html)

As part of my effort to get up to speed with Scala I returned to an old favorite for exploring new languages; working through some of the problems at [Project Euler](http://projecteuler.net). Posting complete answers to specific problems isn't terribly interesting and I have no plans to do so here. That said, some facets of these problems do lead to useful digressions which allow for a deeper exploration of the language. We'll dig into one such example here.

The problem under consideration is Problem 4 which asks for "the largest palindrome made from the product of two 3-digit numbers". It doesn't take much imagination to determine that part of the solution will involve a method, most likely a function of some kind, for identifying whether a number is in fact a palindrome. Okay, but what would such a method look like? The problem lends itself to a simple recursive implementation: divide the integer into a leading digit, a middle integer and a trailing digit and return false if the leading and trailing digits differ, otherwise return the result of the a recursive call with the middle integer.

Easy enough, but a moments reflection tells us we have a problem; we can access the trailing digit in an integer using the mod function and the base of the input integer but there's no easy way to access the leading digit. It's not as if Int (or even RichInt) extend Seq[Int] or even Seq[Byte]. Fortunately we can shift the problem domain by observing that an input integer is equivalent to an input String in which each digit of the integer maps to a Char in the String. Even better, Scala offers built-in support for regular expressions, and a fairly simple regex should give us access to both the leading and trailing characters and everything in the middle. So, how does Scala's regex support stack up?

{% codeblock lang:scala %}
// Pattern to be used by the regex-based predicate.  
// Absolutely must use conservative matching and 
// the backref here to make this work.
val palindromePattern = """(\d)(\d*?)\1""".r

// Recursive helper function to check for a 
//palindrome using a regex
def byRegexHelper(n:String):Boolean = {

  // Base case; empty string and single characters 
  // are by definition palindromes.  Place this 
  // test up front so that we can handle input values
  // of a single character.
  if (n.length == 0 || n.length == 1)
    return true
  palindromePattern.unapplySeq(n) match {

    case Some(matches) => byRegexHelper(matches(1))
    case None => false
  }
}

// Regex-based predicate; convert to a string and call
// the recrusive function
def byRegex(arg:Int):Boolean = byRegexHelper(arg.toString)
{% endcodeblock %}

Not bad, actually. It's not Perl or Ruby but the deep support for pattern matching in the language combined with relatively easy generation of a Regex from a String makes for a fairly clean syntax. Regex literals would be a small improvement but this is still cleaner than what one finds in Java or even Python.

So we've solved the problem, but can we do better? Do we really need the regex? Strings (or the StringOps they're implicitly converted to in 2.8) make use of the SeqLike[Char] trait allowing easy access to the leading and trailing characters using simple method calls. This leads to the following implementation which avoids the overhead of evaluating the regular expression:

{% codeblock lang:scala %}
// Recursive helper function to perform the check 
// based on comparisons of the head and last 
// characters in a string
def byStringHelper(n:String):Boolean = {

  // Base case; empty string and single characters
  // are by definition palindromes.  Place this test
  // up front so that we can handle input values of
  // a single character.
  if (n.length == 0 || n.length == 1)
    return true
  if (n.head != n.last)
    false
  else
    byStringHelper(n.substring(1,n.length - 1))
}

// String-based predicate; convert to string and call
// the recursive function
def byString(arg:Int):Boolean = byStringHelper(arg.toString)
{% endcodeblock %}

Also not bad, but still not completely satisfying. Moving away from the use of regular expressions minimizes the benefit of mapping the input integer onto a String in order to solve the problem. Recall that the primary argument for doing so was the inability to access leading digits in an input integer. How significant is this constraint? Is it something we can work around? More on this next time.
