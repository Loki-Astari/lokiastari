---
title: 'Mug: A C++ server with hot-reloadable plugins'
date: 2026-03-04T12:00:00-0800
author: Loki Astari, (C)2026
comments: true
categories:
- C++
- Mug
- Nisse
- C++-By-Example
- Coding
series:
- Mug
tags:
- Mug
subtitle: C++ By Example
description: Mug is a small HTTP server executable that loads REST endpoints from
  shared libraries and hot-reloads them when rebuilt. It’s aimed at making the edit/build/test
  loop feel closer to Python Bottle, while keeping the core server fast and C++-native.
draft: false
cover:
  image: /images/post/post-2.png
  hidden: false
  caption: Photo by ThisisEngineering RAEng
---

I like Python’s Bottle/Flask development loop: change code, hit refresh, keep moving.

In C++ we traditionally pay for “real builds” with server restarts, cold state, and the mental overhead of wiring a server skeleton for every prototype. **Mug** is my attempt to keep the runtime production grade, while making the development loop feel lightweight: **build a shared library, Mug loads it, and when you rebuild the library Mug hot-reloads it**.

Mug is also intentionally small. It delegates the hard work to the underlying stack:

* `NisseServer`: event loop + coroutines + worker threads
* `NisseHTTP`: request parsing + routing + responses
* `ThorsSerializer`: parse JSON config
* `ThorsLogging`: logging + “log-and-throw” helpers

### Goal
* Keep a stable `mug` executable running.
* Put your application code in **plugins** (shared libraries).
* Rebuild just the plugin while Mug is running.
* Mug detects the file changed and reloads it.

### Warning
This is a sharp tool.

* Hot-reload is fantastic for development speed.
* Hot-reload is also where you discover which parts of your code accidentally rely on process lifetime.

If your plugin “works” only because you have global state that survives forever, you’re going to have a bad day (and Mug is going to help you find that day quickly).

## How Mug is structured (at a high level)
Mug is a standalone executable that:

1. Loads a JSON config file.
2. Starts one or more HTTP/HTTPS listeners.
3. Loads shared libraries configured for those ports.
4. Calls your plugin’s `start()` to register routes.
5. Periodically checks the `.so/.dylib` file timestamps and reloads on change.

Internally it’s essentially:

* `MugServer` (inherits from `NisseServer`)
* one `HTTPHandler` per configured port
* one `PyntHTTPControl` on a control port for stop/ping commands
* `DLLibMap` that owns the dynamic libraries and the plugin instances

## Running Mug
Mug is configured primarily by a JSON file. You can specify it explicitly, or let Mug search defaults.

### Command line arguments
Mug supports the following flags (these are the actual ones used by the current code):

* `--config=<file>`: path to the config (if missing it searches `./mug.cfg`, `/etc/mug.cfg`, `/opt/homebrew/etc/mug.cfg`)
* `--silent`: headless mode (don’t print startup info)
* `--help`: show help
* `--logFile=<file>`: log to file
* `--logSys[=<app>]`: log to syslog (default app name is argv[0])
* `--logLevel=<Level>`: `Error|Warn|Info|Debug|Track|Trace|All` (or a number)

## The config file
The config is JSON. Here is a minimal version that:

* exposes a control port on `8079`
* listens on `8080`
* loads one plugin
* enables hot reload by checking every 500ms

#### mug.cfg
```json
{
  "controlPort": 8079,
  "libraryCheckTime": 500,
  "servers": [
    {
      "port": 8080,
      "actions": [
        {
          "pluginPath": "/absolute/path/to/libHelloBot.dylib",
          "config": { "greeting": "Hello" }
        }
      ]
    }
  ]
}
```

Notes:

* `libraryCheckTime` is milliseconds.
  + `0` disables checking entirely.
  + `500` is a decent development value.
* `servers` lets you run multiple ports and attach multiple plugins to each.
* `certPath` is optional per server:
  + if present, Mug expects files named `fullchain.pem` and `privkey.pem` inside that directory and will start HTTPS on that port.

## Writing a plugin (the real API)
The plugin interface is `ThorsAnvil::ThorsMug::MugPlugin`:

* `start(handler)`: register routes
* `stop(handler)`: unregister routes

If you want the easy path, inherit from `MugPluginSimple` and return a list of routes.

### The one symbol Mug looks for
Your shared library must export this exact C symbol: **`mugCreateInstance`**

Signature:

* `extern "C" ThorsAnvil::ThorsMug::MugPlugin* mugCreateInstance(int init, char const* config);`

Mug passes a stringified version of the plugin’s JSON `config` value (so it can be an object/string/number/etc.) and expects a `MugPlugin*` back.

