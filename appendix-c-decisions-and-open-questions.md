# Appendix C: Decisions and open questions

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). This appendix records what the authors have already decided, so reviewers can focus their feedback on what remains open.*

*Revised 2026-11-03 for livepeer-node, Clearinghouse Batteries and Livepeer Builder Core.*
- *Decisions D1 to D17 keep their numbers. Those marked "revised" changed with the new stack.*
- *D18 onward are new. Several come from the first draft of the [implementation plan](implementation-plan.md).*

## C.1 Decisions taken

| # | Decision | Reason |
|---|---|---|
| D1 | *Revised.* Identities use `did:plc` by default, with `did:web` as an option. The Ethereum-signed binding, naming the repository's signing key, is the root of trust. A repository can be hosted by the participant's telemetry agent, by a community PDS, or by the participant's own PDS. | AT Protocol allows `did:web` only on port 443, and many orchestrators serve on other ports or bare IP addresses (R7). The binding limits the PLC directory to availability. The participant keeps its keys under every hosting mode (R2). |
| D2 | *Revised.* Orchestrators sign each runner record in their discovery responses, and each payment challenge, with an operating key their binding lists. | Makes price, capacity and hardware claims attributable. Per-record signatures survive discovery services, which merge runners from many sources. |
| D3 | Scores begin in a shadow phase, then filter selection, then weight it. | Scores must be shown to predict outcomes before they affect traffic. |
| D4 | Publishing is not mandated. Applications and signers may treat absent data as lower confidence. | Keeps participation voluntary while rewarding it. |
| D5 | The operator is always accountable for its runners, including remote pool members. | Operators need per-runner measurements to remove poor pool members. That is also their first reason to run the telemetry. |
| D6 | History is preserved in three tiers: participants' repositories, independent mirrors, and cold per-round exports. | Participants can delete their own records; accountability needs copies they do not control. |
| D7 | *Revised.* Regions are defined relative to the observing application or prober, using measured latency. | The network has no authoritative region assignment. Selection needs performance from the observer's own position. |
| D8 | Publish at the operator level, never at the end-user or customer level. | Operators are public; their customers and users are not. Public archives cannot honor deletion requests. |
| D9 | Self-dealing is flagged by labelers and down-weighted, not ignored. Participants may declare affiliations, which are kept out of independent corroboration but not penalized. | It cannot be eliminated. Diverse applications and probes mitigate it, and declaration rewards honesty. |
| D10 | *Revised.* The `network.livepeer.*` schemas are governed through a LIP. The pilot publishes under an experimental namespace controlled by Cloud SPE, and migrates at adoption. | Published Lexicons cannot change incompatibly, so versioning must be agreed before anything is published under the final names. |
| D11 | *Revised.* The pilot covers Live Runner work on livepeer-node, by applications built on Builder Core. Its participants are ChatLPT and inference-gateway, the gateways that pay for them, Cloud SPE, volunteer orchestrators, and Livepeer Inc's Livepeer Agent once the system is built. | The Network Engineering SPE is onboarding applications of this kind, and each of them curates orchestrators by hand today. |
| D12 | *Revised.* Phase A compares measurements from published records with each participant's own records: the Batteries ledger, Builder Core's attempts and costs, livepeer-node's metrics, and the chain. Cloud SPE's first formula, `lr-v1`, has no special standing. | Measurements should agree between honest pipelines. Formulas are policy and must be allowed to compete. NaaP's baseline was built on go-livepeer gateway events, which the new stack does not produce. |
| D13 | *Revised.* The clients in scope are Builder Core applications. Applications that use the Python SDK directly can read the same attested discovery view. | Builder Core is the engine the onboarded applications use, and it records the client's view. |
| D14 | *Revised.* go-livepeer, and so transcoding, is outside this proposal's scope. | go-livepeer is deprecated. Work that still runs on it would need the go-livepeer design in this repository's history. |
| D15 | Host abuse is detected and recorded, not prevented. Applications that handle sensitive data use only orchestrators they have vetted until attestation is affordable. | No published record can prove an orchestrator left an application's secrets alone. Test traffic can catch some abuse, and labels make each catch follow the orchestrator to every consumer. |
| D16 | *Revised.* Vetting is published as `approved` labels on an orchestrator's offering of one work key and version, not kept in private lists. | Applications can share evidence without sharing a list, and no one depends on a single party's view. |
| D17 | Host-level scanning on orchestrator machines is not proposed. | The operator controls the machine being scanned, so a scan's report is another self-report. |
| D18 | Each application publishes under its own identity. A gateway run by a different party publishes under its own. A party that plays two roles publishes both from one repository. | Each party is accountable for its own claims, and the relationship between two roles of one party stays visible. |
| D19 | A gateway is a remote signer together with the Ethereum address that holds its TicketBroker deposit and reserve. That address is its identity. | It is the only on-chain anchor a payer has, and every ticket it signs names it. |
| D20 | Publishing is optional for each application. Every publisher chooses a publication mode, `full`, `delayed` or `coarse`, and declares it in a signed policy record. | An application's volume with each orchestrator is commercial information (R8). Corroboration works in every mode. |
| D21 | Every publisher runs the same telemetry agent: one Go binary, separate from the software it reads, and outside every request path. | One artifact for every role keeps participation cheap (R7), and stopping the agent stops publication and nothing else. |
| D22 | One record per publisher per window, carrying an array of rows, under a key derived from the window. | Keeps every repository far below its size ceiling, and makes a republish replace the record instead of duplicating it. |
| D23 | Records are matched on the recipient rand hash, published as 8-byte prefixes. A row lists at most 2,000, with a Bloom filter for the rest. Each publisher chooses five-minute or per-round windows. | No single party chooses the rand hash, all three parties hold it, and redemptions reveal it on-chain. Prefixes cut its size by three quarters, and collisions are negligible. |
| D24 | Phase A is split. A0 needs no upstream change; A1 adds the changes to livepeer-node and Batteries. | Evidence can begin before upstream review, which no one in the pilot controls. |
| D25 | Scores and labels reach selection through an attested discovery view in livepeer-node's `/discovery` format, and through a Builder Core selection policy plugin. | Builder Core, the remote signer and the discovery service all accept that format already, so none of them changes. |
| D26 | The design assumes Builder Core's milestones M1 to M5 are complete, including its events feed. | They will be complete before this work begins. |
| D27 | Cloud SPE is the first prober, built as an ordinary Builder Core application, and pays through a gateway that real applications also use. | It measures what an application would get, and its payments do not single out its probes. |
| D28 | From Phase B, five safety rails apply: fail open, a candidate floor, an exploration share that is never zero, a cap on how fast a score can move, and a quorum of independent scorers. | Scores must not be able to concentrate traffic, starve new orchestrators, or let one scorer control selection. |
| D29 | Labels about an orchestrator and a work key together attach to an `orch.offering` record, one per work key the orchestrator offers. | AT Protocol labels attach to a DID or a record. Approvals and capabilities concern a work key, not the whole orchestrator. |
| D30 | Cloud SPE runs a relay scoped to Livepeer participants from Phase A0. A second operator runs another by the end of Phase A. | Every further aggregator would otherwise subscribe to every repository host. |
| D31 | Every summary commits to a Merkle root over its already-redacted raw events. Serving raw batches is optional until Phase B. | Keeps audits possible without making every participant run an audit service from day one. |
| D32 | From A1, an application proves which sessions it paid for with a client tag that Batteries derives from a per-key secret. Gateways publish only a hash of each tag. | Rand hash prefixes are public, so without tags anyone could copy a gateway's sessions into an observation and misreport them. |
| D33 | Verification is per commit. A record counts if the commit it arrived in is signed by a key that a binding valid at that commit's time names. A commit's time is when an independent relay or mirror first saw it. | AT Protocol signs commits, not records. Checking the binding at the record's window would reject a repository's whole history after a key rotation. |
| D34 | This design's mirrors keep records after their authors delete them, and the LIP says so. | Accountability is the mirrors' purpose. Network mirrors such as Hubble honor deletion, so they cannot serve it. Records hold no personal data, which makes keeping them acceptable. |

