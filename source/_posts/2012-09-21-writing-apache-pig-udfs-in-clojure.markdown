---
layout: post
title: "Writing Apache Pig UDFs in Clojure"
date: 2012-09-21 18:00
comments: true
categories: 
---

A previous [post](http://ianrumford.github.com/blog/2012/09/17/writing-apache-pig-udfs-in-jruby/) explored writing 
Apache [Pig](http://pig.apache.org/) *user defined functions* (UDFs) in [JRuby](http://jruby.org/).

This post explores writing equivalent UDFs in [Clojure](http://clojure.org/).

*Prima facie*, a Clojure UDF, because it compiles directly to Java bytecode,
should integrate extremely well with Pig's Java code.  There
shouldn't be any *impedance mismatches* between the two languages and
performance should be essentially the same as if the UDF were to be
written in Java itself.

But there doesn't seem to have been much prior experimentation with Clojure
UDFs.  Perhaps this quote from a Pig wiki
[page](http://wiki.apache.org/pig/TuringCompletePig) when Clojure was
a candidate for *adding control flow and modularity constructs to Pig*
may help to explain the reticence:

> "Clojure is a functional language, a paradigm which seems to engender one of love, terror, or confusion. As such, it probably is not a good choice for Pig Latin. Also, Cascalog already exists for those who like Clojure."

Even us junior members of the *Parentherati* would view this as
shorted sighted.

However I did find one old-ish but particularly useful example: Matt
Kangas's proof-of-concept
[mountain-pig](https://github.com/kangas/mountain-pig).  Matt's code
was useful to me on both the Java - Clojure interop (new territory to
me) and also how to comply with Pig's UDF implementation requirements (API).
You will see snippets of Matt's code in the example below.  Thanks
Matt!
 
It turns out that writing Pig UDFs in Clojure is as easy to
writing them in JRuby; the setup is a bit more involved but after that
it is plain sailing.  

But the potential to exploit fully the Java UDF
API in Clojure
is (IMHO) more obvious than JRuby, Python or JavaScript:  I think (qualitatively) the Clojure - Java interop would make it
easier to write more closely integrated  UDFs in
Clojure. 

The rest of this post follows broadly the same structure as the JRuby
post and it is worth reading that post  as its very much a scene setter.  In
this post I focus on
writing Clojure UDFs to return a scalars (string), maps (hashes) and
(Pig) *tuples*.  As before, the results are destined to be stored in HBase.

Also, as before, the log file was from a
Ubuntu audit subsystem log (*auditd*).

<!-- more -->

# Preparation #

The JRuby
[post](http://ianrumford.github.com/blog/2012/09/17/writing-apache-pig-udfs-in-jruby/)
has  details on:

* Installing HBase and Pig
* Creating the HBase
table and column family

    The tables names are given in the various Pig Latin scripts, the
    column family is always *record*.
    
* How to execute a Pig Latin script

    But note the change to how to set the classpath using *lein* given below.
    
* Some examples of *auditd* log records (stored in *./data/blog_audit1.log* in the Pig Latin scripts below).

# Installing and Using Leiningen #

[Leiningen](https://github.com/technomancy/leiningen) (*lein*) is
broadly the equivalent of Java's [maven](http://maven.apache.org/) and
describes  its *raison d'etre*  as

> "for automating Clojure projects without setting your hair on fire."

Installing *lein* is a matter of downloading the
[script](https://raw.github.com/technomancy/leiningen/preview/bin/lein)
and saving it somewhere e.g.

``` bash

cd /tmp
wget https://raw.github.com/technomancy/leiningen/preview/bin/lein
mv lein /usr/bin
chown root.root /usr/bin/lein
chmod 755 /usr/bin/lein

```

To create a new Leiningen project is as simple as:

``` bash

cd /home/me/where/i/keep/my/projects
lein new blog_cljudf

```

The new folder (*blog_cljudf*) will have, amongst other things, a
[project.clj](https://github.com/technomancy/leiningen/blob/preview/sample.project.clj)
file which specifies dependencies, source code directories, etc.  The
project.clj file is a Clojure program.

All of the following *lein*  commands must be run while in
the *blog_cljudf* folder.

Once written, all of the Clojure and Java code must  be compiled and  a
jar created that can be used / specified in a Pig Latin script:

``` bash

lein jar

```

The jar (e.g. *blog_cljudf-0.1.0-SNAPSHOT.jar*) will be created in the
*./target* folder.

The classpath for the Pig Latin scripts should be set by executing
*lein* with the *classpath* option:

``` bash

export PIG_CLASSPATH="`lein classpath`"

```

# Configuring the project.clj file #

The project.clj is shown below:

``` clojure

(defproject blog_cljudf "0.1.0-SNAPSHOT"

  :description "Pig UDFs written in Clojure"

  :url "http://example.com/FIXME"

  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  
  :dependencies [[org.clojure/clojure "1.4.0"]
                 [org.apache.hadoop/hadoop-core "0.20.2"]
                 [org.apache.hbase/hbase "0.94.1"]
                 [org.apache.pig/pig "0.10.0"]
                 ]

  :aot [#"blog_cljudf*" ]

  :java-source-paths ["src/java"] ; Java source is stored separately.
  
  :repositories [["apache" "https://repository.apache.org/content/repositories/releases"]])

```

Notes:

*  *:dependencies* includes version 1.4.0 of the Clojure compiler and
    runtime
    
* the *:dependencies* list Hadoop Core (0.20.2) HBase (0.94.1) and  Pig (0.10.0) as required

* *:repositories*  adds the Apache maven repo to the search
  path
    
* *:aot*  specifies the Clojure code that must be
  compiled *ahead of time* 
  
    aot compilation is necessary when a Java-compatible class file must be
  generated by Clojure (*gen-class*). The value given above is a regular expression:  all Clojure
  namespaces begining with *blog_cljudf* will be aot-ed.

* *:java-sources-paths* tell Leiningen to compile the Java in these directories
*before* Clojure; these Java classes will be imported into the Clojure code.

Once the clojure.clj has been configured correctly, Leiningen can be told to
resolve (download) all the dependencies for this project:

``` bash

lein deps

```

# Enabling Clojure UDFs to extend Pig's EvalFunc abstract class #

Clojure can't create a class (*gen-class*) that extends Pig's abstract
*EvalFunc* Java class.  To get over the limitation, stub concrete classes have to be
created for the various return value types in this example i.e. string, map
and (Pig's) tuple.

In the project folder, these java source files live in the
*:java-source-paths* folder (*src/java*). They are automatically
compiled by *lein jar*.

The map concrete class (*MapEvalFunc.java*)  looks like this:

``` java

package pig_udf.stub;
import org.apache.pig.EvalFunc;
import java.util.HashMap;

/**
 * Stub to specify a concrete type of EvalFunc.
 * Clojure's "gen-class" can't extend parameterized classes yet.
 */
 
public abstract class MapEvalFunc extends EvalFunc <HashMap> {
}

```

*TupleEvalFunc.java* is

``` java

package pig_udf.stub;
import org.apache.pig.EvalFunc;
import org.apache.pig.data.Tuple;

/* from Matt Kangas's mountain-pig */

/**
 * Stub to specify a concrete type of EvalFunc.
 * Clojure's "gen-class" can't extend parameterized classes yet.
 */


public abstract class TupleEvalFunc extends EvalFunc <Tuple> {

```

# Using a Clojure UDF returning scalars to etl audit data #

## The Scalar Clojure UDF ##

There appears to be no way, in a Java (Clojure) *EvalFunc* UDF to call arbitrary
methods (functions) from Pig Latin:  the *only* method called is
*exec*.  Contrast this with a JRuby (and Python it seems) UDF where
additional methods can be defined and called (e.g. *uuid* in my JRuby example)

This makes Java/Clojure UDFs returning scalars of more limited utility
(IMHO) as one would need a separate UDF for each return value needed.

*It would be possible to pass some sort of *directive* to the UDF to
tell it to return the *type*, *node* or whatever.  But, in this
example, its just simpler to return a map - see below.*   

For completeness though, two UDFs are given below - one
(*UUIDStringUDF*) returning a *uuid* 
and another (*TypeStringUDF*) returning the *type* of the audit record.

*UUIDStringUDF*:

``` clojure

(ns blog_cljudf.clopigstringuuid

  (:import  [org.apache.pig.EvalFunc] 
            [org.apache.pig.data DataType Tuple TupleFactory]
            [java.util UUID]
            [pig_udf.stub StringEvalFunc]
            )
  
  (:gen-class
   :name blog_cljudf.UUIDStringUDF
   :main true
   :extends pig_udf.stub.StringEvalFunc)

  (:require 
   [clojure.string :as str]
   )
  )

(defn make-uuid
  []
  (let [ uuid (.toString (UUID/randomUUID))]
    uuid))

(defn -exec
  "Return a uuid - no input needed"
  [& any]
  (make-uuid))

```

*TypeStringUDF*:

``` clojure

(ns blog_cljudf.clopigstringtype

  (:import  [org.apache.pig.EvalFunc] 
            [org.apache.pig.data DataType Tuple TupleFactory]
            [java.util UUID]
            [pig_udf.stub StringEvalFunc]
            )
  
  (:gen-class
   :name blog_cljudf.TypeStringUDF
   :main true
   :extends pig_udf.stub.StringEvalFunc)

  (:require 
   [clojure.string :as str]
   )
  )

(defn parse_input_record
  "Parse the text of the input record into fields in a map"
  [input_record]
  (let [prefix_string (get (str/split input_record #"\: ") 0)
        prefix_pairs (str/split prefix_string #" ")
        prefix_pair_vecs (map #(str/split % #"=") prefix_pairs )
        prefix_map (into {} prefix_pair_vecs)
        ]
    prefix_map
    )
  )

(defn parse_input_tuple
  "Parses the input tuple into a map"
  [#^Tuple input_tuple]
  (parse_input_record (.get input_tuple 0))
  )

(defn -exec
  "Return the type of the audit record"
  [this #^Tuple input_tuple]
  (let [m (parse_input_tuple input_tuple )
        {:strs [type]} m]
    type))



```

## The Scalar Pig Latin script ##

The Pig Latin script goes like this:

``` java

/* 

Pig Clojure Scalar UDF Example

Auditd Record Parsed and Loaded into Hbase

Uses a Clojure UDFa returning Strings

*/

/*

Register the Clojure and Java jar

*/

register ./target/blog_cljudf-0.1.0-SNAPSHOT.jar

/*

Load the auditd data

*/

auditd_records = LOAD './data/blog_audit1.log' USING PigStorage() AS (record: chararray);

/*

Creates the fields using the UDFs

*/

auditd_fields = foreach auditd_records generate (chararray) blog_cljudf.UUIDStringUDF(),
                                                (chararray) blog_cljudf.TypeStringUDF(record);

/*

Store the Audit fields into HBase records

*/  

store auditd_fields into 'hbase://blog_cljudf_scalar' using
  org.apache.pig.backend.hadoop.hbase.HBaseStorage (
  'record:type record:node', 'loadkey true');

```

Notes:

* Each UDF is instantiated once for each record

8 The string returned by each UDF must be cast

# Using a Clojure UDF returning a hash / map to etl audit data #

## The Hash / Map Clojure UDF ##

As with a JRuby UDF, a more natural way to return a collection of
fields and their values is  in a hash/map.

The example below  returns all of the prefix fields
(*type*, *node*, *msg*) from the audit record, 
as well as an *id* key with a value generated by the *uuid* method.

Again, as with the JRuby example, two new fields has been added (*unique* and *passed*) to
demonstrate returning integer and boolean values.

``` clojure

(ns blog_cljudf.clopigmap

  (:import  [org.apache.pig.EvalFunc] 
            [org.apache.pig.data DataType Tuple TupleFactory]
            [org.apache.pig.impl.util WrappedIOException]
            [java.io IOException]
            [java.util UUID Map]
            [pig_udf.stub MapEvalFunc]
            )
  
  (:gen-class
   :name blog_cljudf.MapUDF
   :main true
   :extends pig_udf.stub.MapEvalFunc)

  (:require 
   [clojure.string :as str]
   )
  )

(defn make-uuid
  []
  (let [ uuid (.toString (UUID/randomUUID))]
    uuid))

(defn parse_input_record
  "Parse the text of the input record into fields in a map"
  [input_record]
  (let [prefix_string (get (str/split input_record #"\: ") 0)
        prefix_pairs (str/split prefix_string #" ")
        prefix_pair_vecs (map #(str/split % #"=") prefix_pairs )
        prefix_map (into {} prefix_pair_vecs)
        ]
    prefix_map
    )
  )

(defn parse_input_tuple
  "Parses the input tuple into a map"
  [#^Tuple input_tuple]
  (parse_input_record (.get input_tuple 0))
  )

(defn make_output_map
  "Takes the input tuple and creates the output map"
  [#^Tuple input ]
  (let [r (parse_input_tuple input )
        unique (rand-int 1000000)
        passed true 
        s (assoc r "id" (make-uuid) "unique" unique "passed" passed)
        ]
    s
    )
  )

(defn -exec
  "Entry point for Pig evaluation. Invoked on every Tuple of a given dataset."
  [this #^Tuple input ]
  (let [m (make_output_map input )]
    (try
      (new java.util.HashMap m)
      (catch Exception e
        (throw (WrappedIOException/wrap "****woot****" e))))))

```

Notes:

* the name of the UDF class is the *gen-class* name: blog_cljudf.MapUDF

* the *-exec* method is the "entry point" for the Pig API

* the class extends Pig's *EvalFunc* via the local stub *MapEvalFunc*


## The Hash / Map Pig Latin script ##

The Pig Latin script is more complicated as it has to accept the Clojure
Hash, extract and cast the wanted keys' values and store all of them in
the HBase table:

``` java

/* 

Pig Clojure Map UDF Example

Auditd Records Parsed and Loaded into Hbase

Uses a Clojure UDF returning a HashMap 

*/

/*

Register the Clojure and Java jar

*/

register ./target/blog_cljudf-0.1.0-SNAPSHOT.jar

/*

Load the auditd data

*/

auditd_records = LOAD './data/blog_audit1.log' USING PigStorage() AS (record: chararray);

/*

Creates the maps using the UDF

*/

auditd_maps = foreach auditd_records generate blog_cljudf.MapUDF(record);

/*

Create the fields from the maps

(Note it appears to be impossible to combine the UDF call and keys selection into one statement)

*/

auditd_fields = foreach auditd_maps generate (chararray) $0#'id', (chararray) $0#'type', (chararray) $0#'node', (int) $0#'unique', (boolean) $0#'passed';

/*

Store the Audit fields into HBase records

*/

store auditd_fields into 'hbase://blog_cljudf_map' using
  org.apache.pig.backend.hadoop.hbase.HBaseStorage (
  'record:type record:node record:unique record:passed', 'loadkey true');

```

Notes:

* the names of the UDF is the *gen-class* name (*blog_cljudf.MapUDF*)
  
* the syntax for extracting the map's values into a tuple is different to
  the JRuby example; this way is preferred as it prevents repeated
  instantiations of the UDF.


# Using a Clojure UDF returning a tuple to etl audit data #

## The Tuple Clojure UDF ##

Pig allows a UDF to define a
[schema](http://pig.apache.org/docs/r0.8.1/piglatin_ref2.html#Schemas ):

> "Schemas enable you to assign names to and declare types for fields.
  Schemas are optional but we encourage you to use them whenever
  possible; type declarations result in better parse-time error
  checking and more efficient code execution."
  
Adding a schema to the UDF complicates it (compared to returning a map)
but makes the Pig Latin simpler, and arguably more intuitive.

The code below returns the same data as the map example but as a Pig
Tuple with a schema attached.  Much of the code
is the same though, the notable exception being the addition of the
*outputSchema* function that defines the returned tuple's schema:

``` clojure

(ns blog_cljudf.clopigtuple

  (:import  [org.apache.pig.EvalFunc] 
            [org.apache.pig.data DataType Tuple TupleFactory]
            [org.apache.pig.impl.util WrappedIOException]
            [org.apache.pig.impl.logicalLayer.schema Schema Schema$FieldSchema]
            [java.io IOException]
            [java.util UUID Map]
            [pig_udf.stub TupleEvalFunc]
            )
  
  (:gen-class
   :name blog_cljudf.TupleUDF
   :main true
   :extends pig_udf.stub.TupleEvalFunc)

  (:require 
   [clojure.string :as str]
   )
  )

(defn make-uuid
  []
  (let [ uuid (.toString (UUID/randomUUID))]
    uuid))

(defn parse_input_record
  "Parse the text of the input record into fields in a map"
  [input_record]
  (let [prefix_string (get (str/split input_record #"\: ") 0)
        prefix_pairs (str/split prefix_string #" ")
        prefix_pair_vecs (map #(str/split % #"=") prefix_pairs )
        prefix_map (into {} prefix_pair_vecs)
        ]
    prefix_map
    )
  )

(defn parse_input_tuple
  "Parses the input tuple into a map"
  [#^Tuple input_tuple]
  (parse_input_record (.get input_tuple 0))
  )

(defn make_output_map
  "Takes the input tuple and creates the output map"
  [#^Tuple input ]
  (let [r (parse_input_tuple input )
        unique (rand-int 1000000)
        passed true 
        s (assoc r "id" (make-uuid) "unique" unique "passed" passed)
        ]
    s
    )
  )

(defn -exec
  "Entry point for Pig evaluation. Invoked on every Tuple of a given dataset."
  [this #^Tuple input ]
  (let [m (make_output_map input )
        ;;{:strs ["id" "type" "node" "unique" "passed"]} m
        {:strs [id type node unique passed]} m 
        p (java.util.ArrayList. [id type node unique passed])]
    (try
      (.newTuple (TupleFactory/getInstance) p)
      (catch Exception e
        (throw (WrappedIOException/wrap "****woot****" e))))))

(defn -outputSchema
  "Output Schema with field names"
  [this #^Schema input_schema ]
  (let [tupleSchema (Schema.)
        schemaClassName (.. this getClass getName toLowerCase)
        schemaName "tuple_schema"
        ]
    (.add tupleSchema (Schema$FieldSchema. "id" DataType/CHARARRAY))
    (.add tupleSchema (Schema$FieldSchema. "type" DataType/CHARARRAY))
    (.add tupleSchema (Schema$FieldSchema. "node" DataType/CHARARRAY))
    (.add tupleSchema (Schema$FieldSchema. "unique" DataType/INTEGER))
    (.add tupleSchema (Schema$FieldSchema. "passed" DataType/BOOLEAN))
    (Schema. (Schema$FieldSchema. schemaName tupleSchema DataType/TUPLE ))))

```

Notes:

* the *outputSchema* method create a Pig-compliant schema

* The (arbitrary) schema name is *tuple_schema*

* The *exec* method returns a Pig Tuple populate by a Java ArrayList holding the values of the
  fields (keys)

* the name of the UDF class is the *gen-class* name
  (*blog_cljudf.TupleUDF*)
  
* the class extends Pig's *EvalFunc* via the local stub *TupleEvalFunc*



## The Tuple Pig Latin script ##

The Pig Latin script is rather different to the map script in the way
it extract the fields from the tuple returned by the UDF:. 

``` java

/* 

Pig Clojure Tuple UDF Example

Auditd Record Parsed and Loaded into Hbase

Uses a Clojure UDF returning a Pig Tuple with Schema

*/

/*

Register the Clojure and Java jar

*/

register ./target/blog_cljudf-0.1.0-SNAPSHOT.jar

/*

Load the auditd data

*/

auditd_records = LOAD './data/blog_audit1.log' USING PigStorage() AS (record: chararray);

/*

Creates the tuples using the UDF

*/

auditd_tuples = foreach auditd_records generate blog_cljudf.TupleUDF(record);

/*

Create the fields from the tuples

(Note it appears to be impossible to combine the UDF call and field selection into one statement)

*/

auditd_fields = foreach auditd_tuples generate tuple_schema.id, tuple_schema.type, tuple_schema.node, tuple_schema.unique, tuple_schema.passed;

/*

Store the Audit fields into HBase records

*/

store auditd_fields into 'hbase://blog_cljudf_tuple' using
  org.apache.pig.backend.hadoop.hbase.HBaseStorage (
  'record:type record:node record:unique record:passed', 'loadkey true');

```

Notes:

* The schema name (*tuple_schema*) is used to extract
  ([dereference](http://pig.apache.org/docs/r0.8.1/piglatin_ref2.html#deref)) the wanted
  fields from the returned tuple

# Final Words #

There is quite a lot in this post to digest, especially if Clojure and
its ecosystem is unfamiliar.  

But the *takeaway* is simple:
its quite straightforward to write Pig UDFs in Clojure.  This is hardly
a surprise of course.  And if one relegates the *ceremony* of the Java Pig
API to a *recipe*, one can get the job done quite quickly.

Even though I came to this post thinking JRuby would remain  my language of
choice for a UDF, I've come away thinking Clojure gives the best
integration and flexibility working with the Pig API, and probably the
best performance.  

And writing
Clojure, parentheses and all, doesn't feel at all awkward or alien.




