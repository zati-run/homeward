# Homeward

Homeward is a proposed independently usable bridge to home APIs. A home agent
dials a cloud gateway over gRPC/TLS; callers use REST/WebSocket endpoints in the
cloud, which connect to explicitly configured equivalent services at home.

The repository is at the RFC stage. No working agent or gateway is shipped yet.

- [RFC 002: Homeward architecture and API bridge](docs/rfc/002-homeward.md)
- [RFC 001: Proposed extensions to Zamu](docs/rfc/001-zamu-extensions.md)

Voice orchestration belongs in Zamu. Homeward handles connectivity and scoped API
transport and must remain usable without Zati or Zamu.
