---
title: 'SlackActions: Slack Bot handling Slash commands'
date: 2026-05-06T00:00:00-0800
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
description: Handling Slack slash commands
draft: false
cover:
  image: /images/post/post-5.png
  hidden: false
  caption: Photo by Oscar Nilsson
---

## Overview

Slash commands let users trigger your Bot by typing a command like `/todo` or `/deploy` directly into the Slack message box. Unlike message handlers that passively listen for text, slash commands are explicit invocations — they feel like calling a function. The user types the command, optionally followed by arguments, and your Bot receives the request and responds.

If you are familiar with the [Slack Bolt](https://docs.slack.dev/tools/bolt-python/) Python library, the interface should feel familiar. In Bolt Python you would write:

````python
@app.command("/todo")
def handle_todo(ack, command):
    ack(f"Got it! You said: {command['text']}")
````

The NisseBolt equivalent is:

````cpp
command("/todo", [](Ack const& ack, Response const&, ThorsAnvil::Slack::SlashCommand const& command)
{
    ack(std::string("Got it! You said: ") + command.text);
});
````

The key difference is that in NisseBolt the `ack()` call acknowledges receipt of the command (Slack requires a response within 3 seconds), and you use the `Say` object or `response_url` to send a visible reply. But the shape of the API — registering a command name and a handler that receives the command payload — is the same.

## Prerequisites

This article assumes you have completed the steps in the [previous articles](/blog/slackmug) and have:

- A working Mug server with your Bot plugin loaded.
- The `botToken` and `signingSecret` in your `config.plugin` file.
- Your Bot added to a channel.

## What Are Slash Commands?

Slash commands are custom commands that users type in the Slack message input box. They always start with `/` and can include additional text as arguments. For example:

- `/todo buy milk` — create a todo item
- `/deploy staging` — trigger a deployment
- `/status` — check the status of something

When a user types a slash command, Slack sends an HTTP POST to your Bot's configured URL. Your Bot processes the command and responds. The command and its response are only visible to the user who typed it (ephemeral) unless your Bot explicitly posts a message to the channel.

Slash commands are useful when you want:

- An explicit action rather than passive listening.
- Private interaction — the command is not visible to others in the channel.
- Structured input — the command name makes the intent clear.
- Discoverability — users can see available commands by typing `/` in Slack.

## Step 1: Configure the Slash Command in Slack

Go to [api.slack.com/apps](https://api.slack.com/apps) and select your app.

1. Click "Slash Commands" in the left sidebar.

2. Click the "Create New Command" button.

3. Fill in the form:
   - **Command**: The slash command users will type (e.g., `/todo`).
   - **Request URL**: Your Bot's slash command endpoint. Check your `config.plugin` for the "slot" value. The URL format is: `<Domain-Name>/<slot>/slash/<command-without-slash>`.
     For example, if your domain is `thors-anvil.com`, your slot is `/slack/Bot`, and your command is `/todo`, the Request URL is: `https://thors-anvil.com/slack/Bot/slash/todo`.
   - **Short Description**: A brief description shown to users (e.g., "Manage your todo list").
   - **Usage Hint**: Optional hint shown to users about what arguments to provide (e.g., "[task description]").

4. Click "Save".

5. You may be prompted to reinstall your app to the workspace. If so, go to "OAuth & Permissions" and click "Reinstall to <Workspace> then click "Allow".

## Step 2: Handle the Slash Command in Code

Here is a Bot that handles the `/todo` command:

````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"

namespace Bolt  = ThorsAnvil::Nisse::Bolt;
namespace Slack = ThorsAnvil::Slack;

class Bot: public Bolt::App
{
    public:
        Bot(Bolt::AppConfig const& config)
            : Bolt::App(config)
        {
            command("/todo", [](Bolt::Ack const& ack, Bolt::Response const&, Slack::SlashCommand const& cmd)
            {
                ack();
            });
        }
};

THORS_ANVIL_NISSE_BOLT_SERVER_INIT(Bolt::AppConfig, Bot);
````

The `command()` method takes two arguments: the command name (with or without the leading `/`) and a handler. When Slack sends the command to your Bot, the handler is called with three parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `ack` | `Bolt::Ack` | Acknowledges receipt of the command. Must be called within 3 seconds. |
| `response` | `Bolt::Response` | Reserved for future use. |
| `cmd` | `Slack::SlashCommand` | The command payload from Slack. |

### Acknowledging the Command

Slack requires your Bot to respond within 3 seconds. The `ack()` call sends back a 200 OK to Slack, confirming that your Bot received the command. If you don't call `ack()` within the time limit, Slack will show the user an error: "dispatch_failed".

````cpp
ack();          // 200 OK with empty body
ack(200);       // Explicit status code
````

## The `SlashCommand` Object

The handler receives a `const` reference to the [SlashCommand](https://github.com/Loki-Astari/NisseBolt/blob/master/src/ThorsSlack/SlashCommand.h) struct. Its key fields are:

| Field | Type | Description |
|-------|------|-------------|
| `command` | `std::string` | The command that was typed (e.g., `/todo`) |
| `text` | `std::string` | Everything the user typed after the command |
| `user_id` | `std::string` | The Slack user ID of the person who typed the command |
| `user_name` | `std::string` | (Deprecated) Plain text name of the user |
| `channel_id` | `std::string` | The channel where the command was typed |
| `team_id` | `std::string` | The workspace ID |
| `trigger_id` | `std::string` | A short-lived ID for opening modals |
| `response_url` | `std::string` | A temporary webhook URL for sending follow-up messages |

The `text` field is the most commonly used — it contains the arguments the user passed to the command. For example, if the user types `/todo buy milk`, then `cmd.text` is `"buy milk"`.

## The `command()` Method

````cpp
namespace ThorsAnvil::Nisse::Bolt
{

using SlashCommandHandler = std::function<void(Ack const&, Response const&, ThorsAnvil::Slack::SlashCommand const&)>;

class App
{
    public:
        void command(std::string const& command, SlashCommandHandler&& handler);
};

}
````

The command name can be passed with or without the leading `/` — NisseBolt normalizes it:

````cpp
command("/todo", handler);   // These are
command("todo", handler);    // equivalent
````

Each command name can only have one handler. If you register the same command twice, the second handler replaces the first.

## A More Complete Example

Here is a Bot that handles a `/todo` command with simple argument parsing:

````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"

namespace Bolt  = ThorsAnvil::Nisse::Bolt;
namespace Slack = ThorsAnvil::Slack;

class Bot: public Bolt::App
{
    public:
        Bot(Bolt::AppConfig const& config)
            : Bolt::App(config)
        {
            command("/todo", [](Bolt::Ack const& ack, Bolt::Response const&, Slack::SlashCommand const& cmd)
            {
                ack();

                if (cmd.text.empty()) {
                    // User typed just "/todo" with no arguments.
                    // You could list their todos here.
                }
                else {
                    // User typed "/todo <something>".
                    // cmd.text contains the arguments.
                    std::string task = cmd.text;
                    // Process the task...
                }
            });

            command("/status", [](Bolt::Ack const& ack, Bolt::Response const&, Slack::SlashCommand const& cmd)
            {
                ack();
                // Return system status...
            });
        }
};

THORS_ANVIL_NISSE_BOLT_SERVER_INIT(Bolt::AppConfig, Bot);
````

### Combining with Message Handlers

Slash commands and message handlers work independently. You can register both in the same Bot:

````cpp
Bot(Bolt::AppConfig const& config)
    : Bolt::App(config)
{
    // Respond to messages containing "hello".
    message("hello", [](ThorsAnvil::Slack::Event::Message const& msg, Bolt::Say const& say)
    {
        say("Hello!");
    });

    // Handle the /deploy slash command.
    command("/deploy", [](Bolt::Ack const& ack, Bolt::Response const&, Slack::SlashCommand const& cmd)
    {
        ack();
        // Trigger deployment for cmd.text (e.g., "staging" or "production")
    });
}
````

## URL Routing

Looking at the `config.plugin` file, you will notice that slash commands use a different URL path than events:

- Events: `<slot>/event`
- Slash commands: `<slot>/slash/<command-name-without-slash>`

For example, with slot `/slack/Bot` and command `/todo`:

- Event URL: `https://thors-anvil.com/slack/Bot/event`
- Slash command URL: `https://thors-anvil.com/slack/Bot/slash/todo`

Each slash command you create in the Slack configuration needs its own Request URL pointing to the correct path. If you have commands `/todo` and `/deploy`, you need two URLs:

- `https://thors-anvil.com/slack/Bot/slash/todo`
- `https://thors-anvil.com/slack/Bot/slash/deploy`

## Tips

### Ephemeral vs. Channel Messages

Slash commands are private by default — only the user who typed the command sees the interaction. If you want the Bot to post a visible message to the channel after processing a command, you can use a `Say` object (if you store a reference to the client) or use the `response_url` from the command payload to send a follow-up message.

### Argument Parsing

The `text` field is a single string. If your command accepts multiple arguments, you will need to parse them yourself. A common pattern is to use the first word as a sub-command:

````cpp
command("/todo", [](Bolt::Ack const& ack, Bolt::Response const&, Slack::SlashCommand const& cmd)
{
    ack();

    std::string text = cmd.text;
    auto spacePos = text.find(' ');
    std::string subCommand = (spacePos != std::string::npos) ? text.substr(0, spacePos) : text;
    std::string args = (spacePos != std::string::npos) ? text.substr(spacePos + 1) : "";

    if (subCommand == "add") {
        // Add a todo item: /todo add buy milk
    }
    else if (subCommand == "list") {
        // List all todos: /todo list
    }
    else if (subCommand == "done") {
        // Mark a todo as done: /todo done 3
    }
});
````

### Required Scopes

Slash commands themselves don't require additional OAuth scopes beyond what you already have. However, if your command handler needs to post messages to channels, you will need the `chat:write` scope (which you should already have from the [handlers article](/blog/slackhandlers)).

### Testing with ngrok

If you are using ngrok for development, remember that each time you restart ngrok, you get a new URL. You will need to update the Request URL for each slash command in the Slack configuration.

## What's Next

This article covered slash commands — explicit, user-triggered interactions with your Bot. In the next article, I will cover interactive components such as buttons, menus, and modal dialogs that let users interact with rich UI elements posted by your Bot.

