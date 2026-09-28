# Appendix D: Sequence diagrams

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). Each diagram traces one flow from the proposal step by step. Steps marked "today" already happen in go-livepeer; everything else is proposed.*

Record type names refer to the schemas in [Appendix A](appendix-a-record-schemas.md). Code changes refer to [Appendix B](appendix-b-change-list.md).

## D.1 An operator joins

An orchestrator becomes visible to every aggregator without asking anyone. The chain already lists it; its repository proves that the repository and the on-chain address belong to the same operator.

```mermaid
sequenceDiagram
  autonumber
  actor Op as Orchestrator operator
  participant Chain as Arbitrum
  participant Orch as go-livepeer orchestrator
  participant PDS as Operator's PDS
  participant Idx as Chain indexer
  participant Agg as Aggregator

  Op->>Chain: setServiceURI to https://orch.example.com:8935 (today)
  Op->>Orch: start with the publishing flag
  Orch->>PDS: create repository for did:web:orch.example.com
  Orch->>Orch: serve DID document at /.well-known/did.json
  Orch->>PDS: write operator.binding, signed with the Ethereum key
  Orch->>PDS: write orch.fleet and orch.quote
  Idx->>Chain: read active orchestrators and service URIs
  Idx-->>Agg: participant list with service hosts
  Agg->>Orch: fetch /.well-known/did.json
  Orch-->>Agg: DID document naming the signing key and the PDS
  Agg->>PDS: getRepo, a full signed copy of the repository
  Agg->>Agg: verify the commit signature with the DID's key
  Agg->>Agg: recover the binding signer and compare it with the on-chain address
  Agg->>PDS: subscribeRepos, to receive every later change
  Note over Op,Agg: No credential, invitation, or approval was needed at any step
```

A remote signer joins the same way, except that its participant status comes from its TicketBroker deposit rather than from bonding, and it may also publish `operator.delegation` records naming the gateways it pays for.

## D.2 A live video-to-video session

A go-livepeer gateway runs a live video-to-video stream on an orchestrator. Both sides end up publishing a five-minute summary of the same session, and both summaries carry the same `recipient_rand_hash` values, which is what lets an aggregator match them later.

```mermaid
sequenceDiagram
  autonumber
  participant G as go-livepeer gateway
  participant O as Orchestrator
  participant R as Live pipeline runner
  participant K as Gateway's local Kafka
  participant Sc as Gateway sidecar
  participant GP as Gateway's PDS
  participant Pub as Orchestrator publisher
  participant OP as Orchestrator's PDS

  G->>O: GetOrchestrator with capability (today)
  O-->>G: OrchestratorInfo with prices, ticket params, and signed quote
  G->>G: verify quote signature and keep its hash
  G->>O: start live-video-to-video with first payment (today)
  O->>O: ProcessPayment credits tickets and updates counters
  O->>R: start pipeline (today)
  loop every payment interval
    G->>O: payment tickets (today)
    O->>O: ProcessPayment, then DebitFees per second, updating counters
  end
  R-->>G: status events over the trickle events channel (today)
  G->>K: stream_trace, ai_stream_status, create_new_payment (today)
  loop every five minutes
    Sc->>K: read the window's events
    Sc->>Sc: drop client IPs, prompts, and stream IDs
    Sc->>Sc: build counts, histograms, and the Merkle root of the raw batch
    Sc->>GP: write gateway.observation
    Pub->>O: read payment counters for the window
    Pub->>OP: write orch.serviceReport
  end
  Note over GP,OP: Both records list the same recipient_rand_hash values
```

