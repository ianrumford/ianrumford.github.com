---
layout: post
title: "A service-oriented administrative architecture"
date: 2012-09-15 18:30
comments: true
categories: 
published: true
---


About a year ago, over a few beers with a fellow infrastructure
veteran, we fell to discussing the challenges of
managing large scale infrastructures. 

He had been facing similar headaches
  in his environment. I had been discussing the same
challenges with another infrastructure mate who manages 1000s of
servers, and explaining some
thoughts I had had on how a scalable, coherent and consistent
administration approach could be implemented.

We decided to put our thoughts down on paper to show to some others
and I pulled together a
first cut.
  The following notes are broadly those thoughts, with some updates (by
me) to fix typos and general improvements for clarity.

The solution described below is something of a swiss army knife - it can be
sensibly used not only the manage the *bread'n'butter* administration of an
infrastructure but also perform many *value add* functions easily. To
illustrate, in [another post](http://ianrumford.github.com/blog/2012/09/15/some-notes-on-using-ruby-handlers-with-mongrel2)  I've shown  an example "data acquisition" handler
written in Ruby.

<!-- more -->


# The Complexity Challenge #

All large, growing or volatile infrastructures are susceptible to
increased complexity as more and more elements are added to the
portfolio of hardware and software needing to be managed, and the
dictates of the business requires rapid response.

Complex infrastructures have a higher risk of instability and
unreliability; predictable change can be a lottery because
dependencies are not fully appreciated by the sysadmins. 

They take
more effort to operate day in day out – a continuous, continuing and growing
commitment to assigning expensive, expert sysadmins (and perhaps even
developers) to ensure complexity doesn’t spill over into service
unreliability. 

Complex infrastructures are also inflexible and often
have to ring fence resources because the latter are effectively black boxes. It
becomes more difficult to leverage investments in hardware and
software because the current usage is not well appreciated or
understood and the
consequences of changing any element of the configuration are
potentially fraught. 

It becomes very difficult for the sysadmin team
to remain truly in control of a rapidly changing infrastructure that
has not been architected with scale or agility in mind and or according to simple
principals, the most important being: “build once, use many”


# Scripts to the rescue (but only partially) #

Many of the scale and complexity challenges can be mitigated by
scripting common tasks used for day to day operations and e.g newly
commissioned facilities (provisioning). 

Scripts implicitly document the
logic to be followed to complete a given task and concomitantly
explicitly standardise the task: the task and the script become
synonymous. 

Using a script for repetitive and (possibly) complex tasks
are wins on multiple counts especially ensuring predictable outcomes,
automation and efficiency. 

The downside of scripts is that they can
become “owned” by individual sysadmins; often (and usually) the original author.
But each sysadmin ("DevOps"), left to their own devices, is likely to have their own preferred language;
opinions on interfaces, standards, scheduling; views on change
control; approaches to error handling and logging, etc, etc. And
“documentation” may often be vestigial or a *cinderella* task waiting
for a "quiet moment". 

Over time, in fast changing
environments, the infrastructure may become managed by a large corpus
of scripts which are collectively not understood but  still
hugely dependent on individual sysadmins who may take critical
knowledge with them when they move on internally or externally. 

So
many *bus factors* of one is not a tenable status quo for the manager
responsible for the overall wellbeing of the infrastructure and 
charged with responding to changing business needs promptly and at
reasonable cost. So it becomes imperative for the scripts to evolve into
a useful,  usable and reliable collective team resource, with high 
utility, reasonable
documentation and adhering to the agreed consensus standards: in short,
*a standardised scripting execution environment* (SSEE) that can support
 the implementation of a
*service-oriented administrative architecture* (SOAA).


# A Standardised Scripting Execution Environment (SSEE) #

## Summary ##

The SSEE  envisages all scripts being initiated (run) by a HTTP
request originating from any http client (human or automaton - browser, another
script, custom code, cron, curl, etc).

HTTP requests would be [RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer)

Each http request would be categorised by the web server and  forwarded (routed)
to the appropriate
script “sitting behind” the web server and acting as the request
“handler”.

The handler script would analyse 
the parameterised http request, perform whatever
action has been requested, and return a http
response of whatever format (JSON, XML, etc) is required.

## Benefits ##

* Familiar and very common RESTful frontend API using the standard GET, PUT, POST
  and DELETE verbs

    Handlers can interact with browsers on computers, phones, whatever; stand
  alone programs such as curl or other scripts generating
   http requests.
 
* Opportunity to “wrap” scripts for audit, instrumentation, etc

    Common wrappers could be used around each handler as and when
  necessary for many purposes such as
  provide an audit trail, instrumentation, monitoring, alerting, etc,
  etc.
  
* Opportunity to use uniform Authentication and Authorisation techniques 

    Handlers could vet the requestor, insist on authentication and
enforce authorization in a standardised and consistent way (possibly
using utility "wrappers").


* Enforced documentation and standardisation stage as part of the migration

* Change Control And Version Control using e.g. git

    Synchronisation and distribution becomes a simple git push / pull

## Technical Requirements ##

* high speed, low latency http parsing

* use of a well design communication protocol between web server and handlers
   
    A well-defined and implemented messaging middleware

* language agnostics communications

    Handlers can be written in Java, Perl, Python, Lua, etc - any that support the comms protocol
      
* ability to support a large number of different handlers

    No architectural limit on the number of different handlers.

* location independence

    Handlers can run anywhere, not necessarily on the web server
 
* source and support is available


# Candidate Stack:  mongrel2, ZeroMQ and your favourite scripting language #

[mongrel2](http://mongrel2.org/) is a very quick, low latency http
engine that allows backend *handlers* to process various types on
incoming http requests, performing some actions and returning a http
response of whatever format is desired / requested..  

mongrel2
and its handlers communicate using the
[ZeroMQ](http://www.zeromq.org/) middleware layer thereby offering unprecedented flexibility in the languagues used to write the handlers or
where the handlers execute.

Another
[post](http://ianrumford.github.com/blog/2012/09/15/some-notes-on-using-ruby-handlers-with-mongrel2/)
explains more fully what mongrel2, how to install it,  and provides an example of a Ruby handler.


# Getting from here to there #

The prospect of completely revamping their current, possible unwieldy
and vintage scripting
environment probably fills many infrastructure manager with dread.

A strength of implementing the approach advanced here is that it can be evolutionary -
the elephant can be eaten one bite at a time.

Also, by creating a suitable wrapper environment, the existing
portfolio of script would not necessarily have to change much, if at all:  the
wrapper could / would handle the ZeroMQ / mongrel2 messaging and support the
existing script API.  

It wouldn't be that hard (especially in Ruby) to write a "wrapper
generator":  a DSL would define each script's API and the generator
would create the correct, script-specific wrapper.  A more
sophisticated 
generator could support  also other features
"for free" such as audit, authentication, authorization, etc.






