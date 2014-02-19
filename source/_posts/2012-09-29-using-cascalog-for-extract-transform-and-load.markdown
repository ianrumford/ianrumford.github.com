---
layout: post
title: "Using Cascalog for extract transform and load"
date: 2012-09-29 19:30
comments: true
categories: 
---

In two previous posts I explored writing Apache
[Pig](http://pig.apache.org/) *user defined functions* (UDFs) in
[JRuby](http://ianrumford.github.com/blog/2012/09/17/writing-apache-pig-udfs-in-jruby/)
and
[Clojure](http://ianrumford.github.com/blog/2012/09/21/writing-apache-pig-udfs-in-clojure/)
 to process log data, specifically an *auditd* log.


Pig is a very good option for *extract transform and load* of log files but it
is by no means the only option available.  Others include
[Fluentd](https://github.com/fluent/fluentd),
[Flume](https://cwiki.apache.org/FLUME/),
[Scalding](https://github.com/twitter/scalding) and
[Cascalog](https://github.com/nathanmarz/cascalog).  Most of the
others also
have or use Hadoop as the processing / execution environment.

This post focuses on the same log data processing scenario as the
earlier posts, this time using Cascalog, to parse the
auditd log (available [here](https://gist.github.com/3804014))
and load the fields into a HBase table.

<!-- more -->

Nathan Marz, the author of Cascalog,
[describes](http://nathanmarz.com/blog/introducing-cascalog-a-clojure-based-query-language-for-hado.html)
it as

> a Clojure-based query language for Hadoop

Cascalog [source](https://github.com/nathanmarz/cascalog) is available from github and it has a [wiki](https://github.com/nathanmarz/cascalog/wiki)
with some useful articles, albeit incomplete and lagging behind the
code.

Like Pig, Cascalog takes care of building the low-level Haddoop job
artefacts but offers a higher level data abstractions.  However,
Cascalog uses [Cascading](http://www.cascading.org/) for managing its
interface with Hadoop.  

Cascading describes itself on its home page
as:

> Cascading is an application framework for Java developers to quickly and easily develop robust Data Analytics and Data Management applications on Apache Hadoop


# Why Cascalog? #

Nathan gave a [presentation](http://www.slideshare.net/nathanmarz/cascalog) to the Bay Area Clojure User Group in which
he summarised Cascalog's features and differentiators as

* Features (slide 6)
   * Inner and outer joins
   * Aggregators
   * Functions
   * Subqueries
   * Sorting
   * High Performance

and

* Differentiators (slide 6) as
   * Super simple
   * Full power of Clojure always available
   * Easy to extend with custom operations
   * Dynamic queries
   * Arbitrary inputs and output

He could have added
[composability](http://en.wikipedia.org/wiki/Function_composition_(computer_science),
a feature (of Clojure really) which allows complex functions to be
easily built out of simpler building block functions ("functional lego".

It is also worth emphasising the *Full power of Clojure always
available* differentiator:  a Cascalog program *is* a Clojure program and the former can
use the functions and facilities of the latter in a very natural way.
Contrast this flexibility  with Pig Latin and its *user defined functions* where the API
between the former and latter is defined and constrained.


# The Cascalog project.clj file #

Since Cascalog programs are  Clojure ones, the natural way
to manage a Cascalog project  is to
use [Leiningen](https://github.com/technomancy/leiningen) (*lein*). (The Clojure Pig UDF
[post](http://ianrumford.github.com/blog/2012/09/21/writing-apache-pig-udfs-in-clojure)
delves into Leiningen a bit deeper.)

My project.clj for this  project looks like this

<script src="https://gist.github.com/3800631.js?file=project.clj"> </script>

Notes:

* Cascalog is just another dependency; the latest version (1.10.0) is
  used)

* the *conjars* repo is required for Twitter's Maple Cascading *taps*
  for  HBase support


# Explaining Cascalog #

This isn't a tutorial about Cascalog per se, I don't
understand anywhere near enough yet to do it justice.

The two original and most cited tutorials are [here](http://nathanmarz.com/blog/introducing-cascalog-a-clojure-based-query-language-for-hado.html) and [here](http://nathanmarz.com/blog/new-cascalog-features-outer-joins-combiners-sorting-and-more.html).

Notwithstanding,  I will try to explain the *very* basics else the rest of the
post will be nearly impenetrable.

# A simple Cascalog query #

Cascalog query syntax is based on
[Datalog](http://en.wikipedia.org/wiki/Datalog) which is itself a
subset of Prolog. 

Cascalog queries look perfectly at home to anybody at all familiar with
Clojure and Lisp-like syntax.  Below  is the key line in the program below to read all the lines
from the file and print them to *stdout*.

``` clojure

(?<- (stdout) [?line] (file-tap :> ?line))

```

Some notes:

* the *?<-* both defines the query (*<-*) and executes (*?-*) it

    ?<- is actually a Clojure (compile-time) macro

* the *file-tap* is a *[generator]((https://github.com/nathanmarz/cascalog/wiki/Guide-to-custom-operations)* that reads all the lines of the file

    This generator is actually a Cascading
   [source tap](http://docs.cascading.org/cascading/2.0/userguide/html/ch03s05.html).
   
    In very (very!) simplistic terms a generator is an implicit *for
   loop*.  In the query above, the variable *?line* is set to the
   contents of each line
   of the log file for each "loop".
   
    The *lfs-textline* function creates a source tap for the audit log
   held on the local filesystem.  There are many taps
   available, not only for the local filesystem, but also for HDFS
   (e.g. hfs-textline),
   [HBase](https://github.com/Cascading/maple), [Cassandra](http://cassandra.apache.org/), etc.
   
* the *(stdout)* is another tap, this time a *sink* to stdout

* the *[?line]* specified the variable to be output; it can have many
  variables though

The full code looks like this:

<script src="https://gist.github.com/3799669.js?file=print_file.clj"> </script>

To run the code using *lein* with the auditd log file
*./data/blog_audit1.log*, whilst in the project folder, type:

``` bash

lein run -m 'aud_cas.print_file' "./data/blog_audit1.log"

```

# A more useful query #

Simple enough but printing a file using Cascalog is rather an overkill.  Let's turn to
some code to print the audit log *type*, *node* and *msg* i.e. parsing
the
fields in the prefix of each record and printing them.

The complete code is 

<script src="https://gist.github.com/3804493.js?file=print_fields.clj"></script>

Notes:

* *parse_input_record* is exactly the same as by the Pig UDFs

* *query-log-lines* creates (*<-*) a Cascalog query to generate each
   line of the auditd log.
   
    Queries are one of the three types of generators, the other two being
   Clojure sequences and Cascading taps.
    
* *query-log-tuples* creates a query to generate the fields of each log
   record
   
    This function uses the query created by
   *query-log-lines* to *compose* the more complex tuple query.
   
    Each log line is passed to another *generator* called
   *dmo-parse-log-record-to-tuple* to parse the record and return the
   values of *type*, *node* and *msg*.
   
* *dmo-parse-log-record-to-tuple*

    This function is created by the Cascalog
    *[custom operation](https://github.com/nathanmarz/cascalog/wiki/Guide-to-custom-operations)*
    macro *defmapop*.
    
    A function defined by *defmapop* *must* return a tuple.
    (A "tuple" is a Clojure vector e.g. ["a" 6 :my-keyword])
    
    The actual
   parsing is done by the *parse_input_record* function as in the Pig examples.

As before, running the code using *lein*, looks like this:

``` bash

lein run -m "aud_cas.print_fields" "./data/blog_audit1.log"

```

and the output like this:

<script src="https://gist.github.com/3804027.js?file=*scratch*.el"> </script>

# Using Cascalog to save the auditd fields tuple to HBase #

The next step is to show how to do exactly the same as used Pig Latin
scripts +
JRuby and/or Clojure UDFs examples:  parsing the auditd record; creating and adding a *uuid* for each record;
adding a random integer (*unique*) and a boolean (*passed*) fields;
and then
saving the fields to a HBase table (*blog_cascalog_tuple*) and column
family (*record*).

The complete code looks like this, a lot is common to the print
example:

<script src="https://gist.github.com/3804543.js?file=save_fields.clj"></script>

Notes:

* three new *defmapop* functions are defined

    The three new functions (*dmo-uuid*, *dmo-unique*, and
    *dmo-passed*) generate their respective variables.
    
* a function *hbase-tap* to make a HBase tap is added

Its worth looking at the key function (*save-fields*) and walking
through it:

``` clojure

(defn save-fields
  "Use cascalog to save the prefix fields of an auditd log into HBase"
  [log-path]
  (let [q-tuples  (query-log-tuples log-path)
        hbase-sink (hbase-tap "blog_cascalog_tuple" "?uuid" "record"  "?type" "?node" "?unique" "?passed"  )]
    (?<- hbase-sink  [?uuid ?type ?node ?unique ?passed] (q-tuples :> ?type ?node ?msg) (dmo-uuid :> ?uuid) (dmo-passed :> ?passed) (dmo-unique :> ?unique))))

```

1. the generator for the log tuples (*q-tuples*) is created in the
same way as the print example above.

2. a HBase tap is created for the table
(*blog_cascalog_tuple*), the *row-key* is identified as the *?uuid*
variable, column family given as *record* and
the  column qualifiers *?type*, *?node*, *?unique* and *?passed* are given.

    Note the '?' prefix to each qualifier and row-key.

3. the Cascalog query itself now includes additional generator
calls (*dmo_uuid*, etc) to set the *?uuid*, *?unique* and *?passed* variables.

4. the *[?uuid ?type ?node ?unique ?passed]* specifies the
variables to be output (saved) to HBase; *msg* has been ignored.

5. *hbase-sink* is given as the output tap.

For completeness, running the code using *lein*, looks like this:

``` bash

lein run -m "aud_cas.save_fields" "./data/blog_audit1.log"

```

You can use  the HBase shell to see the table e.g.:

``` bash

/usr/lib/hbase/bin/hbase shell

scan 'blog_cascalog_tuple'

```

# Doing a bit more - filtering records on field values #

The examples so far haven't touch on Cacaslog ability to filter and
constrain variables using
*predicates*.  Filters can be easily composed to allow arbitrary and powerful
processing pipelines to be built.

The example predicate clause below filters all values of ?age
greater  or equal to 30 is, the *<* is the actual predicate and the
rest input parameters.  Note the (Clojure) *<* operator has the
familiar
less-than semantics you'd expect.

``` clojure

(< ?age 30)

```

As a trivial example, say we wanted to filter on auditd records
selecting only
login requests (where  *type* is *LOGIN*).

The code is available [here](https://gist.github.com/3804733)
but its nearly identical with the code above, the crucial changes
are in the old
*save-fields* function, now called *filter-fields*:

``` clojure

(defn filter-fields
  "Use cascalog to filter save the prefix fields of an auditd log into HBase"
  [log-path]
  (let [q-tuples  (query-log-tuples log-path)
        f-tuples  (<- [?t ?n ?m] (q-tuples :> ?t ?n ?m) (= ?t "LOGIN"))  ;; Filter on LOGIN records
        hbase-sink (hbase-tap "blog_cascalog_tuple" "?uuid" "record"  "?type" "?node" "?unique" "?passed"  )]
    (?<- hbase-sink  [?uuid ?type ?node ?unique ?passed] (f-tuples :> ?type ?node ?msg) (dmo-uuid :> ?uuid) (dmo-passed :> ?passed) (dmo-unique :> ?unique))))

```

Notes

* a new query has been defined called *f-tuple*

    *f-tuples* uses the *q-tuples* query and applies a filter that
     compare the *type* with '*LOGIN*'; if the compare fails, the record
     will not appear in the *f-tuples* query.
     
* the main query now uses *f-tuples* rather than *q-tuples*

    By using *f-tuples* *only* login request will be saved to HBase.
    
    
It doesn't require much imagination to think of some of the useful
permutations filtering offers.

# Final Words #



Cascalog is hugely impressive and, together with Cascading and Clojure
 presents a compelling and powerful ecosystem for manipulating
 and analysing large data sets.  Investing in learning the ecosystem well
 will pay rich dividends.
 
 The down side is that the learning curve for a complete beginner is
 quite steep and whilst of great use, the limited documentation,
 tutorial and blog material available make the hill harder to climb.
 Maybe this post will help a fellow traveller.

# Fess Up Time #

*The code to save the fields to HBase doesn't work in the desired way:
the name of the column qualifiers retain the '?' in them.  Whether my
ignorance, a
bug or whatever, I don't know yet.  I've asked
a [question](https://groups.google.com/forum/?hl=en&fromgroups=#!topic/cascalog-user/zu0gXPjmj8s) on the Cascalog Google Group and will update this post when I
have more to tell.*
