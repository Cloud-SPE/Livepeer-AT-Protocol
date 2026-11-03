# Appendix A: Draft record schemas

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). These drafts are for review. None is published, and every name may change before the governing LIP adopts them.*

*Revised 2026-11-03 for livepeer-node, Clearinghouse Batteries and Livepeer Builder Core. The go-livepeer version remains in git history.*

## A.1 What changed in this revision

| Previous draft | This draft | Why |
|---|---|---|
| `operator.binding` with DID, address and signature | Adds the repository key hash, the role, a nonce and a validity of at most a year | Under `did:plc`, the Ethereum binding is the root of trust ([§A.5](#a5-networklivepeeroperatorbinding)) |
| `operator.delegation` | Removed | Applications publish under their own identity. A gateway never has to name the applications it serves. |
| `orch.fleet` | `orch.inventory` | Runners and their prices come from livepeer-node's `/discovery` |
| `orch.quote` | Removed | Prices are part of the inventory. Offers made to one client are signed and referenced by hash. |
| — | `orch.offering` | Labels need a record to attach to when they concern an orchestrator and a work key together |
| `gateway.observation` | `client.observation` | The client is a Builder Core application, not a go-livepeer gateway |
| `signer.ledgerDigest` | `gateway.ledgerDigest` | A gateway is a remote signer together with the address that holds its deposit |
| — | `client.demand`, `operator.policy`, `operator.affiliation` | Unmet demand, commercial confidentiality (R8), and declared relationships |
| One record per counterparty, work and window | One record per publisher per window, with an array of rows | The repository ceiling |
| `jobFamily` with four families | Removed. The work key is the runner app, model and version | The new stack has one job family, the Live Runner |
| Full rand hashes, at most 1,000 | 8-byte prefixes, at most 2,000 per row, with a Bloom filter for the rest | Size, and the decision recorded in [Appendix C](appendix-c-decisions-and-open-questions.md) |
| Live video-to-video measures and NaaP's excused errors | Removed | Out of scope ([proposal §12](README.md#12-what-this-proposal-does-not-do)) |

## A.2 Conventions

### Data types

1. **No floating-point numbers.** The AT Protocol data model forbids floats, because they do not re-encode identically across machines and would break signatures. Ratios are in basis points (1 = 0.01%), durations in milliseconds or seconds as each field names.
2. **Amounts are decimal strings.** Wei amounts can exceed a 64-bit integer. Prices copied from discovery keep their own currency and unit.
3. **Ethereum addresses are lowercase hex strings** with the `0x` prefix, 42 characters long.
4. **Times are UTC**, in the AT Protocol `datetime` format.

### Privacy

5. **Nothing identifies an end user, a customer or an API key.** No schema has a field for a client IP address, prompt, request body, path, header, job ID, manifest ID, allocation or customer.
6. **No free text.** Every string is one of these: a known value, an address, a DID, a hash, a timestamp, a URL or runner ID that livepeer-node discovery already publishes, or a runner app name or version as discovery publishes it. The telemetry agent checks every outbound record against its Lexicon and against this rule, so a configuration mistake cannot leak private data into a field.

### Summary records

7. **Summaries are additive.** Every numeric field is a count, a sum, or a histogram of counts. An aggregator can merge windows and publishers without error, and consumers compute rates from counts.
8. **One record per publisher per window.**
   - Each record carries an array of rows. The record key is derived from the window: `<windowStart>.<durationSec>`, such as `20261104T1405Z.300`, or `round-<n>` for a round window.
   - When late events arrive, the agent republishes under the same key, replacing the record. The repository keeps only the latest version, but the change stream carries every version, so mirrors keep them all.
   - A record is final 24 hours after its window ends, or once its delay has passed in `delayed` mode. A later change is an amendment: allowed, visible to every reader, and counted as such by scorers.
   - A record must stay under 1 MB. One that would exceed 800 KB is split into pages, with keys ending `.p2`, `.p3` and so on.
9. **Events are placed by the time they were recorded.** A call or attempt belongs to the window in which it completed, a ticket to the window in which it was signed, and a session to the window in which it ended. A long session therefore appears in several of a gateway's windows but in only one of the orchestrator's. Aggregators match rand hashes across a session's whole span, not window by window.
10. **Every summary carries a policy stamp**: the publication mode it was produced under and when it was published ([§A.6](#a6-networklivepeeroperatorpolicy)).
11. **Every summary carries a commitment** to the redacted raw events it was computed from. The events stay with the publisher.

### Payment references

12. **Rows that describe paid sessions carry rand hash prefixes.**
    - The recipient rand hash is the 32-byte value in the ticket parameters of each payment challenge. livepeer-node draws a new one for every challenge, so it identifies one payment session.
    - A prefix is its first 8 bytes. A row lists its prefixes in ascending order, concatenated into one `bytes` field, up to 2,000 of them.
    - A row with more sessions lists the lowest 2,000 and puts every prefix into a Bloom filter. Choosing the lowest keeps the list deterministic. It is not a guaranteed random sample, because orchestrators choose the secrets behind the hashes.
    - From A1, rows also carry a client tag, or its hash, for each listed session ([proposal §6.1](README.md#61-three-sided-reports)). Applications and probers publish the 16-byte tags; gateways publish the first 8 bytes of each tag's SHA-256 hash. An application can therefore claim only sessions it paid for.
    - Collisions are negligible. At a hundred million sessions a year, the chance of any two 8-byte prefixes colliding is about 0.03%, and matching is already narrowed to one gateway, one orchestrator and adjacent windows.

### Histograms

13. **Histograms name a bound set** instead of carrying their own edges. Bound sets are fixed per schema version, so histograms from different publishers can be summed.

| Bound set | Unit | Upper edges | Chosen to match |
|---|---|---|---|
| `latency-ms-v1` | ms | 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000, 30000, 60000 | livepeer-node's default histogram buckets, so orchestrator rows built from its metrics are exact |
| `duration-s-v1` | s | 1, 5, 15, 30, 60, 300, 900, 3600, 10800 | livepeer-node's `session_duration_seconds` and `app_call_duration_seconds` buckets |

`latency-ms-v1` is coarse between 250 ms and 2.5 s, where first-token latency for language models usually falls. A denser version 2 would need livepeer-node's buckets to change as well. Alternatively, it could be built from the session outbox, which records each single-shot call exactly. Phase A's parity test ([proposal §7.3](README.md#73-proving-the-new-pipeline-counts-correctly)) will show whether version 1 is good enough.

### Namespace

14. **Names.** The schemas are written under their intended `network.livepeer` names. The pilot publishes them under an experimental authority controlled by Cloud SPE, and migrates when the LIP is adopted. A published Lexicon cannot change incompatibly, so only optional fields may be added later; a breaking change needs a new name.
    - Constraints such as maximum lengths are fixed for the life of a name too, so the limits below are set with headroom.
    - Closed `enum` lists are avoided for the same reason. Open `knownValues` lists can grow.

The Lexicon JSON below is abbreviated where fields repeat a shared definition. Descriptions carry the intended semantics.

## A.3 Record types at a glance

| NSID | Publisher | Record key | Purpose | Phase |
|---|---|---|---|---|
| `network.livepeer.defs` | — | — | Shared definitions | — |
| `network.livepeer.operator.binding` | Orchestrator, gateway | The registered address | Proves that a DID and an on-chain address belong to one participant | A0 |
| `network.livepeer.operator.policy` | Every publisher | `self` | Publication mode, windows and retention | A0 |
| `network.livepeer.operator.affiliation` | Any participant | TID | Declares a relationship with another participant | A0 |
| `network.livepeer.orch.inventory` | Orchestrator | TID | Endpoints and runners, hourly and on change | A0 |
| `network.livepeer.orch.offering` | Orchestrator | Hash of the work key | One record per work key offered. Approval and capability labels attach to it. | A0 |
| `network.livepeer.orch.serviceReport` | Orchestrator | Window | Quality and metered payments per runner, and payments per payer | Runner rows A0; payer rows A1 |
| `network.livepeer.gateway.ledgerDigest` | Gateway | Window or round | Payments signed, per orchestrator and work key | A0; some fields A1 |
| `network.livepeer.client.observation` | Application | Window | The application's experience, per orchestrator, runner and gateway | A0; rand hash prefixes A1 |
| `network.livepeer.client.demand` | Application | Window | Jobs that no runner could take | A0 |
| `network.livepeer.probe.result` | Prober | Window | Test results, with capability and integrity checks | A0 |
| `network.livepeer.probe.canaryReveal` | Prober | TID | Opens a canary commitment after the canary has been used | A0 |
| `network.livepeer.score.published` | Scorer | Formula, window and shard | A score with its formula and inputs, so anyone can recompute it | A0, in shadow |

"A1" marks fields that need an upstream change, listed in [Appendix B](appendix-b-change-list.md). Judgments such as approval or suspected self-dealing are published as standard AT Protocol labels, not as records ([§A.16](#a16-labels)).

## A.4 Shared definitions

```json
{
  "lexicon": 1,
  "id": "network.livepeer.defs",
  "defs": {
    "ethAddress": {
      "type": "string",
      "minLength": 42,
      "maxLength": 42,
      "description": "Lowercase hex Ethereum address with 0x prefix."
    },
    "weiAmount": {
      "type": "string",
      "maxLength": 80,
      "description": "Non-negative integer amount in wei, as a decimal string."
    },
    "decimal": {
      "type": "string",
      "maxLength": 80,
      "description": "Non-negative decimal number as a string, such as a price copied from discovery."
    },
    "window": {
      "type": "object",
      "required": ["start", "end"],
      "description": "The period a summary covers. A time window has durationSec. A round window has round and no durationSec.",
      "properties": {
        "start": { "type": "string", "format": "datetime", "description": "Time windows start on a multiple of durationSec." },
        "end": { "type": "string", "format": "datetime" },
        "durationSec": { "type": "integer", "minimum": 60, "description": "300, 3600 or 86400 in this version." },
        "round": { "type": "integer", "minimum": 0, "description": "The Livepeer round in which the window started. Required for round windows." }
      }
    },
    "workKey": {
      "type": "object",
      "required": ["runnerApp"],
      "description": "What the work was.",
      "properties": {
        "runnerApp": { "type": "string", "maxLength": 256, "description": "The runner app as livepeer-node discovery names it, such as 'blueclaw/chat/qwen3.8-27b'." },
        "model": { "type": "string", "maxLength": 256, "description": "The model, when the runner app or the client names it separately." },
        "version": { "type": "string", "maxLength": 256, "description": "The version the runner advertises in discovery: an image digest or a release identifier. Lets labelers approve or exclude one version." }
      }
    },
    "histogram": {
      "type": "object",
      "required": ["bounds", "counts"],
      "description": "Counts per bucket. counts has one more entry than the bound set has edges, for values above the last edge.",
      "properties": {
        "bounds": { "type": "string", "knownValues": ["latency-ms-v1", "duration-s-v1"] },
        "counts": { "type": "array", "maxLength": 32, "items": { "type": "integer", "minimum": 0 } }
      }
    },
    "policyStamp": {
      "type": "object",
      "required": ["mode", "publishedAt"],
      "description": "The publication mode this record was produced under. Must agree with the publisher's operator.policy at publishedAt.",
      "properties": {
        "mode": { "type": "string", "knownValues": ["full", "delayed", "coarse"] },
        "publishedAt": { "type": "string", "format": "datetime" },
        "sampleRateBp": { "type": "integer", "minimum": 1, "maximum": 10000, "description": "Coarse mode: the share of sessions whose quality fields were kept." }
      }
    },
    "producer": {
      "type": "object",
      "required": ["agentVersion"],
      "properties": {
        "agentVersion": { "type": "string", "maxLength": 64 },
        "source": { "type": "string", "knownValues": ["livepeer-node", "livepeer-builder", "livepeer-signer", "batteries"] },
        "sourceVersion": { "type": "string", "maxLength": 64 }
      }
    },
    "rawCommitment": {
      "type": "object",
      "required": ["merkleRoot", "eventCount", "encoding", "retainedUntil"],
      "description": "Commitment to the redacted raw events the summary was computed from. The events stay with the publisher.",
      "properties": {
        "merkleRoot": { "type": "string", "maxLength": 64, "description": "Hex SHA-256 Merkle root." },
        "eventCount": { "type": "integer", "minimum": 0 },
        "encoding": { "type": "string", "knownValues": ["dag-cbor-sorted-v1"], "description": "dag-cbor-sorted-v1: each event reduced to its input's allowlist, encoded as DAG-CBOR, with leaves sorted by timestamp and then event hash." },
        "retainedUntil": { "type": "string", "format": "datetime", "description": "Until when the publisher keeps the batch for auditors." }
      }
    },
    "bloom": {
      "type": "object",
      "required": ["bits", "hashCount", "itemCount"],
      "description": "Bloom filter over 8-byte rand hash prefixes. Bit position i is the first 8 bytes of SHA-256(byte(i) || prefix), read as a big-endian integer, modulo the filter's length in bits.",
      "properties": {
        "bits": { "type": "bytes", "maxLength": 65536 },
        "hashCount": { "type": "integer", "minimum": 1, "maximum": 32 },
        "itemCount": { "type": "integer", "minimum": 0 }
      }
    },
    "paymentRefs": {
      "type": "object",
      "required": ["sessions", "randHashPrefixes"],
      "description": "Links a row to payment sessions the other parties also hold. Reveals nothing that predicts or claims a win.",
      "properties": {
        "sessions": { "type": "integer", "minimum": 0, "description": "Distinct rand hashes in the row." },
        "randHashPrefixes": { "type": "bytes", "maxLength": 16000, "description": "The first 8 bytes of each session's recipient rand hash, ascending and concatenated. At most 2,000; when there are more, the lowest 2,000." },
        "clientTags": { "type": "bytes", "maxLength": 32000, "description": "A1, application and probe rows: the 16-byte client tag of each listed session, in the same order as randHashPrefixes." },
        "clientTagHashes": { "type": "bytes", "maxLength": 16000, "description": "A1, gateway rows: the first 8 bytes of SHA-256 of each listed session's client tag, in the same order as randHashPrefixes." },
        "overflow": { "type": "ref", "ref": "#bloom", "description": "Present when sessions exceeds 2,000. Contains every prefix in the row." }
      }
    },
    "vantage": {
      "type": "object",
      "description": "Where the observer is. Coarse by design.",
      "properties": {
        "geoCell": { "type": "string", "maxLength": 16, "description": "H3 cell at resolution 2 or 3." },
        "rttMs": { "type": "ref", "ref": "#histogram", "description": "Round-trip time to the orchestrator, if the agent measures it. latency-ms-v1." }
      }
    }
  }
}
```

## A.5 `network.livepeer.operator.binding`

A binding proves that a repository belongs to an on-chain participant. Orchestrators and gateways publish one. Applications, probers, scorers and labelers have no on-chain role and publish none. An application that runs its own gateway publishes the gateway's binding in its own repository, with role `gateway`.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.operator.binding",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "Binds this repository to an on-chain address. The record key is the lowercase registered address, so a repository can hold one binding per address.",
      "record": {
        "type": "object",
        "required": ["did", "repoKeyHash", "registeredAddress", "role", "validFrom", "validUntil", "nonce", "sig"],
        "properties": {
          "did": { "type": "string", "format": "did" },
          "repoKeyHash": { "type": "string", "minLength": 66, "maxLength": 66, "description": "keccak256 of the repository's multibase-encoded signing key, as 0x-prefixed hex. The Ethereum key, not the DID document, decides which repository key is trusted." },
          "registeredAddress": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "Orchestrator: the bonded, registered address, which is also its ticket recipient. Gateway: its TicketBroker sender address." },
          "operatingAddresses": {
            "type": "array",
            "maxLength": 32,
            "items": { "type": "ref", "ref": "network.livepeer.defs#ethAddress" },
            "description": "Addresses that act for the registered address, such as the livepeer-node keystore account that redeems tickets and signs runner records and payment challenges."
          },
          "role": { "type": "string", "knownValues": ["orchestrator", "gateway"] },
          "validFrom": { "type": "string", "format": "datetime" },
          "validUntil": { "type": "string", "format": "datetime", "description": "At most one year after validFrom." },
          "nonce": { "type": "integer", "minimum": 0, "description": "Strictly increasing per registered address. Among valid bindings for an address, in any repository, the highest nonce wins." },
          "sig": { "type": "string", "maxLength": 132, "description": "EIP-712 signature by registeredAddress over the Binding message." }
        }
      }
    }
  }
}
```

The signed message:

```
EIP712Domain { name: "LivepeerTelemetryBinding", version: "1", chainId: 42161 }

Binding {
  string    did
  bytes32   repoKeyHash
  address   registeredAddress
  address[] operatingAddresses
  string    role
  uint64    validFrom     // Unix seconds
  uint64    validUntil
  uint64    nonce
}
```

**Verification.** AT Protocol signs commits, not records, so verification starts from the commit in which a reader received the record. An aggregator accepts a record only if all of these hold:
1. The commit verifies with repository key *K*, taken from the DID document.
2. A binding that was valid when the commit was made names this DID, and its `repoKeyHash` equals keccak256(*K*). A commit's time is when an independent relay or mirror first saw it, or its revision if none did.
3. The binding's signature recovers to `registeredAddress`, and no binding for that address with a higher nonce exists in any repository.
4. The address was a participant in the round of the record's window:
   - an orchestrator active in BondingManager, with a service URI in either service registry;
   - a gateway with a deposit and reserve in TicketBroker.

**Who signs.** On livepeer-node, the orchestrator's registered key can stay offline and sign once a year, with a hardware wallet if the operator prefers. A gateway's sender key is always online, because the signer uses it for every ticket.

**Why the Ethereum key is not the repository key.** AT Protocol accepts secp256k1 repository keys, the curve Ethereum uses, so a gateway could in principle sign commits with its sender key. We keep the two separate. AT Protocol signs SHA-256 digests, while Ethereum wallets sign Keccak-256 digests and typed messages, so a hardware wallet could not sign commits. An orchestrator's registered key would also have to stay online.

## A.6 `network.livepeer.operator.policy`

Every publisher declares how it publishes, so that readers know whether a record is exact, delayed or coarsened ([proposal §10](README.md#10-privacy-and-commercial-confidentiality)).

```json
{
  "lexicon": 1,
  "id": "network.livepeer.operator.policy",
  "defs": {
    "main": {
      "type": "record",
      "key": "literal:self",
      "description": "How this publisher publishes. Each summary's policyStamp must agree with the policy in force when the summary was published.",
      "record": {
        "type": "object",
        "required": ["collections", "rawRetentionDays", "updatedAt"],
        "properties": {
          "collections": { "type": "array", "maxLength": 16, "items": { "type": "ref", "ref": "#collectionPolicy" } },
          "coarseBuckets": { "type": "string", "knownValues": ["log-1-2-5-v1"], "description": "How coarse mode rounds volume fields. log-1-2-5-v1 rounds down to 1, 2, 5, 10, 20, 50 and so on; zero stays zero." },
          "rawRetentionDays": { "type": "integer", "minimum": 0, "maximum": 3650 },
          "auditEndpoint": { "type": "string", "format": "uri", "description": "Where an authenticated auditor can request raw batches, if the publisher serves them." },
          "updatedAt": { "type": "string", "format": "datetime" }
        }
      }
    },
    "collectionPolicy": {
      "type": "object",
      "required": ["collection", "mode", "window"],
      "properties": {
        "collection": { "type": "string", "format": "nsid" },
        "mode": { "type": "string", "knownValues": ["full", "delayed", "coarse"] },
        "delaySec": { "type": "integer", "minimum": 0, "description": "For delayed mode, and for coarse mode when it is also delayed. Defaults to 86400." },
        "sampleRateBp": { "type": "integer", "minimum": 1, "maximum": 10000, "description": "Coarse mode: the share of sessions whose quality fields are kept." },
        "window": { "type": "string", "knownValues": ["300s", "round"] }
      }
    }
  }
}
```

In `coarse` mode the agent rounds every volume field in a row: sessions, attempts, calls, tickets, bytes and amounts. Histograms are built from the sampled sessions only. Rand hash prefixes are never coarsened, because matching depends on them. A publisher that does not want its session count exposed should choose `delayed` or per-round windows, or publish nothing.

## A.7 `network.livepeer.operator.affiliation`

```json
{
  "lexicon": 1,
  "id": "network.livepeer.operator.affiliation",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "Declares a relationship with another participant. Declared relationships are kept out of independent corroboration, and are not penalized. A declaration by either side is enough.",
      "record": {
        "type": "object",
        "required": ["relationship", "since"],
        "properties": {
          "did": { "type": "string", "format": "did", "description": "The other party's DID. At least one of did and address is required." },
          "address": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The other party's registered address." },
          "relationship": {
            "type": "string",
            "knownValues": ["same-operator", "operates-for", "funds"],
            "description": "same-operator: one organization runs both. operates-for: this publisher runs infrastructure on the other's behalf. funds: this publisher funds the other's deposit, reserve or grants."
          },
          "since": { "type": "string", "format": "datetime" },
          "until": { "type": "string", "format": "datetime" }
        }
      }
    }
  }
}
```

An application that runs its own gateway does not need an affiliation record for it: both publish from the same repository, so the relationship is already visible.

## A.8 `network.livepeer.orch.inventory`

The agent builds the inventory from the orchestrator's own `/discovery` response. It writes a new record every hour and whenever the inventory changes. That is about 9,000 records a year.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.orch.inventory",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "The operator's orchestrator endpoints and their runners, as livepeer-node discovery reports them. The newest record is current; older records are history.",
      "record": {
        "type": "object",
        "required": ["endpoints", "observedAt"],
        "properties": {
          "endpoints": { "type": "array", "maxLength": 64, "items": { "type": "ref", "ref": "#endpoint" } },
          "observedAt": { "type": "string", "format": "datetime" }
        }
      }
    },
    "endpoint": {
      "type": "object",
      "required": ["serviceUrl", "runners"],
      "properties": {
        "serviceUrl": { "type": "string", "format": "uri", "description": "The orchestrator's HTTP(S) base URL: the address field of its /discovery entry. May be an IP address with a port other than 443." },
        "operatingAddress": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The node account this endpoint signs with. Must be listed in the binding." },
        "geoCell": { "type": "string", "maxLength": 16, "description": "Declared coarse location. Self-declared, and compared with measured latency." },
        "behindLoadBalancer": { "type": "boolean" },
        "runners": { "type": "array", "maxLength": 512, "items": { "type": "ref", "ref": "#runner" } }
      }
    },
    "runner": {
      "type": "object",
      "required": ["runnerId", "work", "mode", "capacity"],
      "properties": {
        "runnerId": { "type": "string", "maxLength": 128, "description": "As it appears in the runner's public URL. The runner chooses it, so it may change when the runner restarts." },
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "mode": { "type": "string", "knownValues": ["single-shot", "persistent"] },
        "gpu": { "type": "ref", "ref": "#gpu" },
        "capacity": { "type": "integer", "minimum": 0 },
        "price": { "type": "ref", "ref": "#price" },
        "attestation": { "type": "ref", "ref": "#attestation" }
      }
    },
    "gpu": {
      "type": "object",
      "description": "Self-declared. The GPU's device ID from discovery is not published.",
      "properties": {
        "name": { "type": "string", "maxLength": 128 },
        "vramMb": { "type": "integer", "minimum": 0 }
      }
    },
    "price": {
      "type": "object",
      "required": ["amount", "currency", "unit"],
      "description": "The runner's price_info from discovery.",
      "properties": {
        "amount": { "type": "ref", "ref": "network.livepeer.defs#decimal" },
        "amountUsd": { "type": "ref", "ref": "network.livepeer.defs#decimal" },
        "currency": { "type": "string", "maxLength": 16 },
        "unit": { "type": "string", "maxLength": 32 }
      }
    },
    "attestation": {
      "type": "object",
      "description": "Reserved for runners on hardware that supports remote attestation. Consumers must not trust it on the operator's word: verifiers check the report and publish an attested label.",
      "required": ["format", "reportUri", "reportHash", "measuredAt"],
      "properties": {
        "format": { "type": "string", "knownValues": ["amd-sev-snp", "intel-tdx", "nvidia-cc"] },
        "reportUri": { "type": "string", "format": "uri" },
        "reportHash": { "type": "string", "maxLength": 64, "description": "Hex SHA-256 of the report." },
        "measuredAt": { "type": "string", "format": "datetime" }
      }
    }
  }
}
```

Discovery's free-form runner `metadata` is left out under the no-free-text rule. Capacity in use is left out too, because it changes on every poll. The service report carries it as capacity-seconds instead.

## A.9 `network.livepeer.orch.offering`

AT Protocol labels attach to a DID or to a record's URI. Approval and capability labels concern an orchestrator *and* a work key together, so each work key an orchestrator offers gets a record of its own for labels to attach to.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.orch.offering",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "One work key this orchestrator offers. The record key is the first 16 bytes of SHA-256 over the canonical work key, in lowercase hex. The canonical work key is runnerApp, model and version joined by newlines, with absent fields empty.",
      "record": {
        "type": "object",
        "required": ["work", "status", "since"],
        "properties": {
          "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
          "status": { "type": "string", "knownValues": ["offered", "withdrawn"] },
          "since": { "type": "string", "format": "datetime", "description": "When the status last changed." },
          "endpoints": { "type": "array", "maxLength": 64, "items": { "type": "string", "format": "uri" }, "description": "Service URLs that offer this work key." }
        }
      }
    }
  }
}
```

The agent changes an offering only when its status or endpoints change. A new runner version is a new work key, and therefore a new offering, so an approval of one version never carries over to the next. A label that omits the record's CID holds until it expires or is withdrawn.

## A.10 `network.livepeer.orch.serviceReport`

One record per orchestrator operator per window. It has two kinds of row:
- **Runner rows** describe quality and metered payments per endpoint, runner and work key. They come from the metrics livepeer-node already exports, so they need no upstream change.
- **Payer rows** describe payments per ticket sender. They come from the session outbox, an upstream change ([Appendix B](appendix-b-change-list.md)), and carry the rand hash prefixes that let the report be matched session by session.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.orch.serviceReport",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "One record per orchestrator operator per window. Record key per §A.2.",
      "record": {
        "type": "object",
        "required": ["window", "policy", "producer", "rows", "raw"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "policy": { "type": "ref", "ref": "network.livepeer.defs#policyStamp" },
          "producer": { "type": "ref", "ref": "network.livepeer.defs#producer" },
          "rows": { "type": "array", "maxLength": 2048, "items": { "type": "ref", "ref": "#runnerRow" } },
          "payerRows": { "type": "array", "maxLength": 2048, "items": { "type": "ref", "ref": "#payerRow" }, "description": "A1: from the session outbox." },
          "counterResets": { "type": "integer", "minimum": 0, "description": "Process restarts detected in the window. Runner rows may undercount across a restart." },
          "raw": { "type": "ref", "ref": "network.livepeer.defs#rawCommitment" }
        }
      }
    },
    "runnerRow": {
      "type": "object",
      "required": ["endpoint", "runnerId", "work", "mode", "calls"],
      "properties": {
        "endpoint": { "type": "string", "format": "uri" },
        "runnerId": { "type": "string", "maxLength": 128 },
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "mode": { "type": "string", "knownValues": ["single-shot", "persistent"] },
        "calls": { "type": "ref", "ref": "#callOutcomes" },
        "headerMs": { "type": "ref", "ref": "network.livepeer.defs#histogram", "description": "Time to the final response header. latency-ms-v1." },
        "callDurationSec": { "type": "ref", "ref": "network.livepeer.defs#histogram", "description": "duration-s-v1." },
        "bytesIn": { "type": "integer", "minimum": 0 },
        "bytesOut": { "type": "integer", "minimum": 0 },
        "reservations": { "type": "ref", "ref": "#reservations" },
        "sessionEnds": { "type": "ref", "ref": "#sessionEnds" },
        "sessionDurationSec": { "type": "ref", "ref": "network.livepeer.defs#histogram", "description": "duration-s-v1." },
        "healthChecks": { "type": "integer", "minimum": 0 },
        "healthCheckFailures": { "type": "integer", "minimum": 0 },
        "capacitySeconds": { "type": "integer", "minimum": 0, "description": "Advertised capacity integrated over the window. Divide by the window's length for average capacity." },
        "availableCapacitySeconds": { "type": "integer", "minimum": 0 },
        "metered": { "type": "ref", "ref": "#metered" }
      }
    },
    "callOutcomes": {
      "type": "object",
      "description": "Calls by livepeer-node's outcome. Only error, transportError and streamError are service failures.",
      "properties": {
        "success": { "type": "integer", "minimum": 0 },
        "error": { "type": "integer", "minimum": 0 },
        "transportError": { "type": "integer", "minimum": 0 },
        "streamError": { "type": "integer", "minimum": 0 },
        "clientCancelled": { "type": "integer", "minimum": 0 },
        "capacity": { "type": "integer", "minimum": 0 },
        "rejected": { "type": "integer", "minimum": 0 },
        "paymentRequired": { "type": "integer", "minimum": 0, "description": "HTTP 402 challenges." },
        "refreshRequired": { "type": "integer", "minimum": 0, "description": "HTTP 480." },
        "noPaymentNeeded": { "type": "integer", "minimum": 0, "description": "HTTP 482." },
        "sessionStopped": { "type": "integer", "minimum": 0 }
      }
    },
    "reservations": {
      "type": "object",
      "properties": {
        "succeeded": { "type": "integer", "minimum": 0 },
        "capacity": { "type": "integer", "minimum": 0 },
        "rejected": { "type": "integer", "minimum": 0 },
        "unavailable": { "type": "integer", "minimum": 0 }
      }
    },
    "sessionEnds": {
      "type": "object",
      "description": "Session ends by livepeer-node's reason.",
      "properties": {
        "clientStop": { "type": "integer", "minimum": 0 },
        "completed": { "type": "integer", "minimum": 0 },
        "released": { "type": "integer", "minimum": 0 },
        "runnerStop": { "type": "integer", "minimum": 0 },
        "reservationFailed": { "type": "integer", "minimum": 0 },
        "runnerUnregistered": { "type": "integer", "minimum": 0 },
        "runnerExpired": { "type": "integer", "minimum": 0 },
        "runnerUnhealthy": { "type": "integer", "minimum": 0 },
        "shutdown": { "type": "integer", "minimum": 0 },
        "billingError": { "type": "integer", "minimum": 0 },
        "insufficientBalance": { "type": "integer", "minimum": 0 }
      }
    },
    "metered": {
      "type": "object",
      "description": "From livepeer-node's payment counters. Amounts are converted from approximate ETH values and are not exact. Exact amounts per payer are in payerRows.",
      "properties": {
        "ticketsAccepted": { "type": "integer", "minimum": 0 },
        "ticketsWinning": { "type": "integer", "minimum": 0 },
        "creditWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "chargeWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "billedSeconds": { "type": "integer", "minimum": 0 },
        "fixedCharges": { "type": "integer", "minimum": 0 }
      }
    },
    "payerRow": {
      "type": "object",
      "required": ["payer", "work", "payments"],
      "properties": {
        "payer": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The ticket sender: a gateway's address." },
        "endpoint": { "type": "string", "format": "uri" },
        "runnerId": { "type": "string", "maxLength": 128 },
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "payments": { "type": "ref", "ref": "network.livepeer.defs#paymentRefs" },
        "ticketsAccepted": { "type": "integer", "minimum": 0 },
        "ticketsRejected": { "type": "integer", "minimum": 0 },
        "ticketsWinning": { "type": "integer", "minimum": 0 },
        "creditWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount", "description": "Expected value credited: the sum of face value times win probability over accepted tickets." },
        "chargeWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "billedSeconds": { "type": "integer", "minimum": 0 },
        "fixedCharges": { "type": "integer", "minimum": 0 },
        "sessionEnds": { "type": "ref", "ref": "#sessionEnds" }
      }
    }
  }
}
```

**Where the fields come from.** Metric names below omit the `livepeer_` prefix.

| Fields | Source | Phase |
|---|---|---|
| `calls` | `app_calls_total{app,runner_id,mode,protocol,outcome}` | A0 |
| `headerMs`, `callDurationSec` | `app_response_header_duration_seconds`, `app_call_duration_seconds` | A0 |
| `bytesIn`, `bytesOut` | `stream_transfer_bytes_total{direction}` | A0 |
| `reservations` | `session_reservations_total{outcome}` | A0 |
| `sessionEnds`, `sessionDurationSec` | `session_ends_total{reason}`, `session_duration_seconds` | A0 |
| `healthChecks`, `healthCheckFailures` | `runner_health_checks_total{outcome}` | A0 |
| `capacitySeconds`, `availableCapacitySeconds` | `runner_capacity` and `runner_capacity_available`, integrated across scrapes | A0 |
| `metered` | `payment_tickets_total{kind}`, `payment_credit_eth_total`, `payment_charge_eth_total`, `payment_billed_seconds_total`, `payment_fixed_charges_total` | A0 |
| `payerRows` | The session outbox: one durable record per payment session | A1 |

The agent computes each window's counts as counter increases between scrapes. The last scrape at or before a window's end closes that window and opens the next, so no increment is counted twice or skipped. A counter that falls has been reset by a restart. Increments between the last scrape and the restart are lost, and `counterResets` records that this happened. Histograms are summed across protocols, because rows are not split by protocol.

## A.11 `network.livepeer.gateway.ledgerDigest`

One record per gateway per window. The gateway chooses five-minute or per-round windows in its policy. The digest never names the applications the gateway serves.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.gateway.ledgerDigest",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "Payments this gateway's signer authorized in the window, by orchestrator and work key. Record key per §A.2.",
      "record": {
        "type": "object",
        "required": ["window", "policy", "producer", "rows", "raw"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "policy": { "type": "ref", "ref": "network.livepeer.defs#policyStamp" },
          "producer": { "type": "ref", "ref": "network.livepeer.defs#producer" },
          "sender": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The sender address that signed. Required when the repository holds bindings for more than one gateway address." },
          "rows": { "type": "array", "maxLength": 2048, "items": { "type": "ref", "ref": "#row" } },
          "clientCount": { "type": "integer", "minimum": 0, "description": "Distinct API keys that paid in the window, always rounded down with log-1-2-5-v1. A count only; clients are never listed." },
          "raw": { "type": "ref", "ref": "network.livepeer.defs#rawCommitment" }
        }
      }
    },
    "row": {
      "type": "object",
      "required": ["orchestrator", "work", "payments"],
      "properties": {
        "orchestrator": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The orchestrator address as the signer reported it. Aggregators resolve it to a registered address through bindings." },
        "orchestratorUrl": { "type": "string", "format": "uri" },
        "runnerId": { "type": "string", "maxLength": 128, "description": "A1: a new signer event field." },
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "payments": { "type": "ref", "ref": "network.livepeer.defs#paymentRefs" },
        "ticketsSigned": { "type": "integer", "minimum": 0 },
        "billableSeconds": { "type": "integer", "minimum": 0 },
        "feeWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount", "description": "The sum of the fees the signer computed for the work." },
        "expectedValueWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount", "description": "A1: the sum of face value times win probability over tickets signed." },
        "offerHashes": { "type": "array", "maxLength": 100, "items": { "type": "string", "maxLength": 64 }, "description": "A1: hex SHA-256 of each signed payment challenge received. The offers themselves stay with the gateway." },
        "offerSignatureFailures": { "type": "integer", "minimum": 0, "description": "A1: challenges whose signature did not verify against the orchestrator's binding." }
      }
    }
  }
}
```

**Where the fields come from.**

| Fields | Source | Phase |
|---|---|---|
| `orchestrator`, `orchestratorUrl`, `work.runnerApp` | `create_signed_ticket` events: `orch_address`, `orch_url`, `app` | A0 |
| `payments` | Distinct `pm_session_id` values, which are the rand hashes | A0 |
| `payments.clientTagHashes` | Batteries, from the API key that paid for each session | A1 |
| `ticketsSigned`, `billableSeconds`, `feeWei` | `num_tickets`, `billable_secs`, `computed_fee` | A0 |
| `clientCount` | Distinct `auth_id` values, counted and never stored in records | A0 |
| `runnerId`, `expectedValueWei` | New event fields: runner ID, face value and win probability | A1 |
| `offerHashes`, `offerSignatureFailures` | The signer's verification of signed challenges | A1 |

The agent reads the same events Batteries consumes. Matching winning tickets to sessions is left to aggregators, which read redemptions from the chain themselves. Batteries performs the same match for its own operator.

## A.12 `network.livepeer.client.observation`

One record per application per window. Publishing is optional for each application. Rows are keyed by orchestrator, runner, work key and paying gateway.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.client.observation",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "The application's experience of the runners it used in the window. Record key per §A.2.",
      "record": {
        "type": "object",
        "required": ["window", "policy", "producer", "rows", "raw"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "policy": { "type": "ref", "ref": "network.livepeer.defs#policyStamp" },
          "producer": { "type": "ref", "ref": "network.livepeer.defs#producer" },
          "vantage": { "type": "ref", "ref": "network.livepeer.defs#vantage" },
          "rows": { "type": "array", "maxLength": 2048, "items": { "type": "ref", "ref": "#row" } },
          "raw": { "type": "ref", "ref": "network.livepeer.defs#rawCommitment" }
        }
      }
    },
    "row": {
      "type": "object",
      "required": ["orchestratorUrl", "work", "attempts"],
      "properties": {
        "orchestratorUrl": { "type": "string", "format": "uri" },
        "orchestrator": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The recipient named in the payment challenge, when the agent has it. Otherwise aggregators resolve orchestratorUrl." },
        "runnerId": { "type": "string", "maxLength": 128 },
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "payer": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The sender address of the gateway that paid. Absent for unpaid attempts." },
        "mode": { "type": "string", "knownValues": ["single-shot", "persistent"] },
        "attempts": { "type": "ref", "ref": "#attemptOutcomes" },
        "failedOver": { "type": "integer", "minimum": 0, "description": "Attempts after which the job moved to another runner." },
        "paidFailedOver": { "type": "integer", "minimum": 0, "description": "Of those, attempts that had been paid." },
        "paidAttempts": { "type": "ref", "ref": "#costClasses" },
        "feeWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount", "description": "Fees matched to this row's attempts." },
        "firstByteMs": { "type": "ref", "ref": "network.livepeer.defs#histogram", "description": "latency-ms-v1." },
        "durationSec": { "type": "ref", "ref": "network.livepeer.defs#histogram", "description": "Attempt duration. duration-s-v1." },
        "paymentChallenges": { "type": "integer", "minimum": 0 },
        "paymentRefreshes": { "type": "integer", "minimum": 0 },
        "payments": { "type": "ref", "ref": "network.livepeer.defs#paymentRefs", "description": "A1: needs the rand hash on Batteries usage rows." },
        "offerSignatureFailures": { "type": "integer", "minimum": 0, "description": "A1: signed runner records or challenges that failed verification." },
        "extensions": { "type": "union", "refs": [], "closed": false, "description": "Measures specific to a runner app, each defined in its own Lexicon." }
      }
    },
    "attemptOutcomes": {
      "type": "object",
      "description": "Attempts by Builder Core's attempt outcome.",
      "properties": {
        "succeeded": { "type": "integer", "minimum": 0 },
        "refused": { "type": "integer", "minimum": 0, "description": "The orchestrator refused before the app ran: 503, 404 or 409." },
        "unreachable": { "type": "integer", "minimum": 0 },
        "paymentRefused": { "type": "integer", "minimum": 0, "description": "The signer refused to pay this orchestrator, for example for a price above the ceiling." },
        "payment": { "type": "integer", "minimum": 0, "description": "Caller- or signer-side payment refusals that would fail anywhere." },
        "timeout": { "type": "integer", "minimum": 0 },
        "http4xx": { "type": "integer", "minimum": 0 },
        "http5xx": { "type": "integer", "minimum": 0 },
        "abandoned": { "type": "integer", "minimum": 0 },
        "paymentUnknown": { "type": "integer", "minimum": 0 }
      }
    },
    "costClasses": {
      "type": "object",
      "description": "Paid attempts by Builder Core's cost class. A pending attempt moves to its final class in a republish of the same record.",
      "properties": {
        "delivered": { "type": "integer", "minimum": 0 },
        "failedAfterPayment": { "type": "integer", "minimum": 0 },
        "uncertain": { "type": "integer", "minimum": 0 },
        "pending": { "type": "integer", "minimum": 0 }
      }
    }
  }
}
```

