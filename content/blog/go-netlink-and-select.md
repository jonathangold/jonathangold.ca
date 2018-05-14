---
title: "Go Netlink and Select"
date: 2018-04-29T01:37:02-04:00
draft: true
---

Go is a wonderful programming language, nestled comfortably in between
low-level languages like C or Rust, and higher level languages like Python or
Ruby. The convenience one gains by doing systems programming in Go, thanks to
perks like syntactic simplicity and first-class concurrency, is offset by
annoying error handling, lack of direct memory access, and some glaring
omissions from the standard library (generics and binary literals come to mind.)

This article will describe the process of writing the network resource for
[mgmt](http://github.com/purpleidea/mgmt), a tool for next generation
distributed, event-driven, parallel config management. The network resource
is tasked with applying the prescribed network configuration, monitoring the
defined device, and correcting its state if it diverges from the configuration.
In simpler terms, it sets up a network device and makes sure it stays set up
exactly how you want.

Unfortunately Go's net library is only useful when it comes to checking the
network state, and lacks the ability to apply changes to an interface, so it
was necessary to venture outside the comfort of the standard library and employ
a more powerful solution. Enter Netlink.

There are several libraries available to interface with kernel netlink sockets
at various levels of abstraction, but none of them are without problems.
