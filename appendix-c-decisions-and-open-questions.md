# Appendix C: Decisions and open questions

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). This appendix records what the authors have already decided, so reviewers can focus their feedback on what remains open.*

## C.1 Decisions taken

| # | Decision | Reason |
|---|---|---|
| D1 | Each orchestrator operator runs its own PDS. | Operators hold their own data and keys (R2). An embedded PDS in go-livepeer keeps the cost low (R7). |
| D2 | Orchestrators sign their quotes, through a new `OrchestratorInfo` field. | Makes price and hardware claims attributable. The change is backward-compatible. |
| D3 | Scores begin in a shadow phase, then filter selection, then weight it. | Scores must be shown to predict outcomes before they affect traffic. |
| D4 | Publishing is not mandated. Gateways and signers may treat absent data as lower confidence. | Keeps participation voluntary while rewarding it. |
| D5 | The operator is always accountable for its runners, including remote pool members. | Operators need per-runner measurements to remove poor pool members; this is also their first reason to run the telemetry. |
| D6 | History is preserved in three tiers: operator repositories, independent mirrors, and cold per-round exports. | Operators can delete their own records; accountability needs copies they do not control. |
| D7 | Regions are defined relative to the observing gateway, using measured latency. | The network has no authoritative region assignment, and selection needs performance from the gateway's own position. |
| D8 | Publish at the operator level; never at the end-user or customer level. | Operators are public; their customers and users are not. Public archives cannot honor deletion requests. |
| D9 | Self-dealing is flagged by labelers and down-weighted, not ignored. | It cannot be eliminated; diverse gateways and probes mitigate it. |
| D10 | The `network.livepeer.*` schemas are governed through a LIP. | Published Lexicons cannot change incompatibly, so versioning must be agreed before publication. |
| D11 | The first pilot covers live video-to-video and Live Runner. Both paths are instrumented, because no timeline exists for moving live video-to-video onto Live Runner. | Live video-to-video has a NaaP baseline; Live Runner is expected to become the general job model and has no telemetry today. |
| D12 | Phase A compares measurements with NaaP, not scores. The NaaP SLA formula becomes the first published formula, `sla-v1`, without special standing. | Measurements should agree between honest pipelines; formulas are policy and must be allowed to compete. |
| D13 | Clients in scope are the Python gateway SDK and go-livepeer gateways. | These are the public clients in use. |
| D14 | Transcoding is outside the first pilot. Livepeer Inc's stream tester becomes its first prober when it enters scope. | Keeps the pilot small; the stream tester covers transcoding only. |

## C.2 Context the decisions rest on

- **Probers today.** Livepeer Inc sends transcoding test streams and publishes the results through a public HTTP API; it is open to adopting a new model. Cloud SPE, funded by the treasury, sends live video-to-video and Live Runner test jobs.
- **Remote signers today.** Livepeer Inc operates one. Pymthouse may operate one, and Cloud SPE may become one. Whether more appear depends on incentives that are not yet clear.
- **BYOC** is deprecated and replaced by Live Runner.
- **Fleets today.** Transcoding operators run regional nodes behind geographic DNS load balancers. For AI and Live Runner work, gateway operators list an orchestrator's nodes by hand with `-orchAddr`. A single URI describing an operator's fleet has been wanted but not defined.

## C.3 Open questions

### Identity and fleet

1. How many registered service URIs use a bare IP address, which cannot yield a `did:web`? What should those operators use instead?
2. What is the key-rotation and compromise procedure for the binding record, particularly when an operating key is compromised but the registered key is not?
3. Should the fleet record eventually be referenced from a contract, and if so, which contract and through what governance path?
4. Can the reference PDS, or an embedded PDS in go-livepeer, host a single `did:web` account without the per-account subdomain handles the reference PDS is designed around? This should be tested before the pilot depends on it.

### Data

5. Is five minutes the right summary window? Which histogram bucket boundaries should each latency and frame-rate field fix?
6. How long must operators retain raw events for audit? Raw events contain fields the summaries omit, so how is a batch redacted while keeping its Merkle root verifiable? Computing the root over already-redacted events is the simplest option.
7. How should summaries handle clock differences between gateway, orchestrator, and runner hosts? NaaP has no policy for this yet.
8. What salting scheme lets a signer's `relayedFor` hash stay stable for one client over time without being reversible?

### Trust

9. What thresholds should the self-dealing signals use, given that some legitimate operators run both gateways and orchestrators?
10. How should consumers weigh a remote signer's relayed observations against a go-livepeer gateway's direct ones?
11. How many independent scorers must exist before Phase C, and who decides that the number has been reached?

### Operations and funding

12. Who operates mirrors and the cold archive, and how are they funded?
13. Should cold exports be anchored on-chain each round?
14. How should probers' deposits be funded and rotated so that probe traffic cannot be singled out?

### Upstream

15. Which go-livepeer maintainers must review the changes, and in what order should the standalone changes be proposed?
16. Should the embedded PDS live in go-livepeer itself or in a separate sidecar maintained alongside it?
