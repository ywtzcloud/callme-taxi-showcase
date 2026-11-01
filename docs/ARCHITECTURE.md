# Architecture — Call Me Taxi

## Design goal

The application is treated as a multi-role distributed system rather than a single booking UI. Customer intent, driver state, order ownership, trip lifecycle, money-related operations, notifications and staff controls all need consistent authority boundaries.

## Client structure

- Jetpack Compose UI
- ViewModels for screen-level state and orchestration
- coroutine/Flow-based asynchronous state
- repositories and core APIs for data/business boundaries
- realtime subscriptions and snapshot refresh
- FCM for asynchronous notification delivery
- Google Maps/location services for location-aware workflows

## Authority boundary

```text
User action
  -> Compose screen
  -> ViewModel
  -> Repository / protected API
  -> backend function
  -> session + role + lifecycle validation
  -> authoritative write
  -> response
  -> realtime refresh / local state update
  -> optional push notification
```

The device is never considered sufficient evidence for authorization, order ownership, financial state or lifecycle validity.

## Session model

The system separates local UI state from protected application sessions. Session-generation fencing is used to prevent responses belonging to a previous authentication lifecycle from mutating the current session.

## Finance and settlement

Transaction-sensitive operations use stable business identity and idempotent server behavior. A repeated network request should not create a second financial effect simply because the client retried.

## Realtime vs push

Realtime data refresh keeps application state synchronized. Push notifications bring attention to events. Business success is determined by authoritative backend state, not notification delivery.
