---
layout: post
title: "Ruby, Perl and Eloquence"
date: 2012-07-10 12:00
comments: true
categories: [ruby]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2012/07/ruby-perl-and-eloquence.html)

In an attempt to make my Ruby code a bit more idiomatic I've been spending a bit of time recently with Russ Olsen's excellent [Eloquent Ruby](http://eloquentruby.com). There are many reasons to love writing Ruby code, not least of which is that Ruby deploys the same terse but expressive power of Perl while employing better overall principles of programming. The effect isn't universal; on occasion my problems with Ruby look quite a bit like my problems with Perl. Given the overall elegance of the language it seems likely that there's a "better" (or at least more idiomatic) way to accomplish my goal. And so I turn to *Eloquent Ruby*. 

As an example of this tension consider the following example. 

Perl has a well-deserved reputation for efficiently processing text files with regular expressions. We'll consider an example from another text I've been spending a bit of time with: Hofstadter's seminal [Godel, Escher, Bach](http://en.wikipedia.org/wiki/G%C3%B6del,_Escher,_Bach). A simple implementation of the productions of the MIU system in Perl [1] might look like the following: 

{% codeblock lang:perl %}
# Simple implementation of the productions for the MIU system
open INFILE,"<",$ARGV[0];
while(<INFILE>) {

    chomp;
    next if $_ =~ /#.*/;
    print "$1IU\n" if $_ =~ /^(.+?)I$/;
    print "M$1$1\n" if $_ =~ /^M(.+)$/;
    print "$1U$2\n" if $_ =~ /^(.*)III(.*)$/;
    print "$1$2\n" if $_ =~ /^(.*)UU(.*)$/;
}
{% endcodeblock %}

Reasonable enough, but there's a lot of magic going on here. We're relying on the "magic" variable $_ to access the current line, and to make things worse we have to obtain those lines using the INFILE identifier that only has meaning due to a side effect of the open() call [2]. There's also those "magic" $1 and $2 variables for accessing capture groups in a regex. 

The Ruby version is both slightly shorter and a bit cleaner: 

{% codeblock lang:ruby %}
# Simple implementation of the productions for the MIU system
File.new(ARGV[0]).readlines.each do |line |

  next if line =~ /#.*/
  puts "#{$1}IU\n" if line =~ /^(.+)I$/
  puts "M#{$1}#{$1}\n" if line =~ /^M(.+)$/
  puts "#{$1}U#{$2}\n" if line =~ /^(.*)III(.*)$/
  puts "#{$1}#{$2}\n" if line =~ /^(.*)UU(.*)$/
end
{% endcodeblock %}

We've made some nice strides here. The use of File.new() allows us to avoid open() and it's side effects. The use of a code block allows us to remove the global $_ in favor of a scoped variable line. 

But we're still stuck with $1 and $2 for those capture groups. 

One can imagine an elegant object-oriented solution based on match objects. Any such implementation would have to accomplish three things:

* The match object will be used as the condition of an if/unless expression so nil should be returned if there's no match
* The match object should be bound to a variable name in scope
* References to capture groups in the if-clause should use the scoped variable rather than the $1,$2, etc.

But remember, this exercise is only useful if we don't have to compromise on elegance. If all we're after is an explicit object-oriented solution we could go with the Python version: 

{% codeblock lang:python %}
# Simple implementation of the productions for the MIU system
from sys import argv
import re

re_c = re.compile("#.*")
re1 = re.compile("^(.+?)I$")
re2 = re.compile("^M(.+)$")
re3 = re.compile("^(.*)III(.*)$")
re4 = re.compile("^(.*)UU(.*)$")

for line in [line.strip() for line in open(argv[1])]:
    if re_c.match(line):
        continue
    m1 = re1.match(line)
    if m1:
        print m1.group(1) + "IU"
    m2 = re2.match(line)
    if m2:
        print "M" + m2.group(1) + m2.group(1)
    m3 = re3.match(line)
    if m3:
        print m3.group(1) + "U" + m3.group(2)
    m4 = re4.match(line)
    if m4:
        print m4.group(1) + m4.group(2)
{% endcodeblock %}

That's probably not what we want. [3] 

After pondering this question for a bit we realize we may not be in such a bad spot. /regex/.match(str) already returns nil if there is no match so our first requirement is satisfied. Assignment is just another expression, so our match object (or nil) will be returned to the if-expression test, helping us with our second goal. And match objects provide access to capture groups using []. So long as the assigned variable is in scope we should have everything we need. A bit of scuffling [4] brings us to the following: 

{% codeblock lang:ruby %}
# Implementation of the productions for the MIU system with match objects
File.new(ARGV[0]).readlines.each do |line |

  next if line =~ /#.*/

  foo = nil
  puts "#{foo[1]}IU\n" if foo = /^(.+)I$/.match(line)
  puts "M#{foo[1]}#{foo[1]}\n" if foo = /^M(.+)$/.match(line)
  puts "#{foo[1]}U#{foo[2]}\n" if foo = /^(.*)III(.*)$/.match(line)
  puts "#{foo[1]}#{foo[2]}\n" if foo = /^(.*)UU(.*)$/.match(line)
end
{% endcodeblock %}

This example is free of any "magic" variables, although we have sacrificed a bit on the clarity front. It's also worth noting that we could have accomplished something very similar in Perl: 

{% codeblock lang:perl %}
# Implementation of the productions for the MIU system with match objects
open INFILE,"<",$ARGV[0];
while(<INFILE>) {

    chomp;
    next if $_ =~ /#.*/;
    print "$foo[0]IU\n" if (@foo = ($_ =~ /^(.+?)I$/));
    print "M$foo[0]$foo[0]\n" if (@foo = ($_ =~ /^M(.+)$/));
    print "$foo[0]U$foo[1]\n" if (@foo = ($_ =~ /^(.*)III(.*)$/));
    print "$foo[0]$foo[1]\n" if (@foo = ($_ =~ /^(.*)UU(.*)$/));
}
{% endcodeblock %}

This implementation is hardly idiomatic. It's also quite a bit less clear than our earlier efforts in the language. 

Where does this leave us? Do we keep in touch with our Perl roots and live with $1 in order to keep things terse and expressive? Do we sacrifice a bit of clarity and go with an object-oriented approach? Or do we do something else entirely? 

Answers to this question (and others like it) are what I'm hoping to get out of *Eloquent Ruby*. 

[1] We're ignoring nasty things like error handling and complex edge cases in order to keep the conversation focused. 

[2] We could use lexical file handles here but that doesn't really change the underlying point. Even in that case we still have to call open() in order for $fh to be useful. 

[3] Python does a lot of things very, very well, but this solution to this problem seems unnecessarily verbose. 

[4] The requirement to declare foo in advance when using the modifier form of if was a bit surprising. Shifting to an if expression removed this requirement; in that case assignment in the condition. The upcoming Perl version also didn't require this advance declaration when using an equivalent to the modifier form. An MRI quirk, perhaps?
