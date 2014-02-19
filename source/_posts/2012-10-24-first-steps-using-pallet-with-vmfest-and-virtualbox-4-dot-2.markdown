---
layout: post
title: First steps using Pallet, VMFest and VirtualBox (VBox) 4.2
date: 2012-10-24 16:35
comments: true
categories: 
---

[Pallet][PalletOps]
is a
*configuration management* tool (library) for
Linux infrastructures.  Well known alternatives to Pallet include [Chef][Chef]
and [Puppet][Puppet].

A bit more background  on Pallet can be found in a
 [previous post][VMFest VBox 4.2 post].  That post is also worth
 reading for  background on VMFest and how it interacts with VBox

Chef is hugely useful in defining  and managing 
standardised, repeatable and consistent definitions and configuration
of Infrastructure components such as 
 (web servers,
database serevers, etc). Picking up the basics of Chef is not a steep learning curve.
I've learnt the 20% of Chef that does 80% of what I
need to do very easily, albeit coming as I do from a recent Ruby background.

Chef is  a vibrant and compelling  *ecosystem*.  Opscode are doing
a fine job of taking the software forward, providing good
community support, documentation, tutorials, books, etc whilst providing professional
support for these organisation that need it.  That said Chef has its
naff bits as well.  Chef is flexible but within the constraint of the "API" / "DSL".  In many
ways Chef feels like Rails - its  Ruby alright but in a well-defined
box.  Both strengths and weaknesses of course but the strengths fair
outweigh  the weaknesses
for all practical purposes and use.

So why bother with Pallet?  Maybe as a long time Infrastructure guy,
its because
my antennae are twitching and I can see a powerful and flexible potential
addition to the armoury of tools for managing infrastructures. 

It is also because 
Pallet "scripts" are Clojure programs and it
really is *Infrastructure as Code* in the most explicit sense:
Pallet is the library but 
the scripts are *mine* and I feel more in control of what is going on
than with Chef.  

Also Pallet's composability is very attractive feature - it feels very natural (in Clojure) to build high level
abstractions from lower level ones.  Even with my currently limited Pallet
knowledge, I
can already "see" how to build  hierarchical, complete,
comprehensive and manageable infrastructures, something I've  found  harder to imagine 
with  Chef.  Horses for courses of course.

BTW: Although my environment is VBox 4.2 and the examples below were run using my
[VBox 4.2 compatible VMFest][VMFest VBox 4.2 post], there is no reason
why they shouldn't work with the PalletOps team's
[current release][VMFest Github] for VBox 4.1 as long as my extensions to hardware
models (*clipboard-mode*, *utc-time* and *boot-order*) are removed.

<!-- more -->

# Pallet Tutorials #

