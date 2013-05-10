---
layout: post
title: "base64 in Clojure: An Initial Attempt"
date: 2010-10-31 12:00
comments: true
categories: [clojure base64]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2010/10/base64-in-clojure-initial-attempt.html)

After working through most of Stuart Halloway's excellent [Programming Clojure](http://pragprog.com/book/shcloj/programming-clojure) I found myself in search of a meaty example to exercise my Clojure chops. Re-implementing the Project Euler examples in another language didn't seem terribly appealing. Maybe a base64 implementation; I hadn't implemented the encoding in a while and it is a reasonable exercise when learning a new language. So what kind of support does Clojure already have?

As expected there's no base64 support in the core. clojure-contrib does include [some support](http://richhickey.github.io/clojure-contrib/base64-api.html), although the documentation doesn't inspire a great deal of confidence:

    This is mainly here as an example.  It is much slower than the 
    Apache Commons Codec implementation or sun.misc.BASE64Encoder.

Intriguing; looks like we have a winner!

The only complexity in a base64 implementation is handling the one and two byte cases. Any code must support returning the last two bits of the first byte (or the last four bits of the second byte) as distinct values in these cases as well as combining these bits with the leading bits of the second and third bytes (respectively) in other cases. My initial implementation was a fairly straightforward adaptation of this principle. 

{% codeblock lang:clojure %}
(ns fencepost.base64_letfn)

; This could be placed within the defn below but rebuilding this map
; on each invocation seems a bit wasteful.
(def to-base64 
     (zipmap 
      (range 0 64)
      "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/")
     )

