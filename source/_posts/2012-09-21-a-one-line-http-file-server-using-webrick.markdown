---
layout: post
title: "A one-line http file server using Webrick"
date: 2012-09-21 18:39
comments: true
categories: 
---

I needed a simple http file server to serve  files around my home
network.

A bit of googling found this.  It trivial but so, so useful.

It uses the Ruby Webrick server so you have to have Ruby installed.

Assuming the path of the filesystem you want to sere is */home/me/heretheyare*,
run this in a terminal:

``` bash

ruby -rwebrick -e 'WEBrick::HTTPServer.new(:Port=>1234,:DocumentRoot=>"/home/me/heretheyare").start'

```

Use your browser, *wget*, whatever to pull the file(s) you need.
Remember to include the port (if its not 80).

It should work on Windows but I haven't tried; I use it on Ububntu.

