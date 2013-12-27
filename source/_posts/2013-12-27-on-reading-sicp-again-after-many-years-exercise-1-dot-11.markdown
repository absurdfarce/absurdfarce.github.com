---
layout: post
title: "On Reading SICP Again After Many Years: Exercise 1.11"
date: 2013-12-27 15:23
comments: true
categories: [sicp scheme clojure haskell]
---
The last time I had occasion to work my way through [SICP](http://en.wikipedia.org/wiki/Structure_and_Interpretation_of_Computer_Programs) I found myself reading the first edition... because that's all there was.  So when some of my fellow developers at [my day job](http://www.code42.com) started up a technical book club and led off with the second edition of SICP I was quite intrigued.  This post reflects the first of (hopefully) several observations on re-reading a classic.

At a recent meetup for this book club one of the exercises in the book gathered a fair share of attention.  That exercise, 1.11, can be found in the [HTML version of the second edition](http://mitpress.mit.edu/sicp/full-text/book/book.html) and reads as follows:

    Exercise 1.11.  A function f is defined by the rule that f(n) = n if n<3 and 
    f(n) = f(n - 1) + 2f(n - 2) + 3f(n - 3) if n> 3.   Write a procedure that 
    computes f by means of a recursive process. Write a procedure that computes 
    f by means of an iterative process.

We consider this problem from the perspective of Haskell, Scheme and Clojure.  The sample code below was known to run correctly on GHC 6.12.1, Chicken Scheme 4.5.0 and Clojure 1.5.1 via Leiningen [1].

A Haskell version of the recursive process is almost a literal transcription of the problem statement:

{% codeblock lang:haskell %}
recurFn :: Int -> Int
recurFn x 
	| x < 3 = x
	| otherwise = a + (2 * b) + (3 * c)
	where a = recurFn (x - 1)
	      b = recurFn (x - 2)
	      c = recurFn (x - 3)
{% endcodeblock %}

This translates into some fairly straightforward Scheme:

{% codeblock lang:scheme %}
(define recur-fn
  (lambda (x)
    ;(printf "recur-fn, x=~s\n" x)
    (if (< x 3)
	x
	(+ (recur-fn (- x 1))
	   (* 2 (recur-fn (- x 2)))
	   (* 3 (recur-fn (- x 3)))))))
{% endcodeblock %}

The problem is that we've also got a straightforward instance of tree recursion here.  To highlight this fact uncomment the printf statement in the above code and compute the value of (recur-fn 4):

    @stravinsky:~/git/sicp/1-11/scheme$ grep printf 1-11.scm 
        (printf "recur-fn, x=~s\n" x)
    (printf "recur-fn for 4: ~s\n" (recur-fn 4))
    @stravinsky:~/git/sicp/1-11/scheme$ csc 1-11.scm 
    @stravinsky:~/git/sicp/1-11/scheme$ ./1-11 
    recur-fn for 4: recur-fn, x=4
    recur-fn, x=3
    recur-fn, x=2
    recur-fn, x=1
    recur-fn, x=0
    recur-fn, x=2
    recur-fn, x=1
    11

A moment of reflection should make it clear that this is exactly the behaviour you'd expect to see from tree recursion.

So what might an iterative solution look like?  The base case of our recursive definition is f(0), f(1) and f(2) and larger values of f(x) build on smaller values so it seems fairly obvious that we want an ascending iteration from the values that make up our base case.  At each iteration we might add the newly-computed value to a list of observed values, returning the last value from this list once we've iterated up to our input value [2].  Another moment of reflection allows us to see that what we're describing is a *fold* which produces a list as it's value.  Fold (sometimes called *reduce* or, in [really odd languages](http://groovy.codehaus.org), *inject*... for no obvious reason [3]) is a higher-order function, a topic which will be covered in the next chapter.

One might object that a list of values, one for each index in our iteration, sounds a lot more like a *map* operation than a fold/reduce.  And you'd be correct... so long as you don't consider the inputs.  A map operation is allowed a single input, in this case the index of an item in the list, and that input isn't sufficient to compute values of f(x) for x >= 3; the history we need to consider is simply lost.  But a fold/reduce operation introduces the notion of an accumulator (of some arbitrary type) which is also made available to our computation as a parameter.  So long as that accumulator has access to the history of our computation our function has all the inputs it needs.  And if our accumulator is nothing more than a repository of previously-computed values (in the form of, say, a list) we have exactly that history.

The key here is the realization that fold/reduce is isomorphic onto a recursive procedure which is also an iterative process:

{% codeblock lang:scheme %}
(define my-fold
  (lambda (fn acc idxs)
    (if (= 0 (length idxs))
        acc
        (my-fold fn (fn (car idxs) acc) (cdr idxs)))))
{% endcodeblock %}

So we crank out an implementation based on fold, using a few of the methods from [SRFI-1](http://srfi.schemers.org/srfi-1/srfi-1.html) to simplify things just a little bit:

{% codeblock lang:scheme %}
(use srfi-1)

; Use some of the list manipulation methods from SRFI-1 to make our lives easier                                                                                                 (define iter-fn
  (lambda (x)
    (define compute-value
      (lambda (prevs)
        (letrec ((prevs-last (take-right prevs 3))
                 (c (car prevs-last))
                 (b (cadr prevs-last))
                 (a (caddr prevs-last)))
        (+ a (* 2 b) (* 3 c)))))
    (if (< x 3)
        x
        (letrec ((range (iota (- x 2) 3))
                 (fold-fn (lambda (x prevs) (append prevs (list (compute-value prevs)))))
                 (vals (fold fold-fn '(0 1 2) range)))
          (last vals)))))
{% endcodeblock %}

For kicks we do a quick translation of the iterative process into Clojure, and don't tell the Scheme people but there's something to this "syntactic sugar" stuff.  The use of destructuring here makes the function much more concise:

{% codeblock lang:clojure %}
(defn iter-fn [x]
  (if (< x 3)
    x
    (let [compute-fn (fn [prevs]
                       (let [[c b a] (take-last 3 prevs)]
                         (+ a (* 2 b) (* 3 c))))
          reduce-fn (fn [col val] (conj col (compute-fn col)))
          vals (reduce reduce-fn [0 1 2] (range 3 (+ x 1)))]
            (last vals))))
{% endcodeblock %}

Unfortunately the same can't be said of the recursive process.  Clojure can't do much to save us here.  Be forewarned; the code that follows is intended for mature audiences only.  The multiple instances of recursion outside of the loop/recur forms is very likely to induce eye bleeding in any seasoned Clojure programmer:

{% codeblock lang:clojure %}
; Recursive version... with sufficient eye bleeding
(defn recur-fn [x]
    (if (< x 3)
	x
	(+ (recur-fn (- x 1))
	   (* 2 (recur-fn (- x 2)))
	   (* 3 (recur-fn (- x 3))))))
{% endcodeblock %}

[1] I'm finally starting to come around when it comes to Leiningen.  Language-specific builders still irritate me, but at least this one seems sensible.

[2] We really only need to preserve three values in our list at any given time in order to maintain enough state to define the computation at every point.  We could move to something like this in the future if necessary, but doing it now doesn't do anything to enhance the logic of our solution.

[3] Ruby gets a pass here (but just barely) since it offers both inject and reduce.  That said, I expect better from you, Ruby!