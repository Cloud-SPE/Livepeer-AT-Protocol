# Appendix D: Sequence diagrams

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). Each diagram traces one flow from the proposal step by step.*

Each step carries one of three markings:
- **"today"** marks a step that already happens in livepeer-node, Clearinghouse Batteries or Builder Core. Builder Core features from its milestones M1 to M5 count as today, because the pilot assumes they are complete.
- **"A1"** marks a proposed upstream change.
- An unmarked step belongs to the proposed telemetry system: the telemetry agent, the aggregator, scorers and labelers.

Record type names refer to the schemas in [Appendix A](appendix-a-record-schemas.md). Code changes refer to [Appendix B](appendix-b-change-list.md).

## D.1 An operator joins

An orchestrator becomes visible to every aggregator without asking anyone. The chain already lists it. Its binding proves that the repository and the on-chain address belong to the same operator.

```mermaid
sequenceDiagram
  autonumber
  actor Op as Orchestrator operator
  participant Chain as Arbitrum
  participant Ag as Telemetry agent
  participant Repo as Operator's repository
  participant Orch as livepeer-node orchestrator
  participant Idx as Chain indexer
  participant Agg as Relay and aggregator

  Op->>Chain: register a service URI and stay active (today)
  Op->>Ag: agent init
  Ag->>Repo: create a repository for a new did:plc
  Ag-->>Op: binding to sign: DID, repository key hash, node account, role, validity
  Op->>Op: sign with the registered key, for example on a hardware wallet
  Op->>Ag: signed binding
  Ag->>Repo: write operator.binding and operator.policy
  Ag->>Orch: read /discovery (today)
  Ag->>Repo: write orch.inventory, and one orch.offering per work key
  Idx->>Chain: read active orchestrators and service URIs from both registries
  Idx-->>Agg: participant list for the round
  Agg->>Repo: fetch the repository
  Agg->>Agg: verify each commit with the key the binding names
  Agg->>Agg: recover the binding's signer and compare it with the participant list
  Agg->>Repo: subscribe to every later change
  Note over Op,Agg: No credential, invitation, or approval was needed at any step
```

**A gateway** joins the same way, with two differences:
- Its participant status comes from its deposit and reserve in TicketBroker, not from bonding.
- Its sender key is always online, because its remote signer uses it for every ticket. The agent can therefore have the binding signed from the signer's own keystore.

**An application** joins without a binding, because it has no on-chain address.
1. It creates its DID. `did:web` on its own domain suits most applications.
2. It writes its publication policy.
3. It asks the relay to crawl its repository. Such requests are rate-limited, because the relay's normal crawl list comes from chain state.

Its records carry weight only as far as their rand hashes match published gateway digests.

## D.2 A paid call

An application makes a single-shot call through Builder Core and pays through a gateway. Three parties end up publishing a five-minute summary that includes the same session, and all three carry the same rand hash prefix. That shared prefix is what lets an aggregator match them later.

```mermaid
sequenceDiagram
  autonumber
  participant App as Application with Builder Core
  participant D as Discovery service
  participant O as Orchestrator
  participant R as Live runner
  participant S as Gateway's remote signer
  participant B as Gateway's Batteries
  participant Rp as Each party's repository

  D->>O: GET /discovery, periodically (today)
  O-->>D: runners, prices, and capacity, signed by the node account (signature A1)
  App->>D: read the inventory (today)
  App->>App: select a runner and write the attempt before calling (today)
  App->>O: call the runner app's path (today)
  O-->>App: 402 challenge with price and ticket parameters, including the rand hash (signature A1)
  App->>App: store the manifest before asking for payment (today)
  App->>S: generate-live-payment with the allocation's API key (today)
  S->>B: authorize the key (today)
  B-->>S: payment session ID (today)
  S->>S: verify the challenge signature (A1) and sign tickets carrying the rand hash (today)
  S->>S: persist create_signed_ticket with the rand hash as pm_session_id (today)
  S-->>App: payment
  App->>O: paid call (today)
  O->>O: Receive credits the tickets, and Charge debits the fixed price (today)
  O->>R: proxy the request (today)
  R-->>O: response
  O-->>App: response, counted by outcome, latency, and bytes (today)
  O->>O: write the session outbox record with rand hash, payer, runner, credit, and charge (A1)
  App->>App: attempt succeeded, with its cost pending until Batteries reports it (today)
  B->>B: debit the allocation from the signer event (today)
  App->>B: cost sync reads usage rows, now carrying the rand hash (field A1)
  loop every five minutes, through each party's telemetry agent
    App->>Rp: client.observation: outcomes, latency, paid failures, rand hash prefixes
    S->>Rp: gateway.ledgerDigest: sessions, expected value, fees, rand hash prefixes
    O->>Rp: orch.serviceReport: quality per runner, and payments per payer with rand hash prefixes (A1)
  end
  Note over App,Rp: All three records list the same rand hash prefix
```

**What exists today.**
- Builder Core records the attempt, its outcome and its cost class.
- The signer records the payment, and Batteries records the charge.
- The orchestrator's metrics count the call.

