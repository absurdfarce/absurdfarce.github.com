---
layout: post
title: "A Funny Thing Happened While Writing Some Haskell...."
date: 2010-12-20 12:00
comments: true
categories: [haskell scheme]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2010/12/funny-thing-happened-while-writing-some.html)

Suppose that you've just finished Graham Hutton's solid [introduction](http://www.cs.nott.ac.uk/~gmh/book.html) to Haskell and now want to take the language for a quick test drive. You decide to stick with something familiar; after noting that the Prelude doesn't have an equivalent to Python's [range](http://docs.python.org/2/library/functions.html#range) function you decide to whip one up yourself. Haskell's syntax is still a bit unfamiliar, however, so you decide to implement this function in a different "mostly functional" language that you have some experience with. You've been looking for a reason to dip your toes into Scheme again (and it seems to be a good fit here [1]) so you install [Gambit Scheme](http://dynamo.iro.umontreal.ca/wiki/index.php/Main_Page) and [Chicken Scheme](http://www.call-cc.org) and dig in.

Perhaps you begin with a straightforward recursive definition in Scheme:

{% codeblock lang:scheme %}
(define (range start stop step)
 (if (< start stop)
  (cons start (range (+ start step) stop step))
  '()))
{% endcodeblock %}

This works well enough when start < stop but fails completely when start > stop and step is a negative value (a use case supported by Python's range()). To fix this problem we define two procedures via letrec and use whichever is appropriate based on the passed arguments:

{% codeblock lang:scheme %}
(define (newrange start stop step)
  (letrec 
   ((upto (lambda (art op ep) 
     (if (< art op)
      (cons art (upto (+ art ep) op ep)) 
      '())))
    (downto (lambda (art op ep)
     (if (> art op)
      (cons art (downto (+ art ep) op ep)) 
      '()))))
 (if (< start stop)
  (upto start stop step)
  (downto start stop step))))
{% endcodeblock %}

This function now works for both start < stop (with positive step) and start > stop (with negative step). There's still a problem, though; if we change the sign of step in either of these cases we find ourselves in an infinite loop. Python's range() returns an empty list in this case so we probably should too:

{% codeblock lang:scheme %}
(define (newerrange start stop step)
 (letrec
  ((upto (lambda (art op ep)
    (if (< art op)
     (cons art (upto (+ art ep) op ep)) 
     '())))
   (downto (lambda (art op ep)
    (if (> art op)
     (cons art (downto (+ art ep) op ep)) 
     '()))))
    (cond ((and (< start stop) (< step 0)) '())
          ((and (> start stop) (> step 0)) '())
          ((< start stop) (upto start stop step))
          (else (downto start stop step)))))
{% endcodeblock %}

While working on these implementations you discover that SRFI-1 includes a function iota which looks similar to Python's range(), although this function takes the number of elements in the return list rather than the start/stop/step values we're looking for. Still, both Chicken Scheme and Gambit Scheme support SFRI-1 as extensions or modules and we should be able to whip up a thin wrapper around iota to get what we need. Of course we first need to figure out how to load up these modules. And after some initial experimentation we see some curious behaviour in the iota implementation on both platforms...

But wait... weren't we supposed to be writing some Haskell code?

Oh, yeah.

Haskell's guarded equations allow for an easy expression of the same logic we were getting from cons in Scheme. We have no need to translate the recursive logic found in the Scheme version, however, since the list-related functions in the Prelude combined with a few sections gives us everything we need:

{% codeblock lang:haskell %}
range art op ep | art < op && ep < 0 = []
                | art > op && ep > 0 = []
                | art < op = takeWhile (<op) (iterate (+ep) art)
                | art > op = takeWhile (>op) (iterate (+ep) art)
{% endcodeblock %}

In all honesty this is a fairly elegant expression of the algorithm. And while there's clearly a lot to like in Haskell this exercise has rekindled a long-dormant romance with Scheme rather than encouraging exploration of something new.

[1] Yes, I realize Scheme isn't "purely functional". I'm not interested in taking a stand in this particular holy war, but there's little doubt that Scheme's bias against set! and it's kin justify the "mostly functional" qualifier.
