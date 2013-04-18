---
layout: post
title: "Data Models and Cassandra"
date: 2011-03-29 12:00
comments: true
categories: [cassandra python]
---
Originally published at [Heuristic Fencepost](http://heuristic-fencepost.blogspot.com/2011/03/data-models-and-cassandra.html)

[Cassandra](http://cassandra.apache.org) implements a [data model built on ColumnFamilies](http://wiki.apache.org/cassandra/DataModel) which differs from both SQL and document-oriented databases such as CouchDB.  We want to spend a little time with this data model, get to know it a bit, so we setup a simple but quasi-realistic problem space: given a database of tweets, tweet authors and tweet followers attempt to answer a sequence of questions about that data.  The questions should increase in complexity and each new question should explore a different facet of the data model.

The examples below are implemented in Python with a little help from a few excellent libraries:

* [Pycassa](https://github.com/pycassa/pycassa) for interacting with Cassandra.
* [Twitter tools](http://mike.verdone.ca/twitter/) when we have to interact with Twitter via their REST API

We'll begin by looking at how we might query an Cassandra instance populated with data in order to find the answers to the questions in our problem space. We'll close by briefly discussing how to get the data into Cassandra.

We should be all set; bring on the questions!

## FOR EACH AUTHOR: WHAT LANGUAGE DO THEY WRITE THEIR TWEETS IN AND HOW MANY FOLLOWERS DO THEY HAVE?

The organizing structure of the Cassandra data model is the column family defined within a keyspace. The keyspace is exactly what it sounds like: a collection of keys, each identifying a distinct entity in your data model. Each column family represents a logical grouping of data about these keys. This data is represented by one or more columns, which are really not much more than a tuple containing a name, value and timestamp. A column family can contain one or more columns for a key in the keyspace, and as it turns out you have a great deal of flexibility here; columns in a column family don't have to be defined in advance and do not have to be the same for all keys.

We begin by imagining a column family named "authors" in a keyspace defined by the user's Twitter handle or screen name. Assume that for each of these keys the "authors" column family contains a set of columns, one for each property returned by the "user/show" resource found in the Twitter REST API. Let's further assume that included in this data are fields named "lang" and "followers_count" and that these fields correspond to exactly the data we're looking for. We can satisfy our requirement by using a range query to identify all keys that fall within a specified range. In our case we want to include all alphanumeric screen names so we query across the range of printable ASCII characters. The Pycassa API makes this very easy [1]:

{% codeblock lang:python %}
import pycassa

cassandra = pycassa.connect("twitter")
authors_cf = pycassa.ColumnFamily(cassandra,"authors")
for (k,values) in authors_cf.get_range('!','~',["lang","followers_count"]):
    print "Key: %s, language: %s, followers: %s" % (k,values["lang"],values["followers_count"])
{% endcodeblock %}

The result is exactly what we wanted:

    [@varese src]$ python Query1.py 
    Key: Alanfbrum, language: en, followers: 73
    Key: AloisBelaska, language: en, followers: 8
    Key: DASHmiami3, language: en, followers: 11
    Key: DFW_BigData, language: en, followers: 21
    ...

## HOW MANY TWEETS DID EACH AUTHOR WRITE?

Okay, so we've got the idea of a column family; we can use them to define a set of information for keys in our keyspace. Clearly this is a useful organizing principle, but in some cases we need a hierarchy that goes a bit deeper. The set of tweets written by an author illustrates just such a case: tweets are written by a specific author, but each tweet has data of it's own (the actual content of the tweet, a timestamp indicating when it's written, perhaps some geo-location info, etc.). How can we represent this additional level of hierarchy?

We could define a new keyspace consisting of some combination of the author's screen name and the tweet ID but this doesn't seem terribly efficient; identifying all tweets written by an author is now unnecessarily complicated. Fortunately Cassandra provides a super column family which meets our needs exactly. The value of each column in a super column family is itself a collection of regular columns.

Let's apply this structure to the problem at hand. Assume that we also have a super column family named "tweets" within our keyspace. For each key we define one or more super columns, one for each tweet written by the author identified by the key. The value of any given super column is a collection of columns, one for each field contained in the results we get when we search for tweets using Twitter's search API. Once again we utilize a range query to access the relevant keys:

{% codeblock lang:python %}
import pycassa

cassandra = pycassa.connect("twitter")
tweets_cf = pycassa.ColumnFamily(cassandra,"tweets")

# Key in return value from "tweets" super column family is the
# tweet ID, value is a map of per-tweet data. We're only interested
# in the number of tweets so we only need to compute the size
# of the returned hash.
for (k,values) in tweets_cf.get_range('!','~'):
    print "Authors: %s, tweets written: %d" % (k,len(values))
{% endcodeblock %}

Running this script gives us the following:

    [@varese src]$ python Query2.py
    Authors: Alanfbrum, tweets written: 1
    Authors: AloisBelaska, tweets written: 1
    Authors: DASHmiami3, tweets written: 1
    Authors: DFW_BigData, tweets written: 1
    ...
    Authors: LaariPimenteel, tweets written: 2
    Authors: MeqqSmile, tweets written: 1
    Authors: Mesoziinha, tweets written: 2


## HOW MANY TWEET AUTHORS ARE ALSO FOLLOWERS OF @SPYCED?

We're now attempting to replicate functionality that looks very much like a join in a relational database.  Somehow we have to relate one type of resource (the set of followers for a Twitter user) to the set of authors defined in our "authors" column family. The Twitter REST API exposes the set of user IDs that follow a given user, so one approach to this problem might be to obtain these IDs and, for each of them, query the "authors" table to see if we have an author with a matching ID. As for the user to search for... well, [Jonathan Ellis](http://spyced.blogspot.com) (@spyced) seemed like the obvious choice.

Newer versions of Cassandra provide support for indexed slices on a column family. The database maintains an index for a named column within a column family, enabling very efficient lookups for rows in which the column matches a simple query. Query terms can test for equality or whether a column value is greater than or less than an input value [2]. Multiple search clauses can be combined within a single query, but in our case we're interested in strict equality only. Our solution to this problem looks something like the following:

{% codeblock lang:python %}
from twitter.api import Twitter
import pycassa
from pycassa.index import *

cassandra = pycassa.connect("twitter")
authors_cf = pycassa.ColumnFamily(cassandra,"authors")
tweets_cf = pycassa.ColumnFamily(cassandra,"tweets")

twitter = Twitter()

# Iterate through the set of IDs returned by the Twitter API and execute
# an index search against each ID. The Pycassa API will return a generator
# for each query so we make use of the for expression to determine when
# we should increment the total count.
count = 0
for authorid in twitter.followers.ids(id="spyced"):
    author_expr = create_index_expression('id_str',str(authorid))
    author_clause = create_index_clause([author_expr])
    for (authorkey,authorprops) in authors_cf.get_indexed_slices(author_clause):
        print authorkey
        count += 1
print count
{% endcodeblock %}

The result looks pretty promising:

    [@varese src]$ python Query3.py
    tlberglund
    evidentsoftware
    sreeix
    ...
    joealex
    cassandra
    21

Some spot checking of these results using the Twitter Web interface seems to confirm the results.

## POPULATING THE DATA

So far we've talked about accessing data from a Cassandra instance that has already been populated with the data we need. But how do we get it in there in the first place? The answer to this question is a two-step process; first we create the relevant structures within Cassandra and then we use our Python tools to gather and add the actual data.

My preferred tool for managing my Cassandra instance is the pycassaShell that comes with Pycassa. This tool makes it's easy to create the column families and indexes we've been working with:

{% codeblock lang:python %}
SYSTEM_MANAGER.create_keyspace('twitter',replication_factor=1)
SYSTEM_MANAGER.create_column_family('twitter','authors',comparator_type=UTF8_TYPE)
SYSTEM_MANAGER.create_column_family('twitter','tweets',super=True,comparator_type=UTF8_TYPE)
SYSTEM_MANAGER.create_index('twitter','authors','id_str',UTF8_TYPE)
{% endcodeblock %}

There are plenty of similar tools around; your mileage may vary.

When it comes to the heavy lifting, we combine Pycassa and Twitter tools into a simple script that does everything we need:

{% codeblock lang:python %}
from twitter.api import Twitter
import pycassa

from itertools import ifilterfalse

# Query to use when finding tweets.
searchquery = "#cassandra"

# Borrowed from the itertools docs
def unique_everseen(iterable, key=None):
    "List unique elements, preserving order. Remember all elements ever seen."
    seen = set()
    seen_add = seen.add
    if key is None:
        for element in ifilterfalse(seen.__contains__, iterable):
            seen_add(element)
            yield element
    else:
        for element in iterable:
            k = key(element)
            if k not in seen:
                seen_add(k)
                yield element

def populate():

    cassandra = pycassa.connect("twitter")
    authors_cf = pycassa.ColumnFamily(cassandra,"authors")
    tweets_cf = pycassa.ColumnFamily(cassandra,"tweets")

    # The Twitter API returns Unicode vals for all string results.  In addition
    # Pycassa appears to complain when we give it a string encoded in something
    # other than UTF-8 or a non-string value.  To get around this we perform
    # an intelligent string conversion; if we get a Unicode type return the
    # UTF-8 encoding of that string, otherwise return the standard string
    # representation.
    def smart_str(val):
        if isinstance(val,unicode):
            return val.encode('utf-8')
        else:
            return str(val)

    search = Twitter(domain="search.twitter.com")
    twitter = Twitter()

    search_results = search.search(q=searchquery,rpp=100)
    tweets = search_results["results"]

    print tweets[0]
    for tweet in tweets:
        tweet_str = dict([(k,smart_str(v)) for (k,v) in tweet.iteritems()])
        tweets_cf.insert(tweet["from_user"],{tweet["id_str"]:tweet_str})

    print "Found %d tweets" % len(tweets)
    author_names = list(unique_everseen([t["from_user"] for t in tweets]))
    print "Found %d distinct authors" % len(author_names)

    # Convert everything into strings; in Cassandra name and values of a column
    # are apparently normally converted into strings
    for author_info in (twitter.users.show(id=name) for name in author_names):

        author_name = author_info['screen_name']
        author_info_str = dict([(k,smart_str(v)) for (k,v) in author_info.iteritems()])
        authors_cf.insert(author_name,author_info_str)
        print "Added data for author %s" % author_name

if __name__ == "__main__":
    populate()
{% endcodeblock %}

[1] For sake of brevity this discussion omits a few details, including the configuration of a partitioner and tokens in order to use range queries effectively and how keys are ordered. Consult the project documentation and wiki for additional details.

[2] Here "greater than" and "less than" are defined in terms of the type provided at index creation time.
