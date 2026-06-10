---
title: Nisse - Origins of a Server Library
date: 2024-11-04T12:50:31-0800
author: Loki Astari, (C)2024
comments: true
categories:
- C++
- Nisse
- Server
- C++-By-Example
- Coding
series:
- Nisse
tags:
- Nisse
subtitle: C++ By Example
description: Nisse. The step by step creation of a C++ Server architecture.
draft: false
cover:
  image: /images/post/post-2.png
  hidden: false
  caption: Photo by ThisisEngineering RAEng
---

# [Nisse](https://github.com/Loki-Astari/Nisse)

[Nisse](https://github.com/Loki-Astari/Nisse) is a C++ library that simplifies the creation of [C++ web-based applications](https://github.com/Loki-Astari/Nisse/tree/master/src/Examples). It provides a framework for handling incoming requests and executing user-defined code asynchronously. To achieve this, Nisse creates managed socket connections that can be utilized by user code. It will automatically suspend the execution of user code and *RE-USE* the thread if the user code blocks during a read/write operation on a connection, thus offering easily accessible asynchronous functionality.

The concept is to make it simple for beginner engineers to write regular, synchronous-looking C++ code that is easy to reason about and automatically provides the asynchronous processing that is inherently needed for web-based applications to function efficiently.

## Why

I admire the Javascript/NodeJS model that makes it (relatively) easy for beginners to quickly and intuitively build simple working web-based applications. The ability to quickly create and deploy simple web apps is a great teaching platform that has allowed easy experimentation by beginners and thus provides them with a platform to learn new skills that are easy to demonstrate to friends (The `Look what I have built`, word of the month).

In comparison, building simple web-based apps in C++ is nontrivial and unsuitable for teaching. I doubt it will ever be as simple to create production web apps in C++, but I hope that Nisse can help beginners quickly throw together a rapid prototype of a web-based application and shout out to their friends, "Look what I have built; come check it out …".

## How

Nisse is built on some standard libraries ([libEvent](https://libevent.org/) and [Boost CoRoutine2](https://www.boost.org/doc/libs/1_86_0/libs/coroutine2/doc/html/index.html)) that do the heavy lifting. Nisse wires them together in a simple-to-use framework. Boost-CoRoutines provide cooperative multitasking, allowing user code to be suspended when a core/thread is blocked waiting on IO. libEvent provides the dispatch loop keyed on IO operations, allowing user code to be rescheduled when the operation can be resumed.

## Plan

I will create a series of posts explaining how Nisse works while providing mini tutorials on C++ coding and web applications as I go. Once this initial series is complete, I will use Nisse as a platform to build small applications that demonstrate C++ coding and provide lessons in modern C++ techniques that are fun and easy to understand for beginners.

* [Nisse V1](https://lokiastari.com/posts/NisseV1): A Web Server: A very basic C++ Web Server
* [Nisse V2](https://lokiastari.com/posts/NisseV2): C++ Sockets:  A C++ wrapper around C Sockets and SSL.
* [Nisse V3](https://lokiastari.com/posts/NisseV3): SSL Certificates: What is a certificate and where do I get one.
* [Nisse V4](https://lokiastari.com/posts/NisseV4): Multi Threading: Handling multiple request asynchronously.
