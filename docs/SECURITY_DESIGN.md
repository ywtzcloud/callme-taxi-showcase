# Security design

## Threat model (what we defend against)

| Threat | Example | Control |
|---|---|---|
| Tampered or outdated client | A modified app accepts an order during the lock, or settles a trip for someone else | Every protected action is validated on the server: session, role, order state, offer ownership, lock times, money totals. |
| Account sharing / stolen device | One driver account used on two phones | The session is bound to one device. Signing in elsewhere is an explicit takeover that revokes the other device's sessions. |
| Data over-exposure | A driver reads customer orders still in staff review, or another customer's ride | Per-collection read policies with row filters and field stripping, applied to the HTTP API **and** the realtime stream. |
| Price manipulation | A customer edits the fare in the request | Fare quotes are signed with an HMAC, expire after 2 minutes and are re-verified against the current pricing fingerprint when the order is created. |
| Replay / double submit | A retried top-up or settlement | Stable operation keys and transactional uniqueness make the second request a no-op. |
| Insider error | Staff edit the wrong field in the console | Schema validation, per-field conflict detection, a full audit log and a read-only SQL role. |
| Credential leakage | Keys in the APK, git or chat | Secrets exist only on the server. Repository checks block keys, keystores and service-account files. |
| Internet noise | SSH brute force (hundreds of attempts per day on a fresh VPS) | Firewall allows only 22/80/443; fail2ban bans repeated failures; TLS everywhere. |

## Authentication

- **Staff and drivers** sign in with phone + passcode proof.
  - The server issues an opaque session token and stores only its **SHA-256 hash**.
  - Sessions expire, can be revoked and are bound to a device id.
- **Customers** have a separate session type and a separate account collection, so a customer token can never act as a driver.
- **Clients**:
  - The token travels in a header on API and realtime requests, and in the body for business actions.
  - Every async result is tagged with a session epoch, so a response that arrives after logout or re-login is discarded.

## Authorization

- A **business action** runs inside one database transaction. It loads the actor from the session, re-reads the order or wallet row in the same snapshot, and checks role, ownership and state before writing.
- **Document API and realtime**: one policy per collection.
  - Customers see only their own orders and chats.
  - Drivers do not see staff-review orders.
  - Sensitive fields such as passcodes and device ids are always stripped.
- **Push notifications** carry an intended recipient. The app ignores a push addressed to another identity (e.g. after an offline logout on a shared device).

## Admin console

- Password hashed with **argon2** and kept in a root-only file. The cookie is `HttpOnly; Secure; SameSite=Strict`.
- State-changing requests require a custom header, which cross-site forms cannot send.
- Strict Content-Security-Policy (`default-src 'self'`).
- Every create, update and delete is written to an audit table with only the changed fields.
- SQL:
  - runs as a database role that **can only read**, one statement per request;
  - blocks file and program access functions;
  - has a statement timeout.

## Secrets handling

- Database password, push service account, routing API key and quote-signing secret are entered through **hidden prompts on the server**. They live in root-only files and are mounted read-only into containers.
- The Android app ships only public identifiers.
- Test Firebase projects and production Firebase projects are separate, so a test build cannot message production users.
- During live drills on production data, real users' push tokens are set aside and restored afterwards, so no customer or driver is disturbed.

## Host baseline (applied by the server setup script)

- Ubuntu LTS with a firewall (ufw: 22, 80, 443) and fail2ban on SSH.
- Postgres publishes no port; it is reachable only from the containers.
- Caddy terminates TLS with automatic certificates; HTTP is redirected to HTTPS.
- Backups go off-site daily. A scripted restore into a scratch database checks that a dump is usable.
