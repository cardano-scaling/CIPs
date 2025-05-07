---
CIP: ?
Title: Canonical ledger state snapshot and immutable data formats
Category: Ledger
Status: Proposed
Authors:
    - Jean-Philippe Raynaud <jp.raynaud@gmail.com>
    - Paul Clark <paul.clark@iohk.io>
    - TBD
Implementors: 
    - TBD
Discussions:
    - https://github.com/cardano-foundation/CIPs/pull/?
Created: 2025--05-05
License: CC-BY-4.0
---

<!-- Existing categories:

- Meta     | For meta-CIPs which typically serves another category or group of categories.
- Wallets  | For standardisation across wallets (hardware, full-node or light).
- Tokens   | About tokens (fungible or non-fungible) and minting policies in general.
- Metadata | For proposals around metadata (on-chain or off-chain).
- Tools    | A broad category for ecosystem tools not falling into any other category.
- Plutus   | Changes or additions to Plutus
- Ledger   | For proposals regarding the Cardano ledger (including Reward Sharing Schemes)
- Catalyst | For proposals affecting Project Catalyst / the Jörmungandr project

-->

## Abstract

<!-- A short (\~200 word) description of the proposed solution and the technical issue being addressed. -->

We propose the adoption of a canonical format for ledger state snapshots and immutable files across all Cardano node implementations. This standardization aims to enable the Mithril protocol to sign ledger state snapshots and facilitate seamless import/export of ledger state and immutable data between different Cardano node implementations.

## Motivation: why is this CIP necessary?

<!-- A clear explanation that introduces the reason for a proposal, its use cases and stakeholders. If the CIP changes an established design then it must outline design issues that motivate a rework. For complex proposals, authors must write a Cardano Problem Statement (CPS) as defined in CIP-9999 and link to it as the `Motivation`. -->

The Cardano ecosystem is rapidly evolving, with the historical Cardano node and new implementations like Amaru and Acropolis under development. In the future, Stake Pool Operators (SPOs) will likely run diverse node implementations. To ensure seamless interoperability and compatibility of data across these implementations, a standardized approach is essential. This is particularly critical for the Mithril protocol, which depends on consistent signing of ledger state snapshots and immutable files across different nodes.