**Where the fields come from.** The agent reads Builder Core's events feed (`job.finished`, `cost.observed` and `cost.corrected`), with the application's `events.read` scope.

| Fields | Source | Phase |
|---|---|---|
| `orchestratorUrl`, `work`, `mode`, `attempts`, `failedOver`, `paidFailedOver` | Each job's attempts and their outcomes | A0 |
| `paidAttempts`, `feeWei` | Per-attempt cost classes from cost events | A0 |
| `payer` | The allocation's signer, from the agent's configuration | A0 |
| `runnerId`, `firstByteMs` | New attempt fields in Builder Core | A0, small Builder Core additions |
| `payments` | The rand hash and client tag of each matched payment session, once Batteries includes them on usage rows | A1 |
| `offerSignatureFailures` | Builder Core's verification of signed offers | A1 |

An application that runs its own gateway can take the rand hashes from its own signer events, and so publish `payments` before A1.

**Extensions.** Builder Core does not read request or response bodies, so it cannot measure quality that only the payload shows. An application that does read them, such as inference-gateway, can add an extension. An extension is a Lexicon named for the kind of runner app, such as `network.livepeer.ext.chat`. Its fields must be additive, for example output tokens and generation milliseconds summed per row, from which a consumer computes tokens per second. Consumers ignore extensions they do not recognize.

