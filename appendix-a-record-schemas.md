# Appendix A: Draft record schemas

*Companion to [Operator-Owned Telemetry for the Livepeer Network](README.md). These drafts are for review. None is published, and every name may change before the governing LIP adopts them.*

## A.1 Conventions

The schemas are written as AT Protocol Lexicons under the `network.livepeer` namespace. Five conventions apply to all of them.

1. **No floating-point numbers.** The AT Protocol data model forbids floats, because they do not re-encode identically across machines and so would break signatures. Ratios are expressed in basis points (1 = 0.01%), frame rates in milli-frames per second, and durations in milliseconds.
2. **Wei amounts are decimal strings.** Ticket face values and fees can exceed the range of a 64-bit integer.
3. **Ethereum addresses are lowercase hex strings** with the `0x` prefix, 42 characters long.
4. **Nothing identifies an end user or a customer.** No schema has a field for a client IP address, prompt, request parameter, stream identifier, or application customer. Where records must link to each other, they use salted hashes, and the salt never leaves the publisher.
5. **Summaries are additive.** Every numeric field in a summary is a count, a sum, or a histogram of counts, so that an aggregator can merge summaries across time windows and across publishers without error. Rates are computed by consumers from counts.

The Lexicon JSON below is abbreviated where fields repeat a shared definition. Descriptions carry the intended semantics.

## A.2 Record types at a glance

| NSID | Publisher | Record key | Purpose |
|---|---|---|---|
| `network.livepeer.defs` | — | — | Shared definitions |
| `network.livepeer.operator.binding` | Any on-chain participant | `self` or the address | Proves a DID and an Ethereum address belong to one operator |
| `network.livepeer.operator.delegation` | Remote signer | TID | Names the gateway DIDs that pay from the signer's deposit |
| `network.livepeer.orch.fleet` | Orchestrator | `self` | Lists the orchestrator's nodes |
| `network.livepeer.orch.quote` | Orchestrator | TID | Public standing offer: prices, capabilities, hardware |
| `network.livepeer.orch.serviceReport` | Orchestrator | TID | Five-minute summary of work performed and payment received |
| `network.livepeer.gateway.observation` | Gateway or signer | TID | Five-minute summary of the gateway's experience with one orchestrator |
| `network.livepeer.signer.ledgerDigest` | Remote signer | TID | Per-round summary of payments authorized |
| `network.livepeer.probe.result` | Prober | TID | Summary of test traffic sent to one orchestrator |
| `network.livepeer.score.published` | Scorer | TID | A score with its formula and inputs, so anyone can recompute it |

