---
title: 'SlackMug: A Mug plugin for a Slack bot'
date: 2026-04-21T12:00:00-0800
author: Loki Astari, (C)2026
comments: true
categories:
- C++
- Mug
- Slack
- Nisse
- C++-By-Example
- Coding
series:
- Mug
tags:
- NisseBolt
subtitle: C++ By Example
description: Creating a Slack bot using the Mug plugin architecture.
draft: false
cover:
  image: /images/post/post-3.png
  hidden: false
  caption: Photo by Ken Blode
---

## Overview

This article will walk through the steps of creating a Mug plugin that implements a Slack Bot. It utilizes NisseBolt, a C++ library designed to be similar to the [Slack Bolt](https://docs.slack.dev/tools/bolt-python/) Python libraries.

You will create three files: a `Makefile` to build a shared library, a `Bot.cpp` that defines your Slack Bot as a Mug plugin, and a `config.plugin` file that tells the Mug server how to load and configure your Bot. Once built, the Mug server handles all the networking and threading — you just write the bot logic. After that, you will register your bot with Slack so it can start receiving events.

A more detailed explanation of how all the pieces fit together is provided at the end of the article.

I should note that my main development environment is a Mac and thus these instructions are Mac-centric. They should be simple enough that any Linux user can adapt the advice without much hassle. I find that writing all possible variations would make the article more complex than it needs to be. Anything sufficiently complex I will go over in detail.

## Step 1: Install the ThorsAnvil libraries

ThorsAnvil is a set of C++ libraries that I have written (see my other articles) for creating Web Applications in C++. For this article, you will be concentrating on using the [Mug](https://github.com/Loki-Astari/Mug) Server and writing a [Mug Plugin](https://github.com/Loki-Astari/Mug) using [NisseBolt](https://github.com/Loki-Astari/NisseBolt) that provides an interface similar to [Bolt](https://docs.slack.dev/tools/bolt-python/).

````
> brew install thors-anvil
````

## Step 2: The Simplest Bot

In this step, you will create three files. You won't build them yet — there is some configuration to do first.


### Makefile
The minimal Makefile to get started. Please use your own build tools once you have it working.
````Makefile
SRC				= $(wildcard *.cpp)
OBJ				= $(patsubst %.cpp,%.o,$(SRC))
CXXFLAGS		= -std=c++20
LIBS			= -lNisseBolt -lNisse -lThorsSocket -lThorSerialize -lThorsLogging

all:	$(OBJ)
	$(CXX) -dynamiclib -o libBot.dylib $(OBJ) $(LIBS)
````

### Bot.cpp
The code for a Mug plugin that implements a Slack Bot that will respond correctly to events from Slack, thus allowing you to confirm that the application is connecting correctly.
````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"

struct BotConfig: public ThorsAnvil::Nisse::Bolt::AppConfig
{
	std::string     slot;
};

class Bot: public ThorsAnvil::Nisse::Bolt::App
{
	public:
		Bot(BotConfig const& config)
			: ThorsAnvil::Nisse::Bolt::App(config)
		{}
};

ThorsAnvil_ExpandTrait(ThorsAnvil::Nisse::Bolt::AppConfig, BotConfig, slot);

THORS_ANVIL_NISSE_BOLT_SERVER_INIT(BotConfig, Bot);
````

### config.plugin
The configuration for Mug to load and run your Bot. You will fill in some of the Slack configuration as you go through subsequent steps.
````JSON
{
	"controlPort":  8079,
	"libraryCheckTime": 500,
	"servers": [
		{
			"port":     8080,
			"actions": [
				{
					"pluginPath": "<PathToYourDir>/libBot.dylib",
					"config": {
						"slot": "/slack/Bot",
						"botToken": "",
						"userToken": "",
						"signingSecret": ""
					}
				}
			]
		}
	]
}
````

## Step 3: Configuration and HTTPS

Slack requires an HTTPS connection, which implies you know where your certificates live. These can be specified in the above config by adding a "certPath" value after "port" that defines the path to your domain's certificates:

````json
			"port":     8080,
			"certPath": "/etc/credentials/live/YourDomain.com/",
````

Mug expects two files in this folder: "fullchain.pem" and "privkey.pem". These files are used by TLS to set up an encrypted authenticated connection with Slack.

If you don't own a domain or don't have access to the domain certificates on a host, then by not providing "certPath", Mug will use a normal HTTP port. In this situation, you will need to use a tool (reverse proxy or ngrok) as an endpoint that handles the HTTPS connection and forwards traffic to your Mug server.

### Reverse Proxy

Setting up a reverse proxy is beyond the scope of this article.

### Using ngrok to forward traffic

For simple, small-scale development, `ngrok` can be used for free to forward requests to your server.

Install ngrok on your machine.

````bash
> brew install ngrok
````

Run ngrok to forward traffic to your machine:

````bash
> ngrok http 8080
````

You should see a line that looks like:

Forwarding                    https://903d-175-22-89-211.ngrok-free.app -&gt; http://localhost:8080

In the following section, you would use "https://903d-175-22-89-211.ngrok-free.app" as the "Domain-Name".

## Step 4: Configure Slack

Slack Bots receive messages from the Slack service and thus have to be registered with Slack.

1. You will need a workspace that you own or administer, so your app can be installed into it.
 * Setting up your own [workspace](https://slack.com/help/articles/206845317-Create-a-Slack-workspace#create-a-workspace).

2. You will need to create a "Slack App".

### Creating a Slack App

Go to [api.slack.com/apps](https://api.slack.com/apps).

1. In the top right-hand corner is a "Create New App" button. Click this.

2. From the dialog, select the "From scratch" option.

3. Enter an "App Name" and select a "workspace".

4. Click "Create App".

5. You should now be on the "Basic Information" page of your application. On this page, there is a field "Signing Secret". Copy this value and paste it into `config.plugin` you created above. Note: The "Signing Secret" should not be committed to version control — consider adding `config.plugin` to `.gitignore`.

6. Don't close this window — you'll come back to it later.

### Build and Run your Slack App

To build the application into a shared library that can be loaded by Mug, simply run "make".

````bash
> make
````

You should now run the Mug server and load your Slack Bot.

````bash
> mug --config=config.plugin
````

If everything is working, you have a Slack Bot running on port 8080 of your local machine.

### Event Subscription

In the Slack Configuration for your app:

1. Click on the "Event Subscriptions" section in the left toolbar.

2. Click the "Enable Events" button.

3. Check your `config.plugin` file for the "slot" information; this will be used in the next step.

4. Set the "Request URL" to be: "&lt;Domain-Name&gt;/&lt;slot&gt;/event".
   I have the domain `thors-anvil.com` and use the slot `/slack/Bot`, so my Request URL is: `https://thors-anvil.com/slack/Bot/event`.

5. After about 2 seconds, Slack should attempt a connection to the server to validate that it can correctly decode an event.

6. Check that you get the "verified" tick.

7. Click the "Subscribe to bot event" dropdown.

8. Click the "Add Bot User Event" button.

9. Type "message.channels" and select this value.

10. Click "Save Changes".

11. Click on the "OAuth & Permissions" section in the left toolbar.

12. Under "OAuth Tokens", there should be a button marked "Install to &lt;Workspace&gt; Click this button. Click the "Allow" button.

13. This should bring you back to the "OAuth & Permissions" page. You should see a "Bot User OAuth Token" Copy this value and page in into `config.plugin' in the 'botToken' field.

## How It All Fits Together

### Mug

Mug is a C++ server similar to Python's Flask in that it allows developers to focus on the business logic while the server handles all the async sockets and threading automatically.

The Mug server is configured via a config file specified as a command-line parameter. This config file lets you specify a set of dynamically loaded libraries that implement the ThorsMug interface and custom configuration that will be passed to the loaded library (see below).

One of the most useful features of Mug is hot-reloading. It monitors your libraries at runtime and, when it detects an update, safely unloads the old version and loads the new one. This allows you to quickly iterate on your plugin without restarting the server.

### ThorsMug

In the `Bot.cpp` file you will find the line:

````cpp
THORS_ANVIL_NISSE_BOLT_SERVER_INIT(<ConfigType>, <AppType>);
````

This is a macro provided by NisseBolt that implements the ThorsMug interface for you. It creates an instance of `AppType`, passing an instance of `ConfigType` as the only parameter to the constructor. The `ConfigType` object is created from the config file (mentioned above) that was passed to Mug on startup.

In this case, the `config` block from config.plugin is passed to the ThorsMug interface:

The only requirement of the config object is that it has a member "slot" that is used by the Mug interface to distinguish multiple Mug handlers.

````JSON
"config": {
    "slot": "/slack/Bot",
    "botToken": "",
    "userToken": "",
    "signingSecret": ""
}
````

This object is treated as JSON and converted into a C++ object of type `ConfigType` using the `ThorsSerializer` library. This allows you to pass any configuration required by your `AppType` through the configuration. Note: `Bolt::App` is expecting an object of type `Bolt::Config` that has the information needed to configure a standard Slack Bot, but you can include this as part of your configuration object.

### Bolt::App

The `Bolt::App` class is an implementation of `ThorsAnvil::ThorsMug::MugPlugin` that knows how to communicate with the Slack service. It provides an interface similar to the Python Bolt libraries published by Slack, allowing you to configure how the Bot reacts to events from Slack and send appropriate requests back to the Slack service.

At this point, your bot is not very interesting and simply receives events but doesn't respond to them (apart from verifying with a 200 OK that it correctly received the event) — in the following articles, I will walk you through the different parts of the NisseBolt interface and how to configure both the C++ code and the Slack permissions.
