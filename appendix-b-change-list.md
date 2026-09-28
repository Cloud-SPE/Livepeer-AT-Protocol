# Appendix B: Change list

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). File references are to go-livepeer at commit `bd645a09` (upstream `livepeer/go-livepeer`) and to the other repositories as reviewed on 2026-09-28. Line numbers will drift; function names are the durable reference.*

Changes are grouped by component and marked with the phase that needs them. Changes marked **standalone** are useful even if the rest of the proposal is never built, and are good first contributions.

## B.1 go-livepeer: orchestrator

### B.1.1 Cumulative accounting counters — Phase A, standalone

Every job type credits payment in `Orchestrator.ProcessPayment` (`core/orchestrator.go:113`) and charges for work in `Orchestrator.DebitFees` (`core/orchestrator.go:481`), keyed by payer address and manifest ID. Today the only state kept is a net balance per session in `core/accounting.go`, held in memory and evicted after a timeout.

Add counters keyed by (payer, manifest ID, work key): tickets received and rejected, winning tickets, expected value credited, units debited, fees debited, and sessions stopped for insufficient balance. Expose them as Prometheus metrics and as an internal interface the publisher can read in five-minute windows.

Mapping from manifest ID to work key differs by job type:

| Job type | Manifest ID | Where the work key comes from |
|---|---|---|
| Transcoding | AuthToken session ID | The segment's transcoding profiles |
| AI batch | `<capability>_<model>` | The manifest ID itself |
| Live video-to-video | Request ID | Pipeline and model at session start (`ai_http.go`, `StartLiveVideoToVideo`) |
| Live Runner | Session ID (equal to the payment manifest ID, `ai_http.go:306`) | The runner's app id |

### B.1.2 Live Runner proxy measurements — Phase A, standalone

The orchestrator proxies every Live Runner client request (`ai_http.go` session proxy near line 733, single-shot proxy near line 756). The runner registry (`ai/runner/live_runner.go`) handles heartbeats, drops, and session release. Neither emits metrics or events today.

Record per runner and per app: requests by status class, 402 and 503 responses, time to first byte, bytes in and out, session duration, heartbeat misses, and the reason each session ended (client stop, runner stop, payment failure, heartbeat timeout, error). Expose them as Prometheus metrics, and in a local pool-health view so operators can identify and remove poor pool members.

### B.1.3 Per-runner registration credentials — Phase B, standalone

Dynamic runners authenticate with the orchestrator's shared `-orchSecret` (`core/orchestrator.go:85`, `RegistrationSecret`). Pool members are therefore indistinguishable across restarts. Allow the operator to issue one credential per pool member, and key the measurements in B.1.2 to that credential rather than to the runner-proposed `runner_id`.

### B.1.4 Signed wire quotes — Phase B

`OrchestratorInfo` is built in `orchestratorInfoWithCaps` (`server/rpc.go:595-657`) and re-sent by `RefreshPayment` (`server/rpc.go:454`), in segment responses, and in the Live Runner payment challenge (`runnerChallenge`, `ai_http.go:395`). Nothing in it is signed.

