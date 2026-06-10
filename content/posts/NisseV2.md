---
title: C++ Sockets
date: 2024-11-08T18:48:31-0800
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
subtitle: Nisse
description: Nisse. The step by step creation of a C++ Server architecture.
draft: false
cover:
  image: /images/post/post-4.png
  hidden: false
  caption: Photo by Roman Synkevych
---

# [Nisse](https://github.com/Loki-Astari/Nisse)

In the previous article "A Web Server," I created the simplest web server possible.

This article covers the next stage by addressing the issues related to low-level socket programming. To be honest, we are going to take a shortcut. Dealing with ‘＊nix’ variants is complex enough, but adding Windows systems makes the process exceedingly complicated (and thus beyond the scope of this article). Here, we will introduce a C++ library to abstract the socket layer and provide a higher-level interface.

## NisseV2

All new code for this article is in the directory [V2](https://github.com/Loki-Astari/NisseBlogCode/tree/master/V2). It re-uses HTTPStuff.cpp and Stream.h from [V1](https://github.com/Loki-Astari/NisseBlogCode/tree/master/V1). It uses standard libraries and [thors-mongo](https://github.com/Loki-Astari/ThorsMongo). If you have a Unix-like environment, this should be easy to build; if you use Windows, you may need extra work. A “Makefile” is provided just as an example.

### Build & Run

```bash
  > brew install thors-mongo              # A header-only version of thors-mongo can 
                                          # be alternatively installed.
  > git clone https://github.com/Loki-Astari/NisseBlogCode.git
  > cd NisseBlogCode/V2
  > make
  > ./NisseV2 8080 /Directory/To/Server/On/Port/8080
```

## ThorsSocket

I am going to add [ThorsSocket](https://github.com/Loki-Astari/ThorsSocket), a C++ wrapper around 'File Descriptors' (FD), to simplify the web server. ThorsSocket provides a [`std::iostream`](https://en.cppreference.com/w/cpp/io/basic_iostream) interface for FD and is designed to work with the [Boost Co-Routine](https://www.boost.org/doc/libs/1_86_0/libs/coroutine2/doc/html/index.html) library to enable cooperative multitasking.

Because FDs are a very low-level OS resource, ThorsSocket provides a [`std::iostream`](https://en.cppreference.com/w/cpp/io/basic_iostream) interface to several necessary OS resources, such as pipes, files, sockets, and SSL sockets (ssockets). Note that the standard library already provides access to files through [`std::fstream`](https://en.cppreference.com/w/cpp/io/basic_fstream) but only allows blocking read/write access; in contrast, ThorsSocket provides non-blocking read/write access, allowing the executing thread to cooperatively switch to another task when an I/O operation would block and transparently resuming the I/O operation when the FD becomes available.

Another advantage of ThorsSocket is that it wraps both the C socket and Open SSL libraries, allowing the secure socket layer to be used as if it were a normal [`std::iostream`](https://en.cppreference.com/w/cpp/io/basic_iostream) object. Apart from the initial creation of the socket, its usage is entirely transparent and no different from using a standard socket (or even a file).

### [NisseV2.cpp](https://github.com/Loki-Astari/NisseBlogCode/blob/master/V2/NisseV2.cpp)

ThorsSocket provides the class `ThorsAnvil::ThorsSocket::SocketStream` that wraps a FD and implements the `std::iostream` interface. But the HTTPStuff interface uses a `Stream` interface as defined in the file `Stream.h`. So we must provide a simple wrapper.

```cpp
class Socket: public Stream
{
    ThorsAnvil::ThorsSocket::SocketStream    stream;
    public:
        Socket(ThorsAnvil::ThorsSocket::SocketStream&& stream)
            : stream(std::move(stream))
        {}

        virtual std::string_view    getNextLine()               override
        {
            static std::string line;
            std::getline(stream, line);
            return line;
        }
        virtual void ignore(std::size_t size)                   override {stream.ignore(size);}
        virtual void sendMessage(std::string const& message)    override {stream << message;}
        virtual void sync()                                     override {stream.sync();}
        virtual void close()                                    override {stream.close();}
        virtual bool hasData()  const                           override
        {return static_cast<bool>(stream);}
};
```

The previous `Server` class in V1 is replaced by the class `ThorsAnvil::ThorsSocket::Server`.

Replacing these two classes has removed much of the complexity of the server code. Note: I am using a library I am very familiar with (and the author of), but several C++ libraries would provide similar functionality and could be used similarly. My point is that the C interface to FD, though very flexible, is on the complex side, and this complexity can be abstracted using an appropriate C++ library.

### Next Steps

Thus far, we have only used simple sockets. However, modern servers also need to handle SSL connections. The following article will explain how to set up a server to accept and correctly authenticate an SSL connection.



