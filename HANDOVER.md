# Linux-verifiable RFC handover

Last productive source output before the original handover request: 2026-11-03T13:52:11-08:00. Push and PR verification followed in the same turn; their precise completion timestamp was not recorded. Subsequent handover preparation does not change this activity timestamp.

## Existing product lifecycles

- Homeward PR #1: https://github.com/zati-run/homeward/pull/1
  - Branch: docs/rfc-001-zati-live
  - Verified head: f07654f8a2137b52022bac92d66e19f430cfa798
  - Scope: README and RFC 002 for outbound home-to-cloud API transport.
- Zamu PR #5: https://github.com/sirerun/zamu/pull/5
  - Branch: docs/rfc-distributed-backends
  - Verified head: daa3ee579c14ff1c6808af44b45401c65e9c982b
  - Scope: RFC 002 proposing distributed voice backend extensions.

Both PRs are drafts. Continue those branches and PRs for review fixes; do not create duplicate product lifecycles or merge this handover branch into product history.

## Owner direction

Homeward is independently usable: the cloud exposes REST/WebSocket APIs, an internal gRPC/TLS connection is initiated by the home agent, and the agent connects to equivalent configured local APIs. Voice orchestration belongs in Zamu. Its RFC is maintained in Zamu, not Homeward.

The requested document rewrite is complete. Next work is review and acceptance, including final protocol schema, enrollment, limits, HTTP compatibility and Zamu API deltas. Drafts are not implementation authorization or evidence of deployed behavior.

## Verification and execution scope

Owned source worktrees were clean at handover and both heads were verified against GitHub. Markdown whitespace and local-link checks were performed during authoring. No runtime tests, independent review, merge, installation or deployment are claimed.

RFC review, document edits and Markdown checks are Linux-verifiable. No Mac UI, Swift, Flutter, native builds or visual acceptance are part of this remaining task. If implementation expands into those areas, keep their verification local and split it from Linux work.

## Custody and recovery

No task-owned running jobs, resource leases or claims exist. No cloud worker has been launched by this task. Global cloud ownership has not been verified: the coordinator must reconcile assignment before dispatch; this handover is not a new execution claim. Foreign work remains untouched. There is no unique uncommitted source to transfer.

Use the exact pushed PR heads as recovery checkpoints, then check current remote state before editing. No technical blocker prevents Linux review. Ajent context checks previously failed connectivity; do not interpret that as an empty inbox or absence of other owners.
