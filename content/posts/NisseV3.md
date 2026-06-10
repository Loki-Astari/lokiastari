---
title: SSL Certificates
date: 2024-11-10T12:48:30-0800
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
  image: /images/post/post-1.png
  hidden: false
  caption: Photo by Possessed Photography
---

# [Nisse](https://github.com/Loki-Astari/Nisse)

In the previous article, "C++ Sockets," I mentioned that Thors-mongo had built-in support for SLL connections but glossed over the details. In this article, we will review the necessary additions. From the server side, you need an SSL certificate. But what is an SSL certificate, and how do you obtain one?

## SSL Certificates

The SSL certificate has two primary purposes.

1. Securely encrypt all traffic between the client and server.
2. Provides trust that you are talking to a server on the requested domain.

### The Basics

Trusted certificate authorities issue SSL certificates and contain information that validates the certificate cryptographically. When a browser connects to a website, the server returns the certificate, which includes crucial information such as the issuing company. Before establishing a secure connection, the browser validates the certificate against the issuing company’s SSL certificate to ensure it is not forged. If the issuing company is a subsidiary, it recursively looks up the parent company that issued the certificate and validates their certificates until it reaches a root certificate. This process creates a chain of trust leading to a known trusted root certificate.

All modern browsers can find and validate root certificates. They also provide information about compromised certificate authorities whose certificates should no longer be trusted. Therefore, it is essential to download only trusted browsers and keep them current.

Once a certificate has been validated, the browser knows it is communicating with a server for a specific domain (as the domain information is included in the certificate). Modern browsers indicate a secure connection, usually with a green padlock beside the URL. The browser can then use the public key in the certificate to establish a secure connection with the domain you are connecting to.

### Where can you get a certificate

If you are a large company trusted by the Internet, you can create your own root certificate. While this is not difficult, it is of little value if no one trusts you. Therefore, the crucial step for a root certificate holder is becoming trusted by tools that validate certificates. Losing trust would mean having your root certificate removed from the list people use to validate certificates (i.e., the Chrome browser team would remove your certificate from the list of root certificates it trusts). This loss of trust would affect the root certificate authority and all downstream companies that hold certificates based on that root certificate; a browser would mark any sites using an untrusted root certificate as not being trusted.

Obtaining a certificate from a trusted root authority can be expensive and is typically only done by companies that issue certificates. However, you don't necessarily need a certificate from a root authority; you can get a certificate from a vendor that has a certificate issued by a root authority. Every trusted certificate authority has a process to validate that you own a domain, which allows them to create and deliver a signed certificate to you, but each authority's process is different. But for normal people (and small companies), you can get relatively cheap SSL certificates from most domain providers (like `GoDaddy`) that will create and manage your SSL Certificate for your domain.

### Free Certificates

