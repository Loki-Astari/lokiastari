---
title: C++ Wrapper for Socket
date: 2016-05-26T21:13:39-0700
author: Loki Astari (C)2016
comments: true
categories:
- C++
- Sockets
- C++-By-Example
- Coding
series:
- Sockets
tags:
- Sockets
subtitle: So you want to learn C++
description: Socket wrappers in C++
draft: false
cover:
  image: /images/post/post-5.png
  hidden: false
  caption: Photo by Oscar Nilsson
---
The last two articles examined the "C Socket" interface provided by the OS. In this article, I wrap this functionality in a simple C++ class to provide guaranteed closing and apply a consistent exception strategy. The first step is to rewrite the client/server code with all the low-level socket code removed. This will help identify the interface that the wrapper class needs to implement.

The client code becomes trivial. Create a `ConnectSocket` specifying the host and a port. Then, communicate with the server using the `putMessage()` and `getMessage()`. Note: I am continuing to use the trivial protocol that was defined in the last article: `putMessage()` writes a string to the socket then closes the connection; `getMessage()` reads a socket until it is closed by the other end (I will cover more sophisticated protocols in a subsequent article).

#### [client.cpp](https://github.com/Loki-Astari/Examples/blob/master/Version2/client.cpp)
```cpp
ConnectSocket    connect("localhost", 8080);          // Connect to a server
ProtocolSimple   connectSimple(connect);              // Knows how to send/recv
                                                      // a message over a socket
connectSimple.sendMessage("", "A test message going to the server");

std::string message;
connectSimple.recvMessage(message);
std::cout << message << "\n";
```


For the server end, this is nearly as trivial as the client. Create a `ServerSocket` and wait for incoming connections from clients. When we get a connection, we return a `DataSocket` object. The reason for returning a new socket-like object is that this mimics the behavior of the underlying `::accept()` call, which opens a new port for the client to interact with the server. The additional benefit of separating this from the `ServerSocket` is that a subsequent version may allow multiple connections, and we want to interact with each connection independently without sharing state, potentially across threads, so modelling it with an object makes sense in an OO world.

#### [server.cpp](https://github.com/Loki-Astari/Examples/blob/master/Version2/server.cpp)
```cpp
ServerSocket   server(8080);                          // Create a listening connection
while(true)
{
    DataSocket      accept  = server.accept();        // Wait for a client to connect
    ProtocolSimple  acceptSimple(accept);             // Knows how to send/recv
                                                      // a message over a socket

    std::string message;
    acceptSimple.recvMessage(message);
    std::cout << message << "\n";

    acceptSimple.sendMessage("", "OK");
}
```

Surprisingly, this gives us three types of socket interface (not the two most people expect).

* The ServerSocket class has no ability to read/write; just accept connections
* The ConnectSocket class connects and can be used to read/write
* The DataSocket class is an already connected socket that can be used to read/write

Since a socket is a resource we don't want to be able to duplicate it, but it can be moved.

This lets me define a very simple interface like this:

#### [Socket.h](https://github.com/Loki-Astari/Examples/blob/master/Version2/Socket.h)
```cpp
// An RAII base class for handling sockets.
// Socket is movable but not copyable.
class BaseSocket
{
    int     socketId;
    protected:
        // Designed to be a base class not used directly.
        BaseSocket(int socketId);
        int getSocketId() const {return socketId;}
    public:
        ~BaseSocket();

        // Moveable but not Copyable
        BaseSocket(BaseSocket&& move)               noexcept;
        BaseSocket& operator=(BaseSocket&& move)    noexcept;
        void swap(BaseSocket& other)                noexcept;
        BaseSocket(BaseSocket const&)               = delete;
        BaseSocket& operator=(BaseSocket const&)    = delete;

        // User can manually call close
        void close();
};

// A class that can read/write to a socket
class DataSocket: public BaseSocket
{
    public:
        DataSocket(int socketId);

        bool getMessage(std::string& message);
        void putMessage(std::string const& message);
        void putMessageClose();
};

// A class that connects to a remote machine
// Allows read/write access to the remote machine
class ConnectSocket: public DataSocket
{
    public:
        ConnectSocket(std::string const& host, int port);
};

// A server socket that listens on a port for a connection
class ServerSocket: public BaseSocket
{
    public:
        ServerSocket(int port);

        // An accepts() waits for a connection, and returns a socket
        // object that the client can use for communication
        DataSocket accept();
};
```
Taking the existing code and wrapping this interface around it becomes trivial. The complete code is provided [here](https://github.com/Loki-Astari/Examples/tree/master/Version2).

In the previous article, I discussed the different types of errors that can be generated by read/write. I take the following code a step further. Since the code is wrapped inside a class and thus can more cleanly control the socket resources, it feels more natural to use exceptions rather than error codes. Consequently, error codes are not leaked across any public API boundary.

1. domain_error
    * This is caused by an error that theoretically cannot happen (since we have full control of the class). If this type of error occurs, there is a bug in the socket code or massive data corruption. Consequently, you should not try to catch these exceptions as the code has a fundamental bug. It is better to let the application exit as there is likely substantial corruption of any data.
1. logic_error
    * This is caused by an error that theoretically cannot happen if the class is used correctly. This means that the calling code has some form of logic error. It is caused by calling any method on a socket object that was previously closed or moved. Again, this type of error should not be caught (but can be). You should remove all potential for this type of error through good unit tests.
1. runtime_error:
    * This is caused by an unlikely situation that the Socket code cannot handle. This type of error requires a broader context to be resolved. As a result, the socket code will throw an exception that the user can catch and potentially correct.