## C.2 Context the decisions rest on

- **The stack.** go-livepeer is deprecated. Its replacements are:
  - livepeer-node: the orchestrator, the remote signer, the discovery service and the chain tool;
  - Clearinghouse Batteries;
  - the Python gateway SDK.

  Live Runners do the work on GPU servers. Orchestrators advertise them, proxy traffic to them, and get paid.
- **Builder Core and its applications.** Builder Core is the engine that enterprise applications build on. The Network Engineering SPE's sample applications, ChatLPT and inference-gateway, are the first. The SPE plans to onboard more.
- **Curation today.**
  - ChatLPT keeps a hand-picked list of five orchestrators. It found that three of them use self-signed certificates, which Builder Core cannot verify.
  - inference-gateway ran 34 test operations on 26 October 2026 to establish which orchestrators and features work for its models, and committed the results to its repository.
- **Telemetry today.**
  - livepeer-node gained detailed operational metrics per runner app and runner in October 2026. They have no payer dimension and no per-session records.
  - The signer reports every signing to its operator's Batteries.
  - Builder Core records every attempt and its cost in the application's database.
- **Gateways.** An application may run its own gateway, as ChatLPT does, or pay through someone else's.
- **Probers.** Cloud SPE, funded by the treasury, sends test jobs. Livepeer Inc's Livepeer Agent is expected to join once the system is built.
- **Registries.** livepeer-node's chain tool writes the protocol `ServiceRegistry`. Builder Core's on-chain directory reads the `AIServiceRegistry` by default.
- **Runner identity.** livepeer-node lets each runner choose its own ID, and keeps runner credentials in memory only.