Flags such as suspected self-dealing are published as standard AT Protocol labels (`com.atproto.label`), not as a new record type. [§A.12](#a12-flags-as-labels) lists the proposed label values.

## A.3 Shared definitions

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
    "window": {
      "type": "object",
      "required": ["start", "durationSec"],
      "properties": {
        "start": { "type": "string", "format": "datetime", "description": "Window start, UTC, aligned to a multiple of durationSec." },
        "durationSec": { "type": "integer", "enum": [300, 3600, 86400], "description": "Five minutes, one hour, or one day." },
        "round": { "type": "integer", "description": "Livepeer round in which the window started." }
      }
    },
    "jobFamily": {
      "type": "string",
      "knownValues": ["live-v2v", "live-runner", "transcode", "ai-batch"],
      "description": "Extensible. Consumers must ignore families they do not recognize."
    },
    "workKey": {
      "type": "object",
      "required": ["family", "app"],
      "description": "What the work was. For live-v2v, app is the pipeline and model; for live-runner, app is the runner's app id.",
      "properties": {
        "family": { "type": "ref", "ref": "#jobFamily" },
        "app": { "type": "string", "maxLength": 256 },
        "model": { "type": "string", "maxLength": 256 }
      }
    },
    "histogram": {
      "type": "object",
      "required": ["bounds", "counts"],
      "description": "Counts per bucket. bounds are upper bucket edges in the unit named by the field; counts has one more entry than bounds, for values above the last edge. Bounds are fixed per field by the schema version so histograms from different publishers can be summed.",
      "properties": {
        "bounds": { "type": "array", "items": { "type": "integer" } },
        "counts": { "type": "array", "items": { "type": "integer", "minimum": 0 } }
      }
    },
    "rawCommitment": {
      "type": "object",
      "required": ["merkleRoot", "eventCount", "retainedUntil"],
      "description": "Commitment to the raw events a summary was computed from. The raw events stay with the publisher.",
      "properties": {
        "merkleRoot": { "type": "string", "maxLength": 128, "description": "Hex SHA-256 Merkle root over the canonical encoding of the raw events." },
        "eventCount": { "type": "integer", "minimum": 0 },
        "retainedUntil": { "type": "string", "format": "datetime", "description": "Until when the publisher commits to serve the raw batch to auditors." }
      }
    },
    "paymentRef": {
      "type": "object",
      "description": "Links a summary to payment material both sides hold, without revealing session identifiers.",
      "properties": {
        "recipientRandHashes": {
          "type": "array",
          "maxLength": 1000,
          "items": { "type": "string", "maxLength": 66 },
          "description": "recipient_rand_hash values from the ticket parameters used in this window."
        },
        "ticketsSent": { "type": "integer", "minimum": 0 },
        "expectedValueWei": { "type": "ref", "ref": "#weiAmount", "description": "Sum over tickets of faceValue × winProb." }
      }
    },
    "vantage": {
      "type": "object",
      "description": "Where an observation was made from. Coarse by design.",
      "properties": {
        "geoCell": { "type": "string", "maxLength": 16, "description": "Coarse location cell of the observer, e.g. an H3 index at resolution 2 or 3." },
        "rttMs": { "type": "ref", "ref": "#histogram", "description": "Round-trip time from observer to the orchestrator node." }
      }
    }
  }
}
```

## A.4 `network.livepeer.operator.binding`

```json
{
  "lexicon": 1,
  "id": "network.livepeer.operator.binding",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "Binds this repository's DID to an on-chain address. The record key is the lowercase address.",
      "record": {
        "type": "object",
        "required": ["did", "registeredAddress", "validFrom", "sig"],
        "properties": {
          "did": { "type": "string", "format": "did" },
          "registeredAddress": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The address registered on-chain (bonded orchestrator or ticket sender)." },
          "operatingAddresses": {
            "type": "array",
            "items": { "type": "ref", "ref": "network.livepeer.defs#ethAddress" },
            "description": "Addresses the registered address authorizes to sign on its behalf."
          },
          "validFrom": { "type": "string", "format": "datetime" },
          "validUntil": { "type": "string", "format": "datetime" },
          "sig": {
            "type": "string",
            "maxLength": 132,
            "description": "EIP-712 signature by registeredAddress over {did, operatingAddresses, validFrom, validUntil}."
          }
        }
      }
    }
  }
}
```

Verification: recover the signer from `sig`, confirm it equals `registeredAddress`, confirm the DID document of `did` points at this repository, and confirm the address is a current participant on-chain.

## A.5 `network.livepeer.operator.delegation`

```json
{
  "lexicon": 1,
  "id": "network.livepeer.operator.delegation",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "Published by a remote signer. States that a gateway pays from this signer's deposit, so the signer vouches for the gateway's reports.",
      "record": {
        "type": "object",
        "required": ["gatewayDid", "validFrom"],
        "properties": {
          "gatewayDid": { "type": "string", "format": "did" },
          "validFrom": { "type": "string", "format": "datetime" },
          "validUntil": { "type": "string", "format": "datetime" }
        }
      }
    }
  }
}
```

Delegations are optional. A keyless client that does not want a DID of its own is covered by the signer's own observations ([§A.8](#a8-networklivepeergatewayobservation)).

## A.6 `network.livepeer.orch.fleet`

```json
{
  "lexicon": 1,
  "id": "network.livepeer.orch.fleet",
  "defs": {
    "main": {
      "type": "record",
      "key": "literal:self",
      "description": "The orchestrator's nodes. Replaces hand-maintained -orchAddr lists and extends OrchestratorInfo.nodes.",
      "record": {
        "type": "object",
        "required": ["nodes", "updatedAt"],
        "properties": {
          "nodes": { "type": "array", "maxLength": 512, "items": { "type": "ref", "ref": "#node" } },
          "updatedAt": { "type": "string", "format": "datetime" }
        }
      }
    },
    "node": {
      "type": "object",
      "required": ["uri", "roles"],
      "properties": {
        "uri": { "type": "string", "format": "uri" },
        "label": { "type": "string", "maxLength": 64, "description": "Stable operator-chosen name, used as the node key in reports." },
        "roles": {
          "type": "array",
          "items": { "type": "string", "knownValues": ["transcode", "live-v2v", "live-runner", "ai-batch"] }
        },
        "geoCell": { "type": "string", "maxLength": 16, "description": "Declared coarse location. Self-declared; checked against measured latency." },
        "behindLoadBalancer": { "type": "string", "format": "uri", "description": "The shared entry point, if this node is reached through a geographic load balancer." },
        "apps": { "type": "array", "items": { "type": "string", "maxLength": 256 } },
        "capacity": { "type": "integer", "minimum": 0 }
      }
    }
  }
}
```

## A.7 `network.livepeer.orch.quote` and the signed wire quote

Two different things are called a quote in this proposal.

- **The wire quote** is signed by the orchestrator and sent to one gateway inside `OrchestratorInfo`. It may carry gateway-specific prices, so it is never published in full. Gateways publish only its hash, in their observations. Its EIP-712 structure is defined in [Appendix B](appendix-b-change-list.md#b8-the-signed-wire-quote).
- **The public quote** below is the orchestrator's standing offer, published in its repository.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.orch.quote",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "Standing public offer. A new record supersedes the previous one.",
      "record": {
        "type": "object",
        "required": ["offers", "validFrom"],
        "properties": {
          "offers": { "type": "array", "maxLength": 256, "items": { "type": "ref", "ref": "#offer" } },
          "hardware": { "type": "array", "items": { "type": "ref", "ref": "#gpu" }, "description": "Self-declared." },
          "validFrom": { "type": "string", "format": "datetime" },
          "validUntil": { "type": "string", "format": "datetime" }
        }
      }
    },
    "offer": {
      "type": "object",
      "required": ["work", "unit", "pricePerUnitWei"],
      "properties": {
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "unit": { "type": "string", "knownValues": ["pixel", "second", "720p-pixel-second", "fixed", "token", "audio-ms"] },
        "pricePerUnitWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "listPriceUsdMicros": { "type": "integer", "description": "Operator's configured USD price in millionths of a dollar, when pricing is USD-denominated." },
        "nodes": { "type": "array", "items": { "type": "string", "maxLength": 64 }, "description": "Fleet node labels offering this work." }
      }
    },
    "gpu": {
      "type": "object",
      "properties": {
        "model": { "type": "string", "maxLength": 128 },
        "memoryMb": { "type": "integer" },
        "count": { "type": "integer" },
        "node": { "type": "string", "maxLength": 64 }
      }
    }
  }
}
```