There are a couple of videos worth watching:  [Hugo Duncan](http://hugoduncan.org/) at [EuroClojure](http://vimeo.com/45562554)
and [Antoni Batchelli](http://disclojure.org/) at
[Clojure West](http://www.infoq.com/presentations/Pallet-DevOps-for-the-JVM)

The [website][palletops] has
[getting started](http://palletops.com/doc/)  documentation and you can also peruse the  [API](http://palletops.com/pallet/api/0.7/index.html)

The PalletOps team realise that the documentation and tutorial
materials for Pallet needs to be improved, especially for people dipping
their first toe.  I'd second that - today its just a bit too
challenging to make headway promptly. One only has to look at the IRC #pallet channel to see how many
pretty basic questions have to be fielded by Pallet's authors.  Those
guys are exceptionally helpful but the model doesn't scale.

There is quite a bit of documentation but it tends to be *reference*
and you need to know what you are  looking for.  There isn't (that I can
find) too much complete, end to end, tutorial material exploring 
how to get started, get to where you want to be and climb to  the first
plateau of understanding.  

Pallet is definitely worth your time though.  I hope this post helps
others get over that initial hump, know enough to learn more;  and progress to a degree of
competence with Pallet and able to take advantage of many more its features.


#  Create a project  #

As usual, for a Clojure project,
[Leiningen][Leiningen] (*lein*) will be used.  (How to
install Leingen was briefly covered  [here][Leiningen
Background].) 

``` bash
lein new pallet_examples
cd ./pallet_examples
```

# Provisioning Examples - using explicit image and hardware declarations #

The frist example tries to do the same as the first example in the
[previous post][VMFest VBox 4.2 post] but wth Pallet: the simplest and most direct way of
creating new VMs with Pallet.

The code to create the same Ubuntu 12.04 64bit VM  looks
like this, and its importnat features will be explained after:

<script src="https://gist.github.com/3894728.js?file=blog_pallet_image0.clj"></script>

## The compute service ##

A *compute service* called *vbox* is defined
(*pconfig/compute-service*) where the *provider* is *vmfest*

The *identity* and *credential* keys specify the account to use to
drive VBox; the account must be a member of *vboxusers*.


## The image models ##

The *images* in the *vbox* service definition defines the available image models

The contents of the *vmfest-images-models* map is exactly the same as
in the *meta* file *~/.vmfest/models/my-models-list.meta* in the
[previous post][VMFest VBox 4.2 post].
    
Note VMFest will *also* read any *meta* files in the *model-path*
but the explicitly provided images will take preference.

## The hardware models ##

The *hardware-models* in the *vbox* service definition provides the additional hardware models
    
The contents of the *vmftest-hardware-models* map will be
familiar  from *~/.vmfest/models/my-hardware-models.meta* in the [previous post][VMFest VBox 4.2 post].
     
Pallet (i.e. pallet-vmfest) has its own default hardware models:
     
``` clojure
{:micro {:memory 512 :cpu-count 1}
 :small {:memory-size 1024 :cpu-count 1}
 :medium {:memory-size 2048 :cpu-count 2}
 :large {:memory-size (* 4 1024) :cpu-count 4}
```

The  *vmfest-hardware-models* map includes a *small* definition which overrides the default one.

There appears to be no way  to request the hardware
  models be read
  from a file but the code to do so is trivial.  For example, using
  the function *load-models*  in VMFest's *manager.clj*:
  
``` clojure
(def vmfest-hardware-models (vman/load-models :model-path "/home/ian/.vmfest/hardware-models"))
```

Note the *model-path* is fully qualified.

## The network configuration ##

The comments in *pallet.compute.vmfest.clj* explains the VMFest / VBox
network modes succinctly:

> Pallet offers two networking models: local and bridged.

> In Local mode pallet creates two network interfaces in the VM, one for an
   internal network (e.g. vboxnet0), and the other one for a NAT network. This
   option doesn't require VM's to obtain an external IP address, but requires
   the image booted to bring up at least eth0 and eth1, so this method won't
   work on all images.

> In Bridged mode pallet creates one interface in the VM that is bridged on a
   physical network interface. For pallet to work, this physical interface must
   have an IP address that must be hooked in an existing network. This mode
   works with all images.
   
   
The configuration shown below is  the
default for *local mode*:  two interfaces (e.g. eth0 & eth1), one
*host-only* (connected to *vboxnet0*) and the second  *nat*.

``` clojure
:network [{:attachment-type :host-only :host-only-interface "vboxnet0"}
          {:attachment-type :nat}]
```
   
The default connection (*vboxnet0*) for the *host-only* interface can
be overridden in the *compute service* definition.  Similarly the default
bridged interface can be set, else the first bridge-able interface
will be chosen (usually e.g. eth0).  One can also set the default
network type to be *local* or *bridged*.

``` clojure
    :default-local-interface "vboxnet0"
    :default-bridged-interface "br0"
    :default-network-type :local
```

It appears to be impossible to provide a custom networking definition
outside of the constraints of *local* or *bridged*. (So for example,
one can't have a 3 adapter VM.)  There is no doubt that the two modes
available are useful and common configurations, but it seems
unnecessarily limiting not to be able to *roll your own* network.  From
superficial examination of the code this improvement doesn't seem to
be hard to do. 

## The node spec ##

The call to *pcore/node-spec* defines a compute node called *node-ubuntu-1204-small*.
    
The node-spec includes the specification of the *image* to use and
also the *hardware*.
    
Both of these two maps specify the model *id* to use in the respective
maps *vmfest-images-models* and *vmfest-hardware-models*
    
There does not appear to be anyway to define the complete image explicitly
in the node-spec definition; if the image map does not have an
*image-id* key but does have others (e.g. *os-family*), the *best match*
from the known images  will be used.

But the hardware model can be specified  explicitly if wanted e.g.
    
``` clojure
  :hardware  {:hardware-model
              {:memory-size 512
               :cpu-count 1
               :vram-size 16
               etc.
```

One can also include a *network* entry (key-value) in the *node-spec*
but my reading of the pallet-vmfest code indicates the values will not
be used, even *network-type*.  

## The group spec ##

The *group-spec* create a collection with a given name (e.g.
*image0-group-ubuntu-1024-64*) and the *node-spec* to use (e.g.
*node-ubuntu-1204-small*.

The names of the VMs created by this spec will be
*image0-group-ubuntu-1024-64-1*, *image0-group-ubuntu-1024-64-2*, etc.

The group spec can also include, amongst other things,  a *count*
keyword (not shown) to set how member nodes the group will have.

## The converge command ##

The *converge* command does exactly that.  It will make the
infrastructure reflect the definition, creating or destroying VMs as
necessary; and re-applying configuration to running VMs.

In this example, the *count* for the *group-ubuntu-1204-small-servers*
group is explicitly given (i.e. 1).  Converging with a count of 0 will
kill all instances of (in) the group.

## Running the converge ##

Pallet tends to be run from the repl:

```bash
lein repl
(use '[pallet_examples.blog_pallet_image0 :reload :all])
(ns pallet_examples.blog_pallet_image0)
(go-pallet)
```

# Provisioning Examples - tidying things up #

Using VMFest *meta* files (see [previous post][VMFest VBox 4.2 post]) to define both the images and hardware models
makes the code cleaner:

<script src="https://gist.github.com/3906707.js?file=blog_pallet_image1.clj"></script>

Notes:

* the *vmfest-hardware-models are read from the meta files in
  "~/.vmfest/hardware-models*
  
    Note the path to the meta folder has to be fully qualified.

# Provisioning Examples - using Pallet's config.clj file

Pallet has its own configuration file
*~/.pallet/config.clj* that is a call to the *defpallet* macro. The
config file allows  all the boiler
plate / common configuration (service definitions, credentials, etc)
to be externalised and centralised.

(Its also possible to define services in e.g. for vbox 
*~/.pallet/services/vbox.clj*.)

## The Pallet config.clj file ##

The  config.clj for these examples looks like this:

<script src="https://gist.github.com/3879997.js?file=config.clj"></script>

Notes:

* both the image and hardware models for the *vbox*
service are explicitly
defined

* image models will also be picked up from *meta* files the *model-path*
as usual.  

* the new VMs will be created in the *node-path* directory

* there appears to be no way of sourcing the hardware
models from a file.

## Provisioning using Pallet's config.clj ##

The more concise code to create the example Ubuntu 12.04 VM  looks like this:

<script src="https://gist.github.com/3907968.js?file=blog_pallet_config0.clj"></script>

Notes

* the call to *pconfig/compute-service* for service *vbox* pulls  the
  definition from  *~/.pallet/config.clj*
  
* image and hardware models are specified by their respective ids

# Configuration Examples - preparation  #

So far the examples have focused on using Pallet for the  *provisioning* of
VBox  VMs with a minimum of fuss and code, but doing nothing to their
*configuration*.  That's only half the Pallet story.

Pallet does it configuration magic by [ssh-ing][ssh] into the VM and
running bash scripts with *sudo* privilege.  To allow  Pallet to do
its stuff, the
image model must be prepped appropriately.

BTW I create (clone) my  (*multi-attach*) image model from the vdi of a working *base* VM (that
has no snapshots - else the clone will not be up to date). 

## Allowing ssh access ##

The  (*multi-attach*) image model for the VMs  must be
prepped to allow ssh access from the system running the Pallet script. 

Pallet uses an
*admin user* account  (e.g. *pallet-admin*) to ssh into the VM.  The
admin user must be  configured
(in */etc/sudoers* using *visudo*) to  allow  it  to use sudo
without a password (i.e. *pallet-admin ALL = (ALL) NOPASSWD: ALL*).

The admin user key-pair is created in the usual way and the public key
added to the admin user account in the  image model.  Familiar
territory for most but
[just in case](http://en.wikipedia.org/wiki/Ssh-keygen).  Be sure to
test the ssh access (to the base VM) works from the host running the Pallet scripts.

You could (should) also  configure  */etc/ssh/sshd_config*
for enhanced security (e.g. setting *AllowUsers* to restrict who can
log on.  (Configuring good system security is a whole post itself.)

(BTW, Pallet has a function called *automated-admin-user* which will
create the admin user account in the VM if it doesn't exist.  But to
do so needs an existing account
with sudo privilege.  So chicken'n'egg - I feel its simpler to
pre-create the admin user in the base model.)

## Tell Pallet what admin user to use ##

By default, Pallet will use the account running the Pallet script as the admin user.  It is convenient to
[set the real admin user](http://palletops.com/how-to-configure-your-cloud-credentials-in-pa/) in
*/.pallet/config.clj* e.g.

``` clojure
 (defpallet
   :admin-user
     {:username "pallet-admin"
      :private-key-path "/home/ian/.ssh/pallet-admin"
      :public-key-path "/home/ian/.ssh/pallet-admin.pub"
      :passphrase "<the passphrase goes here>"}
         
   :services {
      
      etc
```

To make  Pallet  use this admin users requires the *converge* to be
(e.g.) "wrapped" in a *with-admin-user* call - see the example below.

## Installing *aptitude* ##

Pallet uses the *aptitude* package manager on Ubuntu; support for
*apt* is scheduled for 0.8.0 I understand.  Since I'm using the current production
release (0.7.0), *apitude* need to be installed in the base model:

``` bash
apt-get install aptitude
```

## Reset the network configuration ##

The last
thing I do before shutting the base VM  down to
clone its vdi into the image model vdi is to delete (reset) the base's (and
hence image's) network configuration, especially if the base's
network configuration is not the same as the VMs based on it.

```bash
sudo rm /etc/udev/rules.d/70-persistent-net.rules
```

## Clone the base vdi to a multi-attach image model

The clone command looks like e.g.

```bash
VBoxManage clonehd /path/to/vmfest/base/base_ubuntu_1204_64.vdi /path/to/vmfest/models/model_ubuntu_1204_64.vdi
```

To make it multi-attach:

``` bash
VBoxManage modifyhd  /path/to/vmfest/models/model_ubuntu_1204_64.vdi -type multiattach
```

To confirm VBox knows about the image, and its  multiattach, the below
pipe  should show  it:

```bash
VBoxManage list hdds | grep -E "^UUID|Location|^Type" | awk '{print $2}' | xargs -n 3 | grep "model_ubuntu_1204_64"
```

*Note: I have found that if the image model is not made *multiattach* at this
stage (as opposed to letting VMFest making it so the first time the
image is used), subsequent VMs created  using the same image will not
start but the image is attached  in normal mode.  And if that
failed VM is removed, the image is deleted as well!  There is
something in the was VMFest managing images that needs to be tightened
up but I've not got to the bottom of it yet.*

# Configuration Examples - installing some basic packages  #

In this example, Pallet will  install some basic packages.
The code looks like this and will be explained below:

<script src="https://gist.github.com/3923227.js?file=blog_pallet_config1.clj"></script>

## Using the admin user ##

* the call to *admin-user-from-config-file* pulls the *admin-user* definition from
 *~/.pallet/config.clj* and creates a Pallet *user* called
*config-file-admin-user*.

* the *converge* is "wrapped" in a call to *with-config-user* which
has the *config-file-admin-user* given as a parameter

* the comment at the end shows how the admin-user could be specified
explicitly and passed as  the *user* parameter to the *converge*

## Phases ##

Once a VM has been created and started, Pallet applies a sequence of
[phases](http://palletops.com/doc/reference/phases/) to achieve the
desired configuration.

The
API documentation for [converge](http://palletops.com/pallet/api/0.7/pallet.core.html#var-converge/)
says it

> ... applies the bootstrap phase to all new nodes and the configure
 phase to all running nodes whose group-name matches a key in the node
 map. Additional phases can also be specified in the options, and will
 be applied to all matching nodes. The :configure phase is always
 applied, by default as the first (post bootstrap) phase. You can
 change the order in which the :configure phase is applied by
 explicitly listing it.

Phases can have pre- and post- phases, and sub-phases it seems (not
explored here).

Phase contents are defined using the *phase-fn* macro (in the *pallet.phase*
namespace).  In the example above  the standard *configure* is defined
and included in
the *group spec*:

``` clojure
    :phases {:configure phase-configure-ubuntu-1204-64}
```

The *configure* phase in the example first  creates a file that will tell the *apt*
subsystem  to use an
[apt cacher](http://www.unix-ag.uni-kl.de/~bloch/acng/) running on the
VBox host system to
save downloading the same packages time and time again.  And then it
installs a number of packages
e.g. *build-essential* (the list is not a
necessarily sensible selection and definitely not complete).  Finally
a "flag file" is created in /tmp to show the phase ran ok - I did this
for debugging.

## Actions ##

In the code above, *package*, *packages*, *package-manager* and
*remote-file* are examples of  a Pallet
*[action](http://palletops.com/doc/reference/actions/)*.
   
Actions *do stuff* such as install a package, create a file or directory, run a
script, start a services, etc.  and  are similar to  [Chef's resources](http://wiki.opscode.com/display/chef/Resources).

## Crates ##

[Crates](http://palletops.com/doc/reference/crates/) are

> functions that encapsulate some unit of configuration or administration. 

and 

> actions can be composed in crate functions to implement your configuration and operational requirements

The only *create* used in the example was the
[one](https://github.com/pallet/git-crate) on Github.  I installed the
crate (i.e. *git.clj*)  in my (Leiningen project) *src/pallet/crate*
folder.

Crates can be seen as  equivalent  to [Chef's recipes](http://wiki.opscode.com/display/chef/Recipes).

# Configuration Examples - composing richer configurations  #

The example above wasn't very complicated, complete or indeed terribly useful.  In real life its is common
for a server (VM) to have a base configuration on top of which is
layered configuration for its role (e.g. web server, database server, monitoring,
whatever).

Lets consider an example that creates  a web server
running *nginx* and a database server running *mysql*.  There are
*crates* for both of these on [Github][Pallet Github].

The code for this example looks like this and will be explained below:

<script src="https://gist.github.com/3946124.js?file=blog_pallet_config2.clj"></script>

Immediate notes

* a number of extra crates appear in the *require*

* *clojure.data.json* is required

    I had to specify the "0.1.3" jar (c.f. "0.2.0") in my
    project.clj dependencies to obtain the *json-str" function.
    
```clojure
[org.clojure/data.json "0.1.3"]
```

* four *phase-fn* calls

    The code has four *phase-fn* calls, the names and contents of
 which should be
 reasonably self-evident.

## The server specs ##

The code includes four calls to a new function:
*[server-spec](http://palletops.com/doc/reference/node-types/)*.

*server-specs* are 

> ... used to specify phase functions, which can be invoked using converge or lift
  

*server-specs* can inherit (*extends*) from other *server-specs*
 enabling rich definitions to be *composed* easily. When
 *server-specs* inherit, the *phase-fn* contents for each phase (e.g.
 *configure*) are added together.  So, e.g., *server-web* includes the
 *phase-fns* from the *server-apt-cache* and *server-basenode* *server-specs*, as well as
 its own (*phase-configure-db*).

To save cluttering up the top-level code, a rich collection (library) of phase and server specs could be relegated to their own Clojure files
(namespaces) and *required* as necessary.


## The group spec ##

The *group spec* now includes a *converge* for both the web and
database groups.  These are performed serially.


# Final Words #

This post is probably already too long and I've only scratched the
surface of what Pallet can do.  It does demonstrate the
potential though.

But, today, the "barriers to entry" are too high, far higher than with
e.g. Chef.  One has to "dig around" a lot for information on how to do
some basic stuff.  

For
me that was ok, an opportunity to see more Clojure in the wild and
start to build a serious understanding of Pallet.  For the more casual user
though, I believe this is   a huge disincentive to trying or using
Pallet, and probably  an insurmountable and / or undesirable
hurdle.  Its just far easier to get started with Chef.

But, as I said above, I hope this post will  encourage  others to get started on what
I feel will be a very worthwhile and productive journey.

# Fess Up #

The *nginx* crate does not install successfully.  This may be 
because of plus signs in package names believed fixed in 0.8.


[Clojure]: http:///clojure.org
[PalletOps]: http://www.palletops.com
[Pallet Github]: https://github.com/pallet
[Pallet-VMFest Github]: https://github.com/pallet/pallet-vmfest
[VMFest Github]: https://github.com/tbatchelli/vmfest
[VMFest VBox 4.2 post]: http://ianrumford.github.com/blog/2012/10/13/using-vmfest-with-virtualbox-4-dot-2/
[Chef]: http://www.opscode.com/
[Puppet]: http://puppetlabs.com/
[VBox]: https://www.virtualbox.org
[Vagrant]: http://vagrantup.com
[Leiningen]: https://github.com/technomancy/leiningen
[Leiningen Background]: http://ianrumford.github.com/blog/2012/09/21/writing-apache-pig-udfs-in-clojure/
[ssh]: http://www.openssh.com/


