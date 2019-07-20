---
layout: post
title: MaestroFlow - Composing Reactive Programs
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/YpC7Xy-4jJc" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Interprocess communication has been around for quite some time, but nobody seems to use it outside of simple command line programs. Instead, software exists in a walled ecosystem, where programs are unable to communicate except for copy/paste and saving files. MaestroFlow is a program that aims to solve these issues.

MaestroFlow consists of a centralized server with a GUI display, and programs that use the MaestroFlow API can register themselves. Inside the GUI display, users can set up the data routing between programs. When an event happens in a program, it notifies MaestroFlow by calling a ``notify`` method. To receive an event, a program registers a callback handler which will be invoked with the event data. The sources and sinks are typed so that invalid combinations are not allowed.

{% gist 219512b2e2938f30ba45a43977835dc4 %}

{% gist 360927831305989cc5041eb25f25718a %}

## Tear down this wall!

Pipes are a concept familiar to users of Linux command line programs. In Linux, there are special files that act as dynamic input/output devices. ``/dev/random`` is one of them. The [Plan 9](https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs) operating system takes this a step further - everything is a file. This includes things such as network devices, I/O devices like mice, and pretty much everything else. Of course, Plan 9 never caught on, so we've been languishing in an era where programs are completely oblivious to each other.

Part of the issue is that the interface for setting up the pipes is completely alien to most users. Only programmers use command line interfaces frequently. Piping programs together seems to be relegated to a handful of Linux text processing functions. MaestroFlow aims to solve this issue by presenting a high level directed graph interface for hooking programs together.

Since the Internet won and named pipes didn't, MaestroFlow makes use of the HTTP networking protocol to maximize compatibility across devices and platforms. It is conceivable that programs on remote systems and devices on the local network could be hooked together into a coherent system for doing useful things. The data in MaestroFlow is strongly typed, it should be possible to write interesting and complex behaviours that make use of things such as higher order functions (higher order programs?). I'm sure that someone will be able to fit a monad in there. Getting an API for a new language is fairly simple and only requires a HTTP sender/receiver library. The sample [Python API implementation](https://github.com/calebh/maestroflow/blob/master/maestroflow/python/maestroflow.py) is only 127 lines long!

If you're interested in contributing, feel free to check out the MaestroFlow GUI and Python API in the GitHub repository: [https://github.com/calebh/maestroflow/](https://github.com/calebh/maestroflow/). I've also set up a Gitter chat for a more interactive discussion: [https://gitter.im/MaestroFlow](https://gitter.im/MaestroFlow). Currently the MaestroFlow GUI is written in Electron and uses Cytoscape.js for drawing the directed graphs. There are many obvious improvements that could be made to the interface, so contributions in this direction are welcome. Also I'm not a designer, so the interface layout is pretty ugly.

### Related Work

* [Eros (2007)](http://conal.net/papers/Eros/)
* [Node-RED](https://nodered.org/)
* [Total.js](https://www.totaljs.com/)
