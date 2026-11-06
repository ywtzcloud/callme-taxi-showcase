# Engineering case study: six production incidents

Each incident is told as **symptom → evidence → root cause → fix → verification**. The evidence came from live diagnostics run on the production server, never from guessing.

---

## 1. "Some orders don't ring"

**Symptom.** Drivers reported that some orders arrived silently: no popup, no sound, no notification.

**Evidence.**
- A per-order worker log line was added: queue delay, run time, offers created, pushes sent and failed.
- Push failures were then grouped by FCM error code.

**Root cause.** Two interacting defects in the push path:
1. Recipient lists were cached for 60 s. A driver who had just signed in again (new push token) was still targeted with the *old* token for up to a minute.
2. When FCM rejected the old token, the clean-up step **cleared the token stored on the account**, which was by then the driver's *new, valid* token. That phone received nothing until the app re-registered.

The same clean-up also removed the driver from the dispatch candidate list, so no offer was created at all.

**Fix.**
- The token is cleared only if the stored value still equals the dead one.
- A dead token no longer removes a driver from dispatch: the offer is still created, and only the push is skipped.
- Caches are invalidated when the accounts table changes, using a cheap version check (row count + latest change) instead of a fixed timer.

**Verification.** Live drills after deploy and after a cold restart: every synthetic driver received the order with the correct wave and lock.

---

## 2. Orders reached nobody right after a deploy

**Symptom.** A live drill posted two orders immediately after a worker restart, and neither reached any driver. The same drill a minute later passed.

**Evidence.**
- A database sampler running during the drill showed an empty queue, no lock waits and no slow statements.
- Together with the worker log, this pointed at in-process cold-start work and cached state, not the database.

**Fix.**
- The push client's credentials are warmed at worker start.
- Recipient lists are re-read on account changes (incident 1).
- The drill now always includes a cold run.

**Verification.** Cold, warm and cold again: 3/3 passed.

---

## 3. The last drivers heard orders four seconds late

**Symptom.** In drills with ~1,090 real driver accounts, each test driver's offer showed `offered after 0.4 s` but `visible after 3.5–4.3 s`.

**Root cause.**
- Offers were written one row at a time (48 in parallel), so drivers at the end of the list waited for about 1,000 inserts.
- The push went out only after the last insert.

**Fix.**
- One multi-row `INSERT ... ON CONFLICT DO NOTHING RETURNING *`, in chunks of 400.
- Rows that already existed (re-offers) keep the original per-row path with its concurrency guards.

**Verification.** Production drill: visible after **1.0–1.2 s** warm and 1.8 s right after a restart. Locally with 1,100 drivers: 0.4 s.

---

## 4. The customer saw no car

**Symptom.** After a driver accepted a customer ride, the customer's map never showed the car.

**Evidence.**
- A diagnostic called the customer's tracking endpoint on the server for the latest rides: `driverLocationState=missing`.
- The driver had **no position rows at all**.
- A one-line log of every live position the driver phone sent, accepted or refused with a reason, narrowed it down.

**Root cause.**
- The driver phone's clock was **77 seconds ahead**. Every live fix carried a timestamp from the future.
- The server's freshness rule (at most 30 s in the future) refused all of them.
- The same skew also hid orders and opened accept locks early on other paths.

**Fix: never trust a phone's wall clock.**
- Phones send the fix's **age**, measured with the monotonic `elapsedRealtimeNanos`. The server computes the fix time from its own clock.
- Order creation moves all of the poster's timestamps onto the server clock, keeping the durations the app chose.
- The app learns the server offset from every HTTP response's `Date` header and from function execution timestamps. It uses that offset for countdowns and freshness checks.

**Verification.** The car appears on the customer map. Orders from the skewed phone behave like everyone else's.

---

## 5. Distance dispatch silently stopped after the first ring

**Symptom.** Distance orders were offered to the 1 km ring and then never expanded.

**Root cause.**
- The dispatch step staged a guard update inside a transaction, then re-read the order to check that nothing had changed.
- That read went through the same pinned connection, so it saw the transaction's **own uncommitted write**. Every step then concluded "the order changed under me" and stopped.

**Fix.** Reads without an explicit transaction id run on a separate autocommit connection: committed data only, which matches the semantics the business code was written for.

**Verification.** A regression test covers each ring and the citywide fallback.

---

## 6. The worker exhausted its heap every ~3.5 hours

**Symptom.** The monitor reported periodic worker restarts.

**Evidence.** The worker logs were read *before* the upgrade replaced the container. Heap growth was linear with time even with no traffic.

**Root cause.** Each idle poll registered a wake-up callback that was never removed.

**Fix.**
- Wake-ups use a set that is cleared on notify.
- Later, the whole idle loop was redesigned: `NOTIFY`-driven, with one safety poll.

**Verification.** Flat heap over days. Queue queries dropped from ~2,900 to 183 per minute.

---

## Smaller lessons worth keeping

- **First login, no orders until restart.** The realtime socket opened a moment before the session token existed and stayed anonymous. Fix: reconnect all subscriptions once the session is ready.
- **"Finish trip" refused on fixed-price rides with demands.** The finish path recomputed a fixed price *without* demands, then failed its own consistency check. Fix: one formula everywhere, with a test.
- **A build that "succeeded in 4 seconds."** A long-running Gradle daemon reused stale outputs after git had swapped the sources. Fix: the build script stops the daemon, disables file-system watching and checks that the APK contains a string only the newest commit has.
- **`grep -q` under `pipefail`** reported "not found" for a match, because the upstream `unzip` got `SIGPIPE`. Fix: count instead of quitting early.