The gateway needs no code change in Phase A: the sidecar reads events go-livepeer already sends to Kafka. The orchestrator needs the counters in [§B.1.1](appendix-b-change-list.md#b11-cumulative-accounting-counters--phase-a-standalone) and the publisher in [§B.1.6](appendix-b-change-list.md#b16-embedded-publisher--phase-a).

## D.3 A Live Runner session

An application using the Python SDK runs a persistent Live Runner session, paying through a remote signer. The application holds no key and publishes nothing itself. Its signer checks the application's telemetry against its own payment records and publishes it.

```mermaid
sequenceDiagram
  autonumber
  participant App as Application with Python SDK
  participant S as Remote signer
  participant O as Orchestrator
  participant R as Live runner app
  participant SP as Signer's PDS

  S->>O: GET /discovery, refreshed periodically (today)
  O-->>S: runners, prices, and capacity in a signed envelope
  S->>S: verify the envelope
  App->>S: discover orchestrators for app X (today)
  S->>S: rank candidates by trusted scores (Phase B)
  S-->>App: ranked candidates
  App->>O: POST /apps/X/session with payer address (today)
  O-->>App: 402 challenge with OrchestratorInfo and signed quote
  App->>App: verify the quote signature
  App->>S: generate-live-payment for this orchestrator (today)
  S->>S: authorize with the clearinghouse and sign tickets (today)
  S-->>App: payment
  App->>O: POST /apps/X/session with payment (today)
  O->>O: ProcessPayment, with session ID equal to manifest ID
  O->>R: reserve session (today)
  O-->>App: session ID and app URL
  loop while the session runs
    App->>O: requests to the app URL
    O->>R: proxy, recording status, latency, and bytes
    App->>S: request next payment
    App->>O: payment
    O->>O: DebitFees per second, updating counters
  end
  App->>O: stop session (today)
  O->>R: release (today)
  App->>S: five-minute telemetry summary
  S->>S: check the summary against its payments for this client and orchestrator
  S->>SP: write gateway.observation with relayedFor set to a salted client hash
  S->>SP: write signer.ledgerDigest once per round
  Note over O: The orchestrator publishes its side as in D.2, including per-runner measurements
```

## D.4 Corroboration and scoring

An aggregator receives both sides of a session, matches them, checks payment on-chain, and weighs the reporter. A scorer then turns corroborated measurements into a published score that anyone can recompute.

```mermaid
sequenceDiagram
  autonumber
  participant Rel as Relay or direct subscription
  participant Idx as Chain indexer
  participant Agg as Aggregator
  participant Sco as Scorer
  participant Lab as Labeler
  participant XP as Scorer and labeler PDSs

  Rel-->>Agg: gateway.observation from gateway G
  Rel-->>Agg: orch.serviceReport from orchestrator O
  Agg->>Agg: verify signatures and identity bindings
  Agg->>Agg: match the two records on recipient_rand_hash values
  alt counts agree within tolerance
    Agg->>Agg: mark the window corroborated
  else counts disagree
    Agg->>Agg: record a mismatch for G and O
  end
  Idx-->>Agg: WinningTicketRedeemed events for G's sender and O
  Agg->>Agg: compare reported expected value with redemptions
  Agg->>Agg: weight G's report by payer diversity and self-dealing signals
  Sco->>Agg: read corroborated measurements for the hour
  Sco->>Sco: compute sla-v1 per orchestrator, work, and vantage cluster
  Sco->>XP: write score.published with formula name and inputs hash
  Lab->>Agg: read relationship and mismatch data
  Lab->>XP: publish labels such as report-mismatch or self-dealing-suspected
  Note over Sco,XP: Anyone holding the same inputs can recompute the score
```

## D.5 Scores in selection

Gateways and signers each choose which scorers to trust. How they use the scores depends on the phase.

```mermaid
sequenceDiagram
  autonumber
  participant G as go-livepeer gateway or remote signer
  participant A as Scorer A's PDS
  participant B as Scorer B's PDS
  participant O as Candidate orchestrators

  G->>A: read latest score.published
  G->>B: read latest score.published
  G->>G: verify signatures and bindings
  G->>G: take the median score per orchestrator across trusted scorers
  alt Phase A, shadow
    G->>G: select as today, and log score alongside outcome
  else Phase B, filter
    G->>G: drop candidates below the minimum score
    G->>G: select among the rest as today
  else Phase C, weight
    G->>G: add score as a weighted term with stake, price, and randomness
  end
  G->>O: send job to the selected orchestrator
  Note over G: A random share of selection always remains, so new orchestrators still receive traffic
```

For go-livepeer gateways, this replaces the single `-orchPerfStatsUrl` source. For Live Runner, the remote signer applies it inside its discovery endpoint, as step 5 of D.3 shows.

## D.6 Preserving history

Operators may prune detailed records once independent mirrors hold them. Mirrors keep deleted records, so pruning cannot erase history.

```mermaid
sequenceDiagram
  autonumber
  participant Pub as Orchestrator publisher
  participant OP as Orchestrator's PDS
  participant M as Mirrors
  participant C as Cold archive
  participant Chain as Arbitrum

  OP-->>M: every commit, continuously
  M->>M: verify and store each commit
  loop once per round
    C->>M: export every participant repository
    C->>C: store each export by content hash
    opt on-chain anchoring
      C->>Chain: post the hash of the round's export index
    end
  end
  Pub->>OP: write hourly and daily summaries compacted from five-minute records
  Pub->>M: confirm that mirrors hold the five-minute records
  Pub->>OP: delete five-minute records older than the retention period
  Note over OP,M: Deletions appear in the change stream, and mirrors keep the deleted records
```

## D.7 Auditing a summary

Any summary can be checked against the raw events it was computed from, for as long as the publisher's stated retention period lasts.

```mermaid
sequenceDiagram
  autonumber
  participant Aud as Auditor
  participant P as Publisher's PDS
  participant Op as Publisher

  Aud->>P: read a summary record
  P-->>Aud: summary with Merkle root, event count, and retainedUntil
  Aud->>Op: request the raw batch for that summary
  Op-->>Aud: raw events, with private fields redacted as the schema requires
  Aud->>Aud: recompute the Merkle root and the summary from the raw events
  alt root and summary match
    Aud->>Aud: summary confirmed
  else mismatch, or batch refused before retainedUntil
    Aud->>Aud: publish a report-mismatch label
  end
```

One design question remains open here: raw events contain the private fields that summaries omit, so the specification must define how a publisher redacts a batch while keeping the Merkle root verifiable. The simplest approach is to compute the root over already-redacted events.
