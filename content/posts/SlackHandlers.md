---
title: 'SlackHandlers: Slack Bot handling events'
date: 2026-05-02T00:00:00-0800
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
description: Handling events in the Slack Bot
draft: false
cover:
  image: /images/post/post-4.png
  hidden: false
  caption: Photo by Roman Synkevych
---

## Overview

In the previous article I walked through creating a Slack Bot and registering it with Slack. That article simply verified that events coming from Slack were correctly received by the Bot. This article will expand on that and show how to respond to these events using the `message()` and `event()` methods provided by the NisseBolt library.

If you are familiar with the [Slack Bolt](https://docs.slack.dev/tools/bolt-python/) Python library, the interface should feel familiar. In Bolt Python you would write:

````python
@app.message("knock knock")
def handle_knock(message, say):
    say("_who's there?_")
````

The NisseBolt equivalent is:

````cpp
message("knock knock", [](Event::Message const& message, Say const& say)
{
    say("_who's there?_");
});
````

The key differences are C++ idioms: lambdas instead of decorators, explicit types instead of duck typing, and handler registration in the constructor instead of top-level decorators. But the shape of the API, registering a filter and a handler that receives the event and a `say` callback, is the same.

## Prerequisites

This article assumes you have completed the steps in the [previous article](/blog/slackmug) and have:

- A working Mug server with your Bot plugin loaded.
- Your Bot subscribed to the `message.channels` event in the Slack configuration.
- The `botToken` and `signingSecret` in your `config.plugin` file.

### Add Your Bot to a Channel

Before your Bot can see or respond to messages in a channel, it must be explicitly added to that channel.

In the Slack application:

1. Open the channel you want the Bot to monitor.

2. Click on the channel name to open the "Channel Details" dialog.

3. Click the "Integrations" tab.

4. In the "Apps" section, click "Add Apps".

5. Find your app in the list and click "Add".

Your app will now receive events for messages posted in this channel.

## Your First Handler

In the previous article you created the most trivial Bot that did nothing. Here you add a message handler that responds to "knock knock" with "who's there?".

````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"

class Bot: public ThorsAnvil::Nisse::Bolt::App
{
    public:
        Bot(ThorsAnvil::Nisse::Bolt::AppConfig const& config)
            : ThorsAnvil::Nisse::Bolt::App(config)
        {
            message("knock knock", [](ThorsAnvil::Slack::Event::Message const& message, ThorsAnvil::Nisse::Bolt::Say const& say)
            {
                say("_who's there?_");
            });
        }
};

THORS_ANVIL_NISSE_BOLT_SERVER_INIT(ThorsAnvil::Nisse::Bolt::AppConfig, Bot);
````

The `message()` method takes two arguments: a filter and a handler. The filter is checked against each incoming message; if it matches, the handler is called. In this case the filter is a string: if the incoming message text contains the substring `knock knock`, the Bot replies with "who's there?" (using Slack markup for italics).

### Checking It Works

If you type "knock knock" in the channel you registered your Bot with, **nothing will happen yet**. You will see an error message from the Mug Server:

````
ThorsAnvil::Nisse::Bolt::Say::sendMessage: Failed|Data|{"ok":false,"error":"missing_scope","needed":"chat:write:bot","provided":"channels:history"}
````

The important part is `"error":"missing_scope","needed":"chat:write:bot"`. Slack is telling you that your Bot does not have permission to write to the channel. You need to add this scope.

Go to [api.slack.com/apps](https://api.slack.com/apps):

1. Select your app from the list.

2. Select "OAuth & Permissions" from the left sidebar.

3. Scroll down to the "Scopes" section. In "Bot Token Scopes", click "Add an OAuth Scope".

4. Type "chat:write" and select it.

5. Scroll up to the "OAuth Tokens" section and click "Reinstall to \<Workspace\>".

6. Click "Allow".

Now type "knock knock" in the channel. Your Bot should reply with "_who's there?_".

## The `message()` Method

The `message()` method registers a filter and a handler. When a message event arrives from Slack, each registered filter is tested in order. Every handler whose filter matches is called (not just the first match).

There are four overloads:

````cpp
namespace ThorsAnvil::Nisse::Bolt
{

using Filter         = std::function<bool(ThorsAnvil::Slack::Event::Message const&)>;
using MessageHandler = std::function<void(ThorsAnvil::Slack::Event::Message const&, Say const& say)>;

class App
{
    public:
        void message(std::string filter, MessageHandler&& handler);  // Substring match
        void message(std::regex  filter, MessageHandler&& handler);  // Regex match
        void message(Filter&&    filter, MessageHandler&& handler);  // Custom predicate
        void message(MessageHandler&& handler);                      // Match all messages
};

}
````

### Substring Filter

The simplest form. The filter string is searched for as a substring of the message text:

````cpp
message("help", [](Event::Message const& message, Say const& say)
{
    say("How can I help you?");
});
````

This matches any message containing "help": "help me", "I need help please", "helpful", etc. The match is case-sensitive.

### Regex Filter

For more precise matching, pass a `std::regex`:

````cpp
message(std::regex("\\bhelp\\b"), [](Event::Message const& message, Say const& say)
{
    say("How can I help you?");
});
````

This uses `std::regex_search`, so the pattern can match anywhere in the message text. The example above uses word boundaries (`\b`) to match "help" as a whole word, avoiding false positives like "helpful".

### Custom Filter

For full control, pass a lambda that takes the `Message` object and returns `bool`:

````cpp
message(
    [](Event::Message const& message)
    {
        return message.channel == "C0MAINCH"
            && message.text.find("deploy") != std::string::npos;
    },
    [](Event::Message const& message, Say const& say)
    {
        say("Deployment request received.");
    }
);
````

This lets you filter on any field of the `Message` object, not just the text.

### Catch-All

Omitting the filter matches every message:

````cpp
message([](Event::Message const& message, Say const& say)
{
    // Called for every message in every channel the Bot is in.
    // Useful for logging or analytics.
});
````

### Multiple Handlers

All matching handlers are called, not just the first. This means you can layer specific and general handlers:

````cpp
Bot(AppConfig const& config) : App(config)
{
    // Specific: respond to greetings.
    message(std::regex("\\b(hello|hi|hey)\\b"), [](Event::Message const& msg, Say const& say)
    {
        say("Hello!");
    });

    // General: log every message.
    message([](Event::Message const& msg, Say const& say)
    {
        std::cerr << "Message from " << msg.user << ": " << msg.text << "\n";
    });
}
````

A greeting message would trigger both handlers.

## The `Message` Object

The handler receives a `const` reference to the [Event::Message](https://github.com/Loki-Astari/NisseBolt/blob/master/src/ThorsSlack/EventCallbackMessage.h) struct. Its key fields are:

| Field | Type | Description |
|-------|------|-------------|
| `user` | `std::string` | The Slack user ID of the person who posted |
| `text` | `std::string` | The plain-text content of the message |
| `channel` | `std::optional<std::string>` | The channel ID where the message was posted |
| `ts` | `std::string` | The message timestamp (also serves as a unique ID) |
| `thread_ts` | `std::optional<std::string>` | If the message is part of a thread, the parent timestamp |
| `channel_type` | `std::optional<std::string>` | `"channel"`, `"group"`, or `"im"` |
| `team` | `std::string` | The team/workspace ID |
| `blocks` | `std::optional<BlockKit::Blocks>` | Rich-text block content (if present) |
| `reply_count` | `std::optional<int>` | Number of replies (for threaded messages) |

## The `Say` Object

The second parameter to every handler is a [Say](https://github.com/Loki-Astari/NisseBolt/blob/master/src/NisseBolt/Say.h) object that lets you send messages back to Slack. By default it targets the same channel and thread as the incoming message.

### Simple Reply

````cpp
say("_who's there?_");
````

This posts a message to the same channel, as a reply to the thread of the incoming message (via the `ts` field).

### Reply to a Different Channel

You can override the destination by passing a `Where` struct:

````cpp
say("Alert: bad word detected!", Where{.channel = "C0ADMINCHAN"});
````

### The `Where` Struct

````cpp
struct Where
{
    std::string             channel;        // Channel ID to post to
    std::optional<std::string> icon_emoji;  // Custom emoji icon for the message
    std::optional<std::string> username;    // Custom display name for the message
    std::optional<std::string> ts;          // Thread timestamp (reply in thread)
};
````

A few useful patterns:

````cpp
say("In the same thread", Where{.channel = msg.channel.value_or(""), .ts = msg.thread_ts});

say("As a new message in a specific channel", Where{.channel = "C0MYCHANNEL"});

say("With a custom icon", Where{.channel = msg.channel.value_or(""), .icon_emoji = ":robot_face:"});
````

### Block Kit Messages

For rich formatting beyond simple markup text, the `Say` object also accepts [Block Kit](https://api.slack.com/block-kit) content:

````cpp
say(Slack::BlockKit::Blocks{/* ... block content ... */});
say(Slack::BlockKit::Blocks{/* ... */}, Where{.channel = "C0ADMINCHAN"});
````

### What Is a Channel ID?

You will notice that `Where` takes a channel ID, not a channel name. Slack identifies channels by an opaque ID string (typically 11 characters, e.g. `C09RU2URYMS`).

To find a channel's ID: open the channel in Slack, click the channel name to open "Channel Details", and the Channel ID is shown at the bottom of the dialog.

Remember that your Bot can only write to channels it has been added to (see [Prerequisites](#add-your-bot-to-a-channel) above).

## A More Complete Example

Here is a Bot with several message handlers demonstrating different filter types:

````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"

namespace Bolt  = ThorsAnvil::Nisse::Bolt;
namespace Event = ThorsAnvil::Slack::Event;

class Bot: public Bolt::App
{
    public:
        Bot(Bolt::AppConfig const& config)
            : Bolt::App(config)
        {
            // Respond to "knock knock".
            message("knock knock", [](Event::Message const& msg, Bolt::Say const& say)
            {
                say("_who's there?_");
            });

            // Flag bad language and report to admin channel.
            message(std::regex("Grud|Jovus|Stomm|Drokk|Spug"),
                [](Event::Message const& msg, Bolt::Say const& say)
                {
                    if (msg.channel == "C0MAINCH") {
                        say("2000AD language detected: " + msg.text, {.channel = "C0ADMINCHAN"});
                    }
                }
            );

            // Welcome messages from new thread starters.
            message(
                [](Event::Message const& msg)
                {
                    return !msg.thread_ts.has_value() && msg.channel == "C0HELPCH";
                },
                [](Event::Message const& msg, Bolt::Say const& say)
                {
                    say("Welcome! Someone will be with you shortly.");
                }
            );
        }
};

THORS_ANVIL_NISSE_BOLT_SERVER_INIT(Bolt::AppConfig, Bot);
````

Note the use of namespace aliases (`Bolt` and `Event`) to keep the code readable without pulling everything into the global namespace.

## The `event()` Method

The `message()` method only handles one event type: `Event::Message`. Slack can send many other types of events. The `event()` method lets you handle any of them.

````cpp
namespace ThorsAnvil::Nisse::Bolt
{

template<typename T>
using EventHandler = std::function<void(T const& event, Say const& say)>;

class App
{
    public:
        template<typename T>
        void event(EventHandler<T>&& handler);
};

}
````

The event type `T` is inferred from the handler's parameter type. The compiler determines which Slack event you want to handle based on the type you use in your lambda signature.

### How It Differs from `message()`

| | `message()` | `event()` |
|---|---|---|
| **Event type** | `Event::Message` only | Any of the 70+ event types |
| **Built-in filtering** | String, regex, or custom predicate | None (filter manually in your handler) |
| **Multiple handlers** | All matching handlers are called | All handlers for the matching type are called |

If you call `event()` with `Event::Message`, it delegates to `message()` with no filter (catch-all):

````cpp
// These are equivalent:
event(std::move(handler));
message(std::move(handler));
````

### Example: Reacting to Stars

````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"

namespace Bolt  = ThorsAnvil::Nisse::Bolt;
namespace Event = ThorsAnvil::Slack::Event;

class Bot: public Bolt::App
{
    public:
        Bot(Bolt::AppConfig const& config)
            : Bolt::App(config)
        {
            message("knock knock", [](Event::Message const& msg, Bolt::Say const& say)
            {
                say("_who's there?_");
            });

            event([](Event::StarAdded const& event, Bolt::Say const& say)
            {
                if (event.item.channel == "C0TEAMCHAN") {
                    say("We have a star by: " + event.user, {.channel = "C0ADMINCHAN"});
                }
            });
        }
};

THORS_ANVIL_NISSE_BOLT_SERVER_INIT(Bolt::AppConfig, Bot);
````

The `StarAdded` event provides:

| Field | Type | Description |
|-------|------|-------------|
| `user` | `std::string` | User who added the star |
| `reaction` | `std::string` | The star emoji name |
| `item.type` | `std::string` | What was starred (`"message"`, `"file"`, etc.) |
| `item.channel` | `std::string` | Channel containing the starred item |
| `item.ts` | `std::string` | Timestamp of the starred item |
| `event_ts` | `std::string` | When the star was added |

### Subscribing to Events in Slack

To receive non-message events, you must subscribe to them in your Slack app configuration. Go to [api.slack.com/apps](https://api.slack.com/apps), select your app, and under "Event Subscriptions" add the bot events you want (e.g., `star_added`, `reaction_added`, `member_joined_channel`).

Each event type may also require specific OAuth scopes. Slack will tell you which scopes are needed when you add the subscription.

### Example: Reaction Tracking

````cpp
event([](Event::ReactionAdded const& event, Bolt::Say const& say)
{
    if (event.reaction == "bug") {
        say("Bug report flagged on message " + event.item.ts, {.channel = "C0BUGCHAN"});
    }
});
````

### Example: Welcoming New Members

````cpp
event([](Event::MemberJoinedChannel const& event, Bolt::Say const& say)
{
    say("Welcome <@" + event.user + ">! Please read the pinned messages.");
});
````

## Available Event Types

NisseBolt supports approximately 70 event types, matching the Slack Events API. They are grouped by category:

| Category | Event Types |
|----------|-------------|
| **App** | `AppDeleted`, `AppHomeOpened`, `AppInstalled`, `AppRateLimited`, `AppRequested`, `AppUninstalledTeam`, `AppUninstalled`, `AppMentioned` |
| **Channel** | `ChannelArchive`, `ChannelCreated`, `ChannelDeleted`, `ChannelHistoryChanged`, `ChannelIdChanged`, `ChannelLeft`, `ChannelPostingPermissions`, `ChannelRename`, `ChannelShared`, `ChannelUnshared` |
| **Reaction** | `ReactionAdded`, `ReactionRemoved` |
| **Pin** | `PinAdded`, `PinRemoved` |
| **Star** | `StarAdded`, `StarRemoved` |
| **Member** | `MemberJoinedChannel`, `MemberLeftChannel` |
| **User** | `UserChange`, `UserConnection`, `UserHuddleChanged` |
| **Team** | `TeamAccessGranted`, `TeamAccessRevoked`, `TeamDomainChange`, `TeamJoin`, `TeamRename` |
| **File** | `FileChange`, `FileCommentAdded`, `FileCommentDeleted`, `FileCommentEdited`, `FileCreated`, `FileDeleted`, `FilePublic`, `FileShared`, `FileUnshared` |
| **Group** | `GroupClose`, `GroupDeleted`, `GroupHistoryChanged`, `GroupLeft`, `GroupOpen`, `GroupRename` |
| **IM** | `ImClose`, `ImCreated`, `ImHistoryChanged`, `ImOpen` |
| **Message Metadata** | `MessageMetadataPosted`, `MessageMetadataUpdated`, `MessageMetadataDeleted` |
| **Subteam** | `SubteamCreated`, `SubteamMembersChanged`, `SubteamSelfAdded`, `SubteamSelfRemoved`, `SubteamUpdated` |
| **Shared Channel** | `SharedChannelInviteAccepted`, `SharedChannelInviteApproved`, `SharedChannelInviteDeclined`, `SharedChannelInviteReceived`, `SharedChannelInviteRequested` |
| **Other** | `AssistantThreadContextChanged`, `AssistantThreadStarted`, `CallRejected`, `DndUpdated`, `DndUpdatedUser`, `EmailDomainChanged`, `EmojiChanged`, `EntityDetailsRequested`, `FunctionExecuted`, `GridMigrationFinished`, `GridMigrationStarted`, `InviteRequested`, `LinkShared`, `TokensRevoked` |

All event types live in the `ThorsAnvil::Slack::Event` namespace and are defined in the header files under `ThorsSlack/EventCallback*.h`.

## Tips

### Avoid Bot Loops

If your Bot posts a message in a channel it monitors, that message will come back as a new event. If your handler responds unconditionally, you will create an infinite loop. Use a filter to check the user ID:

````cpp
message(
    [botUserId](Event::Message const& msg) { return msg.user != botUserId; },
    [](Event::Message const& msg, Say const& say)
    {
        say("Echo: " + msg.text);
    }
);
````

Alternatively, check `msg.channel_type` or use a more specific filter.

### Thread Replies

The `Say` object is automatically configured with the `ts` of the incoming message, so calling `say("text")` replies in the context of that message. If you want to start a new top-level message instead of a thread reply, pass a `Where` without a `ts`:

````cpp
say("This is a new top-level message", Where{.channel = msg.channel.value_or("")});
````

## What's Next

This article covered the `message()` and `event()` methods for handling Slack events. In the next article, I will cover additional Slack interactions such as slash commands, interactive components (buttons, menus), and modal dialogs.
