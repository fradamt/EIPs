---
eip: TBD
title: Raise PeerDAS Full Custody Node Threshold
description: Raises BALANCE_PER_ADDITIONAL_CUSTODY_GROUP from 32 ETH to 256 ETH.
author: Francesco D'Amato (@fradamt)
discussions-to: https://ethereum-magicians.org/t/eip-raise-peerdas-full-custody-node-threshold/XXXXX
status: Draft
type: Standards Track
category: Core
created: 2026-02-02
requires: 7594
---

## Abstract

This EIP increases the `BALANCE_PER_ADDITIONAL_CUSTODY_GROUP` constant from 32 ETH to 256 ETH (an 8x increase). This raises the threshold at which a consensus layer node becomes a "full custody node" (custodying all 128 columns) from 4,096 ETH to 32,768 ETH. The change reduces bandwidth requirements for nodes operated by solo stakers and home operators while maintaining sufficient full custody node count for network robustness, most importantly still ensuring that there are enough nodes capable of performing reconstruction if needed.

## Motivation

[EIP-7594](./eip-7594.md) introduces PeerDAS with a custody requirement that scales with validator balance: nodes custody additional columns based on their total validator effective balance, computed as `min(max(floor(total_node_balance / BALANCE_PER_ADDITIONAL_CUSTODY_GROUP), VALIDATOR_CUSTODY_REQUIREMENT), NUMBER_OF_CUSTODY_GROUPS)`. With the current value of 32 ETH (equivalent to one pre-Electra validator), nodes reach full custody node status (custodying all 128 columns) at 4,096 ETH or approximately 128 validators.

This threshold creates challenges for fractional staking protocols (Lido CSM, Rocket Pool, etc.) where operators can run many validators with relatively modest personal capital. For example:

- **Lido CSM**: Operators can run 128 validators with approximately 167 ETH of their own capital (1.3 ETH bond per validator).
- **Rocket Pool**: With 4 ETH minipools, operators can run 128 validators with 512 ETH of bond; with 8 ETH minipools, 128 validators requires 1,024 ETH.

While these amounts are substantial, they are achievable for seasoned home stakers or DVT clusters. These operators would face full custody node bandwidth requirements despite having economic exposure far below 4,096 ETH.

The protocol does not require thousands of full custody nodes for robust data availability. Research indicates approximately 750 nodes currently operate with 100+ validators, and roughly 200 nodes operate with 1000+ validators. An 8x increase to the threshold would still preserve hundreds of full custody nodes, sufficient to ensure reconstruction.

Additionally, the necessity for full custody nodes for reconstruction is temporary. Future protocol upgrades intend to implement partial reconstruction (row by row), reducing reliance on full custody nodes.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Parameters

| Constant | Current Value | New Value |
|----------|--------------|-----------|
| `BALANCE_PER_ADDITIONAL_CUSTODY_GROUP` | `32000000000` (32 ETH in Gwei) | `256000000000` (256 ETH in Gwei) |

### Derived Values

| Metric | Current | New |
|--------|---------|-----|
| Full custody node threshold (ETH) | 4,096 | 32,768 |
| Full custody node threshold (min-balance validators) | 128 | 1,024 |
| ETH per additional custody group | 32 | 256 |

### Custody Calculation

The custody group count calculation remains unchanged. Given a node with total effective balance across all its validators:

```python
def get_validators_custody_requirement(
    state: BeaconState, validator_indices: Sequence[ValidatorIndex]
) -> uint64:
    total_node_balance = sum(
        state.validators[index].effective_balance for index in validator_indices
    )
    count = total_node_balance // BALANCE_PER_ADDITIONAL_CUSTODY_GROUP
    return min(max(count, VALIDATOR_CUSTODY_REQUIREMENT), NUMBER_OF_CUSTODY_GROUPS)
```

With the new constant:

- A node with less than 2,304 ETH custodies `VALIDATOR_CUSTODY_REQUIREMENT` (8) groups
- A node with 2,304+ ETH begins custodying additional groups (`2304 // 256 = 9`)
- A node with 32,768+ ETH custodies all 128 groups (full custody node)

## Rationale

### Why 256 ETH?

The value of 256 ETH (2^8 × 10^9 Gwei) was chosen to:

1. **Maintain sufficient full custody nodes**: With approximately 200 nodes currently operating 1000+ validators, the network retains hundreds of full custody nodes even at the higher threshold.

2. **Protect home stakers**: The threshold of 1,024 validators (32,768 ETH) ensures that operators with tens of validators—typical for solo stakers using fractional staking protocols—do not face full custody node bandwidth requirements. As we increase the blob throughput, these nodes might otherwise fall off the network.

## Backwards Compatibility

This change does not require a hard fork. It is a configuration-only change to the `BALANCE_PER_ADDITIONAL_CUSTODY_GROUP` constant.

Per the consensus specs, nodes SHOULD NOT decrease their advertised `custody_group_count` during operation. This means:

- **Existing running nodes** will continue custodying at their current level after the configuration update. Their behavior does not change.
- **Operators facing bandwidth constraints** can re-sync their node to obtain the lower custody requirement.
- **New nodes** joining the network will start with custody calculated using the new constant.

This gradual transition avoids any disruption to data availability while allowing operators to opt into lower custody requirements as needed.

## Test Cases

### Custody Group Count Calculations

| Effective Balance (ETH) | Current Custody Groups | New Custody Groups |
|------------------------|----------------------|-------------------|
| 32 | 8 | 8 |
| 64 | 8 | 8 |
| 128 | 8 | 8 |
| 256 | 8 | 8 |
| 512 | 16 | 8 |
| 1,024 | 32 | 8 |
| 4,096 | 128 (full custody node) | 16 |
| 8,192 | 128 (full custody node) | 32 |
| 32,768 | 128 (full custody node) | 128 (full custody node) |

## Reference Implementation

The change requires updating a single constant in the consensus layer configuration:

```yaml
# configs/mainnet.yaml
BALANCE_PER_ADDITIONAL_CUSTODY_GROUP: 256000000000  # 256 ETH in Gwei
```

The custody calculation function requires no changes.

## Security Considerations

The primary security consideration is that fewer nodes will custody all columns. However, estimates place the number of nodes with 1000+ validators at 200+, providing a robust full custody node backbone even at the higher threshold. In addition, L2 operators are still expected to run full custody nodes.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
