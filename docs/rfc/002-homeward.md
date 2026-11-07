# RFC 002: Homeward — outbound access to home APIs

- Status: Draft; architecture direction confirmed, protocol details proposed.
- Date: 2026-11-03
- Revised: 2026-11-07
- Repository: `zati-run/homeward`
- Owner direction: Independently usable home dial-in component. The cloud exposes REST/WebSocket and other supported APIs, the home agent connects to equivalent local services, and gRPC is the internal cloud-to-home transport.

## Purpose

Homeward lets a home agent establish an authenticated outbound connection to a cloud gateway. Authorized callers use APIs exposed by that gateway to reach explicitly configured services at home. It requires no inbound home port forwarding and no dependency on Zati, Zamu, a voice model, or a particular cloud vendor.

The deliverable is a reusable Go library plus separately runnable home agent and cloud gateway. A standalone example must expose a local HTTP service and WebSocket echo service through a self-hosted gateway. Voice is one consumer, not the protocol's domain model.

## Architecture

```text
Cloud caller                         Home network
 REST / WebSocket                    Local REST / WebSocket service
        |                                         ^
        v                                         |
 Cloud gateway <==== internal gRPC/TLS ===== Home agent
                       outbound dial <-------|
```

The home agent is the gRPC client. It opens long-lived bidirectional RPCs to the gateway, which sends reverse requests over those streams. A gRPC connection alone does not turn the home agent into an inbound gRPC server.

External and local APIs retain their native protocols. Internal envelopes use versioned Protobuf messages. Ordinary Go interfaces connect components within each process; no loopback RPC is required. Additional protocol adapters require explicit contracts and qualification rather than unrestricted TCP tunneling.

## Internal protocol proposal

Use a versioned `homeward.v1` service. A control stream authenticates the agent, negotiates capabilities and limits, announces locally configured service identifiers, and carries operation offers, cancellation and connection status. The gateway intersects announcements with server-side grants; registration does not grant authority.

To isolate slow operations, the gateway offers an operation on the control stream and the agent opens a separate bidirectional exchange RPC for that operation. Bind it to the authenticated agent identity, current connection generation and a short-lived single-use offer. Reject duplicate, stale, wrong-agent and expired offers. Each exchange carries one HTTP request/response or one WebSocket session; it does not multiplex every home operation into a single unbounded queue.

All exchanges share bounded connection capacity. HTTP/2 flow control does not replace per-operation byte limits, concurrency limits, bounded queues or fair admission. Reserve control capacity; qualify whether a separate control channel is needed under load. Never hold a global lock while waiting for stream I/O.

The schema must distinguish request head, body chunk, response head, response chunk, WebSocket frame, half-close, cancellation, terminal error and completion. Carry operation ID, connection generation, sequence numbers and deadline where applicable. Protobuf field numbers are never reused. Incompatible major versions fail negotiation; unknown optional fields do not weaken authorization. Final schema and default limits are implementation prerequisites.

## REST and HTTP bridge

Cloud routes map to configured service IDs and allowed path/method patterns. A caller cannot supply an arbitrary local destination URL, host or port. The agent resolves each service ID from local configuration and enforces local policy independently.

Preserve allowed methods, paths, query strings, end-to-end headers, HTTP status and streaming body bytes. Strip hop-by-hop headers and reconstruct framing at each boundary. Explicitly configure Host rewriting and credential injection. Never forward gateway authorization credentials to a local service by default. Retain repeated headers where semantics require them, including Set-Cookie; do not log authorization values or bodies by default.

Enforce request/header/body limits, deadlines and response backpressure before buffering. Support incremental responses such as SSE through streaming HTTP bodies. Specify trailers and informational response support in the compatibility matrix before claiming transparent HTTP support. Map transport failure to a gateway error only before response headers commit; after commit terminate the stream visibly rather than inventing a successful HTTP response.

Authorization and forwarding must use the same validated request target. Reject ambiguous encodings, dot-segment traversal and conflicting framing rather than authorizing one path and forwarding another. Strip headers named by Connection as well as known hop-by-hop headers; reject invalid framing before dispatch. Replace untrusted forwarding/identity headers with gateway-derived values under explicit service policy. Reject CONNECT and unsupported upgrades; WebSocket uses its dedicated adapter.

Local redirects are returned to callers subject to configured rewriting policy; the agent must not follow redirects into unapproved destinations. Service configuration must also constrain DNS resolution and local routing where needed. Proxy access does not confer broader LAN access.

## WebSocket bridge

Terminate the public WebSocket at the gateway and open the configured local WebSocket at home. Carry text/binary message type, payload and ordering through the internal exchange. Preserve negotiated subprotocols only from an explicit supported intersection. Negotiate compression independently on each leg; do not forward compressed wire frames blindly.

Authenticate and authorize before opening either leg. For browser callers, enforce an explicit Origin policy and CSRF protection where ambient credentials are accepted; Origin alone is not authentication. Complete the local handshake before committing the public upgrade, within a bounded handshake deadline. Offer only caller-requested, configured subprotocols locally and return the locally selected protocol publicly; reject an incompatible selection. Local handshake rejection produces a sanitized HTTP failure before public upgrade. If the caller leaves during setup, close any local socket opened by the race.

Bound message size and queued bytes. Define fragmentation/reassembly limits explicitly. Handle ping/pong per terminated connection and propagate application close code/reason where valid. Transport failure closes both legs with a visible failure; reconnect creates a new session, not a transparent replay of the old one. Caller disconnect, local close, deadline and agent cancellation release both sockets and exchange resources.

## Identity, authority and lifecycle

