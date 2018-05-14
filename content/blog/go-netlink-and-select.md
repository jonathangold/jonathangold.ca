---
title: "Go Netlink and Select"
date: 2018-04-29T01:37:02-04:00
draft: true
---

Go is a wonderful programming language, nestled comfortably in between low-level languages like C or Rust, and higher level languages like Python or Ruby. The convenience one gains by doing systems programming in Go, thanks to _ _ _ _  concurrency, is offset by annoying error handling, lack of direct memory access, and some glaring omissions from the language itself (generics and binary literals come to mind.)

This article will describe the process of writing the network resource for [mgmt](http://github.com/purpleidea/mgmt), a tool for next generation distributed, event-driven, parallel config management. The network resource is tasked with applying the prescribed network configuration, monitoring the defined device, and correcting its state if it diverges from the configuration. In simpler terms, it sets up a network device and makes sure it stays set how you want.

Unfortunately Go's net library is only useful when it comes to checking the network state,
