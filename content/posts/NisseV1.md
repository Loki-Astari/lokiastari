---
title: A Web Server
date: 2024-11-06T12:48:31-0800
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
  image: /images/post/post-3.png
  hidden: false
  caption: Photo by Ken Blode
---

# [Nisse](https://github.com/Loki-Astari/Nisse)

Describing all aspects of Nisse in a single article would be complex and overwhelming. I want to start with a beginner's perspective and address a real-world problem many beginners attempt. I will identify some of the issues and introduce solutions one at a time, ultimately leading to the final library, which is Nisse.

The simplest web application I can build is a web server. A web server maintains no state between calls, and the interface is straightforward. The client sends an [HTTP message](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#http_requests) and, in return, receives an [HTTP response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#http_responses).

Even better, we already have applications (browsers) that handle all the difficult aspects of interacting with a web server, so we don't need to write a client-side application.

## NisseV1

All the code for this article is in the directory [V1](https://github.com/Loki-Astari/NisseBlogCode/tree/master/V1). It uses only the standard libraries, so it should be easy for anybody to build. A “Makefile” is provided as an example.

### Build & Run

```bash
  > git clone https://github.com/Loki-Astari/NisseBlogCode.git
  > cd NisseBlogCode/V1
  > make
  > ./NisseV1 8080 /Directory/To/Server/On/Port/8080
```

I will go over a couple of things in the file that I believe are worth explicitly pointing out:

###  [HTTPStuff.cpp](https://github.com/Loki-Astari/NisseBlogCode/blob/master/V1/HTTPStuff.cpp)

The code in this file is just enough to parse an HTTP Get request and provide a file in the HTTP Response object. It is provided here solely so that all examples can be compiled. Note: this is not part of Nisse, I will come back and address the code in a subsequent article on HTTP and expand this to be more comprehensive.

### [Stream.h](https://github.com/Loki-Astari/NisseBlogCode/blob/master/V1/Stream.h)

This is the interface between the server code and the HTTPStuff. For these articles, I wanted to keep the interface as simple as possible, and again, this is not part of Nissa.

### [NisseV1.cpp](https://github.com/Loki-Astari/NisseBlogCode/blob/master/V1/NisseV1.cpp)
 
```cpp
int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        std::cerr << "Usage: NisseV1 <port> <documentPath>" << "\n";
        return 1;
    }
    
    try
    {
        static const int port = std::stoi(argv[1]);
        static const std::filesystem::path  contentDir  = std::filesystem::canonical(argv[2]);
    
        std::cout << "Nisse Proto 1\n";
        WebServer   server(port, contentDir);
        server.run();
    }
    catch(std::exception const& e)
    {
        std::cerr << "Exception: " << e.what() << "\n";
        throw;
    }
    catch(...)
    {
        std::cerr << "Exception: UNKNOWN\n";
        throw;
    }
}
```

The `main()` function obtains and validates user input to initialize the server. If everything is in order, it creates a `WebServer` object and starts the application dispatch loop by calling `run()`.

Here are two main points to note:
1. Unlike most beginner tutorials, web applications are event-driven. They operate with a "dispatch loop" that executes user code when events occur rather than following a sequential list of commands. In this example, the `run()` function represents the dispatch loop and manages all the underlying details. Typically, frameworks allow you to register user code for specific events, but since this is a simple application, it simply handles an HTTP request.

2. I use exceptions to handle critical errors. Many engineers believe exceptions are problematic because they obscure control flow (I partially agree). However, I prefer using exceptions judiciously, as they reduce the explicit error-handling code required for serious issues that necessitate application shutdown. When shutting down a C++ application, it is essential to unwind the stack correctly and ensure all relevant destructors are called to release resources; therefore, `abort()`, `terminate()`, and `exit()` are usually inappropriate (unless there is corruption) in C++ applications (unlike in C). You **MUST** catch exceptions in `main()`, as it is implementation-defined whether the stack unwinds if an exception escapes the `main()` function. By catching the exception in `main()`, you ensure the stack unwinds correctly, and all destructors are called. Then, you can generate appropriate messages and logs before re-throwing the exceptions. Re-throwing allows the OS to take necessary actions when the application exits abnormally.

### Socket Code

A minor call out. The socket code is all C code (and thus in the global namespace). You will see in my code that all C code is prefixed by `::` to make sure I explicitly call the C version of these functions. For example, when creating a server-end socket, I call `::socket(),` `::bind(),` `::listen()`, and `::accept()`. This ensures that I do not accidentally call similarly named methods.
 
A lot of the code in this example creates and handles sockets and does a rudimentary job of checking and handling basic errors that these functions could generate; class [Server](https://github.com/Loki-Astari/NisseBlogCode/blob/master/V1/NisseV1.cpp#L140-L190) is 50 lines, and class [Socket](https://github.com/Loki-Astari/NisseBlogCode/blob/master/V1/NisseV1.cpp#L192-L391) is another 200 lines, representing more than half the code.

### Errors

The C socket code has a simple interface and returns `-1` when an error occurs. An error when creating sockets usually means programming issues, but in all these cases, the error is unrecoverable, so we simply throw an exception to exit the application (and clean up any connections).

```cpp
Server::Server(int port)
{
    fd = ::socket(AF_INET, SOCK_STREAM, 0);
    if (fd == -1) {
        throw std::runtime_error{Message{} << "Failed to create socket: "
                                           << errno 
                                           << " " 
                                           << strerror(errno)};
    }
    
    struct ::sockaddr_in        serverAddr{};
    serverAddr.sin_family       = AF_INET;
    serverAddr.sin_port         = htons(port);
    serverAddr.sin_addr.s_addr  = INADDR_ANY;
    
    int bindStatus = ::bind(fd,
                            reinterpret_cast<struct ::sockaddr*>(&serverAddr),
                            sizeof(serverAddr));
    if (bindStatus == -1) {
        throw std::runtime_error{Message{} << "Failed to bind socket: "
                                           << errno
                                           << " "
                                           << strerror(errno)};
    }
    
    int listenStatus = ::listen(fd, backlog);
    if (listenStatus == -1) {
        throw std::runtime_error{Message{} << "Failed to listen socket: "
                                           << errno
                                           << " "
                                           << strerror(errno)};
    }
}
```

During read/write operations, the return value is either -1 on failure or the number of bytes received/sent. We must address two issues: an error is not always fatal, and success does not always indicate that the operation has been completed as requested. Therefore, code must be explicitly added to check for all scenarios. This typically means that read/write operations are performed in a loop until a critical failure occurs or you decide that enough data has been retrieved/sent.


```cpp
void Socket::readMoreData(std::size_t maxSize, bool required)
{
    // This function reads "MoreData" onto the end of buffer.
    // Note: There may be data already in buffer, so this appends it.
    //       We will read no more than "maxSize" more data into buffer.
    // The required flag indicates if we must read "maxSize" if true
    // the loop will continue until we get all the data; otherwise the
    // function returns after any data is received.
    std::size_t     currentSize = std::size(buffer);
    std::size_t     amountRead  = 0;
    buffer.resize(currentSize + maxSize);
    
    while (readAvail && amountRead != maxSize)
    {
        int nextChunk = ::read(fd, 
                               &buffer[0] + currentSize + amountRead,
                               maxSize - amountRead);
        if (nextChunk == -1 && errno == EINTR) {
            continue;           // An interrupt can be ignored. Simply try again.
        }
        if (nextChunk == -1 && errno == ECONNRESET) {
            readAvail = false;  // The client dropped the connection. Not a problem
            break;              // But no more data can be read from the socket.
        }
        if (nextChunk == -1) {
            buffer.resize(currentSize + amountRead);
            throw std::runtime_error(Message{} << "Catastrophic read failure: "
                                               << errno
                                               << " " << strerror(errno));
        }
        if (nextChunk == 0) {   // The connection was closed gracefully.
            readAvail = false;  // OS handled all the niceties.
        }
        amountRead += nextChunk;
        if (!required) {
            break;  // have some data. Let's exit and see if it is enough.
        }
    }
    // Make sure to set the buffer to the correct size.
    buffer.resize(currentSize + amountRead);
}
```

I want to emphasize that I have done the bare minimum in error checking here. This is adequate for a simple demo program but insufficient for production code. The problem is that error codes are not entirely consistent across ‘＊nix’ variants and differ significantly from Windows systems. I will discuss this in the following article. 

## Next Step

Address inconsistencies between ‘＊nix’ and `Windows` platforms.

