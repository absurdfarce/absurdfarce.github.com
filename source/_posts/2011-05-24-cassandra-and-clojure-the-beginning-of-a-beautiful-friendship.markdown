---
layout: post
title: "Cassandra and Clojure: The Beginning of a Beautiful Friendship"
date: 2011-05-24 12:00
comments: true
categories: [cassandra clojure scala]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2011/05/cassandra-and-clojure-beginning-of.html)

Soon after I began working with [Cassandra](http://cassandra.apache.org) it became clear to me that if you were in the market for a platform for creating applications that interact with this database you could do a lot worse than [Clojure](http://clojure.org). The lack of a query language [1] suggests that filtering and slicing lists of keys and columns might be a fairly common activity for apps powered by Cassandra. And while many languages support the map/filter/reduce paradigm Clojure's use of [sequences](http://clojure.org/sequences) throughout the core suggest a natural means to integrate this data into the rest of your application.

Cassandra itself provides an [API](http://wiki.apache.org/cassandra/API) that uses the [Thrift](http://thrift.apache.org) protocol for manipulating data. We'll use this interface to implement a simple proof-of-concept application that might serve as a testbed for manipulating data managed by Cassandra in idiomatic Clojure. Note that the Clojure ecosystem already includes several open-source projects that connect Clojure to Cassandra: these include [clj-cassandra](https://github.com/robertluo/clj-cassandra) and [clj-hector](https://github.com/pingles/clj-hector), the latter leveraging the [Hector Java client](http://hector-client.github.io/hector/build/html/index.html) to do it's work. In order to keep things simple we choose to avoid any of these third-party libraries; it's not as if the Thrift interface imposes a heavy burden on us. Let's see how far we can get with what's already in the packaging.

That sounds great... so what exactly are we trying to do? Beginning with the database generated during our [previous work](http://absurdfarce.github.io/blog/2011/03/29/data-models-and-cassandra/) with Cassandra we should be able to access sets of keys within a keyspace and a set of columns for any specific key. These structures should be available for manipulation in idiomatic Clojure as sequences. Ideally these sequences would be at least somewhat lazy and transparently support multiple datatypes. [2]

Using the Thrift interface requires working with a fair number of Java objects representing return types and/or arguments to the various exposed functions. My Clojure isn't yet solid enough to hash out Java interop code without flailing a bit so we kick things off with a Scala implementation. This approach allows us to simplify the interop problem without sacrificing the functional approach, all within a language that is by now fairly familiar.

The Scala code includes a fair number of intermediate objects but is otherwise fairly clean:

{% codeblock lang:scala %}
package org.fencepost.cassandra

import scala.collection.JavaConversions

import org.apache.thrift.protocol.TBinaryProtocol
import org.apache.thrift.transport._
import org.apache.cassandra.service._
import org.apache.cassandra.thrift._

object ThriftCassandraClient {

  def connect(host:String,port:Int,keyspace:String):Cassandra.Client = {

    val transport = new TFramedTransport(new TSocket(host, port))
    val protocol = new TBinaryProtocol(transport)
    val client = new Cassandra.Client(protocol)
    transport.open()
    client set_keyspace keyspace
    client
  }

  // Execute a range slice query against the specified Cassandra instance.  Method returns
  // an object suitable for later interrogation by range_slices_keys() or range_slices_columns()
  def get_range_slices(client:Cassandra.Client,cf:String,start:String,end:String):Iterable[KeySlice] = {

    val sliceRange = new SliceRange()
    sliceRange setStart new Array[Byte](0)
    sliceRange setFinish new Array[Byte](0)
    sliceRange setReversed false
    sliceRange setCount 100
    val slicePredicate = new SlicePredicate()
    slicePredicate setSlice_range sliceRange
    val columnParent = new ColumnParent(cf)
    val keyRange = new KeyRange()
    keyRange setStart_key (start getBytes)
    keyRange setEnd_key (end getBytes)

    val javakeys = client.get_range_slices(columnParent,slicePredicate,keyRange,ConsistencyLevel.ONE)

    // Return from Thrift Java client is List<KeySlice> so we have to explicitly convert it here
    JavaConversions asScalaIterable javakeys
  }

  // Return an Iterable for all keys in an input query state object
  def range_slices_keys(slices:Iterable[KeySlice]) = slices map { c => new String(c.getKey) }

  // Return an Option containing column information for the specified key in the input query
  // state object.  If the key isn't found None is returned, otherwise the Option contains a
  // map of column names to column values.
  def range_slices_columns(slices:Iterable[KeySlice], key:String):Option[Map[String,String]] = {
    
    slices find { c => new String(c.getKey()) == key } match {
      case None => None
      case Some(keyval) =>
      
        val urcols = JavaConversions asScalaIterable (keyval getColumns)
        val cols:Seq[Column] = (urcols map (_ getColumn)).toSeq
        Some(Map(cols map { c => (new String(c.getName())) -> (new String(c.getValue())) }:_*))
    }
  }

  def main(args: Array[String]): Unit = {

    val client = connect("localhost", 9160,"twitter")
    val slices = get_range_slices(client,"authors","!","~")
    val fivekeys = range_slices_keys(slices) take 5
    println("fivekeys: " + fivekeys)
    for (key <- fivekeys) {
      range_slices_columns(slices,key) match {
        case None =>
        case Some(cols) =>
          println(
            "Key " + key +
            ": name => " + (cols getOrElse ("name","")) +
            ", following => " + (cols getOrElse ("following",""))
          )
      }
    }
  }
}
{% endcodeblock %}

Translating this code into Clojure is more straightforward than expected:

{% codeblock lang:clojure %}
(import '(org.apache.thrift.transport TFramedTransport TSocket)
        '(org.apache.thrift.protocol TBinaryProtocol)
        '(org.apache.cassandra.thrift Cassandra$Client SliceRange SlicePredicate ColumnParent KeyRange ConsistencyLevel)
        )

(defn connect [host port keyspace]
  "Connect to a Cassandra instance on the specified host and port.  Set things up to use the specified key space."
  (let [transport (TFramedTransport. (TSocket. host port))
        protocol (TBinaryProtocol. transport)
        client (Cassandra$Client. protocol)]
    (.open transport)
    (.set_keyspace client keyspace)
    client
    )
  )

(defn get_range_slices [client cf start end]
  "Simple front end into the get_range_slices function exposed via Thrift"
  (let [slice_range
        (doto (SliceRange.)
          (.setStart (byte-array 0))
          (.setFinish (byte-array 0))
          (.setReversed false)
          (.setCount 100)
          )
        slice_predicate
        (doto (SlicePredicate.)
          (.setSlice_range slice_range))
        column_parent (ColumnParent. cf)
        key_range
        (doto (KeyRange.)
          (.setStart_key (.getBytes start))
          (.setEnd_key (.getBytes end)))
        ]
    (.get_range_slices client column_parent slice_predicate key_range ConsistencyLevel/ONE) 
    )
  )

(defn range_slices_keys [slices]
  "Retrieve the set of keys in a get_range_slices result"
  (map #(String. (.getKey %)) slices) 
  )

(defn range_slices_columns [slices key]
  "Retrieve a map of the columns associated with the specified key in a get_range_slices result"
  (let [match (first (filter #(= key (String. (.getKey %))) slices))]
    (cond (nil? match) nil
          (true? true)
          (let [urcols (.getColumns match)
                cols (map #(.getColumn %) urcols)]
            (zipmap (map #(keyword (String. (.getName %))) cols)
                    (map #(String. (.getValue %)) cols))
            )
          )
    )
  )

(let [client (connect "localhost" 9160 "twitter")
      key_slices (get_range_slices client "authors" "!" "~")
      five_keys (take 5 (range_slices_keys key_slices))]

  (print five_keys)

  (let [formatfn
        (fn [key]
          (let [cols (range_slices_columns key_slices key)]
            (format "Key %s: name => %s, following => %s\n" key (cols :name) (cols :following))
            )
          )]
    (print (reduce str (map formatfn five_keys)))
    )
  )
{% endcodeblock %}

These results seem fairly promising, although we're nowhere near done. This code assumes that all column names and values are strings, a perfectly ridiculous assumption. We also don't offer any support for nested data types, although in fairness this was a failing of our earlier work as well. Finally we haven't built in much support for lazy evaluation; we lazily convert column names to Clojure keywords but that's about it. But fear not, gentle reader; we'll revisit some or all of these points in future work.

[1] At least until [CQL](https://issues.apache.org/jira/browse/CASSANDRA-1703) arrives in Cassandra 0.8

[2] We won't be able to meet these last two goals in our initial implementation, but with any luck we'll be able to revisit them in future work

