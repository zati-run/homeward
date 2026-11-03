# Homeward

Homeward is a proposed independently usable bridge to home APIs. A home agent
dials a cloud gateway over gRPC/TLS; callers use REST/WebSocket endpoints in the
cloud, which connect to explicitly configured equivalent services at home.

The repository is at the RFC stage. No working agent or gateway is shipped yet.

- [RFC 002: Homeward architecture and API bridge](docs/rfc/002-homeward.md)
- [Zamu RFC 002: Proposed voice extensions](https://github.com/sirerun/zamu/pull/5)

Voice orchestration belongs in Zamu. Homeward handles connectivity and scoped API
transport and must remain usable without Zati or Zamu.