## A.8 `network.livepeer.gateway.observation`

One record per gateway, orchestrator, kind of work, and five-minute window. A remote signer uses the same record type for observations relayed from its keyless clients, setting `relayedFor`.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.gateway.observation",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "record": {
        "type": "object",
        "required": ["window", "orchestrator", "work", "core", "raw"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "orchestrator": { "type": "ref", "ref": "network.livepeer.defs#ethAddress" },
          "node": { "type": "string", "maxLength": 64, "description": "Fleet node label, if known." },
          "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
          "vantage": { "type": "ref", "ref": "network.livepeer.defs#vantage" },
          "relayedFor": { "type": "string", "maxLength": 64, "description": "Set by a signer relaying client reports: salted hash of the client's API key. Never the key or customer name." },
          "quoteHashes": { "type": "array", "maxLength": 100, "items": { "type": "string", "maxLength": 66 }, "description": "Hashes of the signed wire quotes received in this window." },
          "core": { "type": "ref", "ref": "#core" },
          "liveV2v": { "type": "ref", "ref": "#liveV2v" },
          "liveRunner": { "type": "ref", "ref": "#liveRunner" },
          "payment": { "type": "ref", "ref": "network.livepeer.defs#paymentRef" },
          "raw": { "type": "ref", "ref": "network.livepeer.defs#rawCommitment" }
        }
      }
    },
    "core": {
      "type": "object",
      "required": ["sessionsStarted", "sessionsSucceeded", "sessionsFailed"],
      "properties": {
        "sessionsStarted": { "type": "integer", "minimum": 0 },
        "sessionsSucceeded": { "type": "integer", "minimum": 0 },
        "sessionsFailed": { "type": "integer", "minimum": 0 },
        "sessionsExcused": { "type": "integer", "minimum": 0, "description": "Failures attributed to the client, per the NaaP excused-error taxonomy." },
        "noOrchestratorAvailable": { "type": "integer", "minimum": 0, "description": "Requests for this work that found no orchestrator. Recorded with orchestrator set to the zero address." },
        "unitsBilled": { "type": "integer", "minimum": 0 },
        "feesWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "signatureFailures": { "type": "integer", "minimum": 0, "description": "Quotes or output hashes whose orchestrator signature failed verification." }
      }
    },
    "liveV2v": {
      "type": "object",
      "properties": {
        "startupMs": { "type": "ref", "ref": "network.livepeer.defs#histogram", "description": "Stream request to first processed segment." },
        "firstFrameMs": { "type": "ref", "ref": "network.livepeer.defs#histogram", "description": "Time to first frame (PTFF)." },
        "outputMilliFps": { "type": "ref", "ref": "network.livepeer.defs#histogram" },
        "statusSamples": { "type": "integer", "minimum": 0 },
        "statusSamplesExpected": { "type": "integer", "minimum": 0 },
        "zeroFpsSamples": { "type": "integer", "minimum": 0 },
        "loadingOnlySessions": { "type": "integer", "minimum": 0 },
        "swapsAway": { "type": "integer", "minimum": 0, "description": "Sessions this gateway moved away from this orchestrator." }
      }
    },
    "liveRunner": {
      "type": "object",
      "properties": {
        "reservationsAttempted": { "type": "integer", "minimum": 0 },
        "reservationsSucceeded": { "type": "integer", "minimum": 0 },
        "capacityRejections": { "type": "integer", "minimum": 0, "description": "HTTP 503 responses." },
        "paymentChallenges": { "type": "integer", "minimum": 0, "description": "HTTP 402 responses." },
        "firstByteMs": { "type": "ref", "ref": "network.livepeer.defs#histogram" },
        "sessionDurationSec": { "type": "ref", "ref": "network.livepeer.defs#histogram" },
        "endReasons": { "type": "ref", "ref": "#endReasons" }
      }
    },
    "endReasons": {
      "type": "object",
      "properties": {
        "clientStop": { "type": "integer", "minimum": 0 },
        "runnerStop": { "type": "integer", "minimum": 0 },
        "paymentFailure": { "type": "integer", "minimum": 0 },
        "heartbeatTimeout": { "type": "integer", "minimum": 0 },
        "error": { "type": "integer", "minimum": 0 }
      }
    }
  }
}
```

The `liveV2v` fields are chosen so that an aggregator can reproduce the NaaP measurements listed in [§7.3 of the proposal](README.md#73-proving-the-new-pipeline-counts-correctly).

## A.9 `network.livepeer.orch.serviceReport`

One record per orchestrator node, payer, kind of work, and five-minute window. It mirrors the gateway observation from the other side, so that the two can be matched on `payment.recipientRandHashes`.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.orch.serviceReport",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "record": {
        "type": "object",
        "required": ["window", "payer", "work", "accounting", "raw"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "node": { "type": "string", "maxLength": 64 },
          "payer": { "type": "ref", "ref": "network.livepeer.defs#ethAddress", "description": "The ticket sender: a gateway or remote signer." },
          "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
          "accounting": { "type": "ref", "ref": "#accounting" },
          "runners": { "type": "array", "maxLength": 256, "items": { "type": "ref", "ref": "#runnerSummary" } },
          "payment": { "type": "ref", "ref": "network.livepeer.defs#paymentRef" },
          "raw": { "type": "ref", "ref": "network.livepeer.defs#rawCommitment" }
        }
      }
    },
    "accounting": {
      "type": "object",
      "description": "From counters in ProcessPayment and DebitFees.",
      "properties": {
        "sessions": { "type": "integer", "minimum": 0 },
        "ticketsReceived": { "type": "integer", "minimum": 0 },
        "ticketsRejected": { "type": "integer", "minimum": 0 },
        "winningTickets": { "type": "integer", "minimum": 0 },
        "creditedWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "unitsDebited": { "type": "integer", "minimum": 0 },
        "debitedWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "insufficientBalanceStops": { "type": "integer", "minimum": 0 }
      }
    },
    "runnerSummary": {
      "type": "object",
      "description": "Per-runner proxy measurements. Runner identity is a salted label, so pool members are distinguishable over time but not identifiable.",
      "properties": {
        "runnerLabel": { "type": "string", "maxLength": 64 },
        "requests": { "type": "integer", "minimum": 0 },
        "status2xx": { "type": "integer", "minimum": 0 },
        "status4xx": { "type": "integer", "minimum": 0 },
        "status5xx": { "type": "integer", "minimum": 0 },
        "firstByteMs": { "type": "ref", "ref": "network.livepeer.defs#histogram" },
        "bytesIn": { "type": "integer", "minimum": 0 },
        "bytesOut": { "type": "integer", "minimum": 0 },
        "heartbeatMisses": { "type": "integer", "minimum": 0 },
        "endReasons": { "type": "ref", "ref": "network.livepeer.gateway.observation#endReasons" }
      }
    }
  }
}
```

