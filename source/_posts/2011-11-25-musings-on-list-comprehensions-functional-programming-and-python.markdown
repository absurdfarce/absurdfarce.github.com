---
layout: post
title: "Musings on List Comprehensions, Functional Programming and Python"
date: 2011-11-25 12:00
comments: true
categories: [list_comprehension functional_programming python haskell clojure scala]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2011/11/musings-on-list-comprehensions.html)

For simplicity and elegance in your programming constructs the [list comprehension](http://en.wikipedia.org/wiki/List_comprehension) is hard to beat. [1] A list comprehension can filter an input list, transform it or do both, all in a single expression and with very readable syntax. At it's best a list comprehension is "beautiful code" distilled: very clear and expressive with no unnecessary noise. You can, of course, make list comprehensions "ugly" but at least you have to try a bit to do so.

List comprehensions have their roots in functional programming and several modern functional languages include support for them. We'll see comprehensions in Haskell, Clojure and Scala. [2] Python also includes support for comprehensions; in fact the basis for [Guido's argument](http://www.artima.com/weblogs/viewpost.jsp?thread=98196) to remove the map and filter functions from py3k was that expressions using these functions could be easily re-written as comprehensions. We also consider comprehensions in Python, including differences between the Python implementation and those in other languages and what effect those differences may have.

Let's consider a fairly straightforward problem problem: given a list of (x,y) coordinates in some two-dimensional space provide a list of the coordinates that are within some fixed distance of the origin of (0,0). Our application also requires that results be displayed in [polar coordinates](http://en.wikipedia.org/wiki/Polar_coordinates), so for extra bonus points we should return our results in that notation. We can solve the problem quite easily in Haskell and Clojure using list comprehensions:

{% codeblock lang:haskell %}
toDegrees :: Floating a => a -> a
toDegrees rad = rad * 180 / pi

euclidean :: Floating a => a -> a -> a
euclidean x y = let foo = x ^^ 2
	                bar = y ^^ 2 
			    in sqrt (foo + bar)

main = do
     print [(euclidean x y,(toDegrees . atan) (y / x))|(x,y)<-[(1,2),(2,3),(3,4)],(euclidean x y) > 3.0]
{% endcodeblock %}

{% codeblock lang:clojure %}
(defn euclidean [x y] (Math/sqrt (+ (Math/pow x 2) (Math/pow y 2))))
(println (for [
	  [x,y] 
	   [[1,2],[2,3],[3,4]] 
	    :when (> (euclidean x y) 3.0)] 
	     [(euclidean x y),(Math/toDegrees (Math/atan (/ y x)))]))
{% endcodeblock %}

In Scala we use for expressions to accomplish something very similar:

{% codeblock lang:scala %}
import scala.math.{atan,pow,sqrt,toDegrees}

val result =
  for {
    (x,y) <- List((1,2),(2,3),(3,4))
    if sqrt((pow(x,2)) + (pow(y,2))) > 3.0 }
  yield (sqrt((pow(x,2)) + (pow(y,2))),toDegrees(atan(y.toFloat/x)))
println(result)
{% endcodeblock %}

And finally, a straightforward solution in Python:

{% codeblock lang:python %}
from math import sqrt, atan, degrees
data=[(1,2),(2,3),(3,4)]
print [(sqrt(x**2 + y**2),degrees(atan(float(y)/x))) for (x,y) in data if sqrt(x**2 + y**2) > 3.0]
{% endcodeblock %}

Note that in each of these solutions we're doing a bit more work than we need to. Our implementation computes the distance from the origin twice, once when filtering values and again when generating the final output of the transformation process. This seems unnecessary; a better option would be to somehow define this value as intermediate state. This state could then be available to both filter and transform expressions. Haskell and Clojure support the introduction of intermediate bindings in their comprehension syntax:

{% codeblock lang:haskell %}
toDegrees :: Floating a => a -> a
toDegrees rad = rad * 180 / pi

euclidean :: Floating a => a -> a -> a
euclidean x y = let foo = x ^^ 2
	                bar = y ^^ 2 
			    in sqrt (foo + bar)

main = do
     print [(dist,(toDegrees . atan) (y / x))|(x,y)<-[(1,2),(2,3),(3,4)],let dist=euclidean x y,dist > 3.0]
{% endcodeblock %}

{% codeblock lang:clojure %}
(defn euclidean [x y] (Math/sqrt (+ (Math/pow x 2) (Math/pow y 2))))

(println (for [
	  [x,y] 
	   [[1,2],[2,3],[3,4]] 
	    :let [dist (euclidean x y)] 
	     :when (> dist 3.0)]
	      [dist,(Math/toDegrees (Math/atan (/ y x)))]))
{% endcodeblock %}

Scala also allows for intermediate bindings within a for expression:

{% codeblock lang:scala %}
import scala.math.{atan,pow,sqrt,toDegrees}

val result =
  for {
    (x,y) <- List((1,2),(2,3),(3,4))
    dist = sqrt((pow(x,2)) + (pow(y,2)))
    if dist > 3.0 }
  yield (dist,toDegrees(atan(y.toFloat/x)))
println(result)
{% endcodeblock %}

What about Python? It turns out we cannot solve this problem in Python using a single comprehension; the syntax doesn't allow for the introduction of intermediate state which can be used in either the predicate or transform expression. On the face of it this seems a bit odd; the language encourages the use of comprehensions for filtering and/or transformation while providing a less robust version of that very construct. To some degree this discrepancy reflects differing language goals. Guido's post on the history of list comprehensions seems to indicate that the motivation for adding these features was pragmatic; the syntax is an elegant way to express most filter and transform operations. Functional languages use list comprehensions as "syntactic sugar" for monadic effects [3] that don't really have an equivalent in standard Python usage. The syntax may look the same, but if you're coming from a functional perspective they can feel just a bit off. The same is true for a few other common functional idioms:

* Lazy evaluation - List comprehensions in Python are not lazily evaluated. Generator expressions, which look very similar to list comprehensions, are lazily evaluated.
* Higher-order functions - Anonymous functions are supported in Python but these functions are famously limited to a single expression. Functions can return functions but for non-trivial functions a named function must be declared and returned.

A couple things should be noted here. First, let us clearly state that Python is not and does not claim to be a functional programming language. While absolutely true, this fact doesn't change the underlying point. Moving from functional concepts back into Python can be a bit jarring; some things look similar but don't behave quite like you'd expect.

It's also worth noting that the inability to solve this problem with list comprehensions in Python doesn't mean that this problem cannot be solved in idiomatic Python. We wish to return our intermediate state as well as filter results based on it's value; this dual use allows us to solve the problem with nested comprehensions. The inner comprehension will generate the final representation (including the intermediate state) and the outer comprehension will filter results based on that representation. In Python this looks something like:

{% codeblock lang:python %}
from math import sqrt, atan, degrees
data=[(1,2),(2,3),(3,4)]
print [(dist,degrees) for (dist,degrees) in [(sqrt(x**2 + y**2),degrees(atan(float(y)/x))) for (x,y) in data] if dist > 3.0]
{% endcodeblock %}

This works only because the intermediate state is also returned in the final result. If that state were not explicitly returned (i.e. if it's values were used as input to a conditional expression which returned, say, a string value describing the distance) this solution would not apply.

We can also solve this problem using generators. Using the state maintained by the generator we can iterate through the list, compute the intermediate state and yield a value only when we've satisfied our predicate. A generator-based solution would look something like:

{% codeblock lang:python %}
from math import sqrt, atan, degrees
data=[(1,2),(2,3),(3,4)]
def somefunc(alist):
    for (x,y) in alist:
    dist = sqrt(x**2 + y**2)
    if (dist > 3.0):
            yield (dist,degrees(atan(float(y)/x)))
print list(somefunc(data))
{% endcodeblock %}

Finally, none of these comments should be construed as a criticism of Python, the design choices that went into the language or the inclusion of list comprehensions generally. The pragmatic case for inclusion of this feature seems very strong. This post is interested only in the interplay between these features and similar features in other languages.

[1] Some languages (perhaps most notably Scala) use the term "for comprehension" or "for expression", and in some of these languages (Scala again) these constructs are more flexible than a list comprehension. That said, it's fairly straightforward to make Scala's for expressions behave like conventional list comprehensions.

[2] A purist might object that Scala is designed to mix features of object-oriented and functional languages, but the bias in favor of functional constructs justifies Scala's inclusion here.

[3] As an example, note that in Haskell list comprehensions can be replaced with do-notation. See the Haskell wiki for details.

