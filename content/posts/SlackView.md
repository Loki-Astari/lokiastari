---
title: 'SlackView: Slack Bot opening Modals'
date: 2026-05-16T00:00:00-0800
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
description: Opening modal dialogs from a Slack Bot
draft: false
cover:
  image: /images/post/post-6.png
  hidden: false
  caption: Photo by Ben Griffiths
---

## Overview

Modals are popup dialogs that collect structured input from users. They appear as an overlay on top of Slack's interface and can contain text inputs, date pickers, checkboxes, radio buttons, and more. Unlike messages that flow passively through a channel, modals are interactive forms that your Bot presents to a user and then processes when submitted.

If you are familiar with [Slack Bolt](https://docs.slack.dev/tools/bolt-python/), you have probably opened modals using `client.views_open()`. In Bolt Python you would write:

````python
@app.command("/feedback")
def open_feedback(ack, body, client):
    ack()
    client.views_open(
        trigger_id=body["trigger_id"],
        view={
            "type": "modal",
            "callback_id": "feedback_modal",
            "title": {"type": "plain_text", "text": "Feedback"},
            "submit": {"type": "plain_text", "text": "Send"},
            "blocks": [
                {
                    "type": "input",
                    "block_id": "message_block",
                    "element": {"type": "plain_text_input", "action_id": "message_input"},
                    "label": {"type": "plain_text", "text": "Your feedback"}
                }
            ]
        }
    )
````

The NisseBolt equivalent is:

````cpp
class FeedbackView: public Bolt::View
{
    public:
        FeedbackView()
            : Bolt::View
              {
                Slack::API::Views::View
                {
                    .title  = PlainText{.text = "Feedback"},
                    .blocks =
                    {
                        Input
                        {
                            .label   = PlainText{.text = "Your feedback"},
                            .element = PlainTextInput{.action_id = "message_input"},
                            .block_id = "message_block",
                        },
                    },
                    .submit      = PlainText{.text = "Send"},
                    .callback_id = "feedback_modal",
                },
                [](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::Views::ViewSubmission const& view)
                {
                    ack();
                }
              }
        {}
};
````

The key difference is that in NisseBolt you define a `View` object that bundles the visual layout with its handlers. You then open the modal by calling `viewOpen()` with a `trigger_id` and the view object. NisseBolt automatically registers the submit and close handlers for you — there is no need to separately listen for `view_submission` callbacks.

## Prerequisites

This article assumes you have completed the steps in the [previous articles](/blog/slackmug) and have:

- A working Mug server with your Bot plugin loaded.
- The `botToken` and `signingSecret` in your `config.plugin` file.
- Your Bot added to a channel.
- At least one slash command configured (from the [SlackActions](/blog/slackaction) article).

## What Are Modals?

Modals are focused, structured dialogs that appear as overlays in Slack. They are useful when you need to:

- Collect multiple pieces of information at once (a form).
- Present a multi-step workflow where each step builds on the previous.
- Show information that requires user confirmation before proceeding.
- Gather input that is too complex for a single slash command argument.

A modal has a title bar, a content area with blocks (inputs, text, dividers), and optional submit/close buttons. When the user clicks submit, Slack sends a `view_submission` event to your Bot with all the form values.

Modals can be stacked — you can push a new modal on top of an existing one (up to 3 levels deep), creating a navigation flow within the dialog.

## Step 1: Enable Interactivity in Slack

Go to [api.slack.com/apps](https://api.slack.com/apps) and select your app.

1. Click "Interactivity & Shortcuts" in the left sidebar.

2. Toggle "Interactivity" to **On** (if not already enabled from the previous article).

3. Set the **Request URL** to your Bot's interactivity endpoint. Check your `config.plugin` for the "slot" value. The URL format is: `<Domain-Name>/<slot>/useraction`.
   For example: `https://thors-anvil.com/slack/Bot/useraction`.

4. Click "Save Changes".

This is the same endpoint used for button clicks and other interactive components. If you already configured it for the [SlackActions](/blog/slackaction) article, you do not need to change anything.

## Step 2: Define a View

In NisseBolt, you create a `View` object that combines the visual definition (what the user sees) with the handlers (what happens on submit/close). The `View` class constructor takes:

1. A `Slack::API::Views::View` struct describing the modal layout.
2. A submit handler called when the user clicks the submit button.
3. An optional close handler called when the user dismisses the modal.

Here is a minimal view that collects a single text input:

````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"
#include "ThorsSlack/APIViews.h"

namespace Bolt  = ThorsAnvil::Nisse::Bolt;
namespace Slack = ThorsAnvil::Slack;

using namespace ThorsAnvil::Slack::BlockKit;

class FeedbackView: public Bolt::View
{
    public:
        FeedbackView()
            : Bolt::View
              {
                Slack::API::Views::View
                {
                    .title  = PlainText{.text = "Send Feedback"},
                    .blocks =
                    {
                        Input
                        {
                            .label   = PlainText{.text = "Message"},
                            .element = PlainTextInput
                            {
                                .action_id   = "feedback_input",
                                .placeholder = PlainText{.text = "What's on your mind?"},
                                .multiline   = true,
                            },
                            .block_id = "feedback_block",
                        },
                    },
                    .submit      = PlainText{.text = "Send"},
                    .callback_id = "feedback_modal",
                },
                [](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::Views::ViewSubmission const& view)
                {
                    ack();

                    using Slack::API::SlackState;
                    SlackState const& values = view.view.state;
                    std::string feedback = values.getStringValue<PlainTextInput>("feedback_block", "feedback_input");

                    // Process the feedback...
                    std::cerr << "Received feedback: " << feedback << "\n";
                }
              }
        {}
};
````

## Step 3: Open the View

To display the modal you call `viewOpen()` from within a handler that provides a `trigger_id`. Slash commands and interactive actions both provide trigger IDs. You cannot open a modal from a plain message handler — Slack requires a user interaction to trigger the modal.

````cpp
class Bot: public Bolt::App
{
    FeedbackView    feedbackView;
    public:
        Bot(Bolt::AppConfig const& config)
            : Bolt::App(config)
            , feedbackView()
        {
            command("/feedback", [&](Bolt::SlashAck const& ack, Bolt::Response const&, Slack::SlashCommand const& cmd)
            {
                ack();
                viewOpen(cmd.trigger_id, feedbackView);
            });
        }
};
````

When the user types `/feedback`, the Bot acknowledges the command and opens the modal. The user fills in the form and clicks "Send", which triggers the submit handler you defined in `FeedbackView`.

## The `View` Class

````cpp
namespace ThorsAnvil::Nisse::Bolt
{

class View
{
    public:
        View(ThorsAnvil::Slack::API::Views::View display, ViewSubmitHandler&& submitHandler);
        View(ThorsAnvil::Slack::API::Views::View display, ViewSubmitHandler&& submitHandler, ViewClosedHandler&& closeHandler);

        void action(std::string const& actionId, ActionHandler&& handler);

        ThorsAnvil::Slack::API::Views::View const& getDisplayView() const;
};

}
````

| Constructor | Description |
|-------------|-------------|
| `View(display, submitHandler)` | Modal with submit handling only. |
| `View(display, submitHandler, closeHandler)` | Modal with both submit and close handling. When you provide a close handler, `notify_on_close` is automatically set to `true`. |

| Method | Description |
|--------|-------------|
| `action(actionId, handler)` | Register a handler for interactive elements within the modal (buttons, selects, etc.) that fire `block_actions` events while the modal is open. |
| `getDisplayView()` | Returns the visual definition. Useful when you need to modify and update a modal dynamically. |

## The View Methods on `App`

````cpp
namespace ThorsAnvil::Nisse::Bolt
{

class App
{
    public:
        void viewOpen(std::string const& triggerId, View const& view);
        void viewPush(std::string const& triggerId, View const& view);
        void viewUpdate(std::string const& viewId, ThorsAnvil::Slack::API::Views::View view);
};

}
````

| Method | Description |
|--------|-------------|
| `viewOpen(triggerId, view)` | Opens the modal as a new top-level dialog. Requires a valid `trigger_id` from a slash command or interactive action. |
| `viewPush(triggerId, view)` | Pushes a new modal onto the existing modal stack. The user sees a "back" button to return to the previous modal. Maximum 3 levels deep. |
| `viewUpdate(viewId, display)` | Updates an already-open modal with new content. The `viewId` is the ID of the view to update (available from `view.view.id` or `view.view.previous_view_id` in submission payloads). |

## The `Views::View` Struct

This struct defines what the user sees:

| Field | Type | Description |
|-------|------|-------------|
| `title` | `PlainText` | The title in the modal's top bar. Max 24 characters. |
| `blocks` | `Blocks` | The content blocks (inputs, text, dividers, etc.). Max 100 blocks. |
| `close` | `OptPlainText` | Text for the close button. Max 24 characters. Defaults to "Cancel". |
| `submit` | `OptPlainText` | Text for the submit button. Max 24 characters. Required if the modal contains `Input` blocks. |
| `callback_id` | `OptString` | An identifier for recognizing this modal in events. Max 255 characters. |
| `private_metadata` | `OptString` | A string passed through to submission events. Max 3000 characters. Use this to carry state between views. |
| `clear_on_close` | `OptBool` | If `true`, clicking close clears all views in the stack. Defaults to `false`. |
| `notify_on_close` | `OptBool` | If `true`, Slack sends a `view_closed` event when the user closes the modal. Set automatically when you provide a close handler. |
| `external_id` | `OptString` | A custom ID unique per team. Useful for updating a view without knowing its Slack-assigned ID. |
| `submit_disabled` | `OptBool` | If `true`, disables submit until the user interacts with an input. |

## The `ViewSubmission` Object

When the user submits a modal, your handler receives a `ViewSubmission` with these fields:

| Field | Type | Description |
|-------|------|-------------|
| `team` | `SlackTeam` | The workspace. |
| `user` | `SlackUser` | The user who submitted. |
| `trigger_id` | `std::string` | A new trigger ID — you can use this to open or push another modal. |
| `view` | `ViewsInfo` | The full view state including all input values. |

To extract input values from the submission, use the `state` field on `view.view`:

````cpp
SlackState const& values = view.view.state;
std::string text = values.getStringValue<PlainTextInput>("block_id", "action_id");
````

The template parameter must match the input element type you used in the block definition (`PlainTextInput`, `DatePicker`, `RadioButtons`, `Checkboxes`, etc.).

## Handling Actions Within a Modal

Interactive elements within a modal (buttons, selects, date pickers with `dispatch_action = true`) fire `block_actions` events while the modal is still open. You register these handlers on the `View` object using the `action()` method:

````cpp
class MyView: public Bolt::View
{
    public:
        MyView()
            : Bolt::View{/* ... display and submit handler ... */}
        {
            action("priority_input", [](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::BlockActions const& block, std::string const& value)
            {
                ack();
                std::cerr << "Priority changed to: " << value << "\n";
            });
        }
};
````

For an input element to dispatch actions while the modal is open, set `dispatch_action = true` on its `Input` block.

## Pushing and Updating Views

### Pushing a View

Use `viewPush()` to stack a new modal on top of the current one. This is useful for multi-step workflows or "drill-down" interactions. The pushed view gets a "back" button (using the `close` text you set) that returns to the previous modal.

You call `viewPush()` from within an action handler that has a `trigger_id`:

````cpp
action("details_button", [&](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::BlockActions const& block, std::string const& value)
{
    ack();
    app.viewPush(block.trigger_id, detailsView);
});
````

### Updating a View

Use `viewUpdate()` to replace the content of an already-open modal. You need the view's ID, which is available from the submission payload:

````cpp
// In a submit handler for a child view, update the parent:
std::string const& parentViewId = *view.view.previous_view_id;
Slack::API::Views::View updatedDisplay = parentView.getDisplayView();
updatedDisplay.blocks.emplace_back(Section{.text = MrkDwn{.text = "New item added."}});
app.viewUpdate(parentViewId, std::move(updatedDisplay));
````

## A More Complete Example

Here is a Bot that opens a "Create Task" modal from a slash command. The modal includes text input, a date picker, priority radio buttons, and a button that pushes a second modal to collect contact information. When the contact form is submitted, it updates the parent modal with the new contact details.

````cpp
#include "NisseBolt/App.h"
#include "NisseBolt/AppConfig.h"
#include "ThorsSlack/APIViews.h"

namespace Bolt  = ThorsAnvil::Nisse::Bolt;
namespace Slack = ThorsAnvil::Slack;

using namespace ThorsAnvil::Slack::BlockKit;

class ContactView: public Bolt::View
{
    Bolt::App&   app;
    Bolt::View&  parentView;
    public:
        ContactView(Bolt::App& app, Bolt::View& parentView)
            : Bolt::View
              {
                Slack::API::Views::View
                {
                    .title  = PlainText{.text = "Add Contact"},
                    .blocks =
                    {
                        Input
                        {
                            .label   = PlainText{.text = "Name"},
                            .element = PlainTextInput{.action_id = "name_input", .placeholder = PlainText{.text = "Full name"}},
                            .block_id = "name_block",
                        },
                        Input
                        {
                            .label   = PlainText{.text = "Email"},
                            .element = PlainTextInput{.action_id = "email_input", .placeholder = PlainText{.text = "Email address"}},
                            .block_id = "email_block",
                        },
                    },
                    .close       = PlainText{.text = "Back"},
                    .submit      = PlainText{.text = "Save"},
                    .callback_id = "contact_modal",
                },
                [&app, &parentView](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::Views::ViewSubmission const& view)
                {
                    ack();
                    using Slack::API::SlackState;
                    SlackState const& values = view.view.state;
                    std::string name  = values.getStringValue<PlainTextInput>("name_block", "name_input");
                    std::string email = values.getStringValue<PlainTextInput>("email_block", "email_input");

                    // Update the parent view to show the new contact.
                    std::string const& parentViewId = *view.view.previous_view_id;
                    Slack::API::Views::View display = parentView.getDisplayView();
                    std::string info = ":bust_in_silhouette: " + name + " (" + email + ")";
                    display.blocks.emplace_back(Context{.elements = {MrkDwn{.text = info}}});
                    app.viewUpdate(parentViewId, std::move(display));
                }
              }
            , app(app)
            , parentView(parentView)
        {}
};

class TaskView: public Bolt::View
{
    Bolt::App&   app;
    ContactView  contactView;
    public:
        TaskView(Bolt::App& app)
            : Bolt::View
              {
                Slack::API::Views::View
                {
                    .title  = PlainText{.text = "Create Task"},
                    .blocks =
                    {
                        Input
                        {
                            .label   = PlainText{.text = "Description"},
                            .element = PlainTextInput
                            {
                                .action_id   = "desc_input",
                                .placeholder = PlainText{.text = "What needs to be done?"},
                            },
                            .block_id = "desc_block",
                        },
                        Input
                        {
                            .label   = PlainText{.text = "Due Date"},
                            .element = DatePicker
                            {
                                .action_id   = "due_input",
                                .placeholder = PlainText{.text = "Pick a date"},
                            },
                            .block_id = "due_block",
                            .optional = true,
                        },
                        Input
                        {
                            .label   = PlainText{.text = "Priority"},
                            .element = RadioButtons
                            {
                                .action_id = "priority_input",
                                .options =
                                {
                                    {.text = PlainText{.text = "High"},   .value = "high"},
                                    {.text = PlainText{.text = "Medium"}, .value = "medium"},
                                    {.text = PlainText{.text = "Low"},    .value = "low"},
                                },
                                .initial_option = ElOption
                                {
                                    .text  = PlainText{.text = "Medium"},
                                    .value = "medium",
                                },
                            },
                            .block_id = "priority_block",
                        },
                        Actions
                        {
                            .elements =
                            {
                                Button
                                {
                                    .text      = PlainText{.text = "Add Contact"},
                                    .action_id = "add_contact_btn",
                                },
                            },
                            .block_id = "actions_block",
                        },
                    },
                    .close       = PlainText{.text = "Cancel"},
                    .submit      = PlainText{.text = "Create"},
                    .callback_id = "task_modal",
                },
                [](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::Views::ViewSubmission const& view)
                {
                    ack();
                    using Slack::API::SlackState;
                    SlackState const& values = view.view.state;
                    std::string desc     = values.getStringValue<PlainTextInput>("desc_block", "desc_input");
                    std::string due      = values.getStringValue<DatePicker>("due_block", "due_input");
                    std::string priority = values.getStringValue<RadioButtons>("priority_block", "priority_input");

                    std::cerr << "Task created: " << desc
                              << " due=" << due
                              << " priority=" << priority << "\n";
                },
                [](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::Views::ViewClosed const&)
                {
                    ack();
                    std::cerr << "Task creation cancelled.\n";
                }
              }
            , app(app)
            , contactView(app, *this)
        {
            action("add_contact_btn", [&](Bolt::Ack const& ack, Bolt::Response const&, Slack::API::BlockActions const& block, std::string const& /*value*/)
            {
                ack();
                app.viewPush(block.trigger_id, contactView);
            });
        }
};

class Bot: public Bolt::App
{
    TaskView    taskView;
    public:
        Bot(Bolt::AppConfig const& config)
            : Bolt::App(config)
            , taskView(*this)
        {
            command("/task", [&](Bolt::SlashAck const& ack, Bolt::Response const&, Slack::SlashCommand const& cmd)
            {
                ack();
                viewOpen(cmd.trigger_id, taskView);
            });
        }
};
````

## Tips

### Trigger IDs Expire Quickly

A `trigger_id` is only valid for about 3 seconds after the interaction that produced it. You must call `viewOpen()` or `viewPush()` within that window. If you have slow processing to do, call `ack()` first, then open the view immediately — do your heavy work in the submit handler.

### The 3-View Stack Limit

Slack allows a maximum of 3 modals in the push stack. If you try to push a fourth, the API call will fail. Design your workflows to stay within this limit.

### Updating vs. Pushing

Use `viewPush()` when you want the user to be able to go "back" to the previous modal. Use `viewUpdate()` when you want to replace the current modal's content in-place (for example, showing a confirmation screen after submission, or adding dynamic content based on a button click in a child view).

### Input Validation

If you need to reject a submission with validation errors, you can respond to the `view_submission` event with an error payload instead of calling `ack()`. However, NisseBolt currently handles this by calling `ack()` — if you need client-side validation, use the `min_length` and `max_length` fields on `PlainTextInput`, or mark fields as required by leaving `optional` unset (it defaults to `false`).

### Private Metadata for State

Use the `private_metadata` field to pass state between views. For example, if you open a modal from a slash command and need to know which channel it came from when the form is submitted:

````cpp
Slack::API::Views::View display = myView.getDisplayView();
display.private_metadata = cmd.channel_id;
// Then open with this modified display, or store channel_id in your Bot's state.
````

### Common Error: "invalid_trigger_id"

If you see this error, it usually means too much time passed between receiving the interaction and calling `viewOpen()`. Make sure you call `ack()` and `viewOpen()` immediately in your handler — do not do expensive work before opening the modal.

## What's Next

This article covered modals — structured dialogs that collect input from users. In the next article, I will cover the Home Tab, a persistent surface where your Bot can display personalized content to each user who visits your app's home page in Slack.