**Example.** One row from a five-minute record, in `full` mode. Bytes fields are shown in AT Protocol's JSON encoding.

```json
{
  "$type": "network.livepeer.client.observation",
  "window": { "start": "2026-11-04T14:05:00Z", "end": "2026-11-04T14:10:00Z", "durationSec": 300, "round": 4123 },
  "policy": { "mode": "full", "publishedAt": "2026-11-04T14:11:02Z" },
  "producer": { "agentVersion": "0.1.0", "source": "livepeer-builder", "sourceVersion": "0.5.0" },
  "rows": [
    {
      "orchestratorUrl": "https://orch.example.com:8935",
      "orchestrator": "0x1111111111111111111111111111111111111111",
      "runnerId": "r1",
      "work": { "runnerApp": "blueclaw/chat/qwen3.8-27b" },
      "payer": "0x2222222222222222222222222222222222222222",
      "mode": "single-shot",
      "attempts": { "succeeded": 41, "refused": 2, "timeout": 1 },
      "failedOver": 2,
      "paidFailedOver": 0,
      "paidAttempts": { "delivered": 41, "uncertain": 1 },
      "feeWei": "8610000000000000",
      "firstByteMs": { "bounds": "latency-ms-v1", "counts": [0, 0, 0, 0, 0, 3, 19, 15, 4, 0, 0, 0, 0, 0] },
      "paymentChallenges": 42,
      "payments": { "sessions": 42, "randHashPrefixes": { "$bytes": "…336 bytes, base64…" } }
    }
  ],
  "raw": { "merkleRoot": "…", "eventCount": 132, "encoding": "dag-cbor-sorted-v1", "retainedUntil": "2026-12-04T14:10:00Z" }
}
```