**What is missing:**
- the agents that publish these records;
- the rand hash on Batteries' usage rows, which lets the application's record join the others;
- the orchestrator's per-payer session record (A1).

Without A1, the orchestrator's report still carries quality per runner. It cannot yet be matched session by session.

**Failure on a paid call.** Suppose the runner is gone, or out of capacity, after the payment went out. Builder Core then moves the job to a runner on another orchestrator. The first attempt's charge is classified `failed_after_payment`. The application's observation counts that charge against the first orchestrator, under the same rand hash that appears in the gateway's digest.

## D.3 A persistent session

An application holds a persistent session with a runner app priced per hour and charged by the second. The session outlives its first ticket parameters, so it spans more than one rand hash.

```mermaid
sequenceDiagram
  autonumber
  participant App as Application with Builder Core
  participant O as Orchestrator
  participant R as Live runner
  participant S as Gateway's remote signer

  App->>O: reserve a session on a persistent runner app (today)
  O-->>App: 402 challenge with rand hash H1
  App->>S: generate-live-payment (today)
  S-->>App: payment, recorded under H1 (today)
  App->>O: reserve with payment (today)
  O->>R: reserved (today)
  O-->>App: session ID (today)
  loop while the session is open
    App->>O: traffic through the session proxy: HTTP, SSE, WebSocket, or trickle (today)
    O->>R: proxy (today)
    O->>O: Charge per elapsed second (today)
    App->>S: next payment (today)
    S-->>App: tickets under the current rand hash (today)
    App->>O: payment, and Receive credits it (today)
  end
  opt ticket parameters expire
    S-->>App: 480: refresh required (today)
    App->>O: refresh the payment (today)
    O-->>App: new ticket parameters with rand hash H2 (today)
  end
  App->>O: stop the session (today)
  O->>R: released (today)
  O->>O: count the session end by cause (today)
  O->>O: write outbox records for H1 and H2, with the end cause on the last (A1)
  Note over App,S: Each rand hash is one payment session in every party's records
```

A session can also end for other reasons:
- an idle timeout or the maximum duration in Builder Core;
- the runner stopping;
- payment failing;
- the engine stopping.

livepeer-node already counts session ends by cause. Builder Core records how each session ended as part of its job record. Both causes appear in the published summaries. `lr-v1` counts causes that lie with the orchestrator or its runner against the orchestrator, such as a runner stopping, expiring or failing its health checks. It does not count a stop the client requested, a normal completion, or a balance the payer let run out.

## D.4 Corroboration and scoring

An aggregator receives up to three accounts of the same sessions, matches them, anchors them on-chain, and weighs each reporter. A scorer then turns corroborated measurements into a published score that anyone can recompute.

```mermaid
sequenceDiagram
  autonumber
  participant Rel as Relay or direct subscription
  participant Idx as Chain indexer
  participant Agg as Aggregator
  participant Sco as Scorer
  participant Lab as Labeler
  participant XP as Scorer repository and labeler service

  Rel-->>Agg: client.observation from application A
  Rel-->>Agg: gateway.ledgerDigest from gateway G
  Rel-->>Agg: orch.serviceReport from orchestrator O
  Agg->>Agg: verify signatures and bindings, and resolve node accounts to registered addresses
  Agg->>Agg: match rand hash prefixes across the three records
  alt the records agree on sessions and value
    Agg->>Agg: mark the sessions corroborated
  else a prefix appears on one side only
    Agg->>Agg: record a mismatch for the parties involved
  end
  Idx-->>Agg: WinningTicketRedeemed events, each revealing the orchestrator's secret
  Agg->>Agg: hash each secret and anchor the session it matches
  Agg->>Agg: compare reported expected value with redemptions
  Agg->>Agg: weight A's report by independence: declared affiliation, payer diversity, self-dealing signals
  Sco->>Agg: read corroborated measurements for the hour
  Sco->>Sco: compute lr-v1 per orchestrator, work key, and vantage cluster
  Sco->>XP: write score.published with formula, code reference, and inputs
  Lab->>Agg: read mismatch, relationship, and probe data
  Lab->>XP: publish labels such as report-mismatch or self-dealing-suspected
  Note over Sco,XP: Anyone holding the same inputs can recompute the score
```

**Three variations change how much evidence an aggregator has:**
- **The application does not publish.** The match is two-sided, between gateway and orchestrator. The application's sessions are still counted, unnamed, in the gateway's digest.
- **The application runs its own gateway.** Its observation and its ledger digest come from one party, so they are not independent. The orchestrator's report and the chain are the independent checks.
- **Before A1.** Only the gateway's digest carries rand hashes. Its sessions are anchored by redemptions. The application and the gateway can be compared only in aggregate, per orchestrator and window. The orchestrator contributes quality per runner only. An application that runs its own gateway can publish rand hashes from its own signer events, but its records are then not independent of its digest.

## D.5 Scores in selection

Each application chooses which scorers and labelers to trust. It reads them through an attested discovery view running next to its engine. How it uses them depends on the phase.