Use TLS with authenticated agent enrollment, rotatable credentials and revocation. The deployment chooses its identity provider and credential storage; Homeward must not require Zati accounts. Authenticate cloud callers separately from agents and authorize each operation against caller, home, service and method capabilities. An agent's connection is not blanket permission for callers to use it.

The home owner explicitly configures which services are exposed and may narrow cloud grants locally. Keep local provider credentials at home. Do not expose a generic shell, arbitrary URL fetch or whole-network proxy by default. Any future MCP adapter should prefer CIMD over DCR and remain separately scoped.

A single gateway owner admits operations for a home connection generation. Replacement connections fence older generations from accepting new operations. Old in-flight effects may still complete; fencing is not rollback. Gateway replication requires a supported ownership store and routing contract before multi-instance operation is claimed. No second task scheduler is introduced.

Use bounded exponential reconnect backoff with jitter. Negotiate heartbeat/keepalive policy with deployment timeouts. A healthy transport does not prove a local service is healthy. Cancellation is best effort and must distinguish requested, acknowledged and completed states.

## Failure and recovery

Assign a stable operation identity before dispatch. Track offered, accepted, dispatched, completed, failed and outcome-unknown states. Disconnection after local dispatch may leave the outcome unknown. Never automatically retry a mutating request merely because its response was lost, including requests nominally using an idempotent HTTP verb when local semantics are unverified.

The gateway owns its operation record; the agent records local dispatch and completion receipts with bounded retention and appropriate durability. These are transport/effect receipts, not duplicate domain jobs. Define retention, reconciliation and expiration behavior before enabling retry. If receipts cannot resolve the result, return outcome-unknown and require service-specific reconciliation. Do not promise exactly-once effects.

Before any bytes can reach the local service, durably record dispatch intent bound to the operation ID, authenticated home, service, method and immutable request identity. If that write fails, do not dispatch. A crash after intent but before a completion receipt is conservatively outcome-unknown, even if dispatch might not have happened. A receipt cannot be atomic with an arbitrary service's effect. A complete transport response is not proof that a service's asynchronous job finished.

Duplicate operation IDs must never initiate a fresh dispatch: reconcile the existing receipt, or reject a conflicting request identity. Reconciliation authenticates the original scope even across connection generations and must not itself dispatch. Missing or expired receipts do not prove non-execution and cannot authorize replay. Define a bounded replay horizon and durable rejection of expired identities before enabling resubmission. Report the operation ID and uncertainty to authorized callers through a documented status contract, including after an HTTP response has already committed.

Read retries require an explicit service policy. Idempotent effect retries require a service-supported idempotency key and reconciliation contract. Homeward cannot manufacture idempotency for arbitrary local APIs. Streaming sessions and audio are not replayed automatically after reconnect.

## Independent use and boundaries

Ship a library, agent and gateway with a documented self-hosted configuration. Public REST/WebSocket endpoints and local service adapters must work without voice software. Zamu may use those endpoints through an optional adapter; Homeward contains no turn-taking, persona, transcript history, model selection or speech-generation policy.

Homeward owns only connection and exchange lifecycle. Existing services retain records, credentials, effect authorization and application job ownership. Deployment infrastructure belongs in the deploying repository's IaC; this RFC provisions nothing.

## Acceptance

1. Reach local HTTP and WebSocket examples from a separate gateway with inbound home connections blocked.
2. Verify status, repeated headers, streamed bodies, SSE, WebSocket text/binary ordering, subprotocols and close behavior.
3. Deny wrong-home callers, unregistered destinations, expired offers, revoked agents and stale connection generations.
4. Exercise caller cancellation, local failure, gateway restart and home disconnect before dispatch, during dispatch and after effect completion but before response delivery.
5. Confirm no duplicate mutating effects on reconnect; demonstrate outcome-unknown and service-specific reconciliation.
6. Slow consumers and large streams remain within configured memory/concurrency bounds; control and cancellation stay responsive under load.
7. Rotate credentials and verify old credentials cannot establish new sessions; document active-session revocation behavior.
8. Run through the intended HTTP/2 ingress and proxy idle timeouts. Record revisions, environment, commands and results; local fixtures do not establish deployed acceptance.
9. Run the examples without Zati or Zamu installed.
10. Reject encoded path-policy bypasses, conflicting HTTP framing, forged forwarding headers, unsupported upgrades and disallowed browser origins before local dispatch.
11. Verify local WebSocket handshake rejection, subprotocol mismatch and caller departure during setup never leave a successful public upgrade or orphaned local socket.
12. Inject crashes before and after durable dispatch intent and completion recording. Reject journal-write failure, conflicting duplicate identities and replay after receipt expiry; demonstrate authorized reconciliation across generations without redispatch.

## Open details

Finalize Protobuf messages, identity/enrollment mechanism, default limits, receipt storage/retention, HTTP compatibility matrix and single-gateway deployment first. Multi-gateway failover, additional protocols and transparent session resumption are not implied by v1. These details require review before implementation; the repository currently contains proposals only.

## References

- [gRPC bidirectional streams and ordering](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [gRPC flow control](https://grpc.io/docs/guides/flow-control/)
- [gRPC keepalive and broken streams](https://grpc.io/docs/guides/keepalive/)
- [Zamu extension proposal](https://github.com/sirerun/zamu/pull/5)

- [HTTP semantics and intermediary requirements (RFC 9110)](https://www.rfc-editor.org/rfc/rfc9110.html)
- [WebSocket handshake and origin requirements (RFC 6455)](https://www.rfc-editor.org/rfc/rfc6455.html)
