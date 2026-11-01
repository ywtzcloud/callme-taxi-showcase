# Engineering Case Study — Call Me Taxi

## Problem

A taxi platform has several actors changing shared state at nearly the same time. A customer can create or cancel an order while drivers receive dispatch events, staff monitor activity, location updates arrive asynchronously and finance logic may depend on trip completion.

## Engineering response

1. **Keep protected state server-authoritative.** The client sends intent; backend logic validates actor identity, role and lifecycle state.
2. **Separate transport from business truth.** Push and realtime are transports, not the source of authoritative success.
3. **Fence authentication generations.** Late responses from an older login cannot silently mutate a newer active session.
4. **Make transaction-sensitive actions repeat-safe.** Settlement and finance operations are designed to recognize retries.
5. **Externalize production credentials.** Production configuration and signing material stay outside public source control.

## Engineering value demonstrated

- mobile architecture
- backend authorization
- realtime systems
- distributed-state reasoning
- financial integrity
- role-aware UX
- operational tooling
- secure release engineering