## C.3 Open questions

### Scope and participation

1. Should an SPE that onboards applications run a shared approval labeler, or should each application publish only its own approvals? A shared labeler saves duplicated vetting. It also makes the SPE a likely default gatekeeper.
2. What would persuade an application to publish its observations, and in which mode?
3. What role will the Livepeer Agent play: application, prober, or both?
4. Does any go-livepeer work, such as transcoding, need coverage during the transition, or can it wait for its own migration?
5. Should NaaP's analytics pipeline ingest the new records, so that its dashboards continue for Live Runner work?

### Identity and inventory

6. Is the rotation and compromise procedure in the implementation plan ([§4.3](implementation-plan.md#43-rotation-and-compromise)) sufficient, particularly when an operating key is compromised but the registered key is not?
7. Which service registry should be authoritative for an orchestrator's service URI, given that livepeer-node and Builder Core use different ones?
8. Should the inventory record eventually be referenced from a contract, and if so, through what governance path?
9. Does the agent's minimal repository host pass conformance tests against the indigo relay and Tap? This must be tested before any operator depends on it.

### Data

10. Are livepeer-node's histogram buckets, adopted as `latency-ms-v1`, dense enough between 250 ms and 2.5 s? If not, should the session outbox carry each single-shot call's header latency, so that a denser bound set can be built without changing the metrics?
11. How long must participants retain raw events for audit?
12. Is the clock policy proposed in the implementation plan ([§5.4](implementation-plan.md#54-clock-policy)) acceptable: refuse to publish beyond a two-second clock offset, and match a rand hash anywhere within its session's span?
13. Is a gateway's coarsened count of paying API keys safe to publish, or should it be dropped?

### Trust

14. What thresholds should the self-dealing signals use, given that some legitimate operators run several roles?
15. How should consumers weigh the observations of an application that runs its own gateway, whose observations and ledger digests are not independent?
16. How many independent scorers must exist before Phase C, and who decides that the number has been reached?
17. If many applications follow the same labels, are the proposed exploration share and candidate floor enough to prevent traffic from concentrating?

### Operations and funding

18. Who operates mirrors and the cold archive, and how are they funded?
19. Should cold exports be anchored on-chain each round?
20. Which shared gateway should probers pay through, and how should their deposits be funded and rotated?
21. Who controls the `livepeer.network` domain, through which Lexicon resolution runs?

### Upstream

22. Will livepeer-node's maintainers accept the session outbox, signed runner records and signed challenges, and in what form?
23. Will Batteries' maintainers accept the rand hash and client tag on usage rows, and can Batteries issue the agent a read-only Kafka credential?

### Workload integrity

24. Who runs canary monitoring, and in what form should evidence of use be published so that others can check it?
25. How can canary secrets be made indistinguishable from real application secrets, given that an operator who recognizes them can leave them alone?
26. Which TLS and DNS checks can a probe run without being recognizable as a probe?
27. Should a runner's version be an image digest, a signed release, or both?
28. Which attestation technologies are affordable enough to support, and when should `attested` labels begin to affect selection?
29. How should an application or SPE that trusts several labelers combine their approval and exclusion labels into one decision?
30. TLS and DNS checks need a probe runner app on the orchestrator's hosts. Will orchestrators host one, and is a check that the orchestrator knows about worth running?

## C.4 What happened to the first draft's questions

| First draft | Now |
|---|---|
| Q1: bare IP addresses and `did:web` | Answered by D1 |
| Q2: key rotation and compromise | Question 6 |
| Q3: referencing the fleet record from a contract | Question 8 |
| Q4: hosting a single `did:web` account on the reference PDS | Replaced by question 9, because repositories need not run on the reference PDS |
| Q5: summary window and histogram bounds | Five-minute or per-round windows (D23); bounds in question 10 |
| Q6: raw-event retention and redaction | Redaction answered by D31; retention in question 11 |
| Q7: clock differences | Question 12 |
| Q8: salting a signer's relayed client hash | Withdrawn. Signers no longer relay client observations. |
| Q9: self-dealing thresholds | Question 14 |
| Q10: relayed versus direct observations | Replaced by question 15 |
| Q11: independent scorers before Phase C | Question 16 |
| Q12 to Q14: mirrors, anchoring, prober deposits | Questions 18 to 20 |
| Q15: go-livepeer maintainers | Replaced by questions 22 and 23 |
| Q16: embedded PDS in go-livepeer | Withdrawn. The agent is a separate component (D21). |
| Q17 to Q22: workload integrity | Questions 24 to 29 |
