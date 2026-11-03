# Appendix B: Change list

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). Revised 2026-11-03 for the stack that replaces go-livepeer. The go-livepeer version remains in git history.*

*Code references are to these commits:*
- *livepeer/node at `e2b3319` (upstream main, 2026-11-03);*
- *Clearinghouse Batteries at `d47d1ff`;*
- *livepeer-builder (Builder Core) at `016c65d`;*
- *livepeer-python-gateway at `cd6ed5e`.*

*Line numbers will drift; function names are the durable reference.*

Changes are grouped by component. Each is marked with the phase that needs it:
- **A0** needs no change to software the pilot does not control. Builder Core belongs to the Network Engineering SPE, so its small additions count as A0.
- **A1** is an upstream change to livepeer-node or Batteries.
- **B** and **C** follow the selection phases in [proposal §7.1](README.md#71-three-phases).

Changes marked **standalone** are useful even if the rest of the proposal is never built. They make good first contributions.

## B.1 Summary

| Component | Already provides | Needs | Phase |
|---|---|---|---|
| livepeer-node orchestrator | Metrics per runner app and runner: calls by outcome, header latency, bytes, reservations, session ends by cause, payment counters | A session outbox; signed runner records; signed payment challenges | A1 |
| livepeer-node remote signer | One `create_signed_ticket` event per signing, with the rand hash as `pm_session_id` | Runner ID, face value and win probability in each event; verification of signed challenges | A1 |
| livepeer-node discovery service | Merged runner inventory, keeping complete runner records | Nothing | — |
| Clearinghouse Batteries | The signer's event log, a ledger, settlement matching by rand hash | The rand hash and a client tag on usage rows | A1 |
| Builder Core | Attempts with outcomes, failover, cost classes, an events feed once M3 to M5 are complete | First-byte time on attempts; the rand hash on attempt sessions; a selection policy plugin | A0, A1 |
| Python SDK | Tries runners in the order discovery returns them | Verification of signed runner records and challenges | A1 |
| Telemetry agent | — | New ([§B.8](#b8-the-telemetry-agent)) | A0 |

## B.2 livepeer-node: orchestrator

### B.2.1 Nothing for A0

The metrics added upstream in October 2026 (`docs/metrics.md`) provide everything the runner rows of `orch.serviceReport` need ([Appendix A §A.10](appendix-a-record-schemas.md#a10-networklivepeerorchservicereport)). The agent scrapes the orchestrator's loopback metrics listener, `127.0.0.1:8936` by default, so it runs on the same host or is granted access.

### B.2.2 Session outbox — A1, standalone

**Today.** Payment state lives in `PaymentEngine` (`orchestrator/payment_engine.go`):
- `MakeChallenge` (line 77) and `RefreshChallenge` (line 152) issue ticket parameters, each with a fresh rand hash;
- `Receive` (line 208) validates tickets and credits their expected value;
- `Charge` (line 313) debits the session.

Every session ends in `Registry.releaseLocked` (`orchestrator/registry.go:625`), which records the end reason. `Server.ChargePaidSessions` (`orchestrator/payment.go`) ends sessions whose charge fails, with `billing_error` or `insufficient_balance`.

Nothing keeps a record per session after it ends. The metrics deliberately omit session IDs and payers from their labels.

**Change.** Write one durable record per payment session when the session ends. A payment session is one rand hash, so a session refreshed with HTTP 480 produces one record per rand hash. Keep the records in a table in the orchestrator's existing payment store (`RedeemerDB`, default `payments.sqlite`), and serve them on the loopback metrics listener:

```
GET /sessions?after=<cursor>&limit=<n>   →   {"sessions": [...], "next_cursor": "..."}
```

| Field | Meaning |
|---|---|
| `rand_hash` | The recipient rand hash of the challenge |
| `payer` | The ticket sender address |
| `app`, `version`, `runner_id`, `mode` | The registered runner identity, as in the metrics labels |
| `started_at`, `ended_at` | Unix milliseconds |
| `tickets_accepted`, `tickets_rejected`, `tickets_winning` | Counts |
| `credit_wei`, `charge_wei` | Exact amounts |
| `billed_seconds`, `fixed_charges` | Billing units |
| `header_ms` | Single-shot only: time to the final response header |
| `end_reason` | The reason `releaseLocked` recorded |

The records follow the metrics' conventions:
- fixed fields only;
- nothing supplied by the request, so no manifest IDs, paths, headers or error text;
- served on loopback only.

Retention is configurable, seven days by default. Delivery is at least once: readers keep their cursor and drop duplicates by rand hash.

**Why it is standalone.** It gives an operator revenue per payer, and a record to settle a dispute with a gateway. The metrics cannot provide either.

### B.2.3 Signed runner records — A1, standalone

**Today.** `Server.discovery` (`orchestrator/http.go:316`) serves `Registry.Discovery` (`orchestrator/registry.go`). Each entry's `address` is the orchestrator's HTTP(S) base URL. Nothing in the response names its Ethereum address, and nothing is signed.

**Change.** Add two fields to each runner record:
- `orchestrator_address`: the registered address;
- `signature`: an EIP-712 signature by the node's keystore account over the record ([§B.10](#b10-signed-structures)).

**Why each runner record is signed, not the whole response.** Discovery services and signers merge runners from many sources and choose each runner by its newest `observed_at`. A signature over the whole response would not survive that merge. Discovery already "preserves complete runner records", so a per-record signature passes through unchanged.

**Cost.** One signature per runner per response. At typical inventory sizes and five-minute polling, this is negligible. A busy orchestrator can cache signatures for the second in which `observed_at` was stamped.

### B.2.4 Signed payment challenges — A1

**Today.** `Server.challenge` (`orchestrator/payment.go:25`) returns the 402 challenge built by `MakeChallenge`, carrying the price and ticket parameters, and the 480 refresh returns a new one. Neither is signed.

**Change.** Sign each challenge with the node account, covering:
- the runner ID;
- the price and unit;
- the ticket parameters, including the rand hash;
- an expiry.

The structure is in [§B.10](#b10-signed-structures). Because the challenge carries the rand hash, the signature ties a price commitment to the payments made under it, and to every published record about that session.

### B.2.5 Stable runner identity — B, standalone

`Registry.Heartbeat` (`orchestrator/registry.go:321`) accepts the runner's own `runner_id`, or generates `runner_<rand>`. It issues a per-runner credential after registration, but keeps it in memory only. A restarted runner may therefore reappear under a different ID.

Persisting the credential, and keying the runner's identity to it, would let pool operators track members across restarts. The pilot does not depend on it: labels and approvals attach to work keys, not runner IDs.

## B.3 livepeer-node: remote signer

### B.3.1 Nothing for A0

Each `create_signed_ticket` event (`signer/events.go`) already carries every field the A0 ledger digest needs:
- `orch_address` and `orch_url`;
- `app`;
- `pm_session_id`, the rand hash;
- `billable_secs`, `computed_fee` and `num_tickets`;
- `auth_id`, which the agent only counts.

### B.3.2 Three event fields — A1, standalone

Add `runner_id`, `face_value` and `win_prob` to `signedTicketEvent`.
- **Runner ID.** It lets the digest, and Batteries, attribute payments to a runner and not just to a runner app.
- **Face value and win probability.** They give the expected value the signer committed, which can differ from the fee it computed for the work.

This is the smallest change in the list. We suggest proposing it first.

### B.3.3 Verify signed challenges — A1

The signer receives the challenge in `POST /generate-live-payment` before it signs any ticket. When the challenge is signed:
1. Verify the signature.
2. Record `offer_hash` and `offer_signer` in the event.

The signer cannot read AT Protocol repositories in its request path. It therefore records the recovering address, and aggregators check that address against the orchestrator's binding later.

Rejecting unsigned or mismatched challenges is a Phase B option. It would need a local map of operating addresses, which the attested discovery view can serve.

### B.3.4 Discovery — no change

`GET /discover-orchestrators` serves runners in the order of the sources in the signer's `Orchestrators` list, and applies its app and GPU filters. Pointing that list at an attested discovery view filters, and orders, what every client of the signer sees.

## B.4 livepeer-node: discovery service

No change. Signed runner records pass through, because discovery keeps complete runner records. The new `--dump` flag (#37) prints the merged inventory once and exits, which gives the agent a simple way to read a discovery service.

## B.5 Clearinghouse Batteries

### B.5.1 Rand hash on usage rows — A1, standalone

**Today.**
- `usage_events` (`migrations/001_initial.sql:43`) stores each signer event's raw payload, with selected columns. The rand hash is not among them.
- `signing_authorizations` and `settlements` do store it, as `pm_session_id`.
- `GET /v1/usage` (the usage query in `internal/store/manage.go`) therefore cannot return it.

**Change.** Store each event's `pm_session_id` in a new `usage_events` column, and return it on usage rows. Any client can then reconcile its usage with on-chain settlements session by session, and Builder Core can attach the rand hash to its attempts.

Return a **client tag** with it: the first 16 bytes of HMAC-SHA256 over the rand hash, keyed by a secret Batteries holds for each API key. Give the gateway's agent only the first 8 bytes of each tag's SHA-256 hash, never the tag. Together they let an application prove which sessions it paid for without revealing itself to anyone else ([proposal §6.1](README.md#61-three-sided-reports)).

### B.5.2 The agent reads the signer's event log — A0

Batteries runs the Kafka broker that the signer publishes to: an embedded `minikafka` (`internal/kafka/listener.go`, `OpenBroker`). The gateway's agent reads the same topic with its own offsets, as Batteries itself does.

This needs a Kafka credential for the agent. If Batteries cannot yet issue a read-only one, that is a small addition.

### B.5.3 Chain indexer — A0

Batteries' TicketBroker listener (`internal/chain/listener.go`) and its settlement store handle reorganizations and match redemptions to rand hashes. The aggregators' chain indexer should reuse it, either extracted as a package or documented for reuse.

## B.6 Livepeer Builder Core

The pilot assumes Builder Core's milestones M1 to M5 are complete, including the events feed (decision D10).

### B.6.1 Events feed — included in M3 to M5

The agent reads `GET /v1/events` with the application's `events.read` scope.
- `job.finished` events carry the job's attempts: `runner_url`, `orchestrator_url`, `outcome`, `payment_sent`, `status_code` and timestamps.
- `cost.observed` and `cost.corrected` events carry the cost class of each attempt.

The runner ID is parsed from `runner_url`, which needs no change.

### B.6.2 First-byte time on attempts — A0, standalone

`lpb_attempts` records `started_at` and `ended_at`. Streams already detect their first byte (decision D31). Add `first_byte_at`, so that time to first byte can be compared with the orchestrator's header latency.

### B.6.3 Rand hash on attempt sessions — A1

Once Batteries returns `pm_session_id` and the client tag on usage rows (B.5.1):
1. Copy both into `lpb_usage_rows`.
2. Carry it on each matched entry of an attempt's `sessions`, using the matched `payment_session_id`.
3. Include both in `job.finished` and cost events.

### B.6.4 Selection policy plugin — Phase A in shadow, then B and C

An application assembles its engine with a `SelectionPolicy` (`protocols/__init__.py:94`, wired in `runtime/assembly.py`). The default is `MostCapacityFirst`. The plugin is a separate package that implements `order(candidates, request)`:

| Phase | Behavior |
|---|---|
| A, shadow | Orders exactly as `MostCapacityFirst` does. For each selection, logs the candidates, the scores available, the runner chosen and, later, the outcome. |
| B, filter | The same order. Filtering happens upstream, in the attested discovery view. |
| C, weight | Orders by a weighted sum of free capacity, score, price and a random term, with the exploration share ([proposal §7.1](README.md#71-three-phases)) |

It reads scores from the local attested discovery view. Builder Core's core does not change.

### B.6.5 Unmet demand — no change

Jobs that fail `no_offering`, and jobs whose every attempt was refused, are already visible in `job.finished`. Jobs excluded by the application's own policy are counted by the attested discovery view.

### B.6.6 Signed offers — A1

Builder Core reaches runners through the SDK. It counts the verification failures the SDK reports (B.7) and adds them to the attempt.

## B.7 livepeer-python-gateway (SDK)

- **Ranking — no change.** `runner_selector` (`selection.py`) builds its candidates in discovery order, and `RunnerSelectionCursor` tries them in that order. An application that uses the SDK directly gets the attested discovery view's order as its ranking.
- **Verify signed runner records and challenges — A1.** When the records are signed, verify them, and report each failure to the caller instead of silently continuing.
- **Telemetry for direct SDK users — optional.** An application that uses the SDK without Builder Core can send per-call outcomes to its local agent over HTTP. This is not needed for the pilot, whose applications all use Builder Core.

## B.8 The telemetry agent

One Go binary per publisher, configured with one file. [Proposal §2.4](README.md#24-the-telemetry-agent) describes its parts.

| Role | Inputs | Records written |
|---|---|---|
| Orchestrator | `/metrics` scrape; `/discovery`; the session outbox (A1) | `orch.inventory`, `orch.offering`, `orch.serviceReport` |
| Gateway | The signer's events, from Batteries' broker | `gateway.ledgerDigest` |
| Application | Builder Core's events feed; the attested discovery view's exclusion counts | `client.observation`, `client.demand` |
| Prober | Its own Builder Core events; its capability checks, canaries and probe runner app results | `probe.result`, `probe.canaryReveal` |
| Every role | — | `operator.binding` (orchestrators and gateways), `operator.policy`, `operator.affiliation` |

**Attested discovery view mode.** The same binary, configured as a view:
1. It reads inventory from discovery services or orchestrators.
2. It reads trusted scores and labels, and verifies them.
3. It serves `/discovery` to the local engine, signer or discovery service.
4. It counts the jobs its filter left without candidates.

The safety rails in proposal §7.1 are enforced here.

**Build risks**, in order:
1. **The minimal repository host.** It is sync-only: no accounts, no OAuth and no blobs, built on indigo's repository packages. It must pass a conformance suite against the indigo relay and Tap before any operator depends on it.
2. **Counter handling.** Turning Prometheus counters into exact window deltas across restarts, and reporting the resets it sees.
3. **Redaction.** An allowlist per input, so that a new field in an upstream event is dropped until someone adds it deliberately.

## B.9 Other new components

| Component | Phase | Notes |
|---|---|---|
| Chain indexer | A0 | Participants from BondingManager, both service registries and TicketBroker; redemptions with their revealed rand values. Built on Batteries' listener. |
| Livepeer-scoped relay | A0 | Crawls only the repository hosts of on-chain participants and of declared applications and probers. Cloud SPE runs the first; a second operator by the end of Phase A. |
| Mirror | A | Follows every participant repository and keeps full history. Several independent operators. |
| Cold archiver | B | One export of every repository per round, stored by content hash; optionally the hash anchored on-chain |
| Aggregator | A0 | Verifies records and bindings, resolves addresses, matches rand hash prefixes and anchors them in redemptions. Reuses NaaP's ingest and rules. |
| Scorer | A0, in shadow | Publishes `lr-v1` as `score.published`, with a daily recompute check |
| Labelers | A0 | Cloud SPE first: capability, TLS and host-abuse labels from its probes. Self-dealing labels follow in Phase B. |
| Prober | A0 | Cloud SPE's prober is an ordinary Builder Core application. It runs capability checks, and sends canaries in its requests. Where an orchestrator hosts a probe runner app, the prober also runs TLS and DNS checks from it. |

## B.10 Signed structures

Both structures are signed by an operating address that the orchestrator's binding lists ([Appendix A §A.5](appendix-a-record-schemas.md#a5-networklivepeeroperatorbinding)).

```
EIP712Domain { name: "LivepeerOrchestratorOffer", version: "1", chainId: 42161 }

RunnerRecord {                    // one per runner in a /discovery response
  address  orchestrator           // the registered address
  string   serviceUrl             // the entry's address field
  string   runnerUrl
  string   app
  string   version
  string   mode
  bytes32  gpuHash                // keccak256 of the canonical gpu object
  uint32   capacity
  bytes32  priceHash              // keccak256 of the canonical price_info object
  uint64   observedAt             // Unix milliseconds
}

PaymentChallenge {                // one per 402 challenge or 480 refresh
  address  orchestrator
  address  payer                  // the sender the challenge was issued to
  string   runnerId
  bytes32  priceHash
  bytes32  recipientRandHash      // from the ticket parameters
  uint256  faceValue
  uint256  winProb
  uint64   expiresAt
}
```

Hashes cover nested objects so the structures stay fixed as discovery adds fields. The pre-images travel in the same response, so a verifier can check the signature and read the price in one step. The gateway publishes only the hash of each signed challenge, in `gateway.ledgerDigest`.

## B.11 Proposing the upstream changes

livepeer-node's maintainers added the operational metrics this design builds on in October 2026. We suggest:

1. **Open an issue first.** Describe the operator's benefit from each change, before the proposal's.
2. **Propose in order of size.** Each change below is useful on its own.
   1. Signer event fields (B.3.2).
   2. The rand hash and client tag on Batteries usage rows (B.5.1).
   3. The session outbox (B.2.2).
   4. Signed runner records (B.2.3).
   5. Signed challenges, and their verification by the signer and the SDK (B.2.4, B.3.3, B.7).
   6. Stable runner identity (B.2.5).
3. **Follow the conventions of the metrics change:** fixed fields, nothing taken from requests, and loopback-only access.

Phase A0 proceeds without any of them.
