---
layout: post
title: "Concurrent Prime Factorization in Go"
date: 2010-10-22 12:00
comments: true
categories: [golang concurrency]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2010/10/concurrent-prime-factorization-in-go.html)

For a little while now I've been looking for a problem that would allow me to sink my teeth into [Google's Go language](http://golang.org) and see if it's as tasty as it appears to be. After coming across Alex Tkachman's [request for concurrency benchmarks](https://profiles.google.com/113183999235628738153/buzz/J8oSTjegB5F) it was clear that I had what I needed. Alex is the driving force behind Groovy++ and his proposal was to implement a common benchmark (a concurrent version of prime factorization using the Sieve of Eratosthenes) in multiple JVM languages in order to compare and contrast what works and what doesn't. An implementation in Go wouldn't help much in benchmarking JVM languages but the highly concurrent nature of the problem does map naturally onto Go's feature set. The language distribution also includes an elegant implementation of the Sieve of Eratosthenes in it's tests so we already have a solid foundation to build from.

Let's begin with sieve.go. The implementation consists of chained goroutines. Each time a candidate makes it through the sieve and is returned as a new prime number a new goroutine is created to check future candidates and reject them if they divide evenly by the new prime. The implementation cleanly demonstrates the concept but isn't useful as a standalone application since it includes no termination condition. Let's begin by adding that in:

{% codeblock lang:go %}
// $G $F.go && $L $F.$A  # don't run it - goes forever

// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package main

import "flag"

// Send the sequence 2, 3, 4, ... to channel 'ch'.
func Generate(max int, ch chan<- int) {

     // Slight optimization; after 2 we know there are no even primes so we only
     // need to consider odd values
     ch <- 2
     for i := 3; i<=max ; i += 2 {
     	 ch <- i // Send 'i' to channel 'ch'.
     }
     ch <- -1 // Use -1 as an indicator that we're done now
}

// Copy the values from channel 'in' to channel 'out',
// removing those divisible by 'prime'.
func Filter(in <-chan int, out chan<- int, prime int) {
     for i := <- in; i != -1; i = <- in {

     	 if i % prime != 0 {
	      out <- i // Send 'i' to channel 'out'.
	 }
     }
     out <- -1
}

// The prime sieve: Daisy-chain Filter processes together.
func Sieve(max int) {
     ch := make(chan int) // Create a new channel.
     go Generate(max,ch)      // Start Generate() as a subprocess.
     for prime := <-ch; prime != -1; prime = <-ch {
     	 print(prime, "\n")
	 ch1 := make(chan int)
	 go Filter(ch, ch1, prime)
	 ch = ch1
     }
}

func main() {

     var max *int = flag.Int("max",1,"Maximum number we wish to return")
     flag.Parse()
     Sieve(*max)
}
{% endcodeblock %}

    [varese factorization_benchmark]$ cd go
    [varese go]$ ls
    factor.go  sieve.go
    [varese go]$ gobuild sieve.go
    Parsing go file(s)...
    Compiling main (sieve)...
    Linking sieve...
    [varese go]$ ./sieve -max 20
    2
    3
    5
    7
    11
    13
    17
    19

That seemed to work nicely. Adding in the factorization code leads us to factor.go:

{% codeblock lang:go %}
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Prime factorization derived from slightly modified version of
// sieve.go in Go source distribution.

package main

import "fmt"
import "flag"
import vector "container/vector"

// Send the sequence 2, 3, 4, ... to channel 'ch'.
func Generate(max int, ch chan<- int) {

     // Slight optimization; after 2 we know there are no even primes so we only
     // need to consider odd values
     ch <- 2
     for i := 3; i<=max ; i += 2 {
     	 ch <- i // Send 'i' to channel 'ch'.
     }
     ch <- -1 // Use -1 as an indicator that we're done now
}

// Copy the values from channel 'in' to channel 'out',
// removing those divisible by 'prime'.
func Filter(in <-chan int, out chan<- int, prime int) {
     for i := <- in; i != -1; i = <- in {

     	 if i % prime != 0 {
	      out <- i // Send 'i' to channel 'out'.
	 }
     }
     out <- -1
}

func main() {

     var target *int = flag.Int("target",1,"Number we wish to factor")
     flag.Parse()

     t := *target
     var rv vector.IntVector

     // Retrieve a prime value and see if we can divide the target evenly by
     // that prime.  If so perform the multiplication and update the current
     // value.
     ch := make(chan int) // Create a new channel.
     go Generate(t,ch)      // Start Generate() as a subprocess.
     for prime := <-ch; (prime != -1) && (t > 1); prime = <-ch {

     	for ;t % prime == 0; {
	     	t = t / prime
		rv.Push(prime)
	}

	// Create a goroutine for each prime number whether we use it or
	// not.  This performs the daisy chaining setup that was being
	// done by the Sieve() function in sieve.go.
        ch1 := make(chan int)
        go Filter(ch, ch1, prime)
        ch = ch1
     }

     fmt.Printf("Results: %s\n",fmt.Sprint(rv))
}
{% endcodeblock %}

Simple enough. This leaves us with something that looks fairly robust:

    [varese go]$ gobuild factor.go 
    Parsing go file(s)...
    Compiling main (factor)...
    Linking factor...
    [varese go]$ ./factor -target 60
    Results: [2 2 3 5]

Examples here employ [gobuild](http://code.google.com/p/gobuild). This tool has proven to be very useful when building and testing Go applications. Complete code for both examples can be found on [github](https://github.com/heuristicfencepost/factorization_benchmark). There is some interest in comparing a solution based on goroutines and channels to one based on actors and messages so in the future some Scala code might join the existing Go code.