---
layout: post
title: "Clojure, Emacs, SLIME and Swank on Ubuntu Precise 12.04"
description: "Building a Clojure development environment"
category: 
tags: [clojure, leinginen, emacs, slime, swank, ubuntu, precise]
---

## Why this combo?

I've been using [jEdit](http://www.jedit.org) for some time as my general purpose editor and "IDE" (especially for Ruby on Windows) so it was natural to start there for my first forays into Clojure, dropping out to e.g. [Leiningen](https://github.com/technomancy/leiningen) for a REPL.

But debugging Clojure is more of a challenge than, say, Ruby because the stack trace is a Java one and its not as easy to associate the Clojure source file and line number with entries in the stack trace.

Also, I've found introducing print statements (e.g. *println*) into functional code make its look far noisier than OO code to the point where I find it obfuscates what the code is doing.  This is partly a matter of taste but to me functional code looks far more elegant and transitive when compared to imperative and anything that interrupts the flow is a distraction.

After reading around the Clojure community for a better solution, the combination of Emacs, SLIME and Swank (supported by Leiningen) appeared to offer a far richer development and debugging environment so I thought I'd give it a go.

<!-- more -->

In the past though getting all these components to play nice has been frustrating for some.  [Phil Hagelberg](http://technomancy.us/list) has contributed a huge amount to making the setup easy for newbies.  So this post, and the recipe in it, is really a paean to Phil (and numerous others who have made contributions!) who made this process straightforward for me.

## What is SLIME and Swank?

[Slime](http://common-lisp.net/project/slime/) is inevitably an acronym:  the *Superior Lisp Interaction Mode for Emacs*

The [manual](http://common-lisp.net/project/slime/doc/slime.pdf "SLIME pdf") say:

> SLIME is constructed from two parts: a user-interface written in Emacs Lisp, and a
> supporting server program written in Common Lisp. The two sides are connected together
> with a socket and communicate using an RPC-like protocol.

Swank is the "supporting server".

## So why would you bother?

What does this mean in practice? Its the ability to edit Clojure code in Emacs and ask SLIME to execute the code in the Swank server giving immediate feedback - truly interactive development.

## How I tested the recipe below

This post encapsulates knowledge picked up over a period of time but I have tested each of the steps in a VirtualBox (4.1.16) guest running Ubuntu Precise Pangolin 12.04 with fairly recent updates, and the usual toolset (git, wget, etc).  Nevertheless mistakes may have crept in so please let me know and I'll correct and refresh.

## Install Emacs 24 

I covered installed Emacs 24 from source in a [previous](http://ianrumford.github.com/blog/2012/06/21/installing-emacs-from-source-on-ubuntu-precise-1204/) post

Using Emacs 24 seems to forestall problems others have had.

## Install Leiningen 2 preview.  

See [here](https://github.com/technomancy/leiningen) for full details but briefly:

``` bash
	cd /tmp
	sudo wget https://raw.github.com/technomancy/leiningen/preview/bin/lein
	sudo mv lein /usr/bin/lein
	sudo chmod 755 /usr/bin/lein
```

Asking `lein` what version it is will cause it to bootstrap itself and download whatever dependencies it needs.

``` bash
lein -v
```

On my machine the last line output said:

`Leiningen 2.0.0-preview6 on Java 1.7.0_03 OpenJDK 64-Bit Server VM`

## Install Phil's Emacs starter kit

[Full details](https://github.com/technomancy/Emacs-starter-kit) again from Phil's github page.  

Following the advice given, my `~/.emacs.d/init.el` looks like this:


``` clj
(require 'package)

(add-to-list 'package-archives
	     '("marmalade" . "http://marmalade-repo.org/packages/") t)

(package-initialize)

(when (not package-archive-contents)
  (package-refresh-contents))

(defvar my-packages '(starter-kit starter-kit-lisp starter-kit-bindings clojure-mode)
  "A list of packages to be installed at launch")

(dolist (p my-packages)
  (when (not (package-installed-p p))
    (package-install p)))
```

When Emacs is next started, the packages will be downloaded automatically.

## Making SLIME/Swank integration available to all my projects

I wanted Swank integration to all my projects rather than project by project.  So my `./.lein/profiles.clj` looks like this:

``` clj
{:user {:plugins [[lein-swank "1.4.4"]]}}
```

## A simple test

To test and see the Emacs integration with SLIME/Swank, a new, test project was created:

``` bash
	cd <where I keep my projects>
	lein new swanktest
	cd ./swanktest
```

I'm using Clojure 1.4 so I edited swanktest's initial `project.clj` to say so:
	
``` clj
(defproject swanktest "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.4.0"]])
```

Any changes needed to be applied and dependencies resolved:

``` bash
	lein deps
```

Next, Emacs was satrted and swanktest's `src/swanktest/core.clj` edited by e.g. *C-x C-f* and navigating to `core.clj`

From within Emacs a Swank server was started using *M-x clojure-jack-in* and after a while an image similar to the following appeared:

{% img center /images/emacs-swank.png %}

BTW A list of SLIME commands can be found on Phil's [swank-clojure](https://github.com/technomancy/swank-clojure) page.

## What's Next?

Learning how to make the most of the integration - some experiences / examples likely to follow in later post.