```mermaid
sequenceDiagram
  autonumber
  participant App as Application with Builder Core
  participant V as Attested discovery view
  participant D as Discovery services
  participant A as Scorer A
  participant B as Scorer B
  participant L as Trusted labelers
  participant O as Candidate orchestrators

  V->>D: read runner inventory (today)
  V->>A: read the latest score.published
  V->>B: read the latest score.published
  V->>L: read labels
  V->>V: verify signatures and bindings
  V->>V: take the median score per orchestrator where enough scorers publish
  App->>V: GET /discovery through LPB_DISCOVERY_URL (today)
  alt Phase A, shadow
    V-->>App: inventory unchanged
    App->>V: selection policy reads scores for the candidates
    App->>App: select as today, and record the score beside the outcome
  else Phase B, filter
    V-->>App: inventory without runners below the minimum score or without required labels
    App->>App: select among the rest as today
  else Phase C, weight
    V-->>App: filtered inventory
    App->>V: selection policy reads scores for the candidates
    App->>App: weight score with free capacity, price, and randomness
  end
  App->>O: call the selected runner (today)
  Note over V,App: An exploration share ignores scores. With no fresh trusted score, the view serves the inventory unfiltered.
```

**The remote signer** uses scores without any change to its code. Its `Orchestrators` list points at an attested discovery view. Every client that discovers through the signer's `/discover-orchestrators` then receives the filtered inventory.

**Applications that use the Python SDK directly** can read the same view. Ranking needs the SDK's runner selector to keep the order it is given.

## D.6 Preserving history

Participants may compact or prune detailed records once independent mirrors hold them. Mirrors keep deleted records, so pruning cannot erase history.

```mermaid
sequenceDiagram
  autonumber
  participant Ag as Telemetry agent
  participant Rp as Participant's repository
  participant M as Mirrors
  participant C as Cold archive
  participant Chain as Arbitrum

  Rp-->>M: every commit, continuously
  M->>M: verify and store each commit
  loop once per round
    C->>M: export every participant repository
    C->>C: store each export by content hash
    opt on-chain anchoring
      C->>Chain: post the hash of the round's export index
    end
  end
  Ag->>Rp: write hourly and daily summaries compacted from five-minute records
  Ag->>M: confirm that mirrors hold the five-minute records
  Ag->>Rp: delete five-minute records older than the retention period
  Note over Rp,M: Deletions appear in the change stream, and mirrors keep the deleted records
```

## D.7 Auditing a summary

Any summary can be checked against the raw events it was computed from, for as long as the publisher's stated retention period lasts.

```mermaid
sequenceDiagram
  autonumber
  participant Aud as Auditor
  participant Rp as Publisher's repository
  participant Ag as Publisher's telemetry agent

  Aud->>Rp: read a summary record
  Rp-->>Aud: summary with Merkle root, event count, and retainedUntil
  Aud->>Ag: request the raw batch for that summary
  Ag-->>Aud: the redacted raw events
  Aud->>Aud: recompute the Merkle root and the summary from the raw events
  alt root and summary match
    Aud->>Aud: summary confirmed
  else mismatch, or batch refused before retainedUntil
    Aud->>Aud: publish a report-mismatch label
  end
```

The Merkle root is computed over events that are already redacted. The agent applies its allowlist before it stores anything, so the batch it serves is exactly the batch it committed to, and it never has to reveal a private field to prove a summary.

## D.8 A leaked canary

A probe carries a test credential issued to one orchestrator only. The prober commits to it publicly before the job runs. If the credential is ever used, anyone can confirm which orchestrator received it. Where the orchestrator hosts a probe runner app, the prober also checks TLS certificates and DNS answers from it, and those results go into the same `probe.result`.

```mermaid
sequenceDiagram
  autonumber
  participant Pr as Prober with Builder Core
  participant PP as Prober's repository
  participant O as Orchestrator
  participant CM as Canary monitor
  participant Lab as Labeler
  participant V as Application's attested discovery view

  Pr->>Pr: create canary K for orchestrator O, with random nonce n
  Pr->>PP: write probe.result with commitment hash(K, O, n)
  Pr->>O: send a paid probe call carrying K, as for any workload
  O-->>Pr: result
  Note over O: O copies K out of the request
  O->>CM: K is used, by O or by anyone O gave it to
  CM->>CM: publish signed evidence of use under its own identity
  Pr->>PP: write probe.canaryReveal with K, O, n, and the evidence
  Lab->>PP: read the commitment and the reveal
  Lab->>Lab: check that hash(K, O, n) matches a commitment written before the use
  Lab->>Lab: publish a secret-misuse label on O
  V->>Lab: read labels from labelers the application trusts
  V->>V: drop O's runners from the inventory it serves
  Note over Pr,Lab: The commitment proves where K went. Evidence of use still depends on the canary monitor.
```

A canary catches only secrets that are used. It does not catch two kinds of operator ([§6.7](README.md#67-host-abuse)):
- one who copies secrets and never uses them;
- one who recognizes probe traffic and leaves it alone.

Paying for probes through a gateway that real applications also use makes the second harder ([§6.4](README.md#64-test-traffic)).
