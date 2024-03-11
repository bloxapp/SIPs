| Author                    | Title                           | Category | Status              |
| ------------------------- | ------------------------------- | -------- | ------------------- |
| Moshe Revah (@moshe-blox) | Support externally built blocks | Core     | approved |

## Summary

Support the production and proposal of blocks built by external entities, allowing operators to access blocks from a marketplace of builders such as [Flashbots](https://boost-relay.flashbots.net/).

## Rational & Design Goals

Block building is a computationally intensive task, and even more so when blockspace utilization and profit maximization are in mind. Additionally, many of the more profitable transactions, so-called MEV, are private and only accessible to validators through external block builders.

Currently, operators wishing to offer competitive returns must be sophisticated block builders with private transaction flows. This SIP aims to change that by allowing them to register with and propose blocks from external builders who comply to [ethereum/builder-specs](https://github.com/ethereum/builder-specs).

## Specification

### Configuration

Operators who wish to propose blinded blocks must configure their SSV node to do so, otherwise it would only propose standard blocks and reject QBFT proposals containing blinded blocks.

Therefore, validators should select operators with matching configuration.

### Validator registration

#### Background

Builders require validators to publish a [`SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration) to set their `fee_recipient` and `gas_limit` preferences

When building a block for a validator, builders pay the `fee_recipient` specified in the registration with the highest timestamp. If there is no registration, the builder should refuse to build a block altogether.

In the wild, some Ethereum validator clients currently publish this message every epoch.

#### Fee recipient addresses

Owners may set the preferred `fee_recipient` address for their validators by calling `setFeeRecipientAddress` in the `SSVNetwork` contract, which emits a `FeeRecipientAddressUpdated` event.

```solidity
function setFeeRecipientAddress(address feeRecipientAddress) external;

event FeeRecipientAddressUpdated(address indexed owner, address recipientAddress);
```

Owners may repeat this call as their preference changes over time.

If an owner has never set called `setFeeRecipientAddress`, their execution address would be used instead.

#### Constructing `ValidatorRegistration`

```go
type ValidatorRegistration struct {
	FeeRecipient bellatrix.ExecutionAddress
	GasLimit     uint64
	Timestamp    time.Time
	Pubkey       phase0.BLSPubKey
}
```

When constructing a `ValidatorRegistration` message for a validator, operators must populate the following fields:

- `FeeRecipient`: the `fee_recipient` address from the most recent event emitted by the validator's owner, or the validator's owner address (if no event was emitted).
- `GasLimit`: determined by social consensus.
- `Timestamp`: the time of the first slot of the current epoch.
- `Pubkey`: the public key of the validator.

Unlike in standard validator clients, in SSV, gas limits are set by operators rather than validators. Since at least `quorum` signatures are needed for BLS aggregation, it is crucial for operators to sign using the same `GasLimit`.

Ideally, operators align their gas limits with the current gas limit in Ethereum and coordinate to modify it at the same time when it changes.

#### Signing `ValidatorRegistration`

At the start of every slot, operators select their active validators with `ShouldRegisterValidatorAtSlot`, and for those, collectively produce a [`SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration) by aggregating BLS partial signatures.

```go
ValidatorRegistrationSlotInterval = 10 * SlotsPerEpoch

func ShouldRegisterValidatorAtSlot(index phase0.ValidatorIndex, slot phase0.Slot) bool {
    return (index + slot) % ValidatorRegistrationSlotInterval == 0
}
```

#### Publishing `SignedValidatorRegistration`

Every epoch, operators should publish the most recent successfully signed registration for their entire validator set.

This SIP suggests to spread registrations between slots. For example, at every slot, operators may publish only for a fraction of their validators (corresponding to `1/SlotsPerEpoch`), ensuring a more evenly distributed load on the Beacon node and builder(s).

> Note: Since relays/builders can have some delay with registrations, producing a blinded block before and shortly thereafter publishing the first registration may result in either:
>
> 1. Builder is unfamiliar with the validator yet and doesn't build a block, and Beacon node falls back to locally-built block from it's execution layer.
> 2. Builder doesn't reward the validator's `fee_recipient` because it isn't aware of it yet.
> 3. Builder rewards a potentially different `fee_recipient` from the validator's latest registration (such as a registration prior to onboarding to SSV.)

### Blinded block proposals

When a validator has a proposal duty, their operators:

1. Produce a `BlindedBeaconBlock` from their Beacon node at [/eth/v1/validator/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Validator/produceBlindedBlock)
2. Reach consensus on and sign the round leader's `BlindedBeaconBlock`
3. Submit the `SignedBlindedBeaconBlock` to their Beacon nodes at [/eth/v1/beacon/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Beacon/publishBlindedBlock)

Under the hood, each operator's Beacon node would attempt to unblind the block from it's connected builder(s) by revealing the signature to them, which would fail if they haven't built or don't know of this block.

Therefore, for unblinding to succeed, operators' Beacon nodes must share the same builder(s) with the round leader's Beacon node.

Since SSV can't enforce a specific set of builder(s) at the protocol-level, operators would have to reach social consensus on which builder(s) to configure their Beacon nodes with.

#### Locally-built blinded blocks

Sometimes, a Beacon node falls back to a locally-built `BlindedBeaconBlock` from it's execution layer, either because it's configured to do so under certain conditions (profitability threshold; chain is unstable) or because the builder(s) are unavailable.

A locally-built blinded block can only be unblinded by the same Beacon node which built it, which unfortunately means that only the round leader would successfully submit the block, thereby reducing the total network outreach of the proposal down to a single Beacon node.

We should consider pushing operators to configure their node for less harsh fallback conditions, so that more of their blocks are externally-built and can be successfully submitted by other operators' Beacon nodes.

#### Blinded block validation

Nowadays, there's no meaningful validation we can do on blinded blocks, but there's [active discussion](https://github.com/flashbots/mev-boost/issues/99) about requiring builders to attach payment proofs on bids so that they can't cheat and reward themselves instead of the validator's `fee_recipient`.

Ideally, payment proof validation should be handled by [mev-boost](https://github.com/flashbots/mev-boost), as it already examines the builders' bids, in which case operators would only have to ensure their Beacon nodes are always updated with the latest [`builder-specs/SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration).