## A.10 `network.livepeer.signer.ledgerDigest`

```json
{
  "lexicon": 1,
  "id": "network.livepeer.signer.ledgerDigest",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "Per-round digest of payments a remote signer authorized, by orchestrator and kind of work.",
      "record": {
        "type": "object",
        "required": ["window", "lines", "ledgerRoot"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "lines": { "type": "array", "maxLength": 2000, "items": { "type": "ref", "ref": "#line" } },
          "ledgerRoot": { "type": "string", "maxLength": 128, "description": "Merkle root over the signer's append-only ledger entries for the window." },
          "clientCount": { "type": "integer", "minimum": 0, "description": "Distinct paying clients. A count only; clients are not listed." }
        }
      }
    },
    "line": {
      "type": "object",
      "required": ["orchestrator", "work"],
      "properties": {
        "orchestrator": { "type": "ref", "ref": "network.livepeer.defs#ethAddress" },
        "work": { "type": "ref", "ref": "network.livepeer.defs#workKey" },
        "sessions": { "type": "integer", "minimum": 0 },
        "ticketsSigned": { "type": "integer", "minimum": 0 },
        "billableSeconds": { "type": "integer", "minimum": 0 },
        "expectedValueWei": { "type": "ref", "ref": "network.livepeer.defs#weiAmount" },
        "winningTicketsRedeemed": { "type": "integer", "minimum": 0, "description": "Matched against TicketBroker WinningTicketRedeemed events." }
      }
    }
  }
}
```

