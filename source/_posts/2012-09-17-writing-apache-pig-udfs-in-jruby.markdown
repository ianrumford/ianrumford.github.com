---
layout: post
title: "Writing Apache Pig UDFs in JRuby"
date: 2012-09-17 12:48
comments: true
categories: 
---

Apache [Pig](http://pig.apache.org/) is part of the
[Hadoop](http://hadoop.apache.org/) MapReduce ecosystem.

Pig describes itself as:

> "a platform for analyzing large data sets that consists of a
high-level language for expressing data analysis programs, coupled with infrastructure for evaluating these programs. The salient property of Pig programs is that their structure is amenable to substantial parallelization, which in turns enables them to handle very large data sets."

Pig pre-empts  the need to write lower-level MapReduce jobs to process
large datasets.  It has its own "high level language" for data
manipulation and relationships called
[Pig Latin](http://pig.apache.org/docs/r0.10.0/basic.html).  Pig Latin
is really an external domain specific language (*DSL*) written in Java.

Pig is a very good tool for extract-transform-and-load (*etl*) 'duct
tape' tasks e.g.
taking data from text-base log files, parsing the records and loading
the extracted fields
into e.g a HBase table.

But Pig Latin is both a strength and weakness.  A strength because, once
the basics have been learnt, useful stuff can be done pretty quickly.
A weakness because its another thing to learn and quite orthogonal to
anything else in the Hadoop world.  Although written in Java, a Pig
Latin script can't contain Java directly.

Contrast this with
[Cascalog](https://github.com/nathanmarz/cascalog/wiki) - a competitor
to Pig - that provides its own data manipulation and relationship DSL but in a [Clojure](http://clojure.org/)
environment - Cascalog programs can directly define and use Clojure
functions and use all the facilities of the Clojure language
to do their work.

Recognising a limitation, the authors of Pig support the creation of
[user defined functions](http://pig.apache.org/docs/r0.10.0/udf.html)
(*UDFs*).  The pretty good documentation for UDFs shows how to create
them in Java, Python, JavaScript and, with the
[latest](http://pig.apache.org/releases.html) release (0.10.0) of
 of Pig, [JRuby](http://jruby.org/).

Writing JRuby UDFs is very straightforward and easy once the *recipe*
has been learnt.  The authors of the JRuby
support have done a very good job at minimising the *impedance mismatch*.
(The chronology in this [link](https://issues.apache.org/jira/browse/PIG-2317)
demonstrates how the shape of the final support was
arrived at by smart,
mutually respectful and talented people. An example, IMHO, of the best in
open source.)

BTW I was lead to looking at JRuby UDFs by this very good [post](http://hortonworks.com/blog/pig-as-hadoop-connector-part-two-hbase-jruby-and-sinatra/) by 
Russell Jurney.

In the rest of the post, I explore the creation of a JRuby UDF to
*etl* a log file into HBase.  In my example, the log file was from a
Ubuntu audit subsystem log (*auditd*).

<!-- more -->

# Install HBase #

The latest version of HBase (0.94.1) will be used.

A simple recipe for HBase was given in Russell's post, here's my minor
variant:

```  bash

cd /tmp
wget http://archive.apache.org/dist/hbase/hbase-0.94.1/hbase-0.94.1.tar.gz
tar -xvf hbase-0.94.1.tar.gz
mv hbase-0.94.1.tar.gz /usr/lib/hbase
chown -R root.root /usr/lib/hbase
mkdir /var/hbase

```

The file /usr/lib/hbase/conf/hbase-site.xml should be like this:

``` xml

<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>file:///var/hbase</value>
  </property>
</configuration>

```

And start HBase:

``` bash

/usr/lib/hbase/bin/start-hbase.sh

```

# Install Pig #

The Pig download and install is similar to HBase:

``` bash

cd /tmp
wget http://mirror.ox.ac.uk/sites/rsync.apache.org/pig/pig-0.10.0/pig-0.10.0.tar.gz
tar -xvf pig-0.10.0.tar.gz
mv pig-0.10.0 /usr/lib/pig
chown -R root.root /usr/lib/pig

```


# Creating the HBase table and column family #

The easiest way to create the table (e.g. *blog_udf1*) and column
family (e.g. *record*) is  using the
HBase shell (which is itself a JRuby program).

``` bash

/usr/lib/hbase/bin/hbase shell

HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.94.1, r1365210, Tue Jul 24 18:40:10 UTC 2012

hbase(main):001:0> create 'blog_udf1', 'record'
0 row(s) in 0.5590 seconds

hbase(main):002:0> quit

``` 


# Executing a Pig Latin script #

To run a Pig Latin script, the Java classpath needs to be set:

``` bash

export PIG_CLASSPATH=/usr/lib/hbase/*:/usr/lib/hbase/lib/*:/usr/lib/pig/*

```

The examples below execute Pig in *local mode* i.e. run the job on the
local machine (c.f. a Hadoop cluster).  For example, where
*blog_udf1.pig* is the  Pig Latin script in my *bin* directory:

``` bash

/usr/lib/pig/bin/pig -x local ../bin/blog_udf1.pig

``` 

# Some example auditd log records #

The auditd log is fairly regularly formatted.  Most lines have a
*prefix*, delimited with a colon (:), including the *type* field,
followed by type-specific data.  Note the *node* is the host name.

NOTE:  this data has been selected for demonstration purposes; not all
records fit into this regular pattern (but most do).

``` bash

node=cdh4flumevm1 type=DAEMON_START msg=audit(1342114506.467:9723): auditd start, ver=1.7.18 format=raw kernel=3.2.0-26-generic auid=4294967295 pid=1054 subj=unconfined  res=success
node=cdh4flumevm1 type=CONFIG_CHANGE msg=audit(1342114506.571:24): audit_backlog_limit=8192 old=64 auid=4294967295 ses=4294967295 res=1
node=cdh4flumevm1 type=CONFIG_CHANGE msg=audit(1342114506.571:25): audit_failure=2 old=1 auid=4294967295 ses=4294967295 res=1
node=cdh4flumevm1 type=CONFIG_CHANGE msg=audit(1342114506.579:105): audit_enabled=1 old=1 auid=4294967295 ses=4294967295 res=1
node=cdh4flumevm1 type=LOGIN msg=audit(1342114506.751:106): login pid=1104 uid=0 old auid=4294967295 new auid=104 old ses=4294967295 new ses=1
node=cdh4flumevm1 type=LOGIN msg=audit(1342114517.503:107): login pid=1447 uid=0 old auid=4294967295 new auid=1000 old ses=4294967295 new ses=2
node=cdh4flumevm1 type=SYSCALL msg=audit(1342114517.511:108): arch=c000003e syscall=87 success=no exit=-2 a0=e273d0 a1=0 a2=e22620 a3=7ffffcd967e0 items=1 ppid=1447 pid=1539 auid=1000 uid=1000 gid=1000 euid=1000 suid=1000 fsuid=1000 egid=1000 sgid=1000 fsgid=1000 tty=(none) ses=2 comm="gnome-keyring-d" exe="/usr/bin/gnome-keyring-daemon" key="delete"
node=cdh4flumevm1 type=CWD msg=audit(1342114517.511:108):  cwd="/"
node=cdh4flumevm1 type=PATH msg=audit(1342114517.511:108): item=0 name="/tmp/keyring-GZxINJ/control" inode=1574066 dev=08:01 mode=040700 ouid=1000 ogid=1000 rdev=00:00
node=cdh4flumevm1 type=SYSCALL msg=audit(1342114517.547:109): arch=c000003e syscall=93 success=yes exit=0 a0=9 a1=3e8 a2=3e8 a3=7fff7d9907c0 items=1 ppid=987 pid=1447 auid=1000 uid=1000 gid=1000 euid=1000 suid=0 fsuid=1000 egid=1000 sgid=0 fsgid=1000 tty=(none) ses=2 comm="lightdm" exe="/usr/sbin/lightdm" key="perm_mod"

```

The log is stored in *./data/blog_audit1.log* in the Pig Latin scripts below.

# Using a JRuby UDF returning scalars to etl audit data #

## The Scalar JRuby UDF ##

Russell's post, linked above, shows a very simple example of using JRuby
to generate a *uuid*, returning it (as a string / chararray).

The below builds on that example to return both a uuid, to be used as
the HBase *rowkey*, and the *type*
of the parsed auditd log entry.

``` ruby

require 'pigudf'

require 'java'  # Magic line for JRuby - Java interworking
 
import java.util.UUID
 
class JRubyUdf < PigUdf
  
  outputSchema "uuid:chararray"
  def uuid()
    java.util.UUID.randomUUID().toString()
  end
  
  outputSchema "type:chararray"
  def type(auditRecord)
    
    (recordPrefix, recordData) = auditRecord.split(':')

    # Generate a hash of all the prefix fields:  node, type and msg
    prefixFields = Hash[*recordPrefix.split(' ').map {|t| t.split('=')}.flatten]

    prefixFields['type'] # return the type

  end
  
end

```

Notes:

* the *require "pigudf"* provide the link with the Pig world and
facilitate easy Pig-to-JRuby communication and interworking.

* the JRuby class must  subclass of *PigUdf*

* the class name *seems* to be arbitrary - the Pig Latin script
  defines the mnemonic used to refer to the class.
  
* the *outputSchema* defines the *data type* of the return value from the
  method named (i.e. a chararray / string for both *uuid* and *type*)

## The Scalar Pig Latin script ##

The Pig Latin script (e.g. *./bin/blog_udf1.pig*) is minimal:

``` java

/* Load auditd data into Hbase using a JRuby UDF for processing */

/* The UDF returns Scalars (chararays / strings)

/* Register the JRuby UDf with Pig */

register './lib/blog_udf1.rb' using jruby as blog_udf1;

auditd_records = load './data/blog_audit1.log' using PigStorage()
                      as (record:chararray);

/*

Generate a collection of tuples with two fields: an id (a uuid) and
the type from the auditd record

*/

auditd_types = foreach auditd_records generate blog_udf1.uuid() as id, 
                       blog_udf1.type(record) as type;


/* 

Store to the HBase table 'blog_udf1', column family 'record' using the uuid
as the row key by specifying  the loadKey option. 

*/

store auditd_types into 'hbase://blog_udf1' using
     org.apache.pig.backend.hadoop.hbase.HBaseStorage('record:type', 'loadKey true');

```


# Using a JRuby UDF returning a hash / map to etl audit data #

## The Hash / Map JRuby UDF ##

So far, so good.  But a more natural way to return a collection of
fields and their values would be in a hash.

The example above has been reworked below to return all of the prefix fields
(*type*, *node*, *msg*) from the audit record, 
as well as an *id* key with a value generated by the *uuid* method.

Also, two new fields has been added (*unique* and *passed*) to
demonstrate returning integer and boolean values.

``` ruby

require 'pigudf'

require 'java'  # Magic line for JRuby - Java interworking
 
import java.util.UUID
 
class JRubyUdf < PigUdf

  outputSchema "parse:map"
  def parse(auditRecord)
    
    (recordPrefix, recordData) = auditRecord.split(':')

    # Generate a hash of all the prefix fields:  node, type and msg
    prefixFields = Hash[*recordPrefix.split(' ').map {|t| t.split('=')}.flatten]

    prefixFields['id'] = uuid # add the id with a uuid

    prefixFields['unique'] = rand(1000000) # Add an *integer*
    prefixFields['passed'] = 'true'  # Add a boolean BUT as a string
    
    prefixFields # return the hash; it will become a Pig Map

  end
  
  private
  
  def uuid()
    java.util.UUID.randomUUID().toString()
  end
  
end

```

Notes:

* the outputSchema for the *parse* method is *map*

* the *parse* method returns a regular JRuby hash


## The Hash / Map Pig Latin script ##

The Pig Latin script is more complicated as it has to accept the JRuby
Hash, extract and cast the wanted keys' values and store all of them in
the HBase table (remember to create it using the HBase shell: create 'blog_udf2', 'record'):

``` java

/* Load auditd data into Hbase using a JRuby UDF for processing */

/* The UDF returns a Map / Hash */

/* Register the JRuby UDf with Pig */

register './lib/blog_udf2.rb' using jruby as blog_udf2;

auditd_records = load './data/blog_audit1.log' using PigStorage()
                      as (record:chararray);

/*

Generate a collection of tuples with the fields from the returned hash

Note the fields' values have to be cast appropriately 

*/

auditd_fields = foreach auditd_records {
     hash = blog_udf2.parse(record);
     generate (chararray) hash#'id', (chararray) hash#'type', (chararray) hash#'node', (int) hash#'unique', (boolean) hash#'passed';
     };

/*

The describe should display:

auditd_fields: {chararray,chararray,chararray,int,boolean}

*/

describe auditd_fields;

/* 

Store to the HBase table 'blog_udf2', column family 'record' using the uuid
as the row key by specifying  the loadKey option. 

*/

store auditd_fields into 'hbase://blog_udf2' using
     org.apache.pig.backend.hadoop.hbase.HBaseStorage('record:type record:node record:unique record:passed', 'loadKey true');

```

Notes:

* although the returned hash has a *msg* key-value pair, it is ignored (discarded)
  in the *generate*

* The string values in the hash have to be cast to *chararray* on the
  generate
  
* the integer (fixnum) values in the hash have to be cast to *int*
    or *long* on the generate (although they will become byte arrays
    in HBase)
    
* a boolean value must be passed back as a string, either
  *true* or *false*, and cast to *boolean* on the generate (but again
  will be stored as a byte array by HBase)
  
* the *describe* command displays the types of the various fields in
  the new tuple e.g
  chararray, int and boolean
  
* the store into HBase now includes the other fields as well as the *type*

# Final Words #

Being able to write Pig UDFs in JRuby (and I guess Python) makes Pig a
very nice solution for data manipulation, especially in an *etl* pipeline.

I'm  intrigued how easy it would be to write a UDF in Clojure, where
the impedance mismatch between Pig and the UDF should
be minimal as well, and performance close to a native Java UDF.  That will be another post.


