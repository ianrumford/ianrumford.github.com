---
layout: post
title: "Installing emacs from source on Ubuntu Precise 12.04"
description: "Brief instruction on how to install emacs on Ubuntu"
category: 
tags: [emacs, Ubuntu, precise, pangolin, "12.04"]
---


The heavy lifting for this recipe was taken from a [post](http://chrisperkins.blogspot.co.uk/2011/07/building-emacs-24.html) by Chris Perkins (thanks Chris!).

Install the Prereqs

``` bash
	sudo apt-get install autoconf automake texinfo libgtk2.0-dev libxpm-dev libjpeg-dev libgif-dev libtiff4-dev libncurses5 libncurses5-dev
```

Clone emacs repo (into /tmp in my example).  This takes a while - emacs is surpisingly big!

``` bash
	cd /tmp
	git clone https://github.com/emacsmirror/emacs.git
	cd ./emacs
```

Pretty standard build process. All the compilations can take a while.

``` bash
	make distclean
	./autogen.sh
	./configure
	make
	sudo make install
```

Quick test:

``` bash
	emacs -Q
```

Just remains to learn how to use it properly ...
