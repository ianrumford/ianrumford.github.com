---
layout: post
title: "Installing ZeroMQ 2.2.0 with the Ruby gem on Ubuntu 12.04"
date: 2012-09-12 13:41
comments: true
categories: 
---

[ZeroMQ](http://www.zeromq.org/) calls itself an intelligent transport
layer or, colloquially, *sockets on steroids*.

Put simply, ZeroMQ is a relatively low level socket-like interface
that allows two cooperating scripts, applications, whatever to
communicate.

The [website](http://zguide.zeromq.org/page:all#-MQ-in-a-Hundred-Words) has a *100 word definition*:

> ØMQ (ZeroMQ, 0MQ, zmq) looks like an embeddable networking library but acts like a concurrency framework. It gives you sockets that carry atomic messages across various transports like in-process, inter-process, TCP, and multicast. You can connect sockets N-to-N with patterns like fanout, pub-sub, task distribution, and request-reply. It's fast enough to be the fabric for clustered products. Its asynchronous I/O model gives you scalable multicore applications, built as asynchronous message-processing tasks. It has a score of language APIs and runs on most operating systems. ØMQ is from iMatix and is LGPLv3 open source.

This post shows how to install ZeroMQ with the Ruby gem using Ubuntu packages and also from source.

<!-- more -->

# Installing the dependencies #

ZeroMQ needs some dependent packages; these work for me:

```  bash

sudo apt-get install uuid uuid-dev uuid-runtime
```


# Installing ZeroMQ using Chris Lea's packages #

If you want to use Chris's packages, you need first of all to add his
apt repos

``` bash

sudo add-apt-repository ppa:chris-lea/zeromq
sudo add-apt-repository ppa:chris-lea/libpgm
sudo apt-get update

```

then install his ZeroMQ packages:

``` bash

sudo apt-get install libzmq-dbg libzmq-dev libzmq1

```

The ZeroMQ Ruby gem bindings installs easily enough. Note though, the gem has to compile the C code so you may need to
install some basic build and compile packages *before* installing the gem e.g.

``` bash

sudo apt-get install build-essential gcc

``` 

The gem should then install ok but keep an eye open for errors; its
has to find zmq.h (/usr/include) and libzmq.so (/usr/lib)

``` bash

gem install zmq

```

The [gem source on github](https://github.com/zeromq/rbzmq) has a very
simple program to test whether ZeroMQ is working:

``` ruby

require "zmq"

context = ZMQ::Context.new(1)

puts "Opening connection for READ"
inbound = context.socket(ZMQ::UPSTREAM)
inbound.bind("tcp://127.0.0.1:9000")

outbound = context.socket(ZMQ::DOWNSTREAM)
outbound.connect("tcp://127.0.0.1:9000")
p outbound.send("Hello World!")
p outbound.send("QUIT")

loop do
  data = inbound.recv
  p data
  break if data == "QUIT"
end

```

Note the program doesn't end, you need to ctrl-c out.


# Installing ZeroMQ from source #

Although Chris's packages has done the biz for me in the past, 
I've wanted to understand how to install from the
source if needed (e.g. to use more recent versions of ZeroMQ than Chris supports).

The ZeroMQ tarball source can be obtained from the
[download page](http://download.zeromq.org/zeromq-2.2.0.tar.gz).
Building it was a familiar process:

``` bash

cd /tmp
wget http://download.zeromq.org/zeromq-2.2.0.tar.gz
tar -xvf zeromq-2.2.0.tar.gz
cd ./zeromq-2.2.0.tar.gz
./configure
make
sudo make install
sudo ldconfig

```

The *ldconfig* is essential.

The gem should install in exactly the same way as when using Chris's
packages, although the locations (by default) of zmq.h will be /usr/local/include and
libzmq.so will be /usr/local/lib.

But if you have any problems the gem can be manually rebuilt which may
make it easier to see where any errors are coming from.

The [Ruby bindings page](http://www.zeromq.org/bindings:ruby) has the instructions:

``` bash

cd /tmp
git clone https://github.com/zeromq/rbzmq
ruby ./extconf.rb
make
sudo make install

```

The same test program as above will test the source install just fine.


BTW Andrew Cholakian has some good [tutorial material](https://github.com/andrewvc/learn-ruby-zeromq) on using Ruby with ZeroMQ.
