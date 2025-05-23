| ACP           | 77                                                                                        |
| :------------ | :---------------------------------------------------------------------------------------- |
| **Title**     | Reinventing Subnets                                                                       |
| **Author(s)** | Dhruba Basu ([@dhrubabasu](https://github.com/dhrubabasu))                                |
| **Status**    | Activated ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/78)) |
| **Track**     | Standards                                                                                 |
| **Replaces**  | [ACP-13](../13-subnet-only-validators/README.md)                                          |

## Abstract

Overhaul Subnet creation and management to unlock increased flexibility for Subnet creators by:

- Separating Subnet validators from Primary Network validators (Primary Network Partial Sync, Removal of 2000 $AVAX requirement)
- Moving ownership of Subnet validator set management from P-Chain to Subnets (ERC-20/ERC-721/Arbitrary Staking, Staking Reward Management)
- Introducing a continuous P-Chain fee mechanism for Subnet validators (Continuous Subnet Staking)

This ACP supersedes [ACP-13](../13-subnet-only-validators/README.md) and borrows some of its language.

## Motivation

Each node operator must stake at least 2000 $AVAX ($70k at time of writing) to first become a Primary Network validator before they qualify to become a Subnet validator. Most Subnets aim to launch with at least 8 Subnet validators, which requires staking 16000 $AVAX ($560k at time of writing). All Subnet validators, to satisfy their role as Primary Network validators, must also [allocate 8 AWS vCPU, 16 GB RAM, and 1 TB storage](https://github.com/ava-labs/avalanchego/blob/master/README.md#installation) to sync the entire Primary Network (X-Chain, P-Chain, and C-Chain) and participate in its consensus, in addition to whatever resources are required for each Subnet they are validating.

Regulated entities that are prohibited from validating permissionless, smart contract-enabled blockchains (like the C-Chain) cannot launch a Subnet because they cannot opt-out of Primary Network Validation. This deployment blocker prevents a large cohort of Real World Asset (RWA) issuers from bringing unique, valuable tokens to the Avalanche Ecosystem (that could move between C-Chain <> Subnets using Avalanche Warp Messaging/Teleporter).

A widely validated Subnet that is not properly metered could destabilize the Primary Network if usage spikes unexpectedly. Underprovisioned Primary Network validators running such a Subnet may exit with an OOM exception, see degraded disk performance, or find it difficult to allocate CPU time to P/X/C-Chain validation. The inverse also holds for Subnets with the Primary Network (where some undefined behavior could bring a Subnet offline).

Although the fee paid to the Primary Network to operate a Subnet does not go up with the amount of activity on the Subnet, the fixed, upfront cost of setting up a Subnet validator on the Primary Network deters new projects that prefer smaller, even variable, costs until demand is observed. _Unlike L2s that pay some increasing fee (usually denominated in units per transaction byte) to an external chain for data availability and security as activity scales, Subnets provide their own security/data availability and the only cost operators must pay from processing more activity is the hardware cost of supporting additional load._

Elastic Subnets, introduced in [Banff](https://medium.com/avalancheavax/banff-elastic-subnets-44042f41e34c), enabled Subnet creators to activate Proof-of-Stake validation and uptime-based rewards using their own token. However, this token is required to be an ANT (created on the X-Chain) and locked on the P-Chain. All staking rewards were distributed on the P-Chain with the reward curve being defined in the `TransformSubnetTx` and, once set, was unable to be modified.

With no Elastic Subnets live on Mainnet, it is clear that Permissionless Subnets as they stand today could be more desirable. There are many successful Permissioned Subnets in production but many Subnet creators have raised the above as points of concern. In summary, the Avalanche community could benefit from a more flexible and affordable mechanism to launch Permissionless Subnets.

### A Note on Nomenclature

Avalanche Subnets are subnetworks validated by a subset of the Primary Network validator set. The new network creation flow outlined in this ACP does not require any intersection between the new network's validator set and the Primary Network's validator set. Moreover, the new networks have greater functionality and sovereignty than Subnets. To distinguish between these two kinds of networks, the community has been referring to these new networks as _Avalanche Layer 1s_, or L1s for short.

All networks created through the old network creation flow will continue to be referred to as Avalanche Subnets.

## Specification

At a high-level, L1s can manage their validator sets externally to the P-Chain by setting the blockchain ID and address of their _validator manager_. The P-Chain will consume Warp messages that modify the L1's validator set. To confirm modification of the L1's validator set, the P-Chain will also produce Warp messages. L1 validators are not required to validate the Primary Network, and do not have the same 2000 $AVAX stake requirement that Subnet validators have. To maintain an active L1 validator, a continuous fee denominated in $AVAX is assessed. L1 validators are only required to sync the P-Chain (not X/C-Chain) in order to track validator set changes and support cross-L1 communication.

### P-Chain Warp Message Payloads

To enable management of an L1's validator set externally to the P-Chain, Warp message verification will be added to the [`PlatformVM`](https://github.com/ava-labs/avalanchego/tree/master/vms/platformvm). For a Warp message to be considered valid by the P-Chain, at least 67% of the `sourceChainID`'s weight must have participated in the aggregate BLS signature. This is equivalent to the threshold set for the C-Chain. A future ACP may be proposed to support modification of this threshold on a per-L1 basis.

The following Warp message payloads are introduced on the P-Chain:

- `SubnetToL1ConversionMessage`
- `RegisterL1ValidatorMessage`
- `L1ValidatorRegistrationMessage`
- `L1ValidatorWeightMessage`

The method of requesting signatures for these messages is left unspecified. A viable option for supporting this functionality is laid out in [ACP-118](../118-warp-signature-request/README.md) with the `SignatureRequest` message.

All node IDs contained within the message specifications are represented as variable length arrays such that they can support new node IDs types should the P-Chain add support for them in the future.

The serialization of each of these messages is as follows.

#### `SubnetToL1ConversionMessage`

The P-Chain can produce a `SubnetToL1ConversionMessage` for consumers (i.e. validator managers) to be aware of the initial validator set.

The following serialization is defined as the `ValidatorData`:

|          Field |       Type |                     Size |
| -------------: | ---------: | -----------------------: |
|       `nodeID` |   `[]byte` |  4 + len(`nodeID`) bytes |
| `blsPublicKey` | `[48]byte` |                 48 bytes |
|       `weight` |   `uint64` |                  8 bytes |
|                |            | 60 + len(`nodeID`) bytes |

The following serialization is defined as the `ConversionData`:

|            Field |              Type |                                                       Size |
| ---------------: | ----------------: | ---------------------------------------------------------: |
|        `codecID` |          `uint16` |                                                    2 bytes |
|       `subnetID` |        `[32]byte` |                                                   32 bytes |
| `managerChainID` |        `[32]byte` |                                                   32 bytes |
| `managerAddress` |          `[]byte` |                            4 + len(`managerAddress`) bytes |
|     `validators` | `[]ValidatorData` |                          4 + sum(`validatorLengths`) bytes |
|                  |                   | 74 + len(`managerAddress`) + sum(`validatorLengths`) bytes |

- `codecID` is the codec version used to serialize the payload, and is hardcoded to `0x0000`
- `sum(validatorLengths)` is the sum of the lengths of `ValidatorData` serializations included in `validators`.
- `subnetID` identifies the Subnet that is being converted to an L1 (described further below).
- `managerChainID` and `managerAddress` identify the validator manager for the newly created L1. This is the (blockchain ID, address) tuple allowed to send Warp messages to modify the L1's validator set.
- `validators` are the initial continuous-fee-paying validators for the given L1.

The `SubnetToL1ConversionMessage` is specified as an `AddressedCall` with `sourceChainID` set to the P-Chain ID, the `sourceAddress` set to an empty byte array, and a payload of:

|          Field |       Type |     Size |
| -------------: | ---------: | -------: |
|      `codecID` |   `uint16` |  2 bytes |
|       `typeID` |   `uint32` |  4 bytes |
| `conversionID` | `[32]byte` | 32 bytes |
|                |            | 38 bytes |

- `codecID` is the codec version used to serialize the payload, and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000000` for this message
- `conversionID` is the SHA256 hash of the `ConversionData` from a given `ConvertSubnetToL1Tx`

#### `RegisterL1ValidatorMessage`

The P-Chain can consume a `RegisterL1ValidatorMessage` from validator managers through a `RegisterL1ValidatorTx` to register an addition to the L1's validator set.

The following is the serialization of a `PChainOwner`:

|       Field |         Type |                             Size |
| ----------: | -----------: | -------------------------------: |
| `threshold` |     `uint32` |                          4 bytes |
| `addresses` | `[][20]byte` | 4 + len(`addresses`) \* 20 bytes |
|             |              | 8 + len(`addresses`) \* 20 bytes |

- `threshold` is the number of `addresses` that must provide a signature for the `PChainOwner` to authorize an action.
- Validation criteria:
  - If `threshold` is `0`, `addresses` must be empty
  - `threshold` <= len(`addresses`)
  - Entries of `addresses` must be unique and sorted in ascending order

The `RegisterL1ValidatorMessage` is specified as an `AddressedCall` with a payload of:

|                   Field |          Type |                                                                      Size |
| ----------------------: | ------------: | ------------------------------------------------------------------------: |
|               `codecID` |      `uint16` |                                                                   2 bytes |
|                `typeID` |      `uint32` |                                                                   4 bytes |
|              `subnetID` |    `[32]byte` |                                                                  32 bytes |
|                `nodeID` |      `[]byte` |                                                   4 + len(`nodeID`) bytes |
|          `blsPublicKey` |    `[48]byte` |                                                                  48 bytes |
|                `expiry` |      `uint64` |                                                                   8 bytes |
| `remainingBalanceOwner` | `PChainOwner` |                                          8 + len(`addresses`) \* 20 bytes |
|          `disableOwner` | `PChainOwner` |                                          8 + len(`addresses`) \* 20 bytes |
|                `weight` |      `uint64` |                                                                   8 bytes |
|                         |               | 122 + len(`nodeID`) + (len(`addresses1`) + len(`addresses2`)) \* 20 bytes |

- `codecID` is the codec version used to serialize the payload, and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000001` for this payload
- `subnetID`, `nodeID`, `weight`, and `blsPublicKey` are for the validator being added
- `expiry` is the time at which this message becomes invalid. As of a P-Chain timestamp `>= expiry`, this Avalanche Warp Message can no longer be used to add the `nodeID` to the validator set of `subnetID`
- `remainingBalanceOwner` is the P-Chain owner where leftover $AVAX from the validator's Balance will be issued to when this validator it is removed from the validator set.
- `disableOwner` is the only P-Chain owner allowed to disable the validator using `DisableL1ValidatorTx`, specified below.

#### `L1ValidatorRegistrationMessage`

The P-Chain can produce an `L1ValidatorRegistrationMessage` for consumers to verify that a validation period has either begun or has been invalidated.

The `L1ValidatorRegistrationMessage` is specified as an `AddressedCall` with `sourceChainID` set to the P-Chain ID, the `sourceAddress` set to an empty byte array, and a payload of:

|          Field |       Type |     Size |
| -------------: | ---------: | -------: |
|      `codecID` |   `uint16` |  2 bytes |
|       `typeID` |   `uint32` |  4 bytes |
| `validationID` | `[32]byte` | 32 bytes |
|   `registered` |     `bool` |   1 byte |
|                |            | 39 bytes |

- `codecID` is the codec version used to serialize the payload, and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000002` for this message
- `validationID` identifies the validator for the message
- `registered` is a boolean representing the status of the `validationID`. If true, the `validationID` corresponds to a validator in the current validator set. If false, the `validationID` does not correspond to a validator in the current validator set, and never will in the future.

#### `L1ValidatorWeightMessage`

The P-Chain can consume an `L1ValidatorWeightMessage` through a `SetL1ValidatorWeightTx` to update the weight of an existing validator. The P-Chain can also produce an `L1ValidatorWeightMessage` for consumers to verify that the validator weight update has been effectuated.

The `L1ValidatorWeightMessage` is specified as an `AddressedCall` with the following payload. When sent from the P-Chain, the `sourceChainID` is set to the P-Chain ID, and the `sourceAddress` is set to an empty byte array.

|          Field |       Type |     Size |
| -------------: | ---------: | -------: |
|      `codecID` |   `uint16` |  2 bytes |
|       `typeID` |   `uint32` |  4 bytes |
| `validationID` | `[32]byte` | 32 bytes |
|        `nonce` |   `uint64` |  8 bytes |
|       `weight` |   `uint64` |  8 bytes |
|                |            | 54 bytes |

- `codecID` is the codec version used to serialize the payload, and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000003` for this message
- `validationID` identifies the validator for the message
- `nonce` is a strictly increasing number that denotes the latest validator weight update and provides replay protection for this transaction
- `weight` is the new `weight` of the validator

### New P-Chain Transaction Types

Both before and after this ACP, to create a Subnet, a `CreateSubnetTx` must be issued on the P-Chain. This transaction includes an `Owner` field which defines the key that today can be used to authorize any validator set additions (`AddSubnetValidatorTx`) or removals (`RemoveSubnetValidatorTx`).

To be considered a permissionless network, or Avalanche Layer 1:

- This `Owner` key must no longer have the ability to modify the validator set.
- New transaction types must support modification of the validator set via Warp messages.

The following new transaction types are introduced on the P-Chain to support this functionality:

- `ConvertSubnetToL1Tx`
- `RegisterL1ValidatorTx`
- `SetL1ValidatorWeightTx`
- `DisableL1ValidatorTx`
- `IncreaseL1ValidatorBalanceTx`

#### `ConvertSubnetToL1Tx`

To convert a Subnet into an L1, a `ConvertSubnetToL1Tx` must be issued to set the `(chainID, address)` pair that will manage the L1's validator set. The `Owner` key defined in `CreateSubnetTx` must provide a signature to authorize this conversion.

The `ConvertSubnetToL1Tx` specification is:

```golang
type PChainOwner struct {
    // The threshold number of `Addresses` that must provide a signature in order for
    // the `PChainOwner` to be considered valid.
    Threshold uint32 `json:"threshold"`
    // The 20-byte addresses that are allowed to sign to authenticate a `PChainOwner`.
    // Note: It is required for:
    //       - len(Addresses) == 0 if `Threshold` is 0.
    //       - len(Addresses) >= `Threshold`
    //       - The values in Addresses to be sorted in ascending order.
    Addresses []ids.ShortID `json:"addresses"`
}

type L1Validator struct {
    // NodeID of this validator
    NodeID []byte `json:"nodeID"`
    // Weight of this validator used when sampling
    Weight uint64 `json:"weight"`
    // Initial balance for this validator
    Balance uint64 `json:"balance"`
    // [Signer] is the BLS public key and proof-of-possession for this validator.
    // Note: We do not enforce that the BLS key is unique across all validators.
    //       This means that validators can share a key if they so choose.
    //       However, a NodeID + L1 does uniquely map to a BLS key
    Signer signer.ProofOfPossession `json:"signer"`
    // Leftover $AVAX from the [Balance] will be issued to this
    // owner once it is removed from the validator set.
    RemainingBalanceOwner PChainOwner `json:"remainingBalanceOwner"`
    // The only owner allowed to disable this validator on the P-Chain.
    DisableOwner PChainOwner `json:"disableOwner"`
}

type ConvertSubnetToL1Tx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID of the Subnet to transform
    // Restrictions:
    // - Must not be the Primary Network ID
    Subnet ids.ID `json:"subnetID"`
    // BlockchainID where the validator manager lives
    ChainID ids.ID `json:"chainID"`
    // Address of the validator manager
    Address []byte `json:"address"`
    // Initial continuous-fee-paying validators for the L1
    Validators []L1Validator `json:"validators"`
    // Authorizes this conversion
    SubnetAuth verify.Verifiable `json:"subnetAuthorization"`
}
```

After this transaction is accepted, `CreateChainTx` and `AddSubnetValidatorTx` are disabled on the Subnet. The only action that the `Owner` key is able to take is removing Subnet validators with `RemoveSubnetValidatorTx` that had been added using `AddSubnetValidatorTx`. Unless removed by the `Owner` key, any Subnet validators added previously with an `AddSubnetValidatorTx` will continue to validate the Subnet until their [`End`](https://github.com/ava-labs/avalanchego/blob/a1721541754f8ee23502b456af86fea8c766352a/vms/platformvm/txs/validator.go#L27) time is reached. Once all Subnet validators added with `AddSubnetValidatorTx` are no longer in the validator set, the `Owner` key is powerless. `RegisterL1ValidatorTx` and `SetL1ValidatorWeightTx` must be used to manage the L1's validator set.

The `validationID` for validators added through `ConvertSubnetToL1Tx` is defined as the SHA256 hash of the 36 bytes resulting from concatenating the 32 byte `subnetID` with the 4 byte `validatorIndex` (index in the `Validators` array within the transaction).

Once this transaction is accepted, the P-Chain must be willing sign a `SubnetToL1ConversionMessage` with a `conversionID` corresponding to `ConversionData` populated with the values from this transaction.

#### `RegisterL1ValidatorTx`

After a `ConvertSubnetToL1Tx` has been accepted, new validators can only be added by using a `RegisterL1ValidatorTx`. The specification of this transaction is:

```golang
type RegisterL1ValidatorTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Balance <= sum($AVAX inputs) - sum($AVAX outputs) - TxFee.
    Balance uint64 `json:"balance"`
    // [Signer] is a BLS signature proving ownership of the BLS public key specified
    // below in `Message` for this validator.
    // Note: We do not enforce that the BLS key is unique across all validators.
    //       This means that validators can share a key if they so choose.
    //       However, a NodeID + L1 does uniquely map to a BLS key
    Signer [96]byte `json:"signer"`
    // A RegisterL1ValidatorMessage payload
    Message warp.Message `json:"message"`
}
```

The `validationID` of validators added via `RegisterL1ValidatorTx` is defined as the SHA256 hash of the `Payload` of the `AddressedCall` in `Message`.

When a `RegisterL1ValidatorTx` is accepted on the P-Chain, the validator is added to the L1's validator set. A `minNonce` field corresponding to the `validationID` will be stored on addition to the validator set (initially set to `0`). This field will be used when validating the `SetL1ValidatorWeightTx` defined below.

This `validationID` will be used for replay protection. Used `validationID`s will be stored on the P-Chain. If a `RegisterL1ValidatorTx`'s `validationID` has already been used, the transaction will be considered invalid. To prevent storing an unbounded number of `validationID`s, the `expiry` of the `RegisterL1ValidatorMessage` is required to be no more than 24 hours in the future of the time the transaction is issued on the P-Chain. Any `validationIDs` corresponding to an expired timestamp can be flushed from the P-Chain's state.

L1s are responsible for defining the procedure on how to retrieve the above information from prospective validators.

An EVM-compatible L1 may choose to implement this step like so:

- Use the number of tokens the user has staked into a smart contract on the L1 to determine the weight of their validator
- Require the user to submit an on-chain transaction with their validator information
- Generate the Warp message

For a `RegisterL1ValidatorTx` to be valid, `Signer` must be a valid proof-of-possession of the `blsPublicKey` defined in the `RegisterL1ValidatorMessage` contained in the transaction.

After a `RegisterL1ValidatorTx` is accepted, the P-Chain must be willing to sign an `L1ValidatorRegistrationMessage` for the given `validationID` with `registered` set to `true`. This remains the case until the time at which the validator is removed from the validator set using a `SetL1ValidatorWeightTx`, as described below.

When it is known that a given `validationID` _is not and never will be_ registered, the P-Chain must be willing to sign an `L1ValidatorRegistrationMessage` for the `validationID` with `registered` set to `false`. This could be the case if the `expiry` time of the message has passed prior to the message being delivered in a `RegisterL1ValidatorTx`, or if the validator was successfully registered and then later removed. This enables the P-Chain to prove to validator managers that a validator has been removed or never added. The P-Chain must refuse to sign any `L1ValidatorRegistrationMessage` where the `validationID` does not correspond to an active validator and the `expiry` is in the future.

#### `SetL1ValidatorWeightTx`

`SetL1ValidatorWeightTx` is used to modify the voting weight of a validator. The specification of this transaction is:

```golang
type SetL1ValidatorWeightTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // An L1ValidatorWeightMessage payload
    Message warp.Message `json:"message"`
}
```

Applications of this transaction could include:

- Increase the voting weight of a validator if a delegation is made on the L1
- Increase the voting weight of a validator if the stake amount is increased (by staking rewards for example)
- Decrease the voting weight of a misbehaving validator
- Remove an inactive validator

The validation criteria for `L1ValidatorWeightMessage` is:

- `nonce >= minNonce`. Note that `nonce` is not required to be incremented by `1` with each successive validator weight update.
- When `minNonce == MaxUint64`, `nonce` must be `MaxUint64` and `weight` must be `0`. This prevents L1s from being unable to remove `nodeID` in a subsequent transaction.
- If `weight == 0`, the validator being removed must not be the last one in the set. If all validators are removed, there are no valid Warp messages that can be produced to register new validators through `RegisterL1ValidatorMessage`. With no validators, block production will halt and the L1 is unrecoverable. This validation criteria serves as a guardrail against this situation. A future ACP can remove this guardrail as users get more familiar with the new L1 mechanics and tooling matures to fork an L1.

When `weight != 0`, the weight of the validator is updated to `weight` and `minNonce` is updated to `nonce + 1`.

When `weight == 0`, the validator is removed from the validator set. All state related to the validator, including the `minNonce` and `validationID`, are reaped from the P-Chain state. Tracking these post-removal is not required since `validationID` can never be re-initialized due to the replay protection provided by `expiry` in `RegisterL1ValidatorTx`. Any unspent $AVAX in the validator's `Balance` will be issued in a single UTXO to the `RemainingBalanceOwner` for this validator. Recall that `RemainingBalanceOwner` is specified when the validator is first added to the L1's validator set (in either `ConvertSubnetToL1Tx` or `RegisterL1ValidatorTx`).

Note: There is no explicit `EndTime` for L1 validators added in a `ConvertSubnetToL1Tx` or `RegisterL1ValidatorTx`. The only time when L1 validators are removed from the L1's validator set is through this transaction when `weight == 0`.

#### `DisableL1ValidatorTx`

L1 validators can use `DisableL1ValidatorTx` to mark their validator as inactive. The specification of this transaction is:

```golang
type DisableL1ValidatorTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID corresponding to the validator
    ValidationID ids.ID `json:"validationID"`
    // Authorizes this validator to be disabled
    DisableAuth verify.Verifiable `json:"disableAuthorization"`
}
```

The `DisableOwner` specified for this validator must sign the transaction. Any unspent $AVAX in the validator's `Balance` will be issued in a single UTXO to the `RemainingBalanceOwner` for this validator. Recall that both `DisableOwner` and `RemainingBalanceOwner` are specified when the validator is first added to the L1's validator set (in either `ConvertSubnetToL1Tx` or `RegisterL1ValidatorTx`).

For full removal from an L1's validator set, a `SetL1ValidatorWeightTx` must be issued with weight `0`. To do so, a Warp message is required from the L1's validator manager. However, to support the ability to claim the unspent `Balance` for a validator without authorization is critical for failed L1s.

Note that this does not modify an L1's total staking weight. This transaction marks the validator as inactive, but does not remove it from the L1's validator set. Inactive validators can re-activate at any time by increasing their balance with an `IncreaseL1ValidatorBalanceTx`.

L1 creators should be aware that there is no notion of `MinStakeDuration` that is enforced by the P-Chain. It is expected that L1s who choose to enforce a `MinStakeDuration` will lock the validator's Stake for the L1's desired `MinStakeDuration`.

#### `IncreaseL1ValidatorBalanceTx`

L1 validators are required to maintain a non-zero balance used to pay the continuous fee on the P-Chain in order to be considered active. The `IncreaseL1ValidatorBalanceTx` can be used by anybody to add additional $AVAX to the `Balance` to a validator. The specification of this transaction is:

```golang
type IncreaseL1ValidatorBalanceTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID corresponding to the validator
    ValidationID ids.ID `json:"validationID"`
    // Balance <= sum($AVAX inputs) - sum($AVAX outputs) - TxFee
    Balance uint64 `json:"balance"`
}
```

If the validator corresponding to `ValidationID` is currently inactive (`Balance` was exhausted or `DisableL1ValidatorTx` was issued), this transaction will move them back to the active validator set.

Note: The $AVAX added to `Balance` can be claimed at any time by the validator using `DisableL1ValidatorTx`.

### Bootstrapping L1 Nodes

Bootstrapping a node/validator is the process of securely recreating the latest state of the blockchain locally. At the end of this process, the local state of a node/validator must be in sync with the local state of other virtuous nodes/validators. The node/validator can then verify new incoming transactions and reach consensus with other nodes/validators.

To bootstrap a node/validator, a few critical questions must be answered: How does one discover peers in the network? How does one determine that a discovered peer is honestly participating in the network?

For standalone networks like the Avalanche Primary Network, this is done by connecting to a hardcoded [set](https://github.com/ava-labs/avalanchego/blob/master/genesis/bootstrappers.json) of trusted bootstrappers to then discover new peers. Ethereum calls their set [bootnodes](https://ethereum.org/developers/docs/nodes-and-clients/bootnodes).

Since L1 validators are not required to be Primary Network validators, a list of validator IPs to connect to (the functional bootstrappers of the L1) cannot be provided by simply connecting to the Primary Network validators. However, the Primary Network can enable nodes tracking an L1 to seamlessly connect to the validators by tracking and gossiping L1 validator IPs. L1s will not need to operate and maintain a set of bootstrappers and can rely on the Primary Network for peer discovery.

### Sidebar: L1 Sovereignty

After this ACP is activated, the P-Chain will no longer support staking of any assets other than $AVAX for the Primary Network. The P-Chain will not support the distribution of staking rewards for L1s. All staking-related operations for L1 validation must be managed by the L1's validator manager. The P-Chain simply requires a continuous fee per validator. If an L1 would like to manage their validator's balances on the P-Chain, it can cover the cost for all L1 validators by posting the $AVAX balance on the P-Chain. L1s can implement any mechanism they want to pay the continuous fee charged by the P-Chain for its participants.

The L1 has full ownership over its validator set, not the P-Chain. There are no restrictions on what requirements an L1 can have for validators to join. Any stake that is required to join the L1's validator set is not locked on the P-Chain. If a validator is removed from the L1's validator set via a `SetL1ValidatorWeightTx` with weight `0`, the stake will continue to be locked outside of the P-Chain. How each L1 handles stake associated with the validator is entirely left up to the L1 and can be treated independently to what happens on the P-Chain.

The relationship between the P-Chain and L1s provides a dynamic where L1s can use the P-Chain as an impartial judge to modify parameters (in addition to its existing role of helping to validate incoming Avalanche Warp Messages). If a validator is misbehaving, the L1 validators can collectively generate a BLS multisig to reduce the voting weight of a misbehaving validator. This operation is fully secured by the Avalanche Primary Network (225M $AVAX or $8.325B at the time of writing).

Follow-up ACPs could extend the P-Chain <> L1 relationship to include parametrization of the 67% threshold to enable L1s to choose a different threshold based on their security model (e.g. a simple majority of 51%).

### Continuous Fee Mechanism

Every additional validator on the P-Chain adds persistent load to the Avalanche Network. When a validator transaction is issued on the P-Chain, it is charged for the computational cost of the transaction itself but is not charged for the cost of an active validator over the time they are validating on the network (which may be indefinitely). This is a common problem in blockchains, spawning many state rent proposals in the broader blockchain space to address it. The following fee mechanism takes advantage of the fact that each L1 validator uses the same amount of computation and charges each L1 validator the dynamic base fee for every discrete unit of time it is active.

To charge each L1 validator, the notion of a `Balance` is introduced. The `Balance` of a validator will be continuously charged during the time they are active to cover the cost of storing the associated validator properties (BLS key, weight, nonce) in memory and to track IPs (in addition to other services provided by the Primary Network). This `Balance` is initialized with the `RegisterL1ValidatorTx` that added them to the active validator set. `Balance` can be increased at any time using the `IncreaseL1ValidatorBalanceTx`. When this `Balance` reaches `0`, the validator will be considered "inactive" and will no longer participate in validating the L1. Inactive validators can be moved back to the active validator set at any time using the same `IncreaseL1ValidatorBalanceTx`. Once a validator is considered inactive, the P-Chain will remove these properties from memory and only retain them on disk. All messages from that validator will be considered invalid until it is revived using the `IncreaseL1ValidatorBalanceTx`. L1s can reduce the amount of inactive weight by removing inactive validators with the `SetL1ValidatorWeightTx` (`Weight` = 0).

Since each L1 validator is charged the same amount at each point in time, tracking the fees for the entire validator set is straight-forward. The accumulated dynamic base fee for the entire network is tracked in a single uint. This accumulated value should be equal to the fee charged if a validator was active from the time the accumulator was instantiated. The validator set is maintained in a priority queue. A pseudocode implementation of the continuous fee mechanism is provided below.

```python
# Pseudocode
class ValidatorQueue:
    def __init__(self, fee_getter):
        self.acc = 0
        self.queue = PriorityQueue()
        self.fee_getter = fee_getter

    # At each time period, increment the accumulator and
    # pop all validators from the top of the queue that
    # ran out of funds.

    # Note: The amount of work done in a single block
    # should be bounded to prevent a large number of
    # validator operations from happening at the same
    # time.
    def time_elapse(self, t):
        self.acc = self.acc + self.fee_getter(t)
        while True:
            vdr = self.queue.peek()
            if vdr.balance < self.acc:
                self.queue.pop()
                continue
            return

    # Validator was added
    def validator_enter(self, vdr):
        vdr.balance = vdr.balance + self.acc
        self.queue.add(vdr)

    # Validator was removed
    def validator_remove(self, vdrNodeID):
        vdr = find_and_remove(self.queue, vdrNodeID)
        vdr.balance = vdr.balance - self.acc
        vdr.refund() # Refund [vdr.balance] to [RemainingBalanceOwner]
        self.queue.remove()

    # Validator's balance was topped up
    def validator_increase(self, vdrNodeID, balance):
        vdr = find_and_remove(self.queue, vdrNodeID)
        vdr.balance = vdr.balance + balance
        self.queue.add(vdr)
```

#### Fee Algorithm

[ACP-103](../103-dynamic-fees/README.md) proposes a dynamic fee mechanism for transactions on the P-Chain. This mechanism is repurposed with minor modifications for the active L1 validator continuous fee.

At activation, the number of excess active L1 validators $x$ is set to `0`.

The fee rate per second for an active L1 validator is:

$$M \cdot \exp\left(\frac{x}{K}\right)$$

Where:

- $M$ is the minimum price for an active L1 validator
- $\exp\left(x\right)$ is an approximation of $e^x$ following the EIP-4844 specification

  ```python
  # Approximates factor * e ** (numerator / denominator) using Taylor expansion
  def fake_exponential(factor: int, numerator: int, denominator: int) -> int:
    i = 1
    output = 0
    numerator_accum = factor * denominator
    while numerator_accum > 0:
        output += numerator_accum
        numerator_accum = (numerator_accum * numerator) // (denominator * i)
        i += 1
    return output // denominator
  ```

- $K$ is a constant to control the rate of change for the L1 validator price

After every second, $x$ will be updated:

$$x = \max(x + (V - T), 0)$$

Where:

- $V$ is the number of active L1 validators
- $T$ is the target number of active L1 validators

Whenever $x$ increases by $K$, the price per active L1 validator increases by a factor of `~2.7`. If the price per active L1 validator gets too expensive, some active L1 validators will exit the active validator set, decreasing $x$, dropping the price. The price per active L1 validator constantly adjusts to make sure that, on average, the P-Chain has no more than $T$ active L1 validators.

#### Block Processing

Before processing the transactions inside a block, all validators that no longer have a sufficient (non-zero) balance are deactivated.

After processing the transactions inside a block, all validators that do not have a sufficient balance for the next second are deactivated.

##### Block Timestamp Validity Change

To ensure that validators are charged accurately, blocks are only considered valid if advancing the chain times would not cause a validator to have a negative balance.

This upholds the expectation that the number of L1 validators remains constant between blocks.

The block building protocol is modified to account for this change by first checking if the wall clock time removes any validator due to a lack of funds. If the wall clock time does not remove any L1 validators, the wall clock time is used to build the block. If it does, the time at which the first validator gets removed is used.

##### Fee Calculation

The total validator fee assessed in $\Delta t$ is:

```python
# Calculate the fee to charge over Δt
def cost_over_time(V:int, T:int, x:int, Δt: int) -> int:
    cost = 0
    for _ in range(Δt):
        x = max(x + V - T, 0)
        cost += fake_exponential(M, x, K)
    return cost
```

#### Parameters

The parameters at activation are:

| Parameter | Definition                                  | Value         |
| --------- | ------------------------------------------- | ------------- |
| $T$       | target number of validators                 | 10_000        |
| $C$       | capacity number of validators               | 20_000        |
| $M$       | minimum fee rate                            | 512 nAVAX/s   |
| $K$       | constant to control the rate of fee changes | 1_246_488_515 |

An $M$ of 512 nAVAX/s equates to ~1.33 AVAX/month to run an L1 validator, so long as the total number of continuous-fee-paying L1 validators stays at or below $T$.

$K$ was chosen to set the maximum fee doubling rate to ~24 hours. This is in the extreme case that the network has $C$ validators for prolonged periods of time; if the network has $T$+1 validators for example, the fee rate would double every ~27 years.

A future ACP can adjust the parameters to increase $T$, reduce $M$, and/or modify $K$.

#### User Experience

L1 validators are continuously charged a fee, albeit a small one. This poses a challenge for L1 validators: How do they maintain the balance over time?

Node clients should expose an API to track how much balance is remaining in the validator's account. This will provide a way for L1 validators to track how quickly it is going down and top-up when needed. A nice byproduct of the above design is the balance in the validator's account is claimable. This means users can top-up as much $AVAX as they want and rest-assured knowing they can always retrieve it if there is an excessive amount.

The expectation is that most users will not interact with node clients or track when or how much they need to top-up their validator account. Wallet providers will abstract away most of this process. For users who desire more convenience, L1-as-a-Service providers will abstract away all of this process.

## Backwards Compatibility

This new design for Subnets proposes a large rework to all L1-related mechanics. Rollout should be done on a going-forward basis to not cause any service disruption for live Subnets. All current Subnet validators will be able to continue validating both the Primary Network and whatever Subnets they are validating.

Any state execution changes must be coordinated through a mandatory upgrade. Implementors must take care to continue to verify the existing ruleset until the upgrade is activated. After activation, nodes should verify the new ruleset. Implementors must take care to only verify the presence of 2000 $AVAX prior to activation.

### Deactivated Transactions

- P-Chain

  - `TransformSubnetTx`

    After this ACP is activated, Elastic Subnets will be disabled. `TransformSubnetTx` will not be accepted post-activation. As there are no Mainnet Elastic Subnets, there should be no production impact with this deactivation.

### New Transactions

- P-Chain
  - `ConvertSubnetToL1Tx`
  - `RegisterL1ValidatorTx`
  - `SetL1ValidatorWeightTx`
  - `DisableL1ValidatorTx`
  - `IncreaseL1ValidatorBalanceTx`

## Reference Implementation

ACP-77 was implemented and will be merged into AvalancheGo behind the `Etna` upgrade flag. The full body of work can be found tagged with the `acp77` label [here](https://github.com/ava-labs/avalanchego/issues?q=sort%3Aupdated-desc+label%3Aacp77).

Since Etna is not yet activated, all new transactions introduced in ACP-77 will be rejected by AvalancheGo. If any modifications are made to ACP-77 as part of the ACP process, the implementation must be updated prior to activation.

## Security Considerations

This ACP introduces Avalanche Layer 1s, a new network type that costs significantly less than Avalanche Subnets. This can lead to a large increase in the number of networks and, by extension, the number of validators. Each additional validator adds consistent RAM usage to the P-Chain. However, this should be appropriately metered by the continuous fee mechanism outlined above.

With the sovereignty L1s have from the P-Chain, L1 staking tokens are not locked on the P-Chain. This poses a security consideration for L1 validators: Malicious chains can choose to remove validators at will and take any funds that the validator has locked on the L1. The P-Chain only provides the guarantee that L1 validators can retrieve the remaining $AVAX Balance for their validator via a `DisableL1ValidatorTx`. Any assets on the L1 is entirely under the purview of the L1. The onus is on L1 validators to vet the L1's security for any assets transferred onto the L1.

With a long window of expiry (24 hours) for the Warp message in `RegisterL1ValidatorTx`, spam of validator registration could lead to high memory pressure on the P-Chain. A future ACP can reduce the window of expiry if 24 hours proves to be a problem.

NodeIDs can be added to an L1's validator set involuntarily. However, it is important to note that any stake/rewards are _not_ at risk. For a node operator who was added to a validator set involuntarily, they would only need to generate a new NodeID via key rotation as there is no lock-up of any stake to create a NodeID. This is an explicit tradeoff for easier on-boarding of NodeIDs. This mirrors the Primary Network validators guarantee of no stake/rewards at risk.

The continuous fee mechanism outlined above does not apply to inactive L1 validators since they are not stored in memory. However, inactive L1 validators are persisted on disk which can lead to persistent P-Chain state growth. A future ACP can introduce a mechanism to decrease the rate of P-Chain state growth or provide a state expiry path to reduce the amount of P-Chain state.

## Acknowledgements

Special thanks to [@StephenButtolph](https://github.com/StephenButtolph), [@aaronbuchwald](https://github.com/aaronbuchwald), and [@patrick-ogrady](https://github.com/patrick-ogrady) for their feedback on these ideas. Thank you to the broader Ava Labs Platform Engineering Group for their feedback on this ACP prior to publication.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
