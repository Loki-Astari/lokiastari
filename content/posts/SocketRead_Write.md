---
title: Socket Read/Write
date: 2016-04-09T21:11:25-0700
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
  image: /images/post/post-4.png
  hidden: false
  caption: Photo by Roman Synkevych
---

## Checking read/write success

The `read()` and `write()` commands can fail in several ways but can also succeed without reading/writing all the data. A common mistake is not checking the amount of data read/written from/to a stream. Interestingly, not all error conditions are fatal, and reading/writing can potentially be resumed after an error.

## Read
To determine whether you have read all the information available on a stream, you need to define a communication protocol (like HTTP). The protocol for the first version of this server is very simple. Messages are passed as strings (not null-terminated), and the end of the message is marked by closing the write stream. Thus, a client can send one message and receive one reply with each connection it makes.

#### getMessage()
```cpp
/*
 * Returns:     0   EOM reached.
 *                  The message is complete. There is no more data to be read.
 *              >0  Message data has been read (and a null terminator added).
 *                  The value is the number of bytes read from the stream
 *                  You should call getMessage() again to get the next section
 *                  of the message.
 *                  Note: the message is terminated when 0 is returned.
 *              -1  An error occurred.
 */
int getMessage(int socketId, char* buffer, std::ssize_t size)
{
    std::ssize_t     dataRead = 0;
    std::ssize_t     dataMax  = size - 1;

    while(dataRead < dataMax)
    {
        ssize_t get = read(socketId, buffer + dataRead, size - dataRead);
        if (get == -1)
        {
            return -1;
        }
        if (get == 0)
        {
            break;
        }
        dataRead += get;
    }
    buffer[dataRead] = '\0';
    return dataRead;
}
```

### Read Errors
This initial version treats all `read()` errors as unrecoverable, and `getMessage()` returns an error state. However, not all error codes need to result in a failure. So, in this section, I will go through some of the error codes and give some potential actions. In subsequent articles, I may revise these actions as we cover more complex ways of interacting with sockets.


The following errors are the result of programming bugs and should not happen in production.

```text
    [EBADF]            fildes is not a valid file or socket descriptor open for reading.
    [EFAULT]           Buf points outside the allocated address space.  
    [EINVAL]           The pointer associated with fildes was negative.
    [ENXIO]            A requested action cannot be performed by the device.
```
If they occur in production, they cannot be corrected pragmatically because the error occurred in a part of the code unassociated with this function.

One could argue that the application can abort because these should never happen, but for now, we will settle for the read operation aborting with an error code. If we wrap this in a C++ class to control the state of the socket, exceptions may be more appropriate, and we will look into that approach in a subsequent article.

The following errors are potentially recoverable.
{/* https://stackoverflow.com/questions/8471577/linux-tcp-connect-failure-with-etimedout */}

```text
    [EIO]              An I/O error occurred while reading from the file system.
    [ENOBUFS]          An attempt to allocate a memory buffer fails.
    [ENOMEM]           Insufficient memory is available.
    [ETIMEDOUT]        A transmission timeout occurs during a read attempt on a socket.
```
But in reality, recovering from them within the context of a read operation is not practical (you need to recover from these operations at a point where resources are controlled or user interaction is possible). So for now, we will abort the read operation with an error code (we will revisit this in a later article).

The following error codes mean no more data will be available because the connection has been interrupted.
{/* https://stackoverflow.com/questions/2974021/what-does-econnreset-mean-in-the-context-of-an-af-local-socket */}
{/* https://stackoverflow.com/questions/900042/what-causes-the-enotconn-error */}

```text
    [ECONNRESET]       The connection is closed by the peer during a read attempt on a socket.
    [ENOTCONN]         A read is attempted on an unconnected socket.
```
How the application reacts to a broken connection depends on the communication protocol. For the simple protocol defined above, we can return any data retrieved from the socket and then indicate to the calling code that we have reached the end of the message (we will revisit this in a later article). This is probably the most iffy decision in handling error codes, and returning an error code could be more appropriate. Still, I want to illustrate that we can potentially continue depending on the situation.

The following error codes are recoverable from.

```text
    [EAGAIN]           The file was marked for non-blocking I/O,
                       and no data were ready to be read.
```
These error codes are generated when you have a non-blocking stream. In a future article, we will discuss taking advantage of non-blocking streams.

```text
    [EINTR]            A read from a slow device was interrupted before
                       any data arrived by the delivery of a signal.
```
The exact action you take will depend on your application (e.g., doing useful work), but for our simple application, simply re-trying the read operation will be the standard action. Again, we will return to this, but taking advantage of timeouts will require a slightly more sophisticated approach than using the sockets API directly.

**EINTR:**

An important note about signals. There are a lot of signals that are nonlethal, which will result in this EINTR error code. However, one should note that lethal signals like SIGINT by default will kill the application and thus will not cause this error code (as the call to read() will never return).

But you can override the SIGINT signal handler and allow your application to continue, and at this point, your read operation will receive this error. How your code interacts with signals like SIGINT is beyond the scope of this article, and it will be discussed just like other signals.

