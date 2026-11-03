# RFC 001: Extend Zamu for distributed voice backends

- Status: Draft proposal; not accepted in Zamu.
- Date: 2026-11-03
- Scope decision: The owner directed that voice orchestration extend Zamu and that this repository become Homeward, an independently usable home-to-cloud dial-in component.
- This proposal is carried in Homeward PR #1 for boundary review. Acceptance and implementation of Zamu changes belong in the Zamu repository and its RFC process.

## Problem and decision

A voice host needs interruptible conversations using independently selectable speech recognition, language model, and speech synthesis backends. Zamu already owns voice-turn coordination, with Listener, Agent, Synthesizer, and Player interfaces. A second voice library with parallel interfaces and a translation layer into Zamu would duplicate cancellation, playback, and failure contracts.

Extend Zamu's existing interfaces and optional adapters where evidence shows a gap. Do not introduce a second conversation engine or a dependency from Zamu core on Homeward or Zati. Zamu remains directly usable by any Go host. Homeward is one optional transport for reaching home capabilities.

## Ownership

| Concern | Owner |
| --- | --- |
| Turn assembly, interruption, stable phrase emission, playback coordination | Zamu |
| Speech recognition implementation | MISTT or another selected provider |
| Speech synthesis implementation and model serving | Narrate or another selected provider |
| Backend adapters and host-configured selection | Optional Zamu packages |
| Persona, history, tools, consent, credentials, budgets and policy | Host application |
| Outbound connection, scoped API exposure and transport lifecycle | Homeward |
| Remote session controls | Existing Prosody boundary; do not duplicate it |

Homeward transports requests; it does not decide conversation turns, execute host tool policy, or acquire ownership of voice records. Credentials stay with the component that owns the provider or local service.

## Proposed Zamu extensions

### Backend integration

Inventory existing adapters before adding new ones. Reuse Listener, Agent, Synthesizer, and Player, extending capabilities only where the existing contract cannot represent required streaming or cancellation behavior. Support local and remote backends without hardcoding topology into the coordinator. A Homeward adapter must be optional and translate transport failure into existing Zamu lifecycle semantics.

Model availability, audio formats, supported languages, streaming support, and cancellation behavior must be explicit capabilities. Unsupported combinations fail before opening capture or playback. Backend selection belongs to host configuration; provider credentials are not coordinator state.

### Interruption and conversational behavior

Preserve one authoritative turn coordinator. Every response and audio segment must remain attributable to its turn and generation. On interruption, cancel generation and synthesis, stop or flush playback, and reject late output from the cancelled generation. A remote cancellation acknowledgement is not evidence that audible playback stopped; qualify that at the player boundary.

Evaluate backchannels, overlapping speech and speculative responses as bounded additions to the existing lifecycle. Speculation must not dispatch host tools or other effects before host authorization. The implementation must define which provisional outputs can be discarded and when a response becomes committed to history.

### Failover

Propose host-configured, ordered fallback among explicitly permitted providers. Do not silently move private audio or text to an unapproved provider. A failure after partial audible output requires a host-visible policy decision; do not replay an entire response automatically. Provider switches must preserve turn identity and cancel superseded work. This fallback policy remains a proposal pending Zamu acceptance.

### Speech synthesis evaluation

Pocket TTS is a candidate through the synthesis owner, not a required dependency of Zamu or Homeward. Choose models using listening tests, target-language quality, streaming latency, interruption behavior, resource use and licensing checks. No model is selected by this RFC.

## Qualification

First establish the current Zamu baseline using its accepted RFC and tests. Then qualify each new behavior independently:

1. Existing local conversation works without Homeward installed or configured.
2. A remote backend works through an optional adapter using the same coordinator.
3. Interruption during generation, synthesis and playback suppresses late output.
4. Disconnect and slow consumption remain bounded; no stale audio plays after reconnect.
5. Provider fallback obeys host permissions and never duplicates a host effect.
6. Record first-audio latency, interruption-to-silence, turn completion, memory bounds and failure outcomes on named hardware and revisions.
7. Evaluate natural turn-taking with Full-Duplex-Bench or an equivalent documented suite, plus human listening. Benchmark selection and thresholds require acceptance; a benchmark score alone is not audible acceptance.

## Delivery and unresolved decisions

This document authorizes no runtime effects. Submit the accepted extension scope to Zamu before implementing there. Resolve API deltas from current source rather than copying the previous standalone-library API. Keep the home transport proposal in [RFC 002](002-homeward.md).

Open decisions are exact Zamu interface changes, fallback commit points, benchmark thresholds and synthesis model selection. The earlier D1–D12 discussion is historical input, not automatic acceptance of replacement contracts. The superseded standalone-library draft is preserved in Git history at c008285.

## References

- [Zamu README](https://github.com/sirerun/zamu/blob/main/README.md)
- [Zamu RFC 001](https://github.com/sirerun/zamu/blob/main/docs/rfc/rfc-001.md)
