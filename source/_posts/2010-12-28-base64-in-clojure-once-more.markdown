---
layout: post
title: "base64 in Clojure: Once More, With Feeling"
date: 2010-12-28 12:00
comments: true
categories: [clojure base64]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2010/12/base64-in-clojure-once-more-with.html)

In a [previous post](http://absurdfarce.github.io/blog/2010/10/31/base64-in-clojure-an-initial-attempt) we sketched out a simple base64 implementation in Clojure and measured it's performance against clojure-contrib and commons-codec. Our initial implementation used the [partition](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/partition) function to split the input into blocks and then applied a sequence of interdependent functions to translate each block into base64-encoded data. This approach worked well enough but it's not terribly efficient; it traverses the input data once to do the partitioning and then that same data is traversed *again* (in block form) when the encoding function is applied. There's no reason to make multiple passes if we attack the problem recursively rather than iteratively. Time for some further investigation.

{% codeblock lang:clojure %}
(ns fencepost.base64-recur)

; This could be placed within the defn below but rebuilding this map
; on each invocation seems a bit wasteful.
(def to-base64-recur
     (zipmap 
      (range 0 64)
      "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/")
     )

(def from-base64-recur
     (zipmap 
      "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
      (range 0 64)
      )
     )

(defn base64-encode-recur [#^String arg]
  "Encode the input string into the base64 alphabet.  Currently uses the platform charset for the conversion
   of the input string to bytes; this could be generalized if need be."
  (loop [bytes (vec (.getBytes arg))
         rv ""]
    (let [samplesize (count bytes)]
      (cond
       (> samplesize 3)
       (let [[byte1 byte2 byte3] (take 3 bytes)
             char1 (bit-shift-right byte1 2)
             char2 (bit-and 0x3F (bit-or (bit-shift-left byte1 4) (bit-shift-right byte2 4)))
             char3 (bit-and 0x3F (bit-or (bit-shift-left byte2 2) (bit-shift-right byte3 6)))
             char4 (bit-and 0x3F byte3)]
         (recur (subvec bytes 3) (str rv (apply str (map to-base64-recur (list char1 char2 char3 char4)))))
         )
       (= samplesize 3)
       (let [[byte1 byte2 byte3] (take 3 bytes)
             char1 (bit-shift-right byte1 2)
             char2 (bit-and 0x3F (bit-or (bit-shift-left byte1 4) (bit-shift-right byte2 4)))
             char3 (bit-and 0x3F (bit-or (bit-shift-left byte2 2) (bit-shift-right byte3 6)))
             char4 (bit-and 0x3F byte3)]
         (str rv (apply str (map to-base64-recur (list char1 char2 char3 char4))))
         )
       (= samplesize 2)
       (let [[byte1 byte2] (take 2 bytes)
             char1 (bit-shift-right byte1 2)
             char2 (bit-and 0x3F (bit-or (bit-shift-left byte1 4) (bit-shift-right byte2 4)))
             char3 (bit-and 0x3F (bit-shift-left byte2 2))]
         (str rv (apply str (map to-base64-recur (list char1 char2 char3))) "=")
         )
       (= samplesize 1)
       (let [byte1 (get bytes 0)
             char1 (bit-shift-right byte1 2)
             char2 (bit-and 0x3F (bit-shift-left byte1 4))]
         (str rv (apply str (map to-base64-recur (list char1 char2))) "==")
         )
       (= samplesize 0) ""
       )
      )
    )
  )

(defn base64-decode-recur [#^String arg]
  "Decode the string of base64-encoded content into a String"
  (loop [currstr arg
         rv ""]
    (let [currstrsize (count currstr)]
      (cond
       ; String longer than 4 characters
       (> currstrsize 4)
       (let [[char1 char2 char3 char4] (map from-base64-recur (take 4 currstr)) 
             byte1 (bit-or (bit-shift-left char1 2) (bit-shift-right char2 4))
             byte2 (bit-and 0xff (bit-or (bit-shift-left char2 4) (bit-shift-right char3 2)))
             byte3 (bit-and 0xff (bit-or (bit-shift-left char3 6) char4))
             newstr (String. (byte-array (map byte [byte1 byte2 byte3])))]
         (recur (.substring currstr 4) (str rv newstr))
         )
       (= currstrsize 0) ""
       (< currstrsize 4)
       (throw (IllegalArgumentException. "Malformed base64 string; must have even multiple of four characters"))
       (identity 1)
       (let [curratom (apply str (take 4 currstr))]
         (cond
          ; Third byte is the bottom two bits of byte 2 plus all six low
          ; bits of byte 3
          (re-matches #"[A-Za-z0-9+/]{4}" curratom)
          (let [[char1 char2 char3 char4] (map from-base64-recur (take 4 currstr))
                byte1 (bit-or (bit-shift-left char1 2) (bit-shift-right char2 4))
                byte2 (bit-and 0xff (bit-or (bit-shift-left char2 4) (bit-shift-right char3 2)))
                byte3 (bit-and 0xff (bit-or (bit-shift-left char3 6) char4))
                newstr (String. (byte-array (map byte [byte1 byte2 byte3])))]
            (str rv newstr)
            )
          ; Second byte is the low four bits of byte 2 plus the top four
          ; bits of byte 3
          (re-matches #"[A-Za-z0-9+/]{3}=" curratom)
          (let [[char1 char2 char3] (map from-base64-recur (take 3 currstr))
                byte1 (bit-or (bit-shift-left char1 2) (bit-shift-right char2 4))
                byte2 (bit-and 0xff (bit-or (bit-shift-left char2 4) (bit-shift-right char3 2)))
                newstr (String. (byte-array (map byte [byte1 byte2])))]
            (str rv newstr)
            )
          ; First byte is the low six bits of byte 1 plus the top two
          ; bits of byte 2
          (re-matches #"[A-Za-z0-9+/]{2}==" curratom)
          (let [[char1 char2] (map from-base64-recur (take 2 currstr))
                byte1 (bit-or (bit-shift-left char1 2) (bit-shift-right char2 4))
                newstr (String. (byte-array [(byte byte1)]))]
            (str rv newstr)
            )
          (identity 1)
          (throw (IllegalArgumentException. "Malformed base64 string"))
          )
         )
       )
      )
    )
  )
{% endcodeblock %}

Clojure optimizes recursive calls when the loop/recur constructs are used so we deploy them here. In addition we make significant use of destructuring. Each input chunk of data is routed to the appropriate function via a sequence of cond expressions. These functions are implemented as let statements which break the chunk into bytes (via destructuring) and then create bindings for the base64 characters representing these bytes by applying the relevant bit manipulations. The body of the let doesn't need to do much beyond combining these character bindings into an appropriate return value.

This refactoring offers the following benefits:

* The code is a lot cleaner and *much* easier to read and understand
* Using the same techniques we should be able to easily implement a decoding function
* We should see some increase in performance

While we're at it, let's update our comparison application to include the new implementation:

{% codeblock lang:clojure %}
(ns fencepost.test)

(import '(org.apache.commons.codec.binary Base64))
(import '(org.apache.commons.lang RandomStringUtils))
(use '[clojure.contrib.base64])
(use '[fencepost.base64_letfn])
(use '[fencepost.base64-recur])

(def sample-size 100)
(def max-string-size 256)

; Build up some sample data using commons-lang.  Sample data is built
; before any tests are run; this allows us to apply uniform test data
; to each implementation and to avoid corrupting timings with the
; generation of random data.
(def sample-data (map #(RandomStringUtils/randomAscii (* % (rand-int max-string-size))) (repeat sample-size 1)))

; Instantiate a Base64 instance from commons-codec
(def codec-base64 (new Base64 -1))

(println "Encoding")

; Time each run.  Mapping a function onto our sample data produces a
; lazy sequence so we have to take the additional step of realizing
; the sequence; thus the conversion to a vector.
(println "Commons-codec")
(def commons-codec-data (time (doall (map #(String. (.encode codec-base64 (.getBytes %))) sample-data))))
(println (first commons-codec-data))

(println "clojure-contrib")
(def clojure-contrib-data (time (doall (map #(clojure.contrib.base64/encode-str %) sample-data))))
(println (first clojure-contrib-data))

; Simple sanity check; take commons-codec output as canonical and
; verify that what our implementations generate match up with the
; canonical sample.
(assert (.equals (first commons-codec-data) (first clojure-contrib-data)))

(println "fencepost/base64_letfn")
(def fencepost-data (time (doall (map #(fencepost.base64_letfn/base64-encode-ascii %) sample-data))))
(println (first fencepost-data))
(assert (.equals (first commons-codec-data) (first fencepost-data)))

(println "fencepost-recur")
(def fencepost-recur-data (time (doall (map #(fencepost.base64-recur/base64-encode-recur %) sample-data))))
(println (first fencepost-recur-data))
(assert (.equals (first commons-codec-data) (first fencepost-recur-data)))

(println "Decoding")

(println "Commons-codec")
(def commons-codec-data2 (time (doall (map #(String. (.decode codec-base64 (.getBytes %))) commons-codec-data))))
(println (first commons-codec-data2))
(assert (.equals (first sample-data) (first commons-codec-data2)))

(println "clojure-contrib")
(def clojure-contrib-data2 (time (doall (map #(clojure.contrib.base64/decode-str %) clojure-contrib-data))))
(println (first clojure-contrib-data2))
(assert (.equals (first commons-codec-data2) (first clojure-contrib-data2)))

(println "fencepost-recur")
(def fencepost-recur-data2 (time (doall (map #(fencepost.base64-recur/base64-decode-recur %) fencepost-recur-data))))
(println (first fencepost-recur-data2))
(assert (.equals (first commons-codec-data2) (first fencepost-recur-data2)))
{% endcodeblock %}

So, how'd we do? The following is fairly representative:

    bartok ~/git/base64-clojure $ clojure fencepost/test/compare_base64.clj 
    Encoding
    Commons-codec
    "Elapsed time: 85.219348 msecs"
    Xm91U1RMbHRAK1ZCcWFqeEsmJEJrfkhIdSsxfStJdGBNc2txUTBkUyQrdCM7Wk9rInN2XFBtKGNTX002aw==
    clojure-contrib
    "Elapsed time: 896.538841 msecs"
    Xm91U1RMbHRAK1ZCcWFqeEsmJEJrfkhIdSsxfStJdGBNc2txUTBkUyQrdCM7Wk9rInN2XFBtKGNTX002aw==
    fencepost/base64_letfn
    "Elapsed time: 315.582591 msecs"
    Xm91U1RMbHRAK1ZCcWFqeEsmJEJrfkhIdSsxfStJdGBNc2txUTBkUyQrdCM7Wk9rInN2XFBtKGNTX002aw==
    fencepost-recur
    "Elapsed time: 215.516695 msecs"
    Xm91U1RMbHRAK1ZCcWFqeEsmJEJrfkhIdSsxfStJdGBNc2txUTBkUyQrdCM7Wk9rInN2XFBtKGNTX002aw==

The recursive encoding operation is just north of a third faster. We still take about twice as long as commons-codec (an improvement on our previous performance) but we're now *over four times faster* than clojure-contrib.

As referenced above a decoding function was also completed using the same techniques described for the recursive encoding function. The performance increase for that operation was just as stark:

    Decoding
    Commons-codec
    "Elapsed time: 33.148648 msecs"
    ^ouSTLlt@+VBqajxK&$Bk~HHu+1}+It`MskqQ0dS$+t#;ZOk"sv\Pm(cS_M6k
    clojure-contrib
    "Elapsed time: 1494.2707 msecs"
    ^ouSTLlt@+VBqajxK&$Bk~HHu+1}+It`MskqQ0dS$+t#;ZOk"sv\Pm(cS_M6k
    fencepost-recur
    "Elapsed time: 268.112751 msecs"
    ^ouSTLlt@+VBqajxK&$Bk~HHu+1}+It`MskqQ0dS$+t#;ZOk"sv\Pm(cS_M6k

Still nowhere near as fast as commons-codec but we're a bit better than five times faster than clojure-contrib.

As always code can be found on [github](https://github.com/heuristicfencepost/base64-clojure).