#### getMessage() Improved
```cpp
/*
 * Returns:     0   EOM reached.
 *                  There is no data in the buffer.
 *              >0  Message data has been read.
 *                  If the buffer is full, then it is not null-terminated.
 *                  If the buffer is partially full, then it is null-terminated
 *                  and the next call to get getMessage() will return 0.
 *              <0  An error occurred.
 */
int getMessage(int socketId, char* buffer, std::ssize_t size)
{
    std::ssize_t     dataRead = 0;
    std::ssize_t     dataMax  = size - 1;

    while(dataRead < dataMax)
    {
        ssize_t get = read(socketId, buffer + dataRead, size - dataRead);
        if (get == -1)
        {
            switch(errno)
            {
                case EBADF:
                case EFAULT:
                case EINVAL:
                case ENXIO:
                    // Fatal error. Programming bug
                    return -3;
                case EIO:
                case ENOBUFS:
                case ENOMEM:
                    // Resource acquisition failure or device error
                    // Can't recover from here, so indicate failure
                    // and exit
                    return -2;
                case ETIMEDOUT:
                case EAGAIN:
                case EINTR:
                    //Temporary error.
                    // Simply retry the read.
                    continue;
                case ECONNRESET:
                case ENOTCONN:
                    // Connection broken.
                    // Return the data we have available and exit
                    // as if the connection was closed correctly.
                    get = 0;
                    break;
                default:
                    return -1;
            }
        }
        if (get == 0)
        {
            break;
        }
        dataRead += get;
    }
    buffer[dataRead] = '\0';
    return dataRead;
}
```

## Write
The `write()` has the same scenario as `read()`.

The following errors are the result of programming bugs and should not happen in production.

```text
     [EINVAL]           The pointer associated with fildes is negative.
     [EBADF]            fildes is not a valid file descriptor open for writing.
     [ECONNRESET]       A write is attempted on a socket that is not connected.
     [ENXIO]            A request is made of a nonexistent device, or the request
                        is outside the capabilities of the device.
     [EPIPE]            An attempt is made to write to a socket of type SOCK_STREAM
                        that is not connected to a peer socket.
```
The following errors are potentially recoverable bugs. However, recovering from them requires some form of awareness of the context not provided at the read level. So, we must generate an error to stop reading and allow the caller to sort out the problem.

```text
     [EDQUOT]           The user's quota of disk blocks on the file system containing
                        the file is exhausted.
     [EFBIG]            An attempt is made to write a file that exceeds the process's
                        file size limit or the maximum file size.
     [EIO]              An I/O error occurs while reading from or writing to the file system.
     [ENETDOWN]         A write is attempted on a socket and the local network interface
                        used to reach the destination is down.
     [ENETUNREACH]      A write is attempted on a socket and no route to the network is present.
     [ENOSPC]           There is no free space remaining on the file system containing the file.
```

The following error codes are recoverable, and we covered them above in the section on `read()`.

```text
     [EAGAIN]           The file is marked for non-blocking I/O, and no data
                        could be written immediately.
     [EINTR]            A signal interrupts the write before it could be completed.
```
The resulting put function then looks like this.

#### putMessage() Improved
```cpp
/*
 * Returns:
 *              >0  Indicates success and the number of bytes written.
 *              <0  Indicates failure.
 */
int putMessage(int socketId, char* buffer, ssize_t size)
{
    ssize_t     dataWritten = 0;

    while(dataWritten < size)
    {
        ssize_t put = write(socketId, buffer + dataWritten, size - dataWritten);
        if (put == -1)
        {
            switch(errno)
            {
                case EINVAL:
                case EBADF:
                case ECONNRESET:
                case ENXIO:
                case EPIPE:
                    // Fatal error. Programming bug
                    return -3;
                case EDQUOT:
                case EFBIG:
                case EIO:
                case ENETDOWN:
                case ENETUNREACH:
                case ENOSPC:
                    // Resource acquisition failure or device error
                    // Can't recover from here, so indicate failure
                    // and exit
                    return -2;
                case EAGAIN:
                case EINTR:
                    //Temporary error.
                    // Simply retry the read.
                    continue;
                default:
                    return -1;
            }
        }
        dataWritten += put;
    }
    return dataWritten;
}
```

# Summary

This article has shown the most important error people skip over when reading and writing to a socket: **Not all the data was transported at the same time**. The read and write command may only read/write a portion of the data you wanted to send/receive, so you must check the amount sent/received.

The next most crucial point is that not all error codes are fatal (most people actually check these) **but** an interrupt (EINTR) can be relatively common, and you can continue reading after it has happened.
# Inspiration

* 2015-Jun-25 [Impromptu TCP sender/receiver](https://codereview.stackexchange.com/q/94608/507)
* 2015-Jul-03 [Raw Text TCP Client v3](https://codereview.stackexchange.com/q/95638/507)
* 2015-Dec-20 [Server / client desynchronisation of messages ](https://codereview.stackexchange.com/q/114551/507)