## A.13 `network.livepeer.client.demand`

Jobs that no runner could take tell orchestrators what to deploy. Published per work key, they reveal nothing about which orchestrators the application uses.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.client.demand",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "Demand the network did not meet in the window, per work key. Record key per §A.2.",
      "record": {
        "type": "object",
        "required": ["window", "policy", "producer", "rows"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "policy": { "type": "ref", "ref": "network.livepeer.defs#policyStamp" },
          "producer": { "type": "ref", "ref": "network.livepeer.defs#producer" },
          "rows": { "type": "array", "maxLength": 512, "items": { "type": "ref", "ref": "#row" } }
        }
      }
    },
    "row": {
      "type": "object",
      "required": ["work", "jobs"],
      "properties": {
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "jobs": { "type": "integer", "minimum": 0, "description": "Jobs requested for this work key." },
        "noOffering": { "type": "integer", "minimum": 0, "description": "Jobs that failed because no runner offered the work." },
        "excludedByPolicy": { "type": "integer", "minimum": 0, "description": "Jobs for which runners existed but the application's policy excluded all of them. Counted by the attested discovery view." },
        "allRefused": { "type": "integer", "minimum": 0, "description": "Jobs in which every runner tried refused for capacity." }
      }
    }
  }
}
```

`excludedByPolicy` matters for the safety rails as much as for orchestrators. A rising count means an application's filter is too strict ([proposal §7.1](README.md#71-three-phases)).

## A.14 `network.livepeer.probe.result` and `network.livepeer.probe.canaryReveal`

A probe result reuses the client observation row, adding three things:
- the test profile, so readers can tell a short synthetic call from a sustained load test;
- capability checks: which features of a runner app work;
- integrity checks: canaries sent in probe requests, and TLS and DNS checks where a probe runner app runs on the orchestrator's hosts ([proposal §6.7](README.md#67-host-abuse)).

Probers should publish in `delayed` mode, so that a probe's paying address is not visible while the probe could still be recognized.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.probe.result",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "Test traffic sent in the window. Record key per §A.2.",
      "record": {
        "type": "object",
        "required": ["window", "policy", "producer", "rows", "raw"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "policy": { "type": "ref", "ref": "network.livepeer.defs#policyStamp" },
          "producer": { "type": "ref", "ref": "network.livepeer.defs#producer" },
          "vantage": { "type": "ref", "ref": "network.livepeer.defs#vantage" },
          "rows": { "type": "array", "maxLength": 2048, "items": { "type": "ref", "ref": "#row" } },
          "raw": { "type": "ref", "ref": "network.livepeer.defs#rawCommitment" }
        }
      }
    },
    "row": {
      "type": "object",
      "required": ["observation", "testProfile"],
      "properties": {
        "observation": { "type": "ref", "ref": "network.livepeer.client.observation#row" },
        "testProfile": { "type": "string", "maxLength": 128, "description": "Name of a published test profile, such as 'chat-short-v1'." },
        "endpointTls": { "type": "string", "knownValues": ["verified", "self-signed", "invalid"], "description": "Whether the orchestrator's certificate verifies against public roots." },
        "capabilities": { "type": "array", "maxLength": 64, "items": { "type": "ref", "ref": "#capabilityCheck" } },
        "integrity": { "type": "ref", "ref": "#integrity" }
      }
    },
    "capabilityCheck": {
      "type": "object",
      "required": ["feature", "status"],
      "properties": {
        "feature": { "type": "string", "maxLength": 64, "knownValues": ["structured-output", "parallel-tools", "reasoning", "image-input", "audio-input", "seed", "stop", "top-k", "top-p"] },
        "status": { "type": "string", "knownValues": ["supported", "unsupported", "inconclusive"] },
        "checks": { "type": "integer", "minimum": 0 }
      }
    },
    "integrity": {
      "type": "object",
      "description": "Canary commitments for credentials sent in probe requests, and TLS and DNS checks run by a probe runner app on the orchestrator's hosts, where the orchestrator hosts one.",
      "properties": {
        "tlsChecks": { "type": "integer", "minimum": 0, "description": "Connections made to endpoints whose certificates the prober controls." },
        "tlsMismatches": { "type": "integer", "minimum": 0, "description": "Connections that presented an unexpected certificate chain." },
        "dnsChecks": { "type": "integer", "minimum": 0, "description": "Names resolved whose correct answers the prober knows." },
        "dnsMismatches": { "type": "integer", "minimum": 0 },
        "canaryCommitments": {
          "type": "array",
          "maxLength": 64,
          "items": { "type": "string", "maxLength": 64 },
          "description": "Hex SHA-256 of (canary, orchestrator address, nonce) for each canary sent in the window. Published before any use, so a later reveal proves where the canary went."
        }
      }
    }
  }
}
```