Mithril, based on a [Stake-based Threshold Multi-signature scheme](https://iohk.io/en/research/library/papers/mithril-stake-based-threshold-multisignatures/), enables SPOs to certify Cardano chain data in a trustless manner. It is currently used to accelerate full-node bootstrapping and secure light wallets. The protocol aggregates individual signatures from SPOs into multi-signatures and certificates. For Mithril to function securely and effectively, all signer nodes must compute the same message to sign from identical data, and the adoption among SPOs must be very high.

Currently, certification of the Cardano node database relies on the internal representation of the historical Cardano node database:

- Immutable files (chunk files and primary/secondary indexes) are deterministically computed by the historical Cardano node. They are signed and served by Mithril.
- Ledger state snapshots, however, are not consistently computed at the same blockchain point across nodes. They are signed using IOG-owned keys instead and they are served by Mithril.

This limitation means only the historical Cardano node fully benefits from ledger state and immutable file snapshots, and the ledger state snapshot is not yet a first-class citizen in the Mithril protocol.

Another use case for the ledger state snapshot format is to facilitate debugging and testing of a node implementation.

## Specification

<!-- The technical specification should describe the proposed improvement in sufficient technical detail. In particular, it should provide enough information that an implementation can be performed solely on the basis of the design in the CIP. This is necessary to facilitate multiple, interoperable implementations. This must include how the CIP should be versioned, if not covered under an optional Versioning main heading. If a proposal defines structure of on-chain data it must include a CDDL schema in its specification.-->

### Ledger state snapshot format

#### Requirements

The ledger state snapshot format should:

- Be deterministically encoded.
- Support both random access and sequential reading.
- Be adaptable to future changes in the Cardano ecosystem.
- Ensure forward and backward compatibility, allowing new implementations to read older versions and vice versa.
- Be defined using a robust schema notation (e.g., CDDL) to guarantee clarity and validation.

We propose deterministically encoded CBOR ([RFC8949 Sec 4.2](https://datatracker.ietf.org/doc/html/rfc8949#section-4.2), with the tighter profile defined in [dCBOR](https://datatracker.ietf.org/doc/draft-mcnally-deterministic-cbor/)) with CDDL schemas as the base format and tooling.

#### Contents

> [!NOTE]
>
> - Do we want to include derived data in the snapshot?
> - Do we want to create a file per type of data or a single file containing all the data?

The essential required data in a snapshot include are:

- UTXOs
- Stake delegation
- Reward accounts
- Protocol parameters
- Accounting pots (reserves, deposits etc.)
- SPO state
- DRep state
- Governance action general state
- Governance action voting state
- Header state (e.g. nonces)

It could maybe also be useful to produce derived data that is commonly useful:

- Payment address balances
- Stake address balances
- Stake pool delegation distribution

##### UTxOs

```cddl
TODO: Add CDDL schema
```

##### Stake delegation

```cddl
TODO: Add CDDL schema
```

##### Reward accounts

```cddl
TODO: Add CDDL schema
```

##### Protocol parameters

```cddl
TODO: Add CDDL schema
```

##### SPO state

```cddl
TODO: Add CDDL schema
```

##### DRep state

```cddl
TODO: Add CDDL schema
```

##### Governance action general state

```cddl
TODO: Add CDDL schema
```

##### Governance action voting state

```cddl
TODO: Add CDDL schema
```

##### Header state

```cddl
TODO: Add CDDL schema
```

##### Payment address balances

```cddl
TODO: Add CDDL schema
```

##### Stake address balances

```cddl
TODO: Add CDDL schema
```

##### Stake pool delegation distribution

```cddl
TODO: Add CDDL schema
```

#### Forward/backward compatibility

We must ensure that the ledger state snapshot format supports forward and backward compatibility. This guarantees that newer implementations can read older snapshot versions and vice versa without issues. Additionally, Mithril certification must access the ledger state snapshot for a specific version, regardless of the maximum version supported by Cardano nodes. This version would be updated during a new Mithril era, ensuring all Mithril signer nodes transition to the new version simultaneously at an epoch boundary, thereby avoiding split-brain scenarios in the protocol.

#### Snapshot schedule for Mithril certification

> [!NOTE]
> Do we want to create a schedule only for Mithril snapshotting purposes, or do we want to create the same schedule for all node implementations?

Once per epoch, every Cardano node implementation should generate a ledger state snapshot to be signed by Mithril.

The snapshot should be created at a specific blockchain point, sufficiently distant from epoch boundaries. We propose the following formula to determine this point:

```
TODO: Add formula to compute the snapshot point
```

The snapshot must reflect the ledger state after processing the specified block number.

### Ledger state snapshot format

TODO: Add requirements

#### Format

TODO: Add format

#### Forward/backward compatibility

TODO: Add forward/backward compatibility

## Rationale: how does this CIP achieve its goals?

<!-- The rationale fleshes out the specification by describing what motivated the design and what led to particular design decisions. It should describe alternate designs considered and related work. The rationale should provide evidence of consensus within the community and discuss significant objections or concerns raised during the discussion.

It must also explain how the proposal affects the backward compatibility of existing solutions when applicable. If the proposal responds to a CPS, the 'Rationale' section should explain how it addresses the CPS, and answer any questions that the CPS poses for potential solutions.
-->

### Why are we proposing this CIP?

#### For Mithril

TODO: Add content

#### For alternative node implementations

TODO: Add content

#### For the historical Cardano node

TODO: Add content

### For the Cardano ecosystem

TODO: Add content

## Path to Active

### Acceptance Criteria

<!-- Describes what are the acceptance criteria whereby a proposal becomes 'Active' -->

1. A robust and adaptable ledger state snapshot format that facilitates import/export between node implementations and supports Mithril's signing of ledger state snapshots.
1. A standardized snapshotting schedule for the ledger state snapshot format to enable Mithril's signing process.
1. A flexible and consistent immutable file format for seamless import/export across different node implementations.

### Implementation Plan

<!-- A plan to meet those criteria or `N/A` if an implementation plan is not applicable. -->

- [ ] Write the specification for the ledger state snapshot format.
- [ ] Write the specification for the ledger state snapshot schedule.
- [ ] Write the specification for the immutable file format.
- [ ] Develop importer/exporter functionality for the ledger state snapshot format:
  - [ ] Historical Cardano node
  - [ ] Amaru
  - [ ] Acropolis
  - [ ] Mithril (certification)
- [ ] Develop importer/exporter functionality for the immutable file format:
  - [ ] Historical Cardano node
  - [ ] Amaru
  - [ ] Acropolis
  - [ ] Mithril (certification)
- [ ] Implement the snapshotting schedule for the ledger state snapshot format:
  - [ ] Historical Cardano node
  - [ ] Amaru
  - [ ] Acropolis
  - [ ] Mithril (certification)

## References

### Historical Cardano node

- **The ChainDB format**: https://cardano-scaling.github.io/cardano-blueprint/storage/chaindb.html
- TODO: Add more references

### Amaru

- **Amaru**: https://github.com/pragma-org/amaru
- TODO: Add more references

### Acropolis

- **Acropolis**: https://github.com/input-output-hk/acropolis
- TODO: Add more references

### Mithril

- **Mithril: Stake-based Threshold Multisignatures**: https://iohk.io/en/research/library/papers/mithril-stake-based-threshold-multisignatures/
- **Mithril certification**: https://mithril.network/doc/mithril/advanced/mithril-certification/
- **Mithril certification of Cardano database**: https://mithril.network/doc/mithril/advanced/mithril-certification/cardano-node-database
- **Mithril certification of Cardano database v2**: https://mithril.network/doc/mithril/advanced/mithril-certification/cardano-node-database-v2
- **Mithril network upgrade strategy**: https://mithril.network/doc/adr/4
- **Mithril Network Architecture**: https://mithril.network/doc/mithril/mithril-network/architecture
- **Mithril Protocol in depth**: https://mithril.network/doc/mithril/mithril-protocol/protocol
- **Mithril Certificate Chain in depth**: https://mithril.network/doc/mithril/mithril-protocol/certificates
- **Fast Bootstrap a Cardano node**: https://mithril.network/doc/manual/getting-started/bootstrap-cardano-node
- **Run a Mithril Signer node (SPO)**: https://mithril.network/doc/manual/getting-started/run-signer-node/
- **Mithril Threat Model**: https://mithril.network/doc/mithril/threat-model

## Copyright

<!-- The CIP must be explicitly licensed under acceptable copyright terms. Uncomment the license you wish to use (delete the other one) and ensure it matches the License field in the header.

If AI/LLMs were used in the creation of the copyright text, the author may choose to include a disclaimer to describe their application within the proposal.
-->

This CIP is licensed under [Apache-2.0](http://www.apache.org/licenses/LICENSE-2.0)
