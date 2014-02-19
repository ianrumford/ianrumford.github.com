---
layout: post
title: "Using VMFest with VirtualBox (VBox) 4.2"
date: 2012-10-13 14:00
comments: true
categories: 
---

[VMFest](https://github.com/tbatchelli/vmfest) is

> "... a [PalletOps](http://palletops.com) project to turn [VirtualBox](https://www.virtualbox.org/) into a light-weight cloud provider."

To understand the context and point of this post requires a minor digression
into Pallet.

[Pallet](http://palletops.com/) describes itself as

>"A fresh look at cloud infrastructure automation"

and

> "DevOps for the JVM"

Suitably cool aphorisms but, more prosaically, Pallet is a
*configuration management* tool (library) for
Linux infrastructures (although, in principle, there does not appear to
be any reason why it wouldn't work with Windows).  

Well known alternatives to Pallet include [Chef](http://www.opscode.com/)
and [Puppet](http://puppetlabs.com/).  Currently I use Chef to manage my own
development environments.

Pallet is written in [Clojure](http://clojure.org/) and hence is part of the Java/JVM
ecosystem.

Pallet
[supports](http://www.jclouds.org/documentation/reference/supported-providers/)
many
*[cloud providers](http://palletops.com/doc/reference/providers/)*
though the use of the [jclouds](http://www.jclouds.org/)  library:  as long as a
provider supports the jclouds API, it can be used by Pallet.

But VBox is (still?) not  supported by jclouds (there doesn't appear
to be a jclouds
provider).

Because there is no jclouds provider for VBox,  the PalletOps team wrote 
[pallet-vmfest](https://github.com/pallet/pallet-vmfest) to support
VBox as a "local" *compute service*.  pallet-vmfest,  in
turn, uses VMFest to control VBox.

Hope that's clear.  More on Pallet in a future post.  Now back to
VMFest.

<!-- more -->


# VMFest and VBox #

VMFest is a standalone package that does the same job broadly as [Vagrant](http://vagrantup.com/)
(although  the latter favours Chef for configuration management).

The current version of VMFest  only supports VBox 4.1 whereas I now
use 4.2 (initially because 4.2   stopped the infuriating VM "workspace
dance" whenever a VM
changed video mode).

## VBox Interfaces and Components##

The "big picture" diagram in section 1.1 of the VBox
[SDK's](http://download.virtualbox.org/virtualbox/4.2.0/VirtualBoxSDK-4.2.0-80737.zip)
 *Programming Guide and Reference* ([pdf](http://download.virtualbox.org/virtualbox/SDKRef.pdf)) is illuminating and puts the
various interfaces 
and components into context.

## The main VBox API ##

The main VBox API is a Microsoft *Component Object Model* ([COM](https://www.microsoft.com/com/default.mspx)) interface; on Linux the
interface is emulated by [XPCOM](https://developer.mozilla.org/en-US/docs/XPCOM).

One can program directly to the COM API; the guide cites Python or
C++ as suitable languages (although from the snippet of code given I
can't see why you couldn't use Ruby if so inclined as - least
on  Windows itself - 
 using COM interfaces from Ruby is simple and *just works*).

## VBox Management Interfaces ##

VBox support three  interfaces "on top" of  the main API:

1. the familiar GUI
2. the VBoxManage command
3. the web service 


## VBox Web Service Interface ##

The programming guide describes the webservice thus:

> VirtualBox comes with a web service that maps nearly the entire Main API. The web service ships in a stand-alone executable (vboxwebsrv) that, when running, acts as an HTTP
server, accepts SOAP connections and processes them.

VMFest uses the web service API to control VBox, specifically by using
the
*vboxjws* Java library (distributed in the SDK).  (There are equivalent artefacts for  driving the
web service from Perl, Python, Ruby, etc)


## Porting VMFest to  VBox 4.2  ##

Porting VMFest to work with VBox 4.2 wasn't hard at least to make my
examples work.  My branch shouldn't be considered a definitive port
yet but it certainly does the basics.

The  most notable changes I had to make were:

* Accommodating the *groups* parameter in calls to *createMachine*

  I just set the parameter to *nil* to signify no group affiliation

* *findMedium* has been withdrawn; use *openMedium*

    As well as making the explicit change, I found it necessary to
    change the semantics in *ensure-image-is-registered*
    
Other changes were made as part of my learning curve and also to add
useful features:

* support for *boot-order*

    Being able to specify the *boot-order* in the machine definition.
    Any boot position not specified will be set to null.  (Note the
    max boot position is 4 - obtained from a system property).
    e.g.
    
``` clojure
    :boot-order [[1 :hard-disk] [2 :dvd]]
```

* support for *utc-time*

    Set the real time clock of the VM to use UTC time.  Linux prefers this
 
``` clojure
    :utc-time true
```

* support for *clipboard-mode*

``` clojure
    :clipboard-mode :birectional
```
* comments sprinkled around the key function

To use the 4.2 compatible code in projects, I created a new snapshot jar using
the following updated  VMFfest project.clj:

``` clojure

(defproject vmfest "0.2.5-vbox4.2.0-SNAPSHOT"
  :description "Manage local VMs from the REPL"
  :dependencies [[org.clojure/clojure "1.4.0"]
                 [slingshot "0.10.2"]
                 [org.clojure/tools.logging "0.2.3"]
                 [local/vboxjws "4.2.0"]
                 ;;[org.clojars.tbatchelli/vboxjws "4.1.8"]
                 [fs "1.0.0"]]
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :multi-deps {"1.4" [[org.clojure/clojure "1.4.0"]]
               ;; "1.4" [[org.clojure/clojure "1.4.0-beta1"]]
               :all [[slingshot "0.10.0"]
                     [org.clojure/tools.logging "0.2.3"]
                     ;;[org.clojars.tbatchelli/vboxjws "4.1.8"]
                     [local/vboxjws "4.2.0"]
                     [fs "1.0.0"]]}
  :dev-dependencies [[robert/hooke "1.1.2"]
                     [log4j/log4j "1.2.16"]
                     [lein-clojars "0.8.0"]]
  :repositories [["project" "file:repo"]]
  :aot [#"vmfest.virtualbox.model"]
  :test-selectors {:default (fn [v] (not (:integration v)))
                   :integration :integration
                   :all (fn [_] true)}
  :jar-exclusions [#"log4j.xml"])

```

The new jar (*vmfest-0.2.5-vbox4.2.0-SNAPSHOT.jar*) can be installed
in a local repo and cited as a dependency - see later.

Note the *aot* specification.  I found it necessary to aot-compile
*vmfest.virtualbox.model* to allow the *defrecords* defined in it to
be imported when not running in a repl.  Not  sure what exactly is
going on here:  without the *aot*, running from the command line (e.g.
*lein run -m etc*) fails but in a repl (e.g. *use etc*) worked.

## Installing the 4.2-compatible vboxjws and VMFest jars ##

Since we are in the Clojure world,
[Leiningen](https://github.com/technomancy/leiningen) (*lein*) will be used to manage
the project.  (An earlier [post](http://ianrumford.github.com/blog/2012/09/21/writing-apache-pig-udfs-in-clojure/) gave a bit more background on
Leiningen.)

Leiningen requires that  local jars  be kept in a local maven repository
(e.g. *~/.m2/repository/local*)  in order to be used (cited) as a dependency. 

BTW You
may need to install maven first (e.g. *sudo apt-get install maven2*)

``` bash

mvn install:install-file -DgroupId=local -DartifactId=vmfest -Dversion="0.2.5-vbox4.2.0-SNAPSHOT" -Dpackaging=jar -Dfile="/path/to/vmfest/target/vmfest-0.2.5-vbox4.2.0-SNAPSHOT.jar"

mvn install:install-file -DgroupId=local -DartifactId=vboxjws -Dversion=4.2.0 -Dpackaging=jar -Dfile="/path/to/sdk/bindings/webservice/java/jax-ws/vboxjws.jar"

```

Obviously the *-Dfile* paths would need changing if you are following along.

You should see (find) the jars have been installed  successfully in
*~/.m2/repository/local*.


## Starting the VBox Web Service ##

The VBox web service must be running for VMFest to do its stuff.  It can be started at boot
time (see the
[User Manual](https://www.virtualbox.org/manual/ch09.html#vboxwebsrv-daemon))
but starting it from a terminal is fine for testing.  Note the user
account used to run it must be part of the *vboxusers* group.

``` bash

vboxwebsrv -t0 -v

```

You also (one-off) need to disable authorisation

``` bash

VBoxManage setproperty websrvauthlibrary null

```


# Creating the project  #

As usual, need to create a Leiningen project

``` bash

lein new vmfest_examples
cd ./vmfest_examples

```
The project file is pretty minimal.

<script src="https://gist.github.com/3865605.js?file=project.clj"></script>

Notes

* the *local/vmfest* pulls the 4.2 compatible VMFest jar from the
  *local* repo in *~/.m2/repository*

* ditto for the VBox 4.2 Java API jar *local/vboxjws* 


Again, as usual, resolve the dependencies

``` bash

lein deps

```

# A simple example - using images #

There is not a whole lot to do (write) to create a VBox machine with
VMFest using an image.
The following example to create a 64-bit instance of Ubuntu 12.04
 has been adapted from the example given on the
VMFest github page:

The code to create and start the VM looks like this:

<script src="https://gist.github.com/3879064.js?file=blog_vmfest_image0.clj"></script>

Notes:

1. *my-server* is the connection to *vboxwebsrv*

    Note the credentials; the user account must be a member of
    *vboxusers* group.
  
2. *boot-order*, *utc-time* and *clipboard-mode* are my additions to VMFest

3. new images registered as *multi-attach*

    If VMFest can't find the image (*uuid*) already in the user's
    *VirtualBox.xml* file, it will register it and mark the image *multi-attach*
    
    The VMFest documentation confuses (terminologically) making an image (medium) *immutable* with
    *multi-attach* - when it says the former it means the latter.
    VMs using a *multi-attach* medium as their base image retain their changes
    over restarts of the VM (the changes are kept in a
    *dfferencing* snapshot).  Using a *immutable* medium causes the
    snapshot to be cleared (reset) at VM restart.  See section 6.45 of
    the [programming guide](http://download.virtualbox.org/virtualbox/SDKRef.pdf) for details.

4. the *my-image-map* defines the *image model*

5. the *my-machine-map* defines the *hardware model*

6. together the *image model* and *hardware model*  can be considered to define
the *profile* of the machine (e.g a "small Ubuntu 12.04 VM")

To run the code 

``` bash

lein run -m 'vmfest_examples.blog_vmfest_image0'

```

You can see from the comment at the bottom its easy to e.g *stop* or
*destroy* a VM.

# A cloudy example - using  models  #

Image and hardware models can be assigned names and used to define a
new VM.  This is especially useful when many VMs have to be created
using the same recipes.

## Hardware Models ##

However the *only*  hardware model defined currently is *:micro* and there is no way of adding additional hardware profiles (that I can
find).  (This is a stopper when the image needs to be attached to a SATA
controller - as in the  examples here.)

``` clojure

{:micro
   {:memory-size 512
    :cpu-count 1
    :network [{:attachment-type :host-only
               :host-only-interface "vboxnet0"}
              {:attachment-type :nat}]
    :storage [{:name "IDE Controller"
               :bus :ide
               :devices [nil nil {:device-type :dvd} nil]}]
    :boot-mount-point ["IDE Controller" 0]}}

```

(*NOTE:  to prove this example I changed the definition of :micro to
include the same SATA controller definition as in the image example above.*)

## Image Models ##

VMFest *seems* to insist on models (actually the *meta* files -
see below) being kept  in
*~/.vmfest/models* (the *model-path*); I can  find no  way to override the default.

Its worth exploring what goes on under the covers when an image model
name is specified.  VMFest scans the *model-path* for all files ending in
*.meta*.  Meta files contain a map of image model names and their 
specifications.  For example, say the file
*~/.vmfest/models/my-models-list.meta* contained

<script src="https://gist.github.com/3879423.js?file=gistfile1.clj"></script>

This *meta* defines two named image models: *model_ubuntu_1204_64* and *model_ubuntu_1110_32*

Note that each image model specification includes a *uuid* key which
is, in this case, the path to the actual image file.  The image will be
registered as *multi-attach* if not already registered.

## A cloudy example with existing model support ##

With the *~/.vmfest/models/my-models-list.meta* file as above, the equivalent code to the image example above would be this:

<script src="https://gist.github.com/3879212.js?file=blog_vmfest_model0"></script>


## A cloudy example with enhanced  model support ##

The lack of any way of providing one's own portfolio of *hardware
models* is a serious limitation.  However, a very small hack to the
*instance* function in *vmfest.manager.clj*  allows hardware models to be kept in a meta
file.

For example, if *~/.vmfest/models/my-hardware-models.meta* contained

<script src="https://gist.github.com/3879518.js?file=gistfile1.clj"></script>

The code to use a different hardware model is
[here](git://gist.github.com/3879655.git) but the only line different
to the model0 example about is the one to specify the *:medium* hardware profile

``` clojure

(def my-machine-model-hardware-name :medium)

```

This hack reserves a(pre-empts) n 'image' model name of *hardware-models* to
store the hardware models.

# Final Words

VMFest is very powerful, its not a lot of code but it does  some
really heavy lifting with VBox.  Which is a tribute to both VMFest's author(s)
and Clojure.

One can see from the image example above, the basic API is fairly
intuitive once the concepts have been understood and its easy to write
no-nonsense code that *gets stuff done*..

That said,  the current model-oriented API features are rather
too infelicitous to  use extensively without some sort of
augmentation.  But as
the comment in *vmfest.manager.clj* makes clear (and fair enough):

> High level API for VMFest. If you don't want to do anything too fancy, start here.

VMFest is the  first   significant
Clojure project I've tried to read, understand and learn.  It
proved very accessible  in that respect  and I've picked up quite a bit of
idiom and style.
Thanks guys!