You can get Free SSL Certificates from a company called [Let’s Encrypt](https://letsencrypt.org/).

## Code

All the code for this article is in the directory [V3](https://github.com/Loki-Astari/NisseBlogCode/tree/master/V3) directory. It uses standard libraries and [thors-mongo](https://github.com/Loki-Astari/ThorsMongo). If you have a Unix-like environment, this should be easy to build; if you use Windows, you may need to do some extra work. A “Makefile” is provided just as an example.

### Build & Run

```bash
  > brew install thors-mongo              # A header-only version of thors-mongo
                                          # can be alternatively installed.
  > git clone https://github.com/Loki-Astari/NisseBlogCode.git
  > cd NisseBlogCode/V3
  > make
  > ./NisseV3 8080 /Directory/To/Server/On/Port/8080 /etc/letsencrypt/live/<MySite.com>/
```

## How to Use ThorsSocket With an SSL Certificate

In ThorsSocket, a normal socket is created with the following code:

```cpp
    // Make the code easier to read.
    namespace TASock = ThorsAnvil::ThorsSocket;
    namespace SFS = std::filesystem;
    
    TASock::Server   server(ServerInit{port});
    
    // A normal bi-direconal socket.
    TASock::Socket   socket = server.accept();
```

To create a secure connection, specify the location of the SSL certificate file on the host file system. If you use [Let’s Encrypt](https://letsencrypt.org/), the default location for the SSL certificate is `/etc/letsencrypt/live/<domainName>/fullchain.pem`, and the private key is located at `/etc/letsencrypt/live/<domainName>/privkey.pem`. You can then create a secure SSL connection with:

```cpp
    // The path where the “thorsanvil.dev” certificates are stored.
    std::string   certPath = "/etc/letsencrypt/live/thorsanvil.dev";
    
    // Make the code easier to read.
    namespace TASock = ThorsAnvil::ThorsSocket;
    namespace SFS = std::filesystem;

    
    // Create a certificate object that contains the SSL Certificate and private key.
    // Note: Some files require you to provide a password to access the certificate;
    // please see the documentation on how to add appropriate lambda’s to retrieve the
    // password from secure storage (as they should not be in the code)

    TASock::CertificateInfo    certificate{
                                   SFS::canonical(SFS::path(certPath) /= "fullchain.pem"),
                                   SFS::canonical(SFS::path(certPath) /= "privkey.pem")
                               };
    TASock::SSLctx             ctx{TASock::SSLMethodType::Server, certificate};
    TASock::Server             server(TASock::SServerInit{port, std::move(ctx)});
    
    // A secure bi-direconal SSL socket.
    Socket   socket = server.accept();
```

## How to handle SSL or regular connections

The `ThorsAnvil::ThorsSocket::Socket` class can be constructed with `TASock::ServerInfo` or `TASock::SServerInfo` object. To simplify this, the constructor also takes a `std::varient<TASock::ServerInfo, TASock::SServerInfo>` (aka `TASock::ServerInit`) that can contain either of these objects. This allows us to write a function to initialize a server connection to initialize either type.

```cpp
namespace TASock = ThorsAnvil::ThorsSocket;

// Always need a port to listen on.
// If we have an SSL certificate, then pass its location in `certPath` (which may be empty)
TASock::ServerInit getServerInit(int port, std::optional<std::filesystem::path> certPath)
{
    // Make the code easier to read.
    namespace TASock = ThorsAnvil::ThorsSocket;
    namespace SFS = std::filesystem;

    
    // If there is only a port.
    // i.e., the user did not provide a certificate path return a `ServerInfo` object.
    // This will create a normal listening socket.
    if (!certPath.has_value()) {
        return TASock::ServerInfo{port};
    }
    
    // If we have a certificate path.
    // Use this to create a certificate object.
    // This assumes the standard names for these files as provided by "Let's encrypt".
    TASock::CertificateInfo     certificate{
                                    SFS::canonical(SFS::path(*certPath) /= "fullchain.pem"),
                                    SFS::canonical(SFS::path(*certPath) /= "privkey.pem")
                                };
    TASock::SSLctx              ctx{TASock::SSLMethodType::Server, certificate};
    
    // Now that we have created the appropriate SSL objects needed.
    // We return an SServierInfo object.
    // Please Note: This is a different type to the ServerInfo returned above (one less S in the name).
    return TASock::SServerInfo{port, std::move(ctx)};
    
    // We can return these two different types because
    // ServerInit is actually a std::variant<ServerInfo, SServerInfo>
}
```

We need to adjust `main()` slightly to use this.

The only change in Nisse's API from V1 is that it allows the user to provide a certificate and key file.

```cpp
int main(int argc, char* argv[])
{
    if (argc != 4 && argc != 3)
    {
        std::cerr << "Usage: NisseV3 <port> <documentPath> [<SSL Certificate Path>]" << "\n";
        return 1;
    }
    ...
        std::optional<std::filesystem::path>    certDir;
        if (argc == 4) {
            certDir = std::filesystem::canonical(argv[3]);
        }
    ...
        WebServer   server(getServerInit(port, certDir), contentDir);
    ...
}
```

### What is the next step

We have a Web Server that can connect over SSL. However, the server only handles requests serially. The following article will look at how to add some basic parallelism to support multiple simultaneous connections.



