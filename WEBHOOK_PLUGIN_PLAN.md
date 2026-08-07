# Event-Driven Webhook Plugin Plan

## Goal

Provide an optional RuneLite plugin that turns selected in-game events into
outbound notifications. The first useful case is notifying a player when
birdhouse traps are set and when they are ready to revisit. The same mechanism
should later support farming patches, timers, contracts, and user-defined
callbacks without changing the game client or requiring a mobile client.

This is a design plan only; the snippets below are illustrative rather than
production code.

## Proposed architecture

```text
RuneLite event bus / tracker state
              |
              v
      Event detector + normalizer
              |
              v
       Notification event model
              |
              v
       Delivery queue (async)
          /             \
         v               v
   ntfy.sh adapter   Discord adapter
```

### Components

1. **Plugin lifecycle and configuration**  
   A standalone plugin owns enablement, selected event types, destinations,
   filtering, and lifecycle cleanup. It should use RuneLite's `Config` system
   and injected shared `OkHttpClient`, as existing client plugins do.
2. **Event detectors**  
   Detectors subscribe to public RuneLite events and/or expose narrowly scoped
   integration points. A birdhouse detector should emit only after state
   changes are confirmed, not on every `GameTick`.
3. **Canonical notification event**  
   Convert game-specific state into a provider-neutral object containing an
   event type, account/profile identifier (only if enabled), occurrence time,
   expected completion time, and human-readable details.
4. **Delivery service**  
   Accept canonical events, render provider-specific requests, and execute
   network calls off the client/event-bus thread. One failed destination must
   not prevent another destination from receiving the event.
5. **Destination adapters**  
   Keep ntfy and Discord formatting and authentication separate. A future
   generic HTTP callback adapter can use the same canonical event and delivery
   service.

## Birdhouse event flow

The existing Time Tracking plugin already derives birdhouse state and stores a
completion timestamp in `BirdHouseTracker`. The preferred first integration is
to expose a small, read-only event or service boundary from time tracking
rather than duplicating varp and region interpretation in the notification
plugin.

The detector should distinguish at least:

- **Trap set:** transition to a seeded/built state, with `readyAt` calculated
  from the confirmed timestamp.
- **Trap ready:** transition from in-progress to completed, emitted once.
- **Reset or unknown:** clear pending state without sending a misleading
  notification.

If a public integration point is not appropriate, the standalone plugin can
observe existing events and maintain its own state, but this duplicates
time-tracking logic and is more vulnerable to login/profile and partial-varp
updates. The implementation should not modify the existing tracker merely to
send a notification.

## Configuration and UX

Suggested configuration groups:

- Master enable/disable switch.
- Event toggles for birdhouse set, birdhouse ready, farming, and timers.
- ntfy topic/server URL, optional title, priority, and tags.
- Discord webhook URL and optional username/avatar policy.
- Generic callback URL, method, headers, and payload template only if a
  narrowly scoped first release can safely support them.
- Retry count, request timeout, and a “send test notification” action.
- “Include character/world/location” privacy toggles.

Secrets must be treated as credentials. Do not log webhook URLs, authorization
headers, or full request bodies. Configuration values should use RuneLite's
secret/config handling where available, and the UI should warn that a webhook
URL grants whoever possesses it permission to post.

## Canonical payload

Use a small, versioned schema so ntfy, Discord, and later callbacks describe
the same event:

```json
{
  "schema": 1,
  "event": "BIRDHOUSE_SET",
  "occurredAt": "2026-08-07T02:33:00Z",
  "readyAt": "2026-08-07T03:23:00Z",
  "character": "optional",
  "world": "optional",
  "details": {
    "location": "Fossil Island",
    "count": 4
  }
}
```

The character name and world should be omitted by default or explicitly
disabled by configuration. Payloads should contain no credentials, chat
contents, or unnecessary gameplay telemetry.

## Delivery behavior

- Enqueue work and return immediately from event handlers; never perform
  blocking I/O on the game thread.
- Use bounded queues and a small, shared executor. Define behavior when the
  queue is full (prefer dropping/coalescing duplicate notifications).