Add a field `OrchestratorQuote quote = 35` to `OrchestratorInfo` in `net/lp_rpc.proto`. Fields 8–31 and 35 onward are unused. Sign it at the end of `orchestratorInfoWithCaps` and in the Live Runner challenge using `SignTypedData` (`eth/client.go:1212`). See [§B.8](#b8-the-signed-wire-quote) for the typed-data structure.

### B.1.5 Signed Live Runner discovery — Phase B

`GET /discovery` (`ai_http.go`, handler near line 882) returns the orchestrator's address and its ready runners with prices and capacity. Wrap the response in an envelope with an EIP-712 signature over its hash and an expiry.

### B.1.6 Embedded publisher — Phase A

A publisher that reads the counters and measurements above every five minutes, builds `orch.serviceReport` records, keeps the raw batch for the retention period, and writes the records to the operator's PDS. It also publishes the binding, fleet, and public quote records. It should run with one flag, for example `-atprotoPublish=<pds-url>`, and support an embedded PDS for operators who do not want to run one separately.

## B.2 go-livepeer: gateway

### B.2.1 Verify orchestrator output signatures — Phase A, standalone

Orchestrators sign the hash of their output (`core/orchestrator.go:814` for transcoding, `core/ai_orchestrator.go:1354` for AI). The signature is returned in `TranscodeData.sig`. We found no gateway code that reads or verifies it. Verify it on receipt and count failures.

### B.2.2 Verify and record wire quotes — Phase B

On receiving `OrchestratorInfo` in `selectOrchestrator` (`server/broadcast.go:864`) and `updateSession` (`server/broadcast.go:1547`), verify the quote signature if present, record the quote's hash, and compare charged prices with quoted prices in `validatePrice` (`server/segment_rpc.go:902`).

### B.2.3 Observation publisher — Phase A

In Phase A this needs no change to go-livepeer. Gateways already emit the required events to Kafka (`monitor/kafka.go`) when started with `-kafkaBootstrapServers` and `-kafkaGatewayTopic`. Point those flags at a local broker and run a sidecar that builds `gateway.observation` records from the events. Later, the sidecar's logic can move into go-livepeer as a direct publisher.

The sidecar must drop the fields forbidden by the privacy rule: `clientIP` from `create_new_payment`, parameters and prompts from `ai_stream_events`, and raw stream and request identifiers.

Note that the Kafka producer drops events when its buffer of 100 is full (`monitor/kafka.go:170`). Phase A parity checks must account for drops on both paths.

### B.2.4 Score sources for selection — Phase B

Today `-orchPerfStatsUrl` fetches one unsigned JSON document every ten minutes (`refreshOrchPerfScore`, `cmd/livepeer/starter/starter.go:2619`), and the score is applied only as a minimum-score filter (`filterByPerfScore`, `server/selection_algorithm.go:38`).

Add `-scoreSources=<did>,<did>,...`. The gateway resolves each DID, reads its latest `score.published` record, verifies it, and uses the median across sources as the score. Keep `-orchPerfStatsUrl` working during the transition.

### B.2.5 Score as a selection weight — Phase C

Add a performance term to `calculateProbabilities` (`server/selection_algorithm.go:92`), alongside the existing stake, price, and random weights, with a flag such as `-selectPerfWeight`. Defaults to be set through governance.

## B.3 go-livepeer: remote signer

### B.3.1 Ledger digests — Phase A

The signer already emits a `create_signed_ticket` event for every payment it signs (`server/remote_signer.go:760`), with application, pipeline, orchestrator, billable seconds, and fee. Build per-round `signer.ledgerDigest` records from these events, or from the clearinghouse's ledger where one is deployed.

### B.3.2 Relay client observations — Phase B

Add an endpoint where authenticated SDK clients submit five-minute observation summaries. The signer checks each against its own payment records for that client and orchestrator, and publishes those that are consistent as `gateway.observation` records with `relayedFor` set to a salted hash of the client's API key.

### B.3.3 Score-ranked discovery — Phase B

`GET /discover-orchestrators` (`server/remote_discovery.go:248`) aggregates every orchestrator's runner list and filters by price. Add ranking by score, drawing on the signer operator's chosen `-scoreSources`, and include each candidate's score and confidence in the response.

## B.4 livepeer-python-gateway (SDK)

- **Telemetry module — Phase A.** Measure reservation outcomes, payment challenges, time to first byte, session duration, end reasons, and, for live video-to-video, frame rate and startup. Summarize every five minutes.
- **Reporting — Phase B.** Send summaries to the client's remote signer by default (B.3.2). Optionally publish directly to a PDS the application controls.
- **Ranked selection — Phase B.** `runner_selector` currently takes `candidates[0]`. Accept a ranked list from the signer, and allow applications to supply their own ranking.
- **Signature checks — Phase B.** Verify signed discovery envelopes and wire quotes. The SDK currently skips TLS verification for orchestrator connections, so signatures are the only protection against a spoofed orchestrator.

## B.5 livepeer-clearinghouse

The clearinghouse already keeps an append-only double-entry ledger, per-session usage events, and settlements matched against on-chain `WinningTicketRedeemed` events, with reorganization handling (`internal/chain/listener.go`). Two contributions:

- **Ledger roots — Phase A.** Compute a Merkle root over each round's ledger entries, for the signer's `ledgerDigest.ledgerRoot`.
- **Reusable chain indexer — Phase A.** Its TicketBroker listener is a sound basis for the chain indexer that aggregators need. Extract it, or document it for reuse.

## B.6 livepeer-naap-analytics

- **Firehose ingest — Phase A.** Add a consumer that subscribes to participants' repositories, verifies records, and writes them into the existing `accepted_raw_events` → canonical → API pipeline. The hard-coded organization literal becomes the publisher's DID.
- **Parity report — Phase A.** For the Cloud SPE gateways publishing on both paths, compare the measurements listed in [§7.3 of the proposal](README.md#73-proving-the-new-pipeline-counts-correctly), per hour and orchestrator, against the proposed tolerances.
- **Scorer — Phase A.** Publish the existing SLA score as `sla-v1` in `score.published` records under Cloud SPE's scorer DID, with the formula's source linked.

## B.7 New components

| Component | Phase | Notes |
|---|---|---|
| Chain indexer | A | Participant list from BondingManager, both service registries, and TicketBroker; ticket redemptions for payment corroboration. Build on the clearinghouse listener. |
| Livepeer-scoped relay | A (optional) | At pilot scale, aggregators can subscribe to each PDS directly. A relay is a convenience, and anyone may run one. |
| Mirror | A | Follows every participant repository and keeps full history. Several independent operators. |
| Cold archiver | B | One repository export per round, stored by content hash; optional on-chain anchor. |
| Self-dealing labeler | B | Implements the signals in the proposal's §6.6 and publishes labels (Appendix A §A.12). |
| Prober adapters | A | Cloud SPE's live video-to-video and Live Runner tests publish `probe.result`. Livepeer Inc's stream tester (`livepeer-stream-tester`, transcoding only) follows when transcoding enters scope. |

## B.8 The signed wire quote

Proposed EIP-712 structure for the `quote` field in `OrchestratorInfo`:

```
EIP712Domain {
  name:    "LivepeerOrchestratorQuote"
  version: "1"
  chainId: 42161                      // Arbitrum One
}

Quote {
  address   orchestrator              // registered or operating address
  address   gateway                   // the requesting sender
  bytes32   pricesHash                // hash of price_info and capabilities_prices, canonically encoded
  bytes32   capabilitiesHash
  bytes32   hardwareHash
  bytes32   recipientRandHash         // ticket_params.recipient_rand_hash
  uint64    issuedAt                  // unix seconds
  uint64    expiresAt
  string    node                      // fleet node label
}
```

The protobuf carries the signature and the pre-image fields the hashes cover, so a gateway can verify the signature and read the prices in one step. Gateways publish only `keccak256` of the signed quote, in `gateway.observation.quoteHashes`. Binding `recipientRandHash` into the quote is what lets an aggregator connect a price commitment, the payments made under it, and both parties' reports.
