# NisseBolt App: Missing Functionality vs Slack Bolt Python

This document compares the NisseBolt `App` class (the Mug plugin interface) against the Slack Bolt Python API and identifies functionality that is missing or incomplete.

Note: some of this functionality already exists in the newer `NBServer` class (`NB/NBServer.h`), which is a standalone server rather than a Mug plugin. Where applicable, this is noted below.

---


## 3. Shortcuts (`shortcut()`)

**Bolt Python:**
```python
@app.shortcut("open_modal")
def handle_shortcut(ack, shortcut, client):
    ack()
    client.views_open(trigger_id=shortcut["trigger_id"], view={...})
```

**NisseBolt App:** Not supported.

**NBServer:** Supported via `shortcut(std::string callbackId, ShortcutHandler handler)`.

**Impact:** Medium. Shortcuts (both global and message shortcuts) allow users to trigger actions from the Slack UI without typing a command. Less commonly used than commands or actions but important for polished integrations.

---

## 5. External Select Options (`options()`)

**Bolt Python:**
```python
@app.options("external_select_action")
def handle_options(ack):
    ack(options=[{"text": {"type": "plain_text", "text": "Option 1"}, "value": "1"}])
```

**NisseBolt App:** Not supported.

**NBServer:** Supported via `options(std::string actionId, OptionsHandler handler)`.

**Impact:** Low-Medium. Only needed when using external data source select menus in Block Kit.

---

## 7. `respond()` in Handlers

**Bolt Python:**
```python
@app.command("/echo")
def handle_echo(ack, respond, command):
    ack()
    respond(f"You said: {command['text']}")  # Uses response_url
```

The `respond()` function posts an ephemeral or in-channel message using the `response_url` provided in the interaction payload. Unlike `say()`, it can send ephemeral messages (visible only to the user who triggered the action).

**NisseBolt App:** Not supported. Only `Say` is available, which always uses the `chat.postMessage` API.

**NBServer:** Supported via `Respond&` parameter in command and action handlers.

**Impact:** Medium. Without `respond()`, Bots cannot send ephemeral responses to slash commands or interactive actions. All responses via `say()` are visible to the entire channel.

---

## 8. Middleware (`use()`)

**Bolt Python:**
```python
@app.use
def log_request(body, next):
    print(f"Received: {body['type']}")
    next()
```

Middleware runs before handlers and can short-circuit the processing chain.

**NisseBolt App:** Not supported. There is no middleware pipeline.

**NBServer:** Supported via `use(Middleware mw)`.

**Impact:** Medium. Middleware is useful for cross-cutting concerns like logging, authorization checks, and rate limiting. Without it, these concerns must be duplicated in every handler.

---

## 9. Error Handling (`onError()`)

**Bolt Python:**
```python
@app.error
def handle_errors(error, body, logger):
    logger.exception(f"Error: {error}")
```

**NisseBolt App:** Errors in `Say::sendMessage` are logged via `ThorsLogError` but there is no user-configurable error handler. If a handler throws an exception, behavior is undefined at the application level.

**NBServer:** Supported via `onError(ErrorHandler handler)`.

**Impact:** Medium. Production bots need a way to handle errors gracefully (alerting, fallback responses, etc.).

---

## Summary

| Feature | Bolt Python | NisseBolt App | NBServer | Priority |
|---------|:-----------:|:-------------:|:--------:|:--------:|
| Slash commands | Yes | No | Yes | High |
| Interactive actions | Yes | No | Yes | High |
| Modal views | Yes | No | Yes | Medium-High |
| Shortcuts | Yes | No | Yes | Medium |
| External options | Yes | No | Yes | Low-Medium |
| `ack()` in handlers | Yes | Auto only | Yes | Needed with above |
| `respond()` | Yes | No | Yes | Medium |
| Middleware | Yes | No | Yes | Medium |
| Error handler | Yes | No | Yes | Medium |
| Regex match groups | Yes | No | No | Low |
| Event filters | String-based | Type-based (better) | Type-based | N/A |

The most impactful gaps are slash commands and interactive components (actions + views). These are fundamental to most Slack Bot integrations beyond simple message listeners. The NBServer already implements all of these, so the functionality exists in the codebase but hasn't been surfaced in the Mug plugin `App` class.
