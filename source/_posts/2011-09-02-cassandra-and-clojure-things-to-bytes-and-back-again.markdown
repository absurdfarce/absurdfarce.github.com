---
layout: post
title: "Cassandra and Clojure: Things To Bytes And Back Again"
date: 2011-09-02 12:00
comments: true
categories: [cassandra clojure ruby avro serialization]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2011/09/cassandra-and-clojure-things-to-bytes.html)

In a [previous post](http://absurdfarce.github.io/blog/2011/05/24/cassandra-and-clojure-the-beginning-of-a-beautiful-friendship/) we briefly discussed the idea of using idiomatic Clojure to access data contained in a Cassandra instance, including transparent conversion to and from Clojure data. We'll explore an implementation of this idea, although we won't address the question of laziness, in part because there are sizable trade-offs to consider. For example, any solution that provides lazy evaluation must do so while also attempting to minimize the number of trips to the underlying data store. This question may be picked up in yet more future work, but for now we'll continue on. We've upgraded to Cassandra 0.8.2 and Clojure 1.2, and we're using a new data model (see below), but for the most part we'll try to pick up where we left off. 

At the core the Cassandra data model is centered around columns. Each of these columns contains both a name and a value, both of which are represented as binary data. This model is very simple, and while it may appear limiting it is in reality quite flexible. The lack of any pre-defined data types avoids any "impedance mismatch" resulting from structured data being shoehorned into data types that don't really fit. We're free to represent column values in any way we see fit; if we can convert it into bytes it's fair game. Our problem thus devolves into a question of serialization, and suddenly there are many suitors vying for our attention. Among the set of well-known serialization packages we find Google's [Protocol Buffers](https://developers.google.com/protocol-buffers/), [Thrift](http://thrift.apache.org) and [Avro](http://avro.apache.org). And since we're working in a language with good Java interop we can always have Java's built-in serialization (or something similar like [Kryo](http://code.google.com/p/kryo)) available. Finally we're always free to roll our own. 

Let's begin by ruling out that last idea. There are already well-tested third-party serialization libraries so unless we believe that all of them suffer from some fatal error it's difficult to justify the risk and expense of creating something new.  We'd also like our approach to have some level of cross-platform support so native Java serialization is excluded (along with Kryo). We also need to be able to encode and decode arbitrary data without defining message types or their payload(s) in advance, a limitation that rules out Protocol Buffers and Thrift. The last man standing is Avro, and fortunately for us he's a good candidate. Avro is schema-based but the library includes facilities for generating schemas on the fly by inspecting objects via Java reflection. Also included is a notion of storing schemas with data, allowing the original object to be easily reconstructed as needed. The Avro [data model](http://avro.apache.org/docs/current/spec.html) includes a rich set of basic types as well as support for "complex" types such as arrays and maps. 

We'll need to implement a Clojure interface into the Avro functionality; this can be as simple as methods to encode and decode data and schemas. At least some implementations of Avro data files use a pre-defined "meta-schema" (consisting of the schema for the embedded data and that data itself) for storing both items. Consumers of these files first decode the meta-schema then use the discovered schema to decode the underlying data. We'll follow a similar path for our library. We'll also extend our Cassandra support a bit in order to support the insertion of new columns for a given key and column family.

We wind up with the following for our Avro library:

{% codeblock lang:clojure %}
(ns fencepost.avro)

; Dependencies
; avro 1.5.2
; paranamer 2.0 (required by Avro)
; jackson 1.8.5 (required by Avro)

(import '(org.apache.avro Schema)
	'(org.apache.avro.generic GenericDatumReader)
	'(org.apache.avro.io BufferedBinaryEncoder EncoderFactory DecoderFactory)
	'(org.apache.avro.reflect ReflectData ReflectDatumWriter)
	'(java.io ByteArrayOutputStream ByteArrayInputStream)
        )

(defn get_meta_schema []
      ; Ideally we'd use a record for this sort of thing but doing so would require setX()
      ; accessors for all fields.  Use of a map here is less precise but quite a bit easier
      ; to code.
      "{\"type\": \"map\", \"values\": \"bytes\"}"
)

(defn encode_with_schema [target]
      "Use Java reflection to generate a schema for the input object and return that schema along with encoded data"
      (let [targetschema (.getSchema (ReflectData/get) (.getClass target))
      	    targetwriter (ReflectDatumWriter. targetschema)
	    buffer (ByteArrayOutputStream.)
      	    encoder (.binaryEncoder (EncoderFactory.) buffer nil)]

	    ; Populate the buffer with Avro data for the target.  Note that we can't bundle the
	    ; whole thing up into a single let expression because of our reliance on side effects.
	    (.write targetwriter target encoder)
	    (.flush encoder)

	    (let [targetdata (.toByteArray buffer)
	    	  metawriter (ReflectDatumWriter. (Schema/parse (get_meta_schema)))]
		  (.reset buffer)
		  (.write metawriter 
		  	  {"schema" (.getBytes (.toString targetschema)) "data" targetdata} 
			  encoder)
		  (.flush encoder)
		  (.toByteArray buffer)
		  )
	    )
)

(defn decode_from_schema [indata]
      "Use the meta_schema to extract data and schema and then extract raw data"
      (let [metareader (GenericDatumReader. (Schema/parse (get_meta_schema)))
      	    buffer (ByteArrayInputStream. indata)
      	    decoder (.binaryDecoder (DecoderFactory.) buffer nil)
	    middata (.read metareader nil decoder)

	    ; The data type returned from the underlying Avro Java code needs a bit
	    ; of massaging before we can move forward.  Avro decoders return strings
	    ; as instances of Utf8 objects so we have to apply "str" to them directly
	    ; in order to get back something we can work with.
	    {schema "schema" data "data"} (zipmap (map str (keys middata)) (vals middata))
	    schemabytes (byte-array (.capacity schema))
	    databytes (byte-array (.capacity data))]

	    ; Ah, more side effects.  The Java Avro implementation makes heavy use of
	    ; NIO ByteBuffers, so we're forced to convert them into byte arrays before
	    ; continuing. 
	    (.get schema schemabytes)
	    (.get data databytes)

	    (let [targetreader (GenericDatumReader. (Schema/parse (String. schemabytes)))
	    	 targetdecoder (.binaryDecoder (DecoderFactory.) databytes nil)]
	    	 (.read targetreader nil targetdecoder)
		 )
	)
)
{% endcodeblock %}

After the improvements described above our Cassandra library now looks like:

{% codeblock lang:clojure %}
(ns fencepost.cassandra)

; Dependencies
; apache-cassandra-thrift-0.8.2.jar
; libthrift-0.6.jar
; slf4j-api-1.6.1.jar
; slf4j-log4j12-1.6.1.jar
; log4j-1.2.16.jar

(import '(org.apache.cassandra.thrift Cassandra$Client SliceRange SlicePredicate ColumnParent KeyRange ConsistencyLevel ColumnPath Column)
	'(org.apache.thrift.transport TFramedTransport TSocket)
        '(org.apache.thrift.protocol TBinaryProtocol)
	'(java.nio ByteBuffer)
        )

; A quick note on usage.  The Cassandra Java API returns "this" for many of the setX() methods on the setters of core objects in the Thrift
; API.  Our usage here consistently employs the doto syntax rather than relying on these return values.  This is largely a matter of 
; convention, but in this case it appears to make the code a bit clearer to read.  Your mileage may vary.
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
  "Get a list of KeySlices for every key in the range beginning with start and ending with end"
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
  "Retrieve the set of keys in a list of KeySlices"
  (map #(String. (.getKey %)) slices) 
  )

(defn range_slices_columns [slices key]
  "Given a list of KeySlices retrieve a map of the columns associated with the specified key"
  (let [match (first (filter #(= key (String. (.getKey %))) slices))]
    (cond (nil? match) nil
          (true? true)
          (let [urcols (.getColumns match)
                cols (map #(.getColumn %) urcols)]
            (zipmap (map #(keyword (String. (.getName %))) cols)
                    (map #(.getValue %) cols))
            )
          )
    )
  )

(defn insert [client key cf colname colval_bytes]
  "Insert the specified column into the specified column family.  At present we don't support super columns"
  (let [key_bytes (.getBytes key)
       key_buffer (ByteBuffer/allocate (alength key_bytes))
       column_parent 
        (doto (ColumnParent.)
	  (.setColumn_family cf)
	)
       colname_bytes (.getBytes colname)
       column
	(doto (Column.)
	  (.setName colname_bytes)
	  (.setValue colval_bytes)
	  (.setTimestamp (System/currentTimeMillis))
	)]

       ; Populate the built ByteBuffer with the contents of the input key
       (.put key_buffer key_bytes)
       (.flip key_buffer)

       (.insert client key_buffer column_parent column ConsistencyLevel/ONE)
       )
)
{% endcodeblock %}

Our core example will be built around a collection of employee records. We want to create a report on these employees using attributes defined in various well-known columns. We'd like to access the values in these columns as simple data types (booleans, integers, perhaps even an array or a map) but we'd like to do so through a uniform interface. We don't want to access certain columns in certain ways, as if we "knew" that a specific column contained data of a specific type. Any such approach is by definition brittle if the underlying data model should shift in flight (as data models are known to do). 

After finishing up with our Clojure libraries we implement a simple app for populating our Cassandra instance with randomly-generated employee information: 

{% codeblock lang:clojure %}
; Populate data for a set of random users to a Cassandra instance.
;
; Users consist of the following set of data:
; - a username [String]
; - a user ID [integer]
; - a flag indicating whether the user is "active" [boolean]
; - a list of location IDs for each user [list of integer]
;
; User records are keyed by username rather than user IDs, mainly because at the moment 
; we only support strings for key values.  The Cassandra API exposes keys as byte arrays 
; so we could extend our Cassandra support to include other datatypes.

(use '[fencepost.avro])
(use '[fencepost.cassandra])

(import '(org.apache.commons.lang3 RandomStringUtils)
	'(java.util Random)
        )


; Utility function to combine our Avro lib with our Cassandra lib
(defn add_user [client username userid active locationids]
      (let [userid_data (encode_with_schema userid)
      	    active_data (encode_with_schema active)
	    locationids_data (encode_with_schema locationids)]
	        (insert client username "employee" "userid" userid_data)
		(insert client username "employee" "active" active_data)
      		(insert client username "employee" "locationids" locationids_data)
	    )
)

; Generate a list of random usernames
(let [client (connect "localhost" 9160 "employees")]
      (dotimes [n 10]
      	       (let [username (RandomStringUtils/randomAlphanumeric 16)
	       	     random (Random.)
		     userid (.nextInt random 1000)
		     active (.nextBoolean random)
		     locationids (into [] (repeatedly 10 #(.nextInt random 100)))]
      	       	    (add_user client username userid active locationids)
		    (println (format "Added user %s: [%s %s %s]" username userid active locationids))
		    )
      )
)
{% endcodeblock %}

    [varese clojure]$ ~/local/bin/clojure add_employee_data.clj 
    Added user gFc0pVnLKPnjrrLx: [158 true [77 73 99 58 31 64 1 37 70 69]]
    Added user 5gGh9anHwFINpr5t: [459 true [34 71 28 1 2 84 11 33 37 28]]
    Added user pGRMeXBTFoBIWhud: [945 true [45 83 51 45 11 4 80 68 73 27]]
    ...

We're now ready to create our reporting application. Using Avro and a consistent meta-schema this comes off fairly easily: 

{% codeblock lang:clojure %}
; Retrieve information from the Cassandra database about one of our employees

(use '[fencepost.avro])
(use '[fencepost.cassandra])

(defn evaluate_user [slices username]
      "Gather information for the specified user and display a minimal report about them"

      ; Note that the code below says nothing about types.  We specify the column names we
      ; wish to access but whatever Cassandra + Avro supplies for the value of that column
      ; is what we get.
     (let [user_data (range_slices_columns slices username)
      	   userid (decode_from_schema (user_data :userid))
      	   active (decode_from_schema (user_data :active))
      	   locationids (decode_from_schema (user_data :locationids))]
      (println (format "Username: %s" username))
      (println (format "Userid: %s" userid))
      (println (if (> userid 0) "Userid is greater than zero" "Userid is not greater than zero"))
      (println (format "Active: %s" active))
      (println (if active "User is active" "User is not active"))

      ; Every user should have at least one location ID.
      ;
      ; Well, they would if we were able to successfully handle an Avro record.
      ;(assert (> (count locationids) 0))
      )
)

(let [client (connect "localhost" 9160 "employees")
      key_slices (get_range_slices client "employee" "!" "~")
      keys (range_slices_keys key_slices)]
      (println (format "Found %d users" (count keys)))
      (dotimes [n (count keys)]
      	       (evaluate_user key_slices (nth keys n))      
      )
    )
{% endcodeblock %}

    [varese clojure]$ ~/local/bin/clojure get_employee_data.clj 
    Found 10 users
    Username: 5gGh9anHwFINpr5t
    Userid: 459
    Userid is greater than zero
    Active: true
    User is active
    ...

And in order to verify our assumptions about simple cross-platform support we create a Ruby version of something very much like our reporting application: 

{% codeblock lang:ruby %}
require 'rubygems'
require 'avro'
require 'cassandra'

def evaluate_avro_data bytes

  # Define the meta-schema
  meta_schema = Avro::Schema.parse("{\"type\": \"map\", \"values\": \"bytes\"}")

  # Read the meta source and extract the contained data and schema
  meta_datum_reader = Avro::IO::DatumReader.new(meta_schema)
  meta_val = meta_datum_reader.read(Avro::IO::BinaryDecoder.new(StringIO.new(bytes)))

  # Build a new reader which can handle the indicated schema
  schema = Avro::Schema.parse(meta_val["schema"])
  datum_reader = Avro::IO::DatumReader.new(schema)
  val = datum_reader.read(Avro::IO::BinaryDecoder.new(StringIO.new(meta_val["data"])))
end

client = Cassandra.new('employees', '127.0.0.1:9160')
client.get_range(:employee,{:start_key => "!",:finish_key => "~"}).each do |k,v|
  userid = evaluate_avro_data v["userid"]
  active = evaluate_avro_data v["active"]
  locationids = evaluate_avro_data v["locationids"]
  puts "Username: #{k}, user ID: #{userid}, active: #{active}"
  puts "User ID #{(userid > 0) ? "is" : "is not"} greater than zero"
  puts "User #{active ? "is" : "is not"} active"

  # Ruby's much more flexible notion of truthiness makes the tests above somewhat less
  # compelling.  For extra validation we add the following
  "Oops, it's not a number" unless userid.is_a? Fixnum
end
{% endcodeblock %}

    [varese 1.8]$ ruby get_employee_data.rb
    Username: 5gGh9anHwFINpr5t, user ID: 459, active: true
    User ID is greater than zero
    User is active
    Username: 76v8iEJcc79Huj9L, user ID: 469, active: false
    User ID is greater than zero
    User is not active
    ...

This code meets our basic requirement, but as always there were a few stumbling blocks along the way. Avro includes strings as a primitive type, but unfortunately the Java API (which we leverage for our Clojure code) returns string instances as a Utf8 type. We can get a java.lang.String from these objects, but unfortunately we need another toString() method call that (logically) is completely unnecessary. We also don't fully support complex types. The Avro code maps the Clojure classes representing arrays and maps onto a "record" type that includes the various fields exposed via getters. Supporting these types requires the ability to reconstruct the underlying object based on these fields, and doing so reliably is beyond the scope of this work. Finally, we were forced to use Ruby 1.8.x for the Ruby examples since the Avro gem apparently doesn't yet support 1.9. 