(defn base64-encode
  "Encode input string into base64 alphabet.  String should be encoded
  in the character set specified by encoding."
  [#^String arg #^String encoding]
  (letfn [
      (do-one-byte [somebyte]
                   {:curr [(bit-shift-right somebyte 2)] :remainder (bit-and 0x3F (bit-shift-left somebyte 4)) }
                   )
      (do-two-bytes [somebytes]
                    (let [[byteone bytetwo] somebytes
                          {curr :curr remainder :remainder} (do-one-byte byteone)]
                      {:curr (conj curr (bit-or remainder (bit-shift-right bytetwo 4))) :remainder (bit-and 0x3F (bit-shift-left bytetwo 2))}
                      )
                    )
      (do-three-bytes [somebytes]
                      (let [[byteone bytetwo bytethree] somebytes
                            {curr :curr remainder :remainder} (do-two-bytes [byteone bytetwo])]
                        (conj (conj curr (bit-or remainder (bit-shift-right bytethree 6))) (bit-and 0x3F bytethree))
                        )
                      )
      (somefn [somechars]
                (let [l (count somechars)]
                  (cond
                   (= l 3)
                   (apply str (map #(to-base64 %) (do-three-bytes somechars)))
                   (= l 2)
                   (let [{curr :curr remainder :remainder} (do-two-bytes somechars)]
                     (str (apply str (map #(to-base64 %) curr)) (to-base64 remainder) "=")
                     )
                   (= l 1)
                   (let [{curr :curr remainder :remainder} (do-one-byte (first somechars))]
                     (str (apply str (map #(to-base64 %) curr)) (to-base64 remainder) "==")
                     ))))]
    (apply str (map somefn (partition 3 3 () (.getBytes arg encoding))))))

(defn base64-encode-ascii
  "Encode input string into base64 alphabet.  Input string is assumed
  to be encoded in UTF-8.  If the input is encoded with a different
  character set base64-encode should be used instead of this method."
  [#^String arg]
  (base64-encode arg "UTF8")
  )
{% endcodeblock %}

The core consists of a letfn definition with functions for handling the one, two and three byte cases. Each function returns two values: a list of all 6-bit elements discovered so far as well a "remainder" value representing the trailing bits in the one and two byte cases. This structure leads to better function re-use; as an example the two-byte function can build on the one-byte function. A wrapper function calls the correct value for a given set of bytes. Now we need only partition our byte array into groups of three or less and apply the wrapper function to each group. A combination of the partition and map functions do exactly that.

So how'd we do? We implement a simple test program to generate a set of random data, map a base64 implementation on to each sample and then realize the lazy sequence by converting it into a vector. We time this conversion (for a common set of sample data) for our implementation, the clojure-contrib implementation and commons-codec. We wind up with the following:

{% codeblock lang:clojure %}
(ns fencepost.test)

(import '(org.apache.commons.codec.binary Base64))
(import '(org.apache.commons.lang RandomStringUtils))
(use '[clojure.contrib.base64])
(use '[fencepost.base64_letfn])

(def sample-size 100)
(def max-string-size 256)

; Build up some sample data using commons-lang.  Sample data is built
; before any tests are run; this allows us to apply uniform test data
; to each implementation and to avoid corrupting timings with the
; generation of random data.
(def sample-data (map #(RandomStringUtils/randomAscii (* % (rand-int max-string-size))) (repeat sample-size 1)))

; Instantiate a Base64 instance from commons-codec
(def codec-base64 (new Base64 -1))

; Time each run.  Mapping a function onto our sample data produces a
; lazy sequence so we have to take the additional step of realizing
; the sequence; thus the conversion to a vector.
(println "Commons-codec")
(def commons-codec-data (time (vec (map #(new String (.encode codec-base64 (.getBytes %))) sample-data))))
(println (get commons-codec-data 0))

(println "clojure-contrib")
(def clojure-contrib-data (time (vec (map #(clojure.contrib.base64/encode-str %) sample-data))))
(println (get clojure-contrib-data 0))

; Simple sanity check; take commons-codec output as canonical and
; verify that what our implementations generate match up with the
; canonical sample.
(assert (.equals(get commons-codec-data 0) (get clojure-contrib-data 0)))

(println "fencepost/base64_letfn")
(def fencepost-data (time (vec (map #(fencepost.base64_letfn/base64-encode-ascii %) sample-data))))
(println (get fencepost-data 0))
(assert (.equals (get commons-codec-data 0) (get fencepost-data 0)))
{% endcodeblock %}

The following run is fairly representative:

    bartok ~/git/base64-clojure $ clojure fencepost/test/compare_base64.clj 
    Commons-codec
    "Elapsed time: 69.074926 msecs"
    RFJlYWkqIF10RCc1PXlpO3FcIDY6dkJuQkhHNWxRJkwkOFdgciFGfHwhYz5EYG15PjxWdFJ9Ml83TGFoeltHSTs+ST9mdj0rfSZrcVNIKn5oKSdTI3U5a1FqIzBvIkRIV0BkK3xCZjtrSWYuTiM1XWB9UW14W2dVLFM=
    clojure-contrib
    "Elapsed time: 1221.124746 msecs"
    RFJlYWkqIF10RCc1PXlpO3FcIDY6dkJuQkhHNWxRJkwkOFdgciFGfHwhYz5EYG15PjxWdFJ9Ml83TGFoeltHSTs+ST9mdj0rfSZrcVNIKn5oKSdTI3U5a1FqIzBvIkRIV0BkK3xCZjtrSWYuTiM1XWB9UW14W2dVLFM=
    fencepost/base64_letfn
    "Elapsed time: 375.829153 msecs"
    RFJlYWkqIF10RCc1PXlpO3FcIDY6dkJuQkhHNWxRJkwkOFdgciFGfHwhYz5EYG15PjxWdFJ9Ml83TGFoeltHSTs+ST9mdj0rfSZrcVNIKn5oKSdTI3U5a1FqIzBvIkRIV0BkK3xCZjtrSWYuTiM1XWB9UW14W2dVLFM=

These results do confirm that that clojure-contrib implementation is very, very slow in relation to commons-codec. We've reduced that runtime by roughly a third but we're still five or six times slower than commons-codec. Surely we can do better; in the next installment we'll see how to do so.

As usual all code can be found on [github](https://github.com/heuristicfencepost/base64-clojure).