The capability features are the ones inference-gateway tested by hand on 26 October 2026. `inconclusive` means the request was accepted but support could not be established. That is the most common result for sampling parameters, whose effect does not show in a single response.

A canary reveal opens one commitment after the canary has been used. The canary is a test credential with no value, so publishing it exposes nothing.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.probe.canaryReveal",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "record": {
        "type": "object",
        "required": ["commitment", "probeResult", "canary", "orchestrator", "nonce", "useEvidence"],
        "properties": {
          "commitment": { "type": "string", "maxLength": 64 },
          "probeResult": { "type": "string", "format": "at-uri", "description": "The probe.result record that published the commitment." },
          "canary": { "type": "string", "maxLength": 512 },
          "orchestrator": { "type": "ref", "ref": "network.livepeer.defs#ethAddress" },
          "nonce": { "type": "string", "maxLength": 64 },
          "useEvidence": { "type": "string", "format": "uri", "description": "Signed evidence of use, published by whoever monitors the canary, under its own identity." },
          "firstUsedAt": { "type": "string", "format": "datetime" }
        }
      }
    }
  }
}
```

## A.15 `network.livepeer.score.published`

Scores are sharded, because entries for every orchestrator, work key and vantage cluster can exceed both a reasonable array size and the 1 MB record limit.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.score.published",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "A score shard, with enough information for anyone to recompute it. Record key: <formula>.<windowStart>.<shard>.",
      "record": {
        "type": "object",
        "required": ["window", "formula", "codeRef", "freezeTime", "shard", "inputsUri", "inputsHash", "entries"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "formula": { "type": "string", "maxLength": 128, "description": "Formula name and version, such as 'lr-v1'." },
          "formulaSource": { "type": "string", "format": "uri", "description": "Where the formula's specification is published." },
          "codeRef": { "type": "string", "maxLength": 128, "description": "The commit of the open-source scorer code that produced the shard." },
          "freezeTime": { "type": "string", "format": "datetime", "description": "Records published after this time were not used." },
          "shard": { "type": "string", "maxLength": 64, "description": "Which slice of work keys and vantage clusters this shard covers." },
          "inputsUri": { "type": "string", "format": "uri", "description": "A content-addressed list of the record CIDs used." },
          "inputsHash": { "type": "string", "maxLength": 64, "description": "Hex SHA-256 of that list." },
          "entries": { "type": "array", "maxLength": 2000, "items": { "type": "ref", "ref": "#entry" } }
        }
      }
    },
    "entry": {
      "type": "object",
      "required": ["orchestrator", "work", "scoreBp"],
      "properties": {
        "orchestrator": { "type": "ref", "ref": "network.livepeer.defs#ethAddress" },
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "vantageCluster": { "type": "string", "maxLength": 32 },
        "scoreBp": { "type": "integer", "minimum": 0, "maximum": 10000, "description": "10000 is a perfect score." },
        "confidenceBp": { "type": "integer", "minimum": 0, "maximum": 10000 },
        "sessionsObserved": { "type": "integer", "minimum": 0 },
        "inputModes": { "type": "ref", "ref": "#inputModes" }
      }
    },
    "inputModes": {
      "type": "object",
      "description": "How many input rows were published in each mode, so readers can see what a score rests on.",
      "properties": {
        "full": { "type": "integer", "minimum": 0 },
        "delayed": { "type": "integer", "minimum": 0 },
        "coarse": { "type": "integer", "minimum": 0 }
      }
    }
  }
}
```