## Minimal example plugin
This plugin registers one endpoint:

* `GET /hello/{name}` → returns `Hello, <name>`

#### HelloBot.cpp
```cpp
#include "ThorsMug/MugPlugin.h"
#include "NisseHTTP/Request.h"
#include "NisseHTTP/Response.h"

#include <memory>
#include <string>

namespace NisHttp = ThorsAnvil::Nisse::HTTP;

class HelloPlugin : public ThorsAnvil::ThorsMug::MugPluginSimple
{
    public:
        std::vector<ThorsAnvil::ThorsMug::Action> getAction() override
        {
            return {
                {
                    NisHttp::Method::GET,
                    // Note: '{name}' is used to indicate that Nisse will capture this part of the path
                    //                as a variable called "name" that you can then access in the code.
                    "/hello/{name}",
                    [](NisHttp::Request const& req, NisHttp::Response& res) {
                        std::string name = req.variables()["name"];
                        std::string body = "Hello, " + name + "\n";
                        res.body(body.size()) << body;
                        return true;
                    }
                }
            };
        }
};

// The library retains ownership of the "Plugin" and is shown dynamically creating the
// plugin object. This is done as a demonstration of a very simple service and you can
// use other techniques.
// Note: A library may be configured to run multiple versions with different configs on different ports.
//       You should take this into consideration.
// Note: The configuration is meant as the way to initialize your object with context.
static std::unique_ptr<HelloPlugin> plugin;

extern "C" ThorsAnvil::ThorsMug::MugPlugin* mugCreateInstance(int init, char const* /*config*/)
{
    if (init != 0)
    {
        if (plugin) {
          throw std::runtime_error("This plugin only supports one running instance");
        }
        plugin = std::make_unique<HelloPlugin>();
        return plugin.get();
    }
    else {
        plugin.reset();
        return nullptr;
  }
}
```

#### Makefile:
The most trivial Makefile I can build. Please use your own build tools once you have it working.
````Makefile
SRC				= $(wildcard *.cpp)
OBJ				= $(patsubst %.cpp,%.o,$(SRC))
CXXFLAGS		= -std=c++20
LIBS			= -lNisseBolt -lNisse -lThorsSocket -lThorSerialize -lThorsLogging

all:	$(OBJ)
	$(CXX) -dynamiclib -o libHelloBot.dylib $(OBJ) $(LIBS)
````

## The development loop (the whole point)
Start Mug once:

```bash
mug --config=./mug.cfg
```

Test the endpoint:

```bash
curl "http://localhost:8080/hello/Loki"
```

Now change the plugin (edit `HelloBot.cpp`) and rebuild the shared library.

If `libraryCheckTime` is non-zero, Mug will detect the updated timestamp and hot-reload:

* calls `stop()` on the old plugin
* unloads the library
* loads the new library
* calls `mugCreateInstance()` again
* calls `start()` again

So your browser refresh (or `curl`) hits the new code without restarting the server.

## Control port (stop/ping)
Mug also listens on a control port (default `8079`, configurable).

It uses a simple HTTP query parameter: `command=...`

```bash
# Graceful shutdown (finish in-flight work)
curl "http://localhost:8079/?command=stopsoft"

# Immediate shutdown
curl "http://localhost:8079/?command=stophard"

# Health check (no-op, should return 200)
curl "http://localhost:8079/?command=ping"
```

## Practical gotchas (things you should get right early)
### 1) Unregister everything in `stop()`
If you don’t remove your routes, you’re leaving the server with function pointers into a library you are about to unload. That’s instant undefined behavior.

### 2) Treat reload like a process restart
Reload is not “live patching”.

It’s closer to:

* destroy old plugin wiring
* load new plugin wiring

If you need persistent state, put it somewhere intentional (database / external cache / an out-of-process service), not in plugin globals that only *seem* stable.

### 3) Keep `getAction()` stable
If you use `MugPluginSimple`, note that it calls `getAction()` in both `start()` and `stop()`. So it must return the same route set (same paths/methods) both times.

## Summary
Mug is a small executable that lets you:

* write REST endpoints as C++ plugins
* load them dynamically at runtime
* hot-reload them when the shared library is rebuilt

It’s not trying to be a huge framework. It’s trying to give you the one thing C++ web dev is missing for iteration speed: a fast edit/build/test loop.

## Next step
In the next article I’ll walk through a “real” plugin:

* parsing a plugin-specific config block
* wiring multiple routes with validation hooks
* serving static files asynchronously (no blocking threads)
* demonstrating hot reload by swapping behavior in-place while keeping the Mug process running

