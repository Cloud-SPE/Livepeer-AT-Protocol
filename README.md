# Operator-Owned Telemetry for the Livepeer Network

*A design proposal for collecting Livepeer job data without a central collector, using AT Protocol for publication and Livepeer's on-chain state for trust.*

**Status:** Draft for design feedback · **Date:** 2026-09-28 · **Scope of first pilot:** live video-to-video and Live Runner jobs

**Appendices:**
[A. Record schemas](appendix-a-record-schemas.md) ·
[B. Change list](appendix-b-change-list.md) ·
[C. Decisions and open questions](appendix-c-decisions-and-open-questions.md)

---

## Summary

Today, the only way to learn how the Livepeer network is performing is to be given the data by someone who collects it. The NaaP analytics pipeline shows the limits of that arrangement. It receives events from two gateway operators, each onboarded by hand with a Kafka credential. It receives almost nothing from orchestrators. Every metric it publishes rests on reports that nobody can verify. And the performance score that go-livepeer gateways use to filter orchestrators comes from a single unsigned URL.

We propose a different arrangement, in which each operator publishes its own data and anyone can collect it:

1. **Every operator publishes signed records about its own work** — orchestrators, gateways, remote signers, and test-traffic probers — into a data repository it controls. The repository format, identity system, and sync protocol come from [AT Protocol](https://atproto.com), the open protocol behind Bluesky.
2. **Livepeer's on-chain state decides who counts.** An orchestrator is legitimate if it is bonded and registered on Arbitrum; a gateway or signer is legitimate if it holds a ticket deposit. Nobody grants access to the data network, because the blockchain already records who the participants are.
3. **Records are claims, not facts.** A record proves who said something, not that it is true. Truth comes from corroboration: gateway and orchestrator reports about the same sessions, payments visible on-chain, and independent test traffic. Anyone may compute scores from the public records, and every score must be recomputable from them.
4. **Scores reach orchestrator selection gradually.** They begin in a shadow phase that measures whether they predict real outcomes, then become a filter, and only then a weight in selection.

We are asking for feedback on the design before anything is built. The questions we most want answered are collected in [§14](#14-questions-for-reviewers).

---

## 1. The problem

### 1.1 How data is collected today

The NaaP analytics platform ingests two Kafka topics. One is mirrored from Daydream's Confluent Cloud; the other is written directly by Cloud SPE gateways using a dedicated credential. Each source is identified in the ingest views by an organization name written into the SQL. Adding a third gateway operator requires the platform's operators to issue a credential, configure a mirror, and edit the ingest views.

Four consequences follow.

- **Participation requires permission.** A gateway operator who wants its traffic reflected in network metrics must be onboarded by whoever runs the pipeline.
- **Orchestrators are nearly invisible.** Every event arrives through a gateway. Of the ten stream-trace event types, only two originate on the orchestrator side, and the orchestrator's hardware and capabilities are known only because gateways relay what orchestrators declared about themselves.
- **Nothing is verifiable.** Events carry no signatures. A reader of the published metrics must trust both the gateway that reported the event and the pipeline that processed it.
- **The newest workload reports nothing.** Live Runner, which is expected to become the network's general job model, emits no Kafka events and no Prometheus metrics from the orchestrator. The Python gateway SDK, which is the main Live Runner client, emits no telemetry at all. The only record of a Live Runner session is the remote signer's ticket event.

### 1.2 How the data is used today

go-livepeer gateways can filter orchestrators by a performance score fetched from a URL given by `-orchPerfStatsUrl`. That URL serves one unsigned JSON document, refreshed every ten minutes, and the score it contains is used only as a pass-or-fail threshold. The network therefore already depends on a central scoring authority. This proposal aims to replace that dependency, not to add a second one.

### 1.3 What the design must achieve

We judge every design choice below against seven requirements:

| # | Requirement | What it rules out |
|---|---|---|
| R1 | Any legitimate operator can contribute data without anyone's approval. | Credential-based onboarding |
| R2 | Operators hold their own data and signing keys. | A shared store that operators write into |
| R3 | Every record is attributable to an operator's on-chain identity. | Anonymous or pipeline-assigned sources |
| R4 | Every claim can be checked against something its author does not control. | Scores built on self-reports alone |
| R5 | Anyone can aggregate the data and publish scores, and no aggregator is privileged. | A single canonical dashboard or score |
| R6 | End users and gateway customers are never exposed. | Publishing client IPs, prompts, or per-customer volume |
| R7 | Participation costs an operator close to nothing. | Designs that only large operators can afford to join |

R7 deserves emphasis. If publishing is difficult, only large operators will do it, and a selection rule that favors operators with published data would then quietly favor large operators.

---

## 2. The design in one picture

The design has three layers. Each answers a different question.

```
LAYER 1 — CHAIN (Arbitrum)          Who is a participant? Who paid whom?
  BondingManager       active orchestrators, stake, rewards
  ServiceRegistry      orchestrator service URIs (transcoding)
  AIServiceRegistry    orchestrator service URIs (AI)
  TicketBroker         gateway and signer deposits; winning-ticket redemptions
  RoundsManager        rounds, used as a shared clock

LAYER 2 — AT PROTOCOL REPOSITORIES  What does each operator claim?
  orchestrator repos   fleet, signed price quotes, service reports
  gateway repos        observations of orchestrators
  signer repos         ledger digests; observations relayed for keyless clients
  prober repos         results of test traffic
  scorer repos         published scores and flags

LAYER 3 — AGGREGATORS AND SCORERS   What is probably true?
  Anyone may read layers 1 and 2, check claims against each other,
  and publish scores. Gateways and signers choose which scorers to trust.
```

On-chain data is never copied into AT Protocol as a source of truth. Records in layer 2 refer to on-chain facts — addresses, rounds, transaction hashes — and aggregators in layer 3 join the two.

---

## 3. Why AT Protocol, and what it does not do

### 3.1 The parts we use

AT Protocol was built for social media, but its core is a general mechanism for publishing signed data that anyone can collect. We use five of its parts and none of its social features.

- **Decentralized identifiers (DIDs).** Each account is identified by a DID, whose document names the account's current signing key and the server hosting its data. We propose `did:web` identifiers derived from each orchestrator's own hostname, which avoids AT Protocol's central identity directory (see [§3.3](#33-where-at-protocol-falls-short)).
- **Repositories.** Each account owns one repository: a collection of JSON records organized as a Merkle tree, whose root is signed on every change. Anyone holding a copy can prove that a record was published by the account and has not been altered.
- **Personal Data Servers (PDSs).** A PDS hosts repositories and serves them to anyone who asks. Under this proposal, each orchestrator operator runs its own.
- **The sync protocol.** Every PDS exposes a stream of repository changes with cursors for resuming, and full exports for catching up. Any consumer can subscribe without permission. A *relay* is an optional service that combines many such streams into one; a relay restricted to recent data now costs about $30 per month to run.
- **Lexicons.** A Lexicon is a schema for a record type, named in reverse-domain form. We would define a `network.livepeer.*` family of schemas. Once published, a Lexicon cannot change incompatibly; a breaking change requires a new name.

AT Protocol also defines *labelers*: accounts that publish signed judgments about other accounts or records, which consumers choose to subscribe to. Our scorers follow that model.

### 3.2 Why not something else

| Alternative | Why we prefer AT Protocol |
|---|---|
| Keep Kafka and federate it | Kafka has no notion of per-producer identity or signed records. Someone must still issue credentials, which violates R1. |
| Publish telemetry on-chain | Even on Arbitrum, telemetry at five-minute granularity for hundreds of operators would cost far more than it is worth, and the data would be permanent and unredactable. |
| Build a custom signed log per operator | This would work, and it is essentially what AT Protocol already is. Building it ourselves would mean designing and maintaining a sync protocol, key rotation, backfill, and client libraries that AT Protocol already provides in Go, TypeScript, and Rust. |

Using AT Protocol is a choice of tooling, not a claim that the problem requires it. The deciding advantage is that the hard parts already exist and are in production use.

### 3.3 Where AT Protocol falls short

Four limitations shape the design.

- **Its default identity system is centralized.** Most accounts use `did:plc`, which depends on a single directory operated by Bluesky. We avoid it by using `did:web`, at the cost of tying each identity to a domain name.
- **Repositories have a size ceiling.** They are designed for up to single-digit millions of records. Operators must publish summaries, not individual events (see [§5.3](#53-summaries-not-events)).
- **Records can be deleted.** An operator can remove a record from its own repository. Accountability therefore depends on independent archives (see [§9](#9-keeping-history)).
- **All data is public.** AT Protocol's support for permissioned data is on its 2026 roadmap but has not shipped. Anything we publish, anyone can read.

---

## 4. Identity: from on-chain address to published records

### 4.1 Who is a participant

The participant list is derived entirely from chain state:

- **Orchestrators:** addresses active in `BondingManager` with a service URI in `ServiceRegistry` or `AIServiceRegistry`. The AI registry is a separate contract instance, so aggregators must read both.
- **Gateways and remote signers:** addresses that hold a deposit in `TicketBroker`.
- **Probers and scorers:** any DID. They have no on-chain role, so their records carry weight only to the extent that consumers choose to trust them.

### 4.2 Binding a DID to an Ethereum address

An orchestrator's DID is `did:web:` followed by the hostname in its registered service URI. Its PDS runs on that host or a subdomain of it.

To prove that the DID and the on-chain address belong to the same operator, the operator publishes a **binding record** containing the address, the DID, a validity window, and a signature over the DID made with the Ethereum key. Anyone can check that signature with the same recovery function go-livepeer already uses. No transaction and no governance action is required.

Two situations need an extension:

- **Operating keys.** Many orchestrators sign day-to-day messages with a key other than the address registered on-chain. The binding record must therefore support delegation: the registered address authorizes an operating address and a DID. The same gap causes attribution errors in the NaaP pipeline today, where an orchestrator's registered address and its operating address are sometimes treated as different orchestrators.
- **Keyless gateways.** A gateway that pays through a remote signer holds no Ethereum key; the signer's address is the on-chain sender. The signer publishes a **delegation record** naming the gateway DIDs that pay from its deposit.

Service URIs that contain a bare IP address instead of a hostname cannot yield a `did:web`. We have not yet measured how many exist (see [Appendix C](appendix-c-decisions-and-open-questions.md)).

### 4.3 Describing an orchestrator's fleet

A large orchestrator is not one machine. Transcoding operators typically run regional nodes behind a geographic DNS load balancer. For AI and Live Runner work, gateway operators currently list an orchestrator's individual nodes by hand with the `-orchAddr` flag. The on-chain `ServiceRegistry` holds one URI per address and cannot describe either arrangement.

We propose a signed **fleet record** in the orchestrator's repository, listing each node's URI, approximate location, roles (transcoding or Live Runner), applications, and capacity. A gateway would then resolve an orchestrator in five steps: on-chain address, service URI, DID, repository, fleet record. It would no longer need a hand-maintained node list.

This requires no contract change. go-livepeer already has a weaker version of the idea: `OrchestratorInfo` carries a `nodes` field listing other URIs that belong to the orchestrator, and discovery follows it one level deep. The fleet record is its signed and richer successor. If a contract change is later judged worthwhile, the registry would need to store only a pointer to it.

---

## 5. What each role publishes

### 5.1 Who can see what

Each role in a job sees something the others cannot. The design asks each role to publish what only it can observe.

| Role | What only it can see | What it publishes |
|---|---|---|
| **Orchestrator** | Work performed and payments received, across every gateway, every job type, and every runner in its pool | Binding, fleet, signed quotes, service reports |
| **go-livepeer gateway** | The client's experience: startup time, time to first frame, frame rate, orchestrator swaps, and demand that found no orchestrator | Observations of the orchestrators it used |
| **Python SDK client** | The same client experience, but the client holds no key and may be short-lived | Nothing directly by default; reports to its remote signer |
| **Remote signer** | Every payment it authorized, with application, orchestrator, billable seconds, and fee | Ledger digests; client observations it has checked against its own payments |
| **Prober** | Controlled test results that are nobody's self-report | Probe results |

Remote signers carry more weight in this design than their payment role suggests, because Live Runner clients mostly pay through them. There are few signers today — Livepeer Inc operates one, and Pymthouse and Cloud SPE may operate others — so a handful of signers would relay most client-side observations. Two measures limit that concentration. A signer's reports receive no special weight; they are one source among many, corroborated or contradicted by probes and by go-livepeer gateways that publish directly. And the SDK will allow any application to publish to its own repository instead.

### 5.2 One accounting hook covers every job type

Every job type in go-livepeer pays through the same two orchestrator functions: `ProcessPayment`, which credits incoming tickets, and `DebitFees`, which charges for work. Both are keyed by the payer and a per-session identifier. Only the billing unit differs:

| Job type | Billing unit |
|---|---|
| Transcoding | output pixels per segment |
| AI batch | pixels, audio milliseconds, tokens, or words, depending on the pipeline |
| Live video-to-video | pixel-seconds at a reference resolution and frame rate |
| Live Runner | seconds, 720p-pixel-seconds, or a fixed price per call |

Cumulative counters placed in those two functions would therefore give orchestrators a uniform service report for every job type, including job types moved onto Live Runner in future. Today the orchestrator keeps only a net balance per session, in memory, discarded after a timeout.

### 5.3 Summaries, not events

Repositories cannot hold every event, and they should not: raw events contain session identifiers and timing detail that serve no public purpose. Operators publish **five-minute summaries** instead: counts, sums, and histograms, grouped by orchestrator, application or pipeline, and model.

Summaries use additive quantities only. An aggregator can merge twelve five-minute summaries into an hour, or combine summaries from many gateways, without error. The NaaP canonical rules already require this; they forbid averaging averages.

Each summary carries a Merkle root of the raw events it summarizes. The raw events stay on the operator's machine for a published retention period. An auditor who doubts a summary can request the raw batch and check it against the root.

### 5.4 Different job types, one record shape

Job types measure different things, so records share a common core and add one extension per job family.

- **Core:** billing units, fees, session outcomes, and payment references.
- **Live video-to-video extension:** startup success, time to first frame, output frame rate, and orchestrator swaps. These come from the events go-livepeer gateways already emit.
- **Live Runner extension:** reservations, payment challenges, time to first byte, capacity rejections, and why each session ended.

Live Runner will probably absorb live video-to-video and transcoding over time, but no timeline exists. When a job type moves, its records carry both extensions and no consumer breaks.

The orchestrator can measure Live Runner work without any cooperation from applications. It sits in the path of every client request as a reverse proxy, so it can record status codes, latency, bytes transferred, session length, and the reason each session ended. The application's content remains opaque, which also serves R6.

Draft schemas for every record type are in [Appendix A](appendix-a-record-schemas.md).

---

## 6. Trust: from attributable claims to probable truth

A signed record proves who made a claim. It does not prove the claim. An orchestrator can overstate its work; a gateway can understate a competitor's quality. The design therefore gives self-reports no weight on their own. Four kinds of evidence give them weight.

### 6.1 Two-sided reports

A gateway's observation of an orchestrator and that orchestrator's service report describe the same sessions from opposite ends. When they agree, each corroborates the other. When they disagree, the disagreement is itself a signal worth publishing.

To match the two sides, we propose joining on `recipient_rand_hash`, a value in the ticket parameters that both parties hold. It changes with every batch of ticket parameters and reveals nothing about content. It is a better join key than a request identifier, which only some job types have, or timestamps, which differ between machines.

### 6.2 Signed quotes

Today, when a gateway asks an orchestrator for its terms, the reply — price, capabilities, hardware, ticket parameters — carries no signature. It includes two authenticators, but both are keyed by a secret only the orchestrator holds, so no third party can check them. A gateway cannot later prove what an orchestrator offered.

We propose that orchestrators sign their quotes. The change adds one field to the `OrchestratorInfo` message, which has unused field numbers available, and old gateways ignore it. The signature uses EIP-712 typed data, which go-livepeer already supports and which a contract could verify if that is ever useful. A quote commits the orchestrator to its address, the requesting gateway's address, prices per capability and model, hashes of its capabilities and hardware, its ticket parameters, and a validity window.

The same field also covers the payment challenge in Live Runner, which carries an `OrchestratorInfo`. The Live Runner discovery response, which lists runners and their prices, needs a signed envelope of its own.

Signed quotes do not make hardware claims true. They make them attributable: an orchestrator that advertises an H100 and performs like a T4 can be shown to have done so.

A related change is small. For transcoding and AI batch jobs, orchestrators already sign the hashes of the output they return, but we found no gateway code that verifies those signatures. Verifying them makes delivery of those jobs provable.

### 6.3 Payments on-chain

A gateway that reports many sessions with an orchestrator should have paid that orchestrator accordingly. Each ticket's expected value is its face value multiplied by its win probability, and winning tickets are redeemed on-chain in public. Redemptions are a random sample of all tickets sent, so an aggregator can estimate, with a confidence interval, how much business two parties really did. Payment is the one signal that costs real money to fake.

### 6.4 Test traffic

Probers send test jobs and publish the results. Their reports are the only data in the system that is nobody's self-report. Two probers exist today: Livepeer Inc tests transcoding with its stream tester, and Cloud SPE, funded by the treasury, tests live video-to-video and Live Runner jobs. Each would publish under its own DID.

Probes have a weakness: an orchestrator that recognizes a prober's paying address can treat its traffic specially. Probers should rotate their deposits and addresses and send traffic indistinguishable from real use. Over time, observations of real traffic should carry more weight than probe results; probes are best at confirming that an orchestrator is alive and really offers what it advertises.

### 6.5 Scores must be recomputable

A scorer publishes each score together with the name and version of its formula and a reference to the exact records it used. Anyone can then recompute the score. This removes a conflict of interest that would otherwise be hard to avoid: an organization such as Cloud SPE may run probes, gateways, an aggregator, and a scorer at once. With recomputable scores, it does not need to be trusted, only checked.

### 6.6 Self-dealing

Payment evidence has a blind spot. An operator that runs both a gateway and an orchestrator can pay itself, recover the payment minus redemption costs, and publish favorable reports on both sides. The same operator can also publish unfavorable reports about competing orchestrators.

Nothing in this design eliminates self-dealing. We propose to detect it as well as possible and to reduce its effect. Detection belongs to labelers — accounts that publish signed flags — not to the protocol. Candidate signals, from cheapest to most involved:

1. **Concentrated revenue.** Most of an orchestrator's payments come from one or two senders.
2. **Traced funding.** A gateway's ticket deposit was funded from wallets that received the orchestrator's fees or rewards. This needs only on-chain data and is the strongest signal.
3. **Shared infrastructure.** A gateway and an orchestrator share an IP address, network operator, TLS certificate, or DID host. Signed fleet records make this visible.
4. **Irrational prices.** A sender consistently pays more than the market.
5. **Uncorroborated quality.** An orchestrator's good results come only from gateways that probes and independent gateways do not corroborate.

Flagged relationships have their observations down-weighted, not discarded. Diverse gateways and probers mitigate the problem; they do not solve it.

---

## 7. From scores to orchestrator selection

### 7.1 Three phases

Scores must earn their influence on selection. We propose three phases, each entered only when the previous phase's criteria are met.

| Phase | What changes | Criteria for moving on |
|---|---|---|
| **A. Shadow** | Scorers publish scores. Gateways record the score available at selection time next to the outcome that followed. Selection does not change. | Scores predict outcomes: rank correlation between an orchestrator's score and its next hour's success and latency; calibration by region and pipeline; coverage of orchestrators and sessions. |
| **B. Filter** | Gateways accept a list of trusted scorer DIDs, verify each score's signature, take the median across sources, and apply it as today's minimum-score filter. | No regression in gateways' own success rates; no sudden concentration of traffic; stable scores. |
| **C. Weight** | Score becomes a weighted term in selection probability, alongside stake, price, and randomness. | Default weights approved through governance. |

Two safeguards apply from the start. A random share of selection must remain, so that new orchestrators without scores still receive traffic and can build a record. And scores should not become weights until several independent scorers exist, so that no single scorer controls selection.

### 7.2 Where scores enter selection

Selection happens in different places for different job types.

- **Live video-to-video through go-livepeer gateways** uses the gateway's selection algorithm. There, the single-URL `-orchPerfStatsUrl` would become a list of trusted scorer DIDs.
- **Live Runner** bypasses that algorithm entirely. Clients choose from a discovery list, and the Python SDK currently takes the first candidate. The remote signer's discovery endpoint already collects every orchestrator's runner list and filters it by price, which makes it the natural place to rank candidates by score for all the clients it serves. The SDK should also accept a ranked list.

Each gateway and each signer chooses its own scorers. There is no network-wide ranking.

### 7.3 Proving the new pipeline counts correctly

Before any score influences selection, we need evidence that the new pipeline measures traffic correctly. Phase A provides that evidence by running the new pipeline beside the existing one on the same traffic.

During Phase A, Cloud SPE gateways keep sending raw events to Kafka, where the NaaP pipeline turns them into metrics as it does today. The same gateways also publish five-minute summaries to their own repositories, and a separate aggregator computes the same metrics from those summaries alone. One set of real traffic therefore produces two sets of numbers:

```
                  ┌─► Kafka ─► NaaP pipeline ─────────────────────► metrics  (existing path)
Cloud SPE gateway ┤
                  └─► summaries in its repository ─► new aggregator ─► metrics  (new path)
```

If the new path is built correctly, the two sets agree for every hour and every orchestrator. The table below shows what a comparison for one orchestrator, one pipeline, and one hour might look like. The numbers are invented for illustration.

| Measurement | Existing path (NaaP) | New path | Tolerance | Result |
|---|---|---|---|---|
| Streams that started successfully | 96.2% | 96.0% | ±1 percentage point | Pass |
| Streams moved to another orchestrator (swap rate) | 3.1% | 3.1% | ±1 percentage point | Pass |
| Streams stuck loading or at zero frames per second (output viability) | 1.4% | 1.5% | ±1 percentage point | Pass |
| Time to first frame, median stream | 1,850 ms | 1,900 ms | ±5% | Pass |
| Time to first frame, 90th-percentile stream | 4,200 ms | 5,100 ms | ±5% | **Fail** |
| Average output frame rate | 17.8 fps | 17.8 fps | ±5% | Pass |
| Status updates received as a share of those expected (coverage) | 92% | 91% | ±2 percentage points | Pass |

A tolerance is the disagreement we accept as noise. The two paths will never match exactly: the Kafka producer drops events when its buffer fills, and summaries round values into histogram buckets. A result outside tolerance, such as the failing row above, points to a real defect. In that example, the five-minute histograms may be too coarse to recover the slowest streams' latency, or the publishing sidecar may be losing events. Either must be fixed before anything is built on the summaries.

All the measurements use the definitions in NaaP's canonical rules, which are the only written and production-tested definitions available. End-to-end latency and jitter are left out because NaaP has itself withdrawn them.

**Phase A compares measurements, not scores.** NaaP also turns these measurements into an SLA score between 0 and 100 for each orchestrator. That score comes from a formula whose weights are choices: 70% reliability and 30% quality, each benchmarked against the network's previous seven days. The two kinds of number call for different treatment:

- **A measurement is a fact** about what happened, counted or timed. Any two honest pipelines should agree on it, so agreement is a fair test of the new pipeline.
- **A score is a judgment** about what those facts mean. Requiring every scorer to reproduce NaaP's score would make NaaP's formula the official one, reintroducing a central authority by another route.

NaaP's formula therefore becomes one published option, `sla-v1`, under Cloud SPE's scorer identity. Anyone may recompute it, adopt it, or publish a competing formula.

---

## 8. Regions

The current network has no clear definition of a region. The existing performance score is keyed by region names, but orchestrators are not assigned to regions by any authority, and a fleet behind a geographic load balancer spans several.

We propose to define region relative to the gateway doing the selecting. Selection needs to know how well an orchestrator serves *this* gateway, not which continent it sits on. Every observation and probe result records the measured round-trip time and the observer's coarse location. Scorers then publish scores for each pair of orchestrator and observer cluster, rather than for named regions.

Orchestrators may still declare a location for each node in their fleet record. The declaration cannot be verified, but it can be compared with measured latency, and a mismatch is attributable.

---

## 9. Keeping history

Preserved history matters for two reasons. Scores need past data, and disputes need evidence. Because an operator can delete records from its own repository, preservation cannot depend on the operator. We propose three tiers:

| Tier | Contents | Operated by | Retention |
|---|---|---|---|
| **Live** | Each operator's repository | The operator | Detailed records until archived, then pruned |
| **Mirror** | Complete copies kept current from each repository's change stream | Anyone: SPEs, the Foundation, scorers | Indefinite |
| **Cold** | One export of every repository per round, stored by content hash; optionally the export's hash anchored on-chain | Treasury-funded | Indefinite |

Mirrors need not be trusted. Every repository change is signed, so a copy held by anyone can be verified against the operator's key. Several independent mirrors mean no single party holds the history.

Retention differs by kind of data. Summaries and Merkle roots are kept indefinitely. Raw events stay with operators for a period the specification must fix, and audits must happen within it.

Size requires one further step. Five-minute summaries across several applications amount to roughly half a million records per orchestrator per year, which would reach AT Protocol's repository ceiling within a few years. Operators will therefore publish at five-minute granularity, compact to hourly and daily summaries, and prune the detail once mirrors hold it. Pruning is safe only because the mirrors exist.

---

## 10. Privacy

Operators are public by design. Their addresses, endpoints, prices, and performance are exactly what this proposal exists to publish. The people and businesses behind their traffic are not, and three kinds of data in today's events would expose them:

- **Client IP addresses,** carried today in the gateway's payment event. Under the GDPR an IP address is personal data, and a public, mirrored archive cannot honor a deletion request.
- **Prompts and parameters,** carried today in live video-to-video parameter-update events. These are end users' content.
- **Application and customer identifiers,** held by the remote signer's clearinghouse. They reveal which businesses use which orchestrators, and at what volume.

The rule is to publish at the level of the operator and never at the level of an end user or customer. The schemas enforce it structurally: they have no fields that could hold these values, and identifiers that must link records are salted hashes.

---

## 11. Incentives and adoption

Publishing is not mandated. Instead, gateways and signers may treat the absence of data as a reason to prefer other orchestrators. We think this is the right incentive, with two qualifications.

- **Absence means lower confidence, not exclusion.** Gateways observe every orchestrator they use, whether or not it publishes. An orchestrator's own reports corroborate those observations; they are not the basis of its score. A new orchestrator with no history must still receive exploratory traffic.
- **The first beneficiary of an orchestrator's telemetry is the orchestrator.** Orchestrators are accountable for every runner in their pools, including remote runners contributed by others, and they need a way to remove members that perform poorly. The per-runner measurements described in [§5.4](#54-different-job-types-one-record-shape) serve that purpose locally, whether or not anything is published. Publishing then costs one flag.

To meet R7, publishing must ship in go-livepeer, be enabled with one flag, and run with an embedded or sidecar PDS that needs no separate operation.

Pool runners currently authenticate with a secret shared by the whole pool, so an orchestrator cannot distinguish one runner from another across restarts. Giving each runner its own registration credential would help pool management even without this proposal.

---

## 12. What this proposal does not do

- **It does not affect stake or slashing.** The data informs selection, explorers, and delegators. Once data can cost an operator stake, the incentive to game it rises sharply. We propose to record this as an explicit non-goal in the governing LIP.
- **It does not make claims trustless.** It makes them attributable and checkable, and relies on corroboration to separate the probable from the doubtful.
- **It does not publish raw events.** Raw events stay with operators, committed to by Merkle roots.
- **It does not replace chain state.** The chain remains the authority on who participates and who paid whom.
- **It does not cover transcoding in the first pilot.** Transcoding follows once the pilot's two job families work, with Livepeer Inc's stream tester as its first prober.

---

## 13. Governance and rollout

**Schemas.** The `network.livepeer.*` Lexicons, their versioning rules, and the non-goals in [§12](#12-what-this-proposal-does-not-do) should be adopted through a LIP. Because a published Lexicon cannot change incompatibly, the versioning process must exist before the first schema is published.

**Pilot scope.** Live video-to-video and Live Runner. Live video-to-video has well-defined metrics and an existing NaaP baseline to check against. Live Runner is expected to become the network's general job model and currently has no telemetry, so instrumenting it now means later migrations inherit the work.

**Pilot participants.** Cloud SPE gateways, publishing alongside Kafka; Cloud SPE as the first prober and scorer; Livepeer Inc as a second prober and scorer, publishing in place of its current HTTP API; and a small number of volunteer orchestrators.

**Upstream changes.** Every go-livepeer change in [Appendix B](appendix-b-change-list.md) must be accepted into upstream go-livepeer. Several are small and useful on their own: verifying output signatures, per-runner credentials, and cumulative accounting counters. We suggest proposing those first.

---

## 14. Questions for reviewers

We would value feedback on any part of the design, and especially on these questions:

1. **Identity.** Is `did:web` on the orchestrator's service hostname acceptable to operators? How many service URIs use bare IP addresses?
2. **Signed quotes.** Are orchestrators willing to sign their prices and hardware claims, knowing that the signatures make misrepresentation attributable?
3. **The accounting hook.** Are cumulative counters in `ProcessPayment` and `DebitFees` acceptable to go-livepeer maintainers, given the extra state they require?
4. **Remote signers.** Would signer operators accept the role of relaying and checking observations from the clients they serve?
5. **Summaries.** Is a five-minute window right? Which histogram boundaries would make the summaries useful to you?
6. **Self-dealing.** Which detection signals would you trust, and which would produce too many false positives among legitimate operators who run both gateways and orchestrators?
7. **Phase criteria.** What evidence would persuade you that scores are ready to filter, and later to weight, orchestrator selection?
8. **History.** Who should operate mirrors and cold storage, and should cold exports be anchored on-chain?
9. **Fleet record.** Does the proposed fleet record describe how you actually run your nodes?

Decisions already taken, and the questions still open, are listed in [Appendix C](appendix-c-decisions-and-open-questions.md).