## A.16 Labels

Scorers, probers and dedicated labelers publish judgments with AT Protocol's standard label mechanism (`com.atproto.label`). Labels are served by the labeler, not stored in repositories. Each label names:
- its source, the labeler's DID;
- its subject, a DID or a record URI, optionally pinned to one version of the record by CID;
- its value;
- optionally an expiry;
- optionally a negation, which withdraws an earlier label.

Labels are signed with the `#atproto_label` key in the labeler's DID document, which may be the same as its repository key. Readers must not fall back to any other key. AT Protocol makes the signature optional, but attested discovery views ignore unsigned labels.

Label values are lowercase, hyphenated and at most 128 bytes.

**Subjects.**

| Subject | Used for |
|---|---|
| An orchestrator's or gateway's DID | Judgments about the participant |
| An `orch.offering` record | Judgments about one orchestrator and one work key together |
| An `orch.inventory` record | Judgments about a specific attestation report |
| A `score.published` record | Judgments about a score |

**Proposed values.**

| Value | Subject | Meaning |
|---|---|---|
| `approved` | Offering | The labeler has vetted this orchestrator for this work key and version. It means only what the labeler's published policy says, and carries an expiry. |
| `cap-<feature>` | Offering | Tests show the feature works, such as `cap-structured-output`. Features are the names in `probe.result`. |
| `nocap-<feature>` | Offering | Tests show the feature does not work, such as `nocap-parallel-tools` |
| `tls-unverified` | DID or offering | The endpoint's certificate does not verify against public roots, so Builder Core cannot use it |
| `attested` | Inventory record | The labeler has verified the attestation report the record names |
| `self-dealing-suspected` | DID | One or more self-dealing signals exceed the labeler's threshold. Its published method names which. |
| `report-mismatch` | DID or record | Reports for the same rand hashes disagree beyond tolerance |
| `hardware-mismatch` | Offering | Measured performance is inconsistent with the declared GPU |
| `location-mismatch` | DID | Measured latency is inconsistent with the declared location |
| `binding-invalid` | DID | The binding failed verification |
| `probe-special-treatment` | DID | Probe traffic performs markedly better than real traffic to the same orchestrator |
| `secret-misuse` | DID | A canary sent to this orchestrator was used. The label cites the `probe.canaryReveal` record and the evidence of use. |
| `tls-interception` | DID | Probes on this orchestrator repeatedly received unexpected certificate chains |
| `dns-tampering` | DID | Probes on this orchestrator repeatedly received wrong DNS answers |
| `score-irreproducible` | Score record | Recomputing the shard from its inputs gave a different result. A scorer applies it to its own output when its daily check fails. |

**How they are used.**
- `approved`, the `cap-` labels and `attested` are positive. They are typically published by an application, or by a party it trusts, about orchestrators it is willing to use.
- `nocap-` and `tls-unverified` are facts that tests establish. They replace the capability reports and certificate notes that applications keep in their own repositories today.
- `secret-misuse`, `tls-interception` and `dns-tampering` report detected host abuse. Absence of `secret-misuse` is not evidence of honesty. It means only that no canary has been caught.

Labels are advisory. Each consumer decides which labelers to subscribe to, and how to weigh their labels.
