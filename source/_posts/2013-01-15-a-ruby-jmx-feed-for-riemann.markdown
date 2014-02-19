---
layout: post
title: "A Ruby JMX Feed for Riemann"
date: 2013-01-15 15:30
comments: true
categories: 
---

There are a plethora of monitoring tools available today, the range of
choice catering for most every need and
for all sorts of application and / or infrastructure complexity.  And size of chequebook too although price is
no real guide to quality, flexibility or ease of use.

Many, perhaps most, of the best solutions are open source and
cross-platform (Linux/Unix and Windows).  Some have great features,
battle-hardened reliability, excellent support and documentation;
and helpful and engaged communities underpinning them.   The
grand-daddy is undoubtedly [Nagios][Nagios].  I'm no expert in Nagios
but there is no doubt  it
is a magnificent achievement.

Do we really need another solution?  Probably not.  But
[Riemann][Riemann] is interesting and 
relevant  because its not a monitoring tool *per se* but an *event stream
processor*.  Riemann is a more a scion of the same family as (the also hugely
impressive) [Esper][Esper].  

Simply put Riemann is, in my words, a *traffic cop* for events coming in
from multiple *sources*
and going out to multiple *sinks*.  

This post is about creating a Ruby source client for JMX data using Hadoop and
HBase as the data sources. It does not go into how to build a Hadoop /
HBase system (Apache [BigTop 0.5.0](https://blogs.apache.org/bigtop/entry/apache_bigtop_0_5_0) has just released and may be the
easier route - I've not used it). 

For completeness, I've included some [background](#riemannsummary) on
Riemann, and simple guidance on how to run a Riemann server, and also how to
[configure](#configurejmx) Hadoop & HBase for JMX.  If
you are unfamiliar with either, you may find it beneficial to  read them first, before the
sections on the feed.

The below was tested on Ubuntu 12.04 using Ruby 1.9.3 (jRuby 1.7.2) and
Cloudera's
[CDH4 YARN distribution](https://ccp.cloudera.com/display/CDH4DOC/CDH4+Quick+Start+Guide)
running in *pseudo-distributed* (i.e. one host) mode.

<!-- more -->

# Some background to Ruby JMX feeds for Riemann  #

After swapping a couple of emails with *@aphyr* I decided to loosely
model my feeds on the way *riemann-tools* worked, specifically using 
Riemann's reference Ruby client to submit events to
Riemann.

It seems one of the first attempts to write a JMX feed in Ruby for
Riemann was [alcy's](http://alcy.github.com/)
[HBase](https://github.com/alcy/riemann-hbase) example.  (This is the
one referred to on the [clients](http://riemann.io/clients.html) page
on the Riemann website.)  alcy's example gave me the critical 'first
steps' knowledge to try something more adventurous.

Although this post focuses on the JMX feed, the code has been written
with the likely need to have (integrate) other types of feeds for other sources, not just
JMX ones.

Ruby is an ideal language to prototype this sort of framework and the
below is more about writing that framework in Ruby since submitting
events to Riemann
side is so easy as to be essentially trivial.


## Preparation ##

There are some pre-requisites.

### Installing jRuby ###

My code takes advantage of the existing support in [jRuby][jRuby] for JMX provided by
[Jeff Mesnil's](http://jmesnil.net/weblog/) 
[jmx4r](https://rubygems.org/gems/jmx4r) gem.  

Full details on how to install jRuby are available on the [wiki](https://github.com/jruby/jruby/wiki) but
the following install of 1.7.2 worked fine for me:

```bash
cd /tmp
wget http://jruby.org.s3.amazonaws.com/downloads/1.7.2/jruby-bin-1.7.2.tar.gz
tar -xvf ./jruby-bin-1.7.2.tar.gz
mkdir /usr/lib/jruby
mv jruby-1.7.2 /usr/lib/jruby/1.7.2
ln -f -s /usr/lib/jruby/1.7.2/bin/jruby /usr/bin/jruby
ln -f -s /usr/lib/jruby/1.7.2/bin/jrubyc /usr/bin/jrubyc 
ln -f -s /usr/lib/jruby/1.7.2/bin/jruby.rb /usr/bin/jruby.rb
```

### Installing the gems ###

Three gems are needed:

```bash
sudo jruby -S gem install jmx4r riemann-client riemann-feeds
```

# Example 0: a simple NameNode feed #

One of the key components of Hadoop is the [HDFS NameNode](http://wiki.apache.org/hadoop/NameNode) daemon. 

## Example 0: the code ##

Here's the code:

<script src="https://gist.github.com/4474713.js?file=blog_feed_jmx0.rb"></script>

## Example 0: Notes ##

Quite a few things to note:

* The first line specifies jruby

* Running with debug enabled (*$DEBUG = true*) will produce lots of
  diagnostics

* The feed handler (*Riemann::Feeds::FeedHandler*) manages feeds

  The feed handler instantiates and runs feeds.
  
  It (will) also do some other
  stuff but that's for a later post.  
  
* The feed specification (*feedspec_namenode*) is just a regular hash

  The feed specification can only have three (symbol) keys: *type*, *name* and
  *configuration*.
  
  The *configuration* will be passed transparently to the instantiated
  instance of the feed type class - see next.

* The feed type is *JMX*
  
  The feed type is the name of the class that must be found in
  *Riemann::Feeds::Type* e.g. *Riemann::Feeds::Type::JMX*.  
  
  If the feed class is not
  already loaded, the feed handler will attempt to do so by requiring
  *'riemann/feeds/type/jmx'*.
  
* Configuring the feed

  I've tried to make it easy to configure feeds flexibly and the
  capabilities look beyond  the needs of this simple example -  the
  following  two example explores this more.
  
  For example, the *config_riemann*, *credentials_jmx* and
  *jmxconfig_namenode* hashes provide 'external' (to the
  feedspec_namenode) hash are 'pulled into' the *configuration* by their
  inclusion in the *include_configuration* array.  The former's key-value
  pairs are used the set the attributes (accessors) of the feed instance.
  
  The purpose of 'external' configuration is three-fold:  First many feeds will have common
  configuration needs e.g. the details of the Riemann server
  (*config_riemann*) to submit events to.  
  
  Secondly, credentials such as the JMX username and
  password really should not be kept in scripts  but in protected (e.g.
  permissions 600) files somewhere else.
  
  Finally, configuration files can be updated without any changes to the
  feed scripts themselves.
  
* Setting the Riemann event defaults

  The *configuration* can contain a hash with the default event fields' values
  to use (e.g. *service* as *'NameNode'*).
  
* Defining the JMX beans to monitor

  The *beans* key in the *configuration* defines the beans to monitor.  It
  must be a array.
  
  Two beans are defined in this examples, *NameNodeActivity* and the
  ubiquitous *OperatingSystem* one containing attributes such as  cpu time,
  swap usage, load average, free memory, etc.
  
  Normally beans are specified by a hash specification (described
  later in Example 1).
  Just supplying the bean name (string) is a shorthand to 'fake up' a
  specification with the
  defaults.
  
  Using the defaults for a bean specification   tells the feed
  instance to ask the bean what attributes it has and report  them
  all to Riemann.
  
## Example 0: the each_event method ##

The feed instance  dynamically defines an *each_event*  method on its
  [*singleton / eigenclass*](http://blog.madebydna.com/all/code/2011/06/24/eigenclasses-demystified.html).
  
The *each_event* method for the JMX class is an [iterator](http://www.ruby-doc.org/docs/ProgrammingRuby/html/tut_containers.html) that
  yields events with *metric* and *description* for all of the
  attributes to be reported on. 
  
Running with $DEBUG enabled will show exactly what the *each_event*
  method does.  Here it is for this example (slightly tidied up for
  ease of reading):
  
<script src="https://gist.github.com/4524629.js?file=blog_feed_jmx0_each_event.rb"></script>

# Example 1:  easy attribute selection and configuration  #

In the previous example all of the attributes for each bean were
reported on (the default).  Sometimes though
this is not what you want and e.g. maybe only a few specific, nominated
attributes should be reported.

Sometimes metric values may need to be normalised, for example to
always report storage in gigabytes, or time in your chosen radix
(e.g. seconds since the epoch).

You  may actually want to do some  transformation of
an event (*map*) e.g. to change the *status* from *ok* to *critical*
when a threshold has been breached.

Finally, you  may also want to filter events using a predicate (*select*) that
returns *truthy* or *falsey*.

This example  shows also how common and / or sensitive information can
be read from  external YAML file(s).

It also  demonstrates how bean and / or attribute level event defaults can be set.

## Example 1: external configuration files ##

In this example the *config_riemann*, *credentials_jmx* and
*jmxconfig_namenode* hashes providing 'external' configuration have
been moved (written) to their own individual YAML files in *./etc*.
Writing  YAML files is straightforward  e.g:

```ruby
require 'yaml'

config_riemann = {riemann_host: '10.16.0.64', riemann_port: 5555, riemann_interval: 10}

credentials_jmx = {jmx_user: 'monitorRole', jmx_pass: 'monitorpass'}

jmxconfig_namenode = {jmx_host: '10.16.0.48', jmx_port: 8004}

config_home = './etc'

yaml_files = {
  config_riemann => File.join(config_home, 'config_riemann.yaml'),
  jmxconfig_namenode => File.join(config_home, 'jmxconfig_namenode.yaml'),
  credentials_jmx => File.join(config_home, 'credentialsJMX.yaml'),
}

yaml_files.each do | yaml_data, yaml_path | 
 File.open(yaml_path, 'w') {|yaml_hndl|  YAML.dump(yaml_data, yaml_hndl) }
end
```

The contents of the e.g. *config_riemann* YAML file looks as you might expect:

```yaml
---
:riemann_host: 10.16.0.64
:riemann_port: 5555
:riemann_interval: 10
```

## Example 1: the code to monitor selected attributes of the HDFS NameNode ##

The code looks like this:

<script src="https://gist.github.com/4524859.js?file=blog_feed_jmx1.rb""></script>

## Example 1: Notes ##

* The *config_riemann_path*, *credentials_jmx_path* and
  *jmxconfig_namenode_path* are YAML files.
  
* The values of both the original beans' specifications are now hashes.

  The only keys allowed in the specification hash are
  *event_defaults* and *attributes*, the values for which are expected to
  be hashes, and the *bean_name* which must be a string.
  
## Example 1:  attribute-level event defaults for the NameNode bean ##

If a bean has its own *event_defaults* these are merged with the
feed-level event defaults with normal hash semantics for duplicate
keys (last value wins).

Note however, *tags* array are *concatenated*.  In this example
events for this bean will have additional tags (*Activity* and *Files*) as well
as the feed-level tags (*YARN* and *NameNode*).

## Example 1:  attribute selection for the NameNode bean ##

The *attributes* key hash value can contain four keys: *include*,
*exclude*, *all* and *definitions*.

If no *all* value (array) is provided, all of the bean's attributes are
automatically supplied.

If  *include*  (array) is provided it  takes precedence over the
*all* list.

However, *exclude* attributes (array) are *always* discarded.

The logic looks like this:

```ruby
attrUse = ((attrInc || attrAll) - attrExc).uniq  # list of unique attributes to report on
```

Note all attributes  in *include* and *exclude* and the  keys (i.e.
attribute names)
in *definitions*  *must* be
present in the *all* array (sanity checking).

*definitions* will be explained in the discussion of  the attributes for
the *OperatingSystem* bean below.

## Example 1: attributes for the OperatingSystem bean ##

The *attributes* hash for the *OperatingSystem* bean is rather more
involved.  As for the *NameNodeActivity* bean, the
*include* lists all the wanted attributes, in this case five of them
(e.g *free_physical_memory_size*).

The *attributes* hash also includes a *definitions* key,  also
a hash, that defines how the attribute should be treated (processed):  

* *event-defaults*

 If present, the *event-defaults* value provides attribute-level
 defaults, in this case just an  extra tag (*FreeMem*).
 
* *metric*

  If present, the *metric* value defines the normalisation to be applied to the
  *metric* value.
  
  In this example, the *free_physical_memory_size* will be converted into
  GB using a *proc* taking the metric as its only argument.
  
  In the background, the feed instance creates a method with a unique name (e.g.
  *metric_free_physical_memory_size_123456*) using the proc for its
  body (using *define_method*).
  
* *map*

  If present, the *map* function takes the event as its only argument and
  returns the (maybe modified) event.  Again, as for *metric*,  a uniquely named method is
  created (e.g. *map_free_physical_memory_size_7263828*).
  
  In this example the *status* has been changed to *critical*' because the
  *select* (see next) will *only* pass through events which have crossed the threshold.
  
* *select*

  If present, the *select* must be a predicate function, with the
  event as its only argument, and returning *truthy* or *falsey*.  If the
  latter, the event is discarded and *not* sent to Riemann.
  
  If this case, only events where the *free_physical_memory_size* is
  less that 1GB are submitted to Riemann. (Note the *metric* has been
  previously normalised to GB.)
  
## Example 1: the each_event method ##

Seeing the  generated *each_event*  method may help clarify how the attribute
*definitions* have been used, particularly *free_physical_memory_size*:

<script src="https://gist.github.com/4524871.js?file=blog_feed_jmx1_each_event.rb"></script>

# Example 2:  advanced configuration #

The example covers the same ground as Example 1 but using a more
advanced configuration technique: using Ruby itself to create
the necessary data structures.

Using YAML is very useful in many situations although it has some
drawbacks. For example it is not possible to easily save (serialise)  a *Proc* in a
YAML file.

However its easy straightforward to use Ruby itself to build the
configuration.  Ruby files can be used in the same ways / places
as YAML files.  Arguably using an *active* Ruby configuration is less
*safe* than a *passive* YAML one since the former is executed
(*instance_eval*) on the feed instance.

## Example 2: the code ##

The feed script itself is now pretty minimal with the feed
specification now defined as  a Ruby file (*./etc/feedspecNameNode.rb*):

<script src="https://gist.github.com/4526324.js?file=blog_feed_jmx2.rb"></script>

## Example 2: the NameNode feed specification  ##

The *./etc/feedconfigNameNode.rb* file looks like this:

<script src="https://gist.github.com/4526444.js?file=blog_feed_jmx2_feedspecNameNode.rb"></script>

Notes:

* Normal Ruby semantics

  The full power of Ruby available.
  
* YAML files are used as before for simple configuration

* The two bean specifications refer to external Ruby files
  
  *./etc/jmxbeanconfigNameNodeActivity.rb* and
   *./etc/jmxbeanconfigOperatingSystem.rb* are listed in the *beans* array.

* The result of evaluating Ruby code is the feed specification hash

  *instance_eval* is used to do the evaluation with the feed instance
   as *self*.

## Example 2: the NameNodeActivity bean configuration ##

The *./etc/jmxbeanconfigNameNodeActivity.rb* file contains the same as
in Example 1:

<script src="https://gist.github.com/4526464.js?file=blog_feed_jmx2_jmxbeanconfigNameNodeActivity.rb"></script>

## Example 2: the OperatingSystem bean configuration ##

The *./etc/jmxbeanconfigOperatingSystem.rb* file contains the more
involved specification, again the same as Example 1:

<script src="https://gist.github.com/4526472.js?file=blog_feed_jmx2_jmxbeanconfigOperatingSystem.rb"></script>


## Example 2: using simple building blocks, mix and match ##

Using YAML and Ruby to create simple building block for a feed
specification and configuration allows you to create rich feed *mix and match*
definitions that can be easily added, changed or augmented as needed.

This abillity  comes into it own when monitoring feeds from  multiple
sources (e.g. a whole Hadoop / HBase cluster) where many common beasn
and attributes
need to be collected (for example most if not all hosts will monitor
 *OperatingSystem*).  
 
Also, it makes it easy to build
*templates* for host  *roles* e.g. in a Hadoop cluster with a number
of HDFS DataNodes, you'd likely want to monitor them in the same way,
collecting the same beans and attributes.

# Final Words on Riemann Feeds #

The above is useful for a small number of hosts  but it
would be clumsy to use for, again and e.g., a Hadoop cluster where the
same monitoring will likely be used across the cluster and depending
on the individual hosts' roles.  Building configurations for multiple
hosts, using templates, will be the subject of the next post.

Also, there is nothing in the above to watch over the feeds
themselves (*who watches the watchers?*) and telling Riemann if and when the
feeds fail.  This will be covered in the
next post as well.

A JMX feed is a useful and versatile solution for the Java world but there are many more feed
sources it would be nice to support e.g.
[auditd](http://linux.die.net/man/8/auditd) for security exceptions.  Also
it would be good to have feeds that *gateway* to other monitors e.g.
sucking from Nagios's [nrpe](http://exchange.nagios.org/directory/Addons/Monitoring-Agents/NRPE--2D-Nagios-Remote-Plugin-Executor/details).

Finally, anyone reading this post hoping to discover how to configure
the *sink* side of Riemann will have been sorely disappointed.  As most
infrastructure folk will tell you, good monitoring is all about the
tuning of what you do will all the data (and when you get the DevOps
out of bed).  This is part science, part art and part experience so a
prescriptive approach will only get you to first base.  But is will be
interesting to explore some utility "recipes" for common concerns.


# <a id="riemannsummary"></a>Riemann Overview #

The Riemann [website][Riemann] has recently had a make over and is
very good, well worth browsing.

Riemann's author is [Kyle Kingsbury](http://aphyr.com/)

## Riemann Features ##

Riemann adds some deceptively simple but very elegant ways to
filter, (re)direct or aggregate inbound events.  

It is completely *agnostic* as to
the characteristics of the event sources and sinks.  

It also doesn't care much (intrinsically) as the
semantics of the event data: as long as the event itself observes a
very simple format, restricting  itself to a hash / map with a  set of
easily understood and familiar
[fields named & value pairs](http://riemann.io/concepts.html),
Riemann will process it successfully. The meaning of the fields' values
are a matter for the sources and sinks: if source X says *The backup
has failed* its up to sink Y to say *get the on-call DevOps out of bed*.

Riemann is written in [Clojure][Clojure] and brings the ability to
*compose* event pipelines in really simple but intuitive ways.
Arguably its most powerful feature.

## Riemann Events ##

A Riemann event is a hash / map.  An example  Ruby one:

```ruby
{host: 'server01', service: 'HBase', status: 'ok', metric: 5, :description: 'No of
HBase users', tags: ['HBase', 'Users']}
```

The valid fields (keys) are listed 

## Riemann Clients ##

Riemann provide  clients for anumber of  different lanagues bindings for building
custome sources including reference ones in
[Ruby](https://github.com/aphyr/riemann-ruby-client), [Java](https://github.com/aphyr/riemann-java-client) and
{Clojure](https://github.com/aphyr/riemann-clojure-client).  There are
also clients for  othe rpopular langaes such as Python, Perl, Node, C, Erlang and Scala.

## Riemann Sinks ##

A number of *sinks* are already available including [Graphite](http://riemann.io/howto.html#forward-to-graphite),
[PagerDuty](http://riemann.io/howto.html#notify-with-pagerduty) and
[email](http://riemann.io/howto.html#send-email).  Riemann servers can
also be daisy-chainen if need be e.g for load sharing, aggreation
(e.g. by location, service, etc), separation
of duties, whatever.

# Running a Riemann Server #

Running a Riemann server and feeding it some events is really very easy and described in the
[QuickStart](http://riemann.io/quickstart.html) section on the
website.  This section is drawn from it.

## Installing Riemann ##

The 0.1.5 version can be installed using the following:

```bash
wget http://aphyr.com/riemann/riemann-0.1.5.tar.bz2
tar xvfj riemann-0.1.5.tar.bz2
cd riemann-0.1.5
```

The expanded tar file creates three folders.  The *lib* folder contains
the Riemann  (v0.1.5) jar (*riemann-jar*), *etc* contains an example
configuration (*riemann.config* - see below)
and *bin* which contains a simple script (*riemann*) to run Riemann with the
example, or your own, configuration.

I've found a problem with *tags* (see later) support when using the 0.1.5  and am
currently using the unreleased 0.1.6-SNAPSHOT from Github.  To install
0.1.6 requires [Leiningen][Leiningen] to be installed (see
[another of my posts](http://ianrumford.github.com/blog/2012/09/21/writing-apache-pig-udfs-in-clojure/)
for some guidance if necessary).

```bash
cd <wherever you want to keep riemann source>
git clone https://github.com/aphyr/riemann.git
cd riemann
lein uberjar
```

The *target* directory will contain the *standalone* jar
*riemann-0.1.6-SNAPSHOT-standalone.jar*.

## Configuring Riemann ##

The *etc* folder contains an example
configuration for running tcp, udp and ws (websockets) servers.

The configuration file is a snippet of Clojure:

```clojure
(logging/init :file "riemann.log")

; Listen on the local interface over TCP (5555), UDP (5555), and websockets
; (5556)
(let [host "127.0.0.1"]
  (tcp-server :host host)
  (udp-server :host host)
  (ws-server  :host host))

; Expire old events from the index every 5 seconds.
(periodically-expire 5)

; Keep events in the index for 5 minutes by default.
(let [index (default :ttl 300 (update-index (index)))]

  ; Inbound events will be passed to these streams:
  (streams

    ; Index all events immediately.
    index

    ; Calculate an overall rate of events.
    (with {:metric 1 :host nil :state "ok" :service "events/sec"}
      (rate 5 index))

    ; Log expired events.
    (expired
      (fn [event] (info "expired" event)))
))
```

This makes Riemann  listen only for events coming from  localhost.
To make it listen for sources external to localhost, change the host
value e.g.

```clojure
(let [host "0.0.0.0"]
```


## Using the 0.1.6 jar ##

The *riemann* script in *bin* needs to be edited to use the 0.1.6
jar.  Change the line near the top of the script:

```bash
JAR="<wherever you want to keep riemann source>/target/riemann-0.1.6-SNAPSHOT-standalone.jar"
```

## Running Riemann ##

Assuming you are in the riemann source folder, the following will start
Riemann with the example configuration:

```bash
./bin/riemann 
```

or with your own

```bash
./bin/rieman /path/to/my-riemann.config
```

## Sending Riemann some events ##

A very simple way to test your Riemann server is by sending it some
events is using *riemann-health* from the Rieman server itself.:

```bash
gem install riemann-health
riemann-health
```

Or if you're Riemann server is running elsewhere (e.g. on ip 10.16.0.64) then:

```bash
riemann-health --host 10.16.0.64
```
The health events can be view using the Riemann Dashboard.  On the
Riemann server:

```bash
gem install riemann-dash
riemann-dash
```

Click [here](http://localhost:4567/) to see the dashboard on localhost running on its default port (4567).

# <a id="configurejmx"></a>Configuring JMX Data Sources #

Each source will have its own way of configuring JMX access.  The
below only applies
to Hadoop & HBase.  Configuring JMX can be a bit fraught if reports on the net are
anything to go by.  The below worked for me.  Note I'm using YARN;
details will be different (but not too dissimilar) for Hadoop v1.  

In this
example authenticated access will be enforced 
i.e  remote  JMX requests must explicitly
include a username and password.  Also, although different files are
used for Hadoop and HBase components, the contents
(usernames & passwords) are the same  for
convenience - they can be different of course.

I use environment variables *$HADOOP_CONF_HOME* and
*$HBASE_CONF_HOME* and,
in my system, these are */etc/hadoop/conf* and */etc/hbase/conf* respectively.

## Hadoop ##

Hadoop YARN has a number of component daemons that all have to be
configured to respond to JMX requests.

* HDFS

 HDFS has one *NameNode*, an optional *SecondaryNameNode* and a
 *DataNode* daemon  that runs on each server that hosts data.
 
* YARN (Map Reduce)
 
  YARN has the *ResourceManager* replacing  the *JobTracker* of version 1
  
  Each mapred *worker* machine has a *NodeManager* daemon.
  
  YARN also has the *JobHistoryTracker* daemon running on the same
  host as the *ResourceManager*. 
  
In *pseudo-distributed* all of these run on the same host and hence  need to listen for JMX requests on
different ports.

### Configure the Hadoop Metrics Properties ###

Edit *$HADOOP_CONF_HOME/hadoop-metric.properties* to contain the following:

```json
# Configuration of the "hadoop" context for null
hadoop.class=org.apache.hadoop.metrics.spi.NullContextWithUpdateThread
hadoop.period=60

# Configuration of the "jvm" context for null
jvm.class=org.apache.hadoop.metrics.spi.NullContextWithUpdateThread
jvm.period=60

# Configuration of the "rpc" context for null
rpc.class=org.apache.hadoop.metrics.spi.NullContextWithUpdateThread
rpc.period=60
```

### Create the JMX Usernames and Password files ###

JMX password files have to have restricted permissions (600) and since HDFS and
YARN daemons run under different users (hdfs and yarn), a password file
is needed for each.


Create file *$HADOOP_CONF_HOME/jmxremote.access* containing:

```json
monitorRole readonly
controlRole readwrite
```

Create file *$HADOOP_CONF_HOME//jmxremote.passwd* with chosen passwords
for monitoring (read-only e.g. 'monitorpass') and
control (read/write e.g. 'controlpass'):

```json
monitorRole monitorpass
controlRole controlpass
```

Run the following command sequence whilst in the conf folder:

```bash
chown root.hadoop ./jmxremote.access
chmod 640 ./jmxremote.access
cp ./jmxremote.pass ./hdfs-jmxremote.passwd
cp ./jmxremote.pass ./yarn-jmxremote.passwd
chmod 600 ./*jmxremote.passwd
chown hdfs.hadoop ./hdfs-jmxremote.passwd
chown yarn.hadoop ./yarn-jmxremote.passwd
```

### Configure the Hadoop daemons to enable JMX ###

Edit *$HADOOP_CONF_HOME/hadoop-env.sh* to contain the following lines

```bash
# JMX Options

export HADOOP_CONF_HOME=/etc/hadoop/conf

HADOOP_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false"

HADOOP_JMX_OPTS="$HADOOP_JMX_OPTS -Djava.net.preferIPv4Stack=true"

HADOOP_JMX_OPTS="$HADOOP_JMX_OPTS -Dcom.sun.management.jmxremote.access.file=$HADOOP_CONF_HOME/jmxremote.access"

export HADOOP_NAMENODE_OPTS="$HADOOP_JMX_OPTS -Dcom.sun.management.jmxremote.port=8004 \
  -Dcom.sun.management.jmxremote.password.file=$HADOOP_CONF_HOME/hdfs-jmxremote.passwd"

export HADOOP_SECONDARYNAMENODE_OPTS="$HADOOP_JMX_OPTS -Dcom.sun.management.jmxremote.port=8005 \
  -Dcom.sun.management.jmxremote.password.file=$HADOOP_CONF_HOME/hdfs-jmxremote.passwd"

export HADOOP_DATANODE_OPTS="$HADOOP_JMX_OPTS -Dcom.sun.management.jmxremote.port=8006 \
  -Dcom.sun.management.jmxremote.password.file=$HADOOP_CONF_HOME/hdfs-jmxremote.passwd"
  
export YARN_RESOURCEMANAGER_OPTS="$HADOOP_JMX_OPTS -Dcom.sun.management.jmxremote.port=8007 \
  -Dcom.sun.management.jmxremote.password.file=$HADOOP_CONF_HOME/yarn-jmxremote.passwd"

export YARN_NODEMANAGER_OPTS="$HADOOP_JMX_OPTS -Dcom.sun.management.jmxremote.port=8008 \
  -Dcom.sun.management.jmxremote.password.file=$HADOOP_CONF_HOME/yarn-jmxremote.passwd"
```

Notes:

* Note the different ports used.  These seem to be arbitrary although
  examples on the Net show the NameNode on 8004.  I have just used the
  next port up for the others.

* Note the different password files used for the HDFS and YARN
  daemons.
  
### Restart the Hadoop daemons ###

```bash
service hadoop-yarn-resourcemanager restart
service hadoop-yarn-nodemanager restart
service hadoop-hdfs-namenode restart
service hadoop-hdfs-secondarynamenode restart
service hadoop-hdfs-datanode restart
```

### Check the Hadoop daemons are listening ###

To check the *NameNode* is listening for example:

```bash
netstat -antp | grep 8004
```

should display something like:

```bash
tcp        0      0 0.0.0.0:8004           0.0.0.0:*               LISTEN      -               
```


## HBase ##

HBase has two main components: the *Master* and *RegionServer*.  Its
the latter that does all the heavy lifting and an instance of it will
run on each HBase server.

BTW HBase also require / uses ZooKeeper and in the *pseudo-distrubuted* confgiuration
there will also be a daemon called *QuorumPeerMain* running.

In *pseudo-distributed* mode both the Master and (one) RegionServer
will run on the same host. Hence they need to listen for JMX requests on
different port.

### Configure the HBase Metrics Properties ###

Edit *$HBASE_CONF_HOME/conf/hadoop-metric.properties* to contain the following:

```json
# Configuration of the "hbase" context for null
hbase.class=org.apache.hadoop.metrics.spi.NullContextWithUpdateThread
hbase.period=60

# Configuration of the "jvm" context for null
jvm.class=org.apache.hadoop.metrics.spi.NullContextWithUpdateThread
jvm.period=60

# Configuration of the "rpc" context for null
rpc.class=org.apache.hadoop.metrics.spi.NullContextWithUpdateThread
rpc.period=60
```

### Create the JMX Usernames and Password files ###

The jmx usernames and password file needs to be created in the same
way as for Hadoop; you could copy the Hadoop file into
*$HBASE_CONF_HOME/conf*, and change the ownerships (hbase.hbase).

### Configure the HBase daemons to enable JMX ###

Edit *$HBASE_CONF_HOME/conf/hbase-env.sh* to have the following lines

```bash
HBASE_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false"

# Need the following line to ensure listen in on ipv4 not ipv6

# JMX Options

export HBASE_CONF_HOME=/etc/hbase/conf

HBASE_JMX_OPTS="$HBASE_JMX_OPTS -Djava.net.preferIPv4Stack=true"

HBASE_JMX_OPTS="$HBASE_JMX_OPTS -Dcom.sun.management.jmxremote.password.file=$HBASE_CONF_HOME/jmxremote.passwd"

HBASE_JMX_OPTS="$HBASE_JMX_OPTS -Dcom.sun.management.jmxremote.access.file=$HBASE_CONF_HOME/jmxremote.access"

export HBASE_MASTER_OPTS="$HBASE_JMX_OPTS -Dcom.sun.management.jmxremote.port=10101"

export HBASE_REGIONSERVER_OPTS="$HBASE_JMX_OPTS -Dcom.sun.management.jmxremote.port=10102"
```

Note the Master is set to listen on port 10101 and the RegionServer
on 10102.

### Restart the HBase daemons ###

```bash
service hbase-master restart
service hbase-regionserver restart
```

### Check the HBase daemons are listening ###

To check the *HBase Master* is listening for example:

```bash
netstat -antp | grep 10101
```

should display something like:

```bash
tcp        0      0 0.0.0.0:10101           0.0.0.0:*               LISTEN      -               
```

## Browsing the beans ##

You should now be able to peruse the Hadoop and HBase JMX beans using the *jconsole*
program distributed as part of the Java JDK:

```bash
jconsole
```

jconsole will put up a fairly intuitive  GUI.



[Esper]: http://esper.codehaus.org/
[Nagios]: http://www.nagios.org/
[Riemann]: http://riemann.io
[Ruby]: http://www.ruby-lang.org/
[jRuby]: http://jruby.org/
[Clojure]: http:///clojure.org
[Leiningen]: https://github.com/technomancy/leiningen