- Deduplicate by `(profile, event type, state transition, readyAt)` so repeated
  ticks, reconnects, and plugin reloads do not spam destinations.
- Retry transient failures with a small exponential backoff and a cap.
  Respect HTTP status codes and `Retry-After`; do not retry most 4xx errors.
- Apply per-destination rate limits and redact failures in user-facing logs.
- Close/cancel outstanding calls during plugin shutdown and avoid sending after
  the plugin is disabled.
- Treat delivery as best effort. The game state remains authoritative even if
  the network is unavailable; optionally show a local warning or delivery
  status in the configuration panel.

Illustrative flow:

```java
void onNotification(NotificationEvent event)
{
    if (dedupe.seen(event.id())) return;
    deliveryQueue.offer(event);
}

void deliver(NotificationEvent event, Destination destination)
{
    Request request = destination.toRequest(event);
    httpClient.newCall(request).enqueue(callbackFor(destination));
}
```

## Provider details

### ntfy.sh

Send a short title and text body to the configured topic using HTTPS. Support
the configured ntfy server rather than hard-coding the public service. Headers
such as `Title`, `Priority`, `Tags`, and an optional bearer token should be
constructed by the ntfy adapter. Validate the server URL and topic, and
document that public topics are discoverable unless access controls are used.

### Discord

Use a Discord webhook URL and a compact embed or plain content message. Keep
the initial formatter conservative to avoid leaking account data, and handle
Discord's payload and rate-limit responses. Discord's webhook URL is a secret;
never include it in diagnostics or clickable log output.

### Generic callbacks

A generic HTTP destination is flexible but increases security and support
surface. Defer templated headers, arbitrary methods, and arbitrary body
expressions until the fixed ntfy/Discord adapters establish validation,
redaction, rate limiting, and documentation patterns. If added, allow only
HTTPS by default, cap body/header sizes, reject local/private network targets
where practical, and make the exact outbound data visible in the UI.

## RuneLite-specific constraints

- Prefer public API events and services over reflection, client hooks, or
  scraping widget text.
- Preserve the existing Time Tracking plugin's login, profile-switch, and
  partial-state safeguards.
- Follow plugin lifecycle conventions: register/unregister event subscribers,
  start/stop scheduled work, and release queued HTTP calls.
- Use the repository's existing OkHttp, event bus, config, logging, and
  executor facilities instead of adding dependencies.
- Keep the feature opt-in and disabled until a destination is configured.
- Review RuneLite plugin distribution and external-network policies before
  proposing inclusion in the built-in client; an external plugin may be the
  appropriate first release.
- Do not make gameplay actions or claim that a notification was delivered when
  only the request was queued.

## Testing strategy

1. Unit-test event detectors with seeded, completed, reset, login, profile
   switch, partial-state, and repeated-tick scenarios.
2. Unit-test canonical payload serialization and omission of disabled private
   fields.
3. Use a fake/mock HTTP client or local test server to verify request method,
   URL, headers, body, timeout, retry, cancellation, and redaction behavior.
4. Verify that a slow or failing destination does not block event processing or
   another destination.
5. Manually test enabling/disabling the plugin, changing profiles, shutting
   down during an in-flight request, and rate-limited provider responses.

## Incremental implementation

1. Define the canonical event and destination interfaces without network
   behavior.
2. Add a birdhouse detector at a stable time-tracking integration boundary and
   verify transition/deduplication tests.
3. Implement one ntfy adapter and a test-notification action.
4. Add the Discord adapter, provider-specific formatting, and rate-limit
   handling.
5. Add optional farming/timer detectors only after their state transitions are
   equally well-defined.
6. Evaluate a generic callback adapter and distribution model after the fixed
   providers have demonstrated safe configuration and delivery behavior.

## Open decisions

- Should the first version be an external plugin or a built-in RuneLite
  feature?
- Is a stable public event needed from Time Tracking, or should notification
  support live inside that plugin?
- Should queued events survive a client restart, or should delivery remain
  in-memory and best effort?
- Which account/profile identifier, if any, is useful enough to justify
  including it in a notification?
- Should provider credentials be stored per RuneLite profile?