## A.11 `network.livepeer.probe.result` and `network.livepeer.score.published`

A probe result has the same shape as a gateway observation, with one addition: the probe's test configuration, so readers can tell a short synthetic clip from a sustained load test.

```json
{
  "lexicon": 1,
  "id": "network.livepeer.probe.result",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "record": {
        "type": "object",
        "required": ["observation", "testProfile"],
        "properties": {
          "observation": { "type": "ref", "ref": "network.livepeer.gateway.observation" },
          "testProfile": { "type": "string", "maxLength": 128, "description": "Name of a published test profile, e.g. 'lv2v-512px-60s'." }
        }
      }
    }
  }
}
```

```json
{
  "lexicon": 1,
  "id": "network.livepeer.score.published",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "description": "A score, with enough information for anyone to recompute it.",
      "record": {
        "type": "object",
        "required": ["window", "formula", "scores", "inputs"],
        "properties": {
          "window": { "type": "ref", "ref": "network.livepeer.defs#window" },
          "formula": { "type": "string", "maxLength": 128, "description": "Formula name and version, e.g. 'sla-v1'." },
          "formulaSource": { "type": "string", "format": "uri", "description": "Where the formula's code or specification is published." },
          "scores": { "type": "array", "maxLength": 5000, "items": { "type": "ref", "ref": "#entry" } },
          "inputs": { "type": "string", "maxLength": 128, "description": "Hash of the sorted list of record CIDs the scores were computed from; the list itself is served by the scorer on request." }
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
        "scoreBp": { "type": "integer", "minimum": 0, "maximum": 10000, "description": "Score in basis points: 10000 is a perfect score." },
        "confidenceBp": { "type": "integer", "minimum": 0, "maximum": 10000 },
        "sessionsObserved": { "type": "integer", "minimum": 0 }
      }
    }
  }
}
```

## A.12 Flags as labels

Scorers and dedicated labelers publish flags with AT Protocol's standard label mechanism. The subject is an orchestrator's or gateway's DID, or a specific record. Proposed values:

| Label value | Meaning |
|---|---|
| `self-dealing-suspected` | One or more self-dealing signals exceed the labeler's threshold. The labeler's published method names which. |
| `report-mismatch` | Gateway and orchestrator reports for the same payment references disagree beyond tolerance. |
| `hardware-mismatch` | Measured performance is inconsistent with declared hardware. |
| `location-mismatch` | Measured latency is inconsistent with declared location. |
| `binding-invalid` | The DID-to-address binding failed verification. |
| `probe-special-treatment` | Probe traffic performs markedly better than real traffic to the same orchestrator. |

Labels are advisory. Each consumer decides which labelers to subscribe to and how to weigh their labels.
