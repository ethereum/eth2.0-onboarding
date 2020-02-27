# Phase 0 for Humans [v0.10.0]

_Warning_: This a work in progress document with many broken links, some unfinished sections, and commands for including code segments. The entirety of the document is subject to change without warning.

## Introduction

This document is an aid to the specification of the [state transition function](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-chain-state-transition-function) and accompanying [data structures](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#containers) for Phase 0 - The Beacon Chain -- the core system level chain at the heart of Ethereum 2.0. 

The [primary Phase 0 specification](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md) is particularly terse, only defining much of the functionality via python code and comments. This document aims to provide a more verbose version of the specification, primarily aimed at helping onboard new contributors to Ethereum 2.0.

## Table of contents

[toc]

## High level overview

The beacon chain is the core system-level blockchain for Ethereum 2.0. The name has its origins in the idea of a [random beacon](https://csrc.nist.gov/projects/interoperable-randomness-beacons)-- a beacon that provides a source of randomness to the rest of the system -- but it could just as easily have been called the "system chain", "spine chain", or something similar.

A major part of the work of the beacon chain is storing and managing the registry of validators -- the set of participants who have placed the required stake of 32 Ether, and who are responsible for running the Ethereum 2.0 system.

This registry is used to:

- Assign validators their duties
- Finalize checkpoints
- Perform a protocol level random number generation (RNG)
- Progress the beacon chain
- Vote on the head of the chain for the fork choice
- Finalize checkpoints
- Link and vote in transitions/data of shard chains

The beacon chain is both the brains behind the operation and the scaffolding upon which the rest of the sharded system is built.

<!-- TODO: diagram of beacon chain, shards, shard transitions -->

The beacon chain's state ([`BeaconState`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-state)) is the core object around which the specification is built.  In addition to maintaining a hash-reference to the Ethereum 1 chain, the `BeaconState` encapsulates all of the information relating to:

- Who the validators are
- Which state the validators are in
- Which chain in the block tree this state belongs to

> **Note:** while there are several possible states a validator can be in, only those marked as **active** can take part in the Ethereum 2.0 protocol. This is why it's important to keep track of which state validators are in.

The linking in of shard chains has evolved over time from including a data root, to linking in more elaborate transition data at a per-block pace for all shards.

Improving the UX of cross-shard communication, has required a reduction in the initial shard count (from 1024 to 64). However this has been offset by an increase in other parameters -- such as increasing the maximum number of shards per slot from 16 to 64.

You can find a discussion of the new shard chain/crosslink structure [here](https://notes.ethereum.org/@vbuterin/HkiULaluS).

> **Note:** the attested inclusion of a shard transition is called a "crosslink".

To enable state transitions, the beacon chain has a `state_transition` function which takes as input a `BeaconState` (pre state) and a `BeaconBlock` and returns a new beacon state (what we call a post state).

Beginning with the genesis state, the post state of a block is considered valid if it passes all of the guards within the `state_transition` function.

> **Note:** the pre-state of a block is defined as being a valid post state of the previous block. This definition extends recursively all the way back to the genesis state. 

### Fork Choice Rule

Given a block tree, the [fork choice rule](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/fork-choice.md) provides a single chain (the canonical chain) and resulting state based upon recent messages from validators.

The fork choice rule consumes the block-tree along with the set of most recent attestations from active validators, and identifies a block as the current head of the chain.

>**Definition:** an attestation is a vote for a block proposal

LMD GHOST ("Latest Message Driven Greedy Heaviest-Observed Sub-Tree"), the fork choice rule of Eth2.0, considers which block the latest attestation from each validator points to and uses this to calculate the total balance that recursively attests to each block in the tree. 

This is done by setting the weight of a node in the tree to be the sum of the balances of the validators whose latest attestations were for that node or any descendent of that node.

The GHOST algorithm then starts at the base of the tree, selecting the child of the current node with the heaviest weight until reaching a leaf node. This leaf node is the head of the chain and recursively defines the canonical chain.

Concretely, validators are given the opportunity to produce a single [attestation](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestation) during an assigned slot at each epoch. The committed to `attestation.data.beacon_block_root` serves as a fork choice vote. A view takes into account all of the most recent of these votes from the current active validator set when computing the fork choice.

### Finality

In addition to voting on the canonical chain, validators also contribute to deciding finality of blocks --  a process that tells us when a Beacon block can be considered final and non-revertible.

In other words, whereas the fork choice rule allows us to choose a single canonical blockchain through the block tree, finality provides guarantees that certain blocks will remain within the canonical chain.

The beacon chain uses a [modified version of Casper FFG](https://github.com/ethereum/research/blob/master/papers/ffg%2Bghost/paper.pdf) for finality (read original Casper FFG paper for basic concepts [here](https://arxiv.org/pdf/1710.09437.pdf).

Casper provides "accountable safety" that certain blocks will always remain in the canonical chain unless some threshold of validators burn their staked capital. This is a "crypto-economic" claim of safety rather than a more traditional safety found in traditional distributed systems consensus algorithms.

Concretely, in order to help finalize blocks, validators are given the opportunity to produce a single [attestation](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestation) during an assigned slot at each epoch -- where an epoch is defined as the span of blocks between checkpoints.

>**Note:** the committed to `attestation.data.source` serves as the FFG source pair discussed in depth in ["Combining GHOST and Casper"](https://github.com/ethereum/research/blob/master/papers/ffg%2Bghost/paper.pdf), while the committed to `attestation.data.target` serves as the FFG target pair.

### Shard transitions

_Shards and crosslinks are not currently contained within the Phase 0 beacon chain. They are the major Phase 1 milestone. A brief discussion is included here in order to provide a more complete view._

To the Beacon Chain, shard transitions are small packages that reference the progression of a shard chain. These are voted on by attestations, and included (via attestations) into the Beacon Chain by block producers.

Crosslinks, references of shard transitions and data, are formed by committees as part of their validator duties. The latest successful crosslink of each shard serves as the root for that shard chain's fork choice. 

Through these crosslinks, asynchronous communication between shards is made possible on layer 1. 

In the normal case, each shard can be crosslinked into the beacon chain once per slot. However, if the validator count is low, crosslinks may occur less frequently (this is to preserve safe size of committees).

### Validator responsibilities

A detailed discussion of the responsibilities of validators in phase 0 can be found [here](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/validator.md#beacon-chain-responsibilities).

The three primary responsibilities in phase 0 are:

1. Attesting to the beacon chain (once per epoch).

2. Aggregating attestations from validators in the same committee (occasionally).

3. Creating beacon blocks when selected (infrequently).

Validators are split into "beacon committees" at each epoch (defined 1 epoch in advance to allow for preparation). Each committee is assigned to a slot (and shard in Phase 1). And each validator in the committee attests to the head of the beacon chain (and the recent data of their assigned shard) at their assigned slot.

In addition, a subset of the beacon committee is selected to aggregate attestations of similar `AttestationData` and to re-broadcast these aggregates to a global gossip channel for wider dissemination.

At each slot, a single beacon block proposer is selected (via `get_beacon_proposer_index`) from one of the active validators. Based on the Beacon RNG, committees are randomly shuffled, and proposers are randomly picked. Proposal assignments are only known within the current epoch, but even then, any amount of public lookahead on block production is cause for DoS concern.

Techniques for single secret leader election are being investigated for integration in subsequent phases of development. [This](https://eprint.iacr.org/2020/025.pdf) is an example of one such proposal.

Validators gain rewards by regularly attesting to the canonical beacon chain and incur penalties if they fail to do so. Proposers gain rewards for inclusion of attestations in block proposals.

In Phase 0, a validator can be _slashed_ (a more severe penalty) if they violate the Casper FFG rules or if they create two beacon blocks in one epoch.

Slashings and other events may prevent a validator from participating, but do not immediately result in the re-assignment of committees/shuffling. This is to ensure the stability of responsibilities during each epoch -- stability needs to be prioritized to ensure sufficient lookahead on duties.

Slashings are like normal voluntary exits, except that their withdrawal is delayed, and penalties are applied (based on total slashings at the time). The more validators coordinate in bad behavior, the more slashings, and the more severe the penalties (see `process_slashings`).

More details on slashing from a validator perspective can be found [here](https://github.com/ethereum/eth2.0-specs/blob/master/specs/validator/0_beacon-chain-validator.md#how-to-avoid-slashing).

## Parameters

### Constants

[Constants](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#constants) are global parameters that are not meant to change, even between chains. They document numbers used in different parts of the specification.

### Configurables

[Configurable global parameters](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#configuration) are used to enable testnets to experiment, avoid collisions, and provide the necessary flexibility in development.

The philosophy behind configuration is that changes to values should only happen between chains, not over time.

This means that in order to change functionality, additional constants must be introduced. For example, if there is a decision to change the slot time in Phase 1, rather than altering `SLOTS_PER_EPOCH`, we'd have to introduce a new constant -- `PHASE_1_SLOTS_PER_EPOCH`.

Note that transitioning from one phase to another is not a hard reset -- the system is still live. Additionally, security analysis and optimizations greatly benefit from compile-time guarantees.

[Two configurations](https://github.com/ethereum/eth2.0-specs/tree/v0.10.0/configs) are actively maintained. They provide client implementers with targets to maintain a shared testing and production focus.

1. `minimal`: A reduced version of Eth2. Used to speed up the dev iteration cycle, as mainnet can be resource intensive and slower paced.

2. `mainnet`: Has the intended mainnet parameters. Note however that these paramenters are not definitive until audit results, testnet analysis and community feedback are collected.

## Data structures

Data structures across the Eth2 specification are defined in terms of [Simple SerialiZe (SSZ)](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/ssz/simple-serialize.md) types: a type system that defines serialization and [merkleization (hash-tree-root)](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/ssz/simple-serialize.md#merkleization), focused on determinism and minimalism.

>**Definition:** determinism in this context means well definedness. Determinism matters because we use the SSZ type and hashing scheme (`hash_tree_root`) for consensus objects and hash chains. Since we're using it in consensus, there cannot be any ambiguity in the serialization spec. Some other (more common) serialization formats/types sometimes have corner cases that aren't fully spec'd and can end up with slightly different results depending on the implementation/machine.

The main benefit of SSZ-tree-hashing is its support for different tree depths when merkleizing the underlying data. This allows for any of the contents of complex data structures to be summarized in-place as a merkle root.

The full expansion can be proven for this root. Compositions, and partial datastructures can be proven as well.

>**Note:** when we merklize a container (B) within another container (A), we can replace the embedded B with it's merkle root, without affecting the merklization of A. In other words, in this tree structure, each sub-container / ssz-type is it's own sub-tree. In other words, because of the merklization rules, sub-trees can be replaced by their roots, without affecting the merklization of outer containers.

The type based tree-structure enables efficient proof navigation, and sophisticated caching techniques to be used on consensus objects (e.g. `BeaconState`). **Techniques like these greatly reduce the amount of hashing work that needs to be done at each slot, without compromising the data-type expressiveness of the consensus types.**

>**Note:** techniques like these allow us to do things like take a `BlockHeader` (with a `body_root`) or a full `Block` (with the entire `body` expressed) and get to the same block root for each. This is really nice because we can choose to use either succinct versions, or full expansions, depending on the use case. And because of the merklization rules, we can make succinct proofs about the contents of what a full expansion might be. For example, say we have a `block_header` (with a `body_root`), we can prove that an attestation is contained within the block body, without needing to have the entire block body at hand.

The serialization of SSZ is focused on determinism and efficient lookups. It is not a streaming encoding, but generally very efficient. And more standard than previous approaches such as RLP. Additionaly the complete coverage of serialization and hash-tree-root on the same type system avoids gaps in functionality and ambiguity.


### Beacon operations

[Beacon operations](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-operations) are datastructures that can be added to a `BeaconBlock` by a block proposer.

They are the means by which new information is introduced into the beacon chain allowing it to update its internal state.

>**Note:** there is a maximum number of beacon operations allowed per block. And different operations may have different maximum values associated with them. These numbers are defined in the constants in the [max operations per block](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#max-operations-per-block) subsection of the Beacon chain specification.

#### `ProposerSlashing`

The [`ProposerSlashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#proposerslashing) operation is used to police potentially malicious validator block proposal activity.

>**Definition:** slashings are major penalties given for malicious operations.

Specifically, validators can be slashed if they sign two different beacon blocks for the same slot. This makes duplicate block proposals expensive, which disincentivizes activity that might lead to forking and conflicting views of the canonical chain.

```python
class ProposerSlashing(Container):
    proposer_index: ValidatorIndex
    signed_header_1: SignedBeaconBlockHeader
    signed_header_2: SignedBeaconBlockHeader
```

It has three fields:

  * `proposer_index` - The validator index of the validator to be slashed for double proposing
  * `signed_header_1` - The signed header of the first of the two slashable beacon blocks
  * `signed_header_2` - The signed header of the second of the two slashable beacon blocks

These fields are all that's needed to prove that a slashable offense has occurred.

```python
class SignedBeaconBlock(Container):
    message: BeaconBlock
    signature: BLSSignature
```

```python
class BeaconBlockHeader(Container):
    slot: Slot
    parent_root: Root
    state_root: Root
    body_root: Root
```

Importantly, since `hash_tree_root(signed_block.message) == hash_tree_root(signed_block_header.message)`, a single signature is valid for both data structures. This means `SignedBeaconBlockHeader` can be used as a proof, which reduces data size. For more on `hash_tree_root` see [here](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#hash_tree_root).


#### `AttesterSlashing`

The [`AttesterSlashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attesterslashing) operation is used to police potentially malicious validator attestation activity that might lead to finalizing two conflicting chains.

Specifically, validators can be slashed if they sign two conflicting attestations -- where conflicting is defined by [`is_slashable_attestation_data`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#is_slashable_attestation_data) which checks if the attestations are slashable according to Casper FFG rules (in particular the "double" and "surround" vote conditions).

```python
class AttesterSlashing(Container):
    attestation_1: IndexedAttestation
    attestation_2: IndexedAttestation
```

It has two fields:

* `attestation_1` - The first of the two slashable attestations (in [`IndexedAttestation`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#indexedattestation) form).
* `attestation_2` - The second of the two slashable attestations (in `IndexedAttestation` form).

```python
class IndexedAttestation(Container):
    attesting_indices: List[ValidatorIndex, MAX_VALIDATORS_PER_COMMITTEE]
    data: AttestationData
    signature: BLSSignature
```

>**Note:** we use an `IndexedAttestation`, as opposed to a bitfield based `Attestation` since it allows us to check if the attestations are slashable without recomputing (historical) committee indices.


#### `Attestation`

An [`Attestation`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestation) is the primary message type that validators create for consensus.

Although beacon blocks are only created by one validator per slot, _all_ validators have a chance to create one attestation per epoch (through their assignment to a Beacon Committee). In the optimal case, all active validators create and have an attestation included into a block during each epoch.

```python
class Attestation(Container):
    aggregation_bits: Bitlist[MAX_VALIDATORS_PER_COMMITTEE]
    data: AttestationData
    signature: BLSSignature
```

 It has three fields:

 * `aggregation_bits` - a list of bits containing a single bit for each member of the committee. Each validator that participated in this aggregate signature is assigned a value of `1`. These bits are ordered by the sort of the associated crosslink committee.
 * `data` - the `AttestationData` that was signed by the validator or collection of validators.
 * `signature` - the aggregate BLS signature of the attestation.
 
As you can see, it contains the aggregate signature (`BLSSignature`) and the participation bitfield required for verification of the signature. 

However, the core of the message is the [`AttestationData`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestationdata).

```python
class AttestationData(Container):
    slot: Slot
    index: CommitteeIndex
    # LMD GHOST vote
    beacon_block_root: Root
    # FFG vote
    source: Checkpoint
    target: Checkpoint
```

`AttestationData` is the primary component committed to by each validator. It contains signalling support for the head of the chain and the FFG vote.

`AttestationData` has five fields:

   * `slot` - the slot that the validator/committee is assigned to attest to
   * `index` - the index of the committee making the attestation (committee indices are mapped to shards in Phase 1)
   * `beacon_block_root` - block root of the beacon block seen as the head of the chain during the assigned slot
   * `source` - the most recent justified `Checkpoint` in the `BeaconState` during the assigned slot
   * `target` - the `Checkpoint` the attesters are attempting to justify (the current epoch and epoch boundary block)

```python
class Checkpoint(Container):
    epoch: Epoch
    root: Root
```

>**Note:** committee indices will be mapped to shards in Phase 1, which will allow `AttestationData` to signal support for shard data.

#### `Deposit`

A [`Deposit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#deposit) represents an incoming validator deposit from the eth1 [deposit contract](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/deposit-contract.md).

```python
class Deposit(Container):
    proof: Vector[Bytes32, DEPOSIT_CONTRACT_TREE_DEPTH + 1]  # Merkle path to deposit root
    data: DepositData
```

It has two fields:

   * `proof` - the merkle path to the deposit root. In other words, the merkle proof against the current `state.eth1_data.root` in the `BeaconState`. Note that the `+ 1` in the vector length is due to the SSZ length mixed into the root.
   * `data` - the submitted [`DepositData`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#depositdata) to the deposit contract. This is verified using the `proof` against the `state.eth1_data.root`.

```python
class DepositData(Container):
    pubkey: BLSPubkey
    withdrawal_credentials: Bytes32
    amount: Gwei
    signature: BLSSignature  # Signing over DepositMessage
```

`DepositData` has four fields:

   * `pubkey` - the BLS12-381 public key to be used by the validator to sign messages
   * `withdrawal_credentials` - a `BLS_WITHDRAWAL_PREFIX` -- concatenated with the last 31 bytes of the hash of an offline pubkey -- to be used to withdraw staked funds after exiting. This key is not used actively in validation and can be kept in cold storage.
   * `amount` - the amount in Gwei that was deposited
   * `signature` - the signature of the `DepositMessage(pubkey, amount, withdrawal_credentials)` using the `privkey` pair of the `pubkey`. This is used as a one-time proof of possession: a requirement for securely using BLS keys. It also signs over the `withdrawal_credentials` -- this is essential to avoid injection of other withdrawal credentials.
   
```python
class DepositMessage(Container):
    pubkey: BLSPubkey
    withdrawal_credentials: Bytes32
    amount: Gwei
```

>**Note:** we don't need to explicitly define a deposit index, as it is already verified through the merkle inclusion proof. (And can easily be derived from the mix-in node in the proof).


#### `VoluntaryExit`

A [`VoluntaryExit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#voluntaryexit) allows a validator to voluntarily exit validation duties. 

```python
class VoluntaryExit(Container):
    epoch: Epoch  # Earliest epoch when voluntary exit can be processed
    validator_index: ValidatorIndex
```

It has two fields:
     
   * `epoch` - the minimum epoch at which this exit can be included on chain. This helps prevent accidental/malicious use in chain forks or reorganisations.
   * `validator_index` - the exiting validator's index within the `BeaconState` validator registry.
   
Importantly, `VoluntaryExit` is wrapped into [`SignedVoluntaryExit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#signedvoluntaryexit) which is included on-chain.

```python
class SignedVoluntaryExit(Container):
    message: VoluntaryExit
    signature: BLSSignature
```
   
`SignedVoluntaryExit` has two fields:
   
   * `message` - the `VoluntaryExit` signed over in this object.
   * `signature` - the signature of the `VoluntaryExit.message` included in this object -- signed by the pubkey associated with the `Validator` defined by `validator_index`.


### Beacon blocks

#### `BeaconBlock`

[`BeaconBlock`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beaconblock) is the primary component of the beacon chain, serving as the main container/header for each block in the chain. Each `BeaconBlock` contains a reference (`parent_root`) to the block root of its parent. These references form a chain of ancestors all the way up to the (parent-less) genesis block -- in other words, a block-chain. In optimal operation, a single `BeaconBlock` is proposed each slot by the selected proposer from the current epoch's active validator set. 

```python
class BeaconBlock(Container):
    slot: Slot
    parent_root: Root
    state_root: Root
    body: BeaconBlockBody
```

It has four fields:

* `slot` - the slot for which this block is created. Must be greater than the slot of the block defined by `parent_root`
* `parent_root` - the block root of the parent block
* `state_root` - the hash root of the post state of running the state transition through this block
* `body` - a `BeaconBlockBody` containing fields for each of the [beacon operation](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-operations) objects as well as a few proposer input fields

#### `BeaconBlockBody`

 [`BeaconBlockBody`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beaconblockbody) contains fields specific to the action of the block on the current state. 

```python
class BeaconBlockBody(Container):
    randao_reveal: BLSSignature
    eth1_data: Eth1Data  # Eth1 data vote
    graffiti: Bytes32  # Arbitrary data
    # Operations
    proposer_slashings: List[ProposerSlashing, MAX_PROPOSER_SLASHINGS]
    attester_slashings: List[AttesterSlashing, MAX_ATTESTER_SLASHINGS]
    attestations: List[Attestation, MAX_ATTESTATIONS]
    deposits: List[Deposit, MAX_DEPOSITS]
    voluntary_exits: List[SignedVoluntaryExit, MAX_VOLUNTARY_EXITS]
```

It has eight fields:

* `randao_reveal` - the `BLSSignature` of the current epoch (by the current block proposer). Constitutes the seed for randomness when mixed in with the other proposers' reveals.
* `eth1_data` - a vote on recent data from the Eth1 chain. It consists of the following fields:
    * `deposit_root` - the SSZ List `hash_tree_root` of all of the deposits in the deposit contract
    * `deposit_count` - the number of successful validator deposits that have been made into the deposit contract so far
    * `block_hash` - the eth1 block hash that contains the `deposit_root`. This `block_hash` is intended to be used for finalization of the Eth1 chain in the future
 * `graffiti` - 32 bytes for validators to decorate as they choose with no defined in-protocol use
 * `proposer_slashings`, `attester_slashings`, `attestations`, `deposits`, `voluntary_exits` are all lists (with max lengths) -- one for each operation type -- that can be included into the `BeaconBlockBody`

#### `SignedBeaconBlock`

`SignedBeaconBlock` is a signed wrapper of `BeaconBlock` that allows for the proposer's signature to be transmitted with a block.

 ```python
class SignedBeaconBlock(Container):
    message: BeaconBlock
    signature: BLSSignature
```
 
It has two fields:

* `message` - the `BeaconBlock` signed over in this object
* `signature` - the signature of the `BeaconBlock` `message` included in this object (signed by the public key of the proposer for the given slot)

#### `BeaconBlockHeader`

Thanks to the properties of [SSZ](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/ssz/simple-serialize.md) trees -- `hash_tree_root(BeaconBlock) == hash_tree_root(BeaconBlockHeader)` which means a single signature is valid for both `BeaconBlock` and `BeaconBlockHeader`.

```python
class BeaconBlockHeader(Container):
    slot: Slot
    parent_root: Root
    state_root: Root
    body_root: Root
```

This allows us to do things like prove that an attestation is contained within the block body, without needing to have the entire block body at hand (the signed header is enough).

#### `SignedBeaconBlockHeader`

```python
class SignedBeaconBlockHeader(Container):
    message: BeaconBlockHeader
    signature: BLSSignature
```

### Beacon state

#### `BeaconState`

The [`BeaconState`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beaconstate) is the resulting state of running the state transition function on a chain of `BeaconBlock`s starting from the genesis state.

It contains the information about fork versioning, historical caches, data about the eth1 chain/validator deposits, the validator registry, information about randomness, information about past slashings, recent attestations, recent crosslinks, and information about finality.

```python
class BeaconState(Container):
    # Versioning
    genesis_time: uint64
    slot: Slot
    fork: Fork
    # History
    latest_block_header: BeaconBlockHeader
    block_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    state_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    historical_roots: List[Root, HISTORICAL_ROOTS_LIMIT]
    # Eth1
    eth1_data: Eth1Data
    eth1_data_votes: List[Eth1Data, SLOTS_PER_ETH1_VOTING_PERIOD]
    eth1_deposit_index: uint64
    # Registry
    validators: List[Validator, VALIDATOR_REGISTRY_LIMIT]
    balances: List[Gwei, VALIDATOR_REGISTRY_LIMIT]
    # Randomness
    randao_mixes: Vector[Bytes32, EPOCHS_PER_HISTORICAL_VECTOR]
    # Slashings
    slashings: Vector[Gwei, EPOCHS_PER_SLASHINGS_VECTOR]  # Per-epoch sums of slashed effective balances
    # Attestations
    previous_epoch_attestations: List[PendingAttestation, MAX_ATTESTATIONS * SLOTS_PER_EPOCH]
    current_epoch_attestations: List[PendingAttestation, MAX_ATTESTATIONS * SLOTS_PER_EPOCH]
    # Finality
    justification_bits: Bitvector[JUSTIFICATION_BITS_LENGTH]  # Bit set for every recent justified epoch
    previous_justified_checkpoint: Checkpoint  # Previous epoch snapshot
    current_justified_checkpoint: Checkpoint
    finalized_checkpoint: Checkpoint
```

It has 20 fields. The following are descriptions of the fields grouped into logical units:

* Fork Versioning
    * `genesis_time` -- tracks the Unix timestamp during which the genesis slot occurred. This allows a client to calculate what the current slot should be according to [wall-clock time](https://wikipedia.org/wiki/Elapsed_real_time)
    * `slot` -- tracks the slot of the containing state. Note that this isn't necessarily the same as the slot according to the local wall-clock. Remember that time is divided into slots of fixed length (`SECONDS_PER_SLOT`) in which actions occur and state transitions happen
    * `fork` -- a mechanism for handling forking (upgrading) the beacon chain. The main purpose here is to handle versioning of signatures and handle objects of different signatures across fork boundaries
* History
    * `latest_block_header` -- a cache of the latest block header seen in the chain defining this state. During the slot transition of the block, the state `Root` (a 32 byte vector) in the header is stored with an empty placeholder (`0x00..00`) instead of the real state root. At the start of the _next slot_ transition -- before anything has been modified within state -- the real state root of the previous slot is calculated and added to the `latest_block_header`. We do this to prevent the state root from being embedded in the block header while the block header is part of the same slot's state -- the state root is a hash and we can’t embed an object's hash into itself without changing that object's hash (what we call a circular dependency). The placeholder helps us get around this circular dependency by allowing us to insert the state root in the header within the beacon state in a subsequent slot.  
    * `block_roots` -- a per-slot store of the recent block roots. The block root for a slot is added at the start of the next slot to avoid the circular dependency that arises from the state root being embedded in the block. For slots that are skipped (no block in the chain for the given slot), the most recent block root in the chain prior to the current slot is stored for the skipped slot. When validators attest to a given slot, they use this store of block roots as an information source to cast their vote
    * `state_roots` -- a per-slot store of the recent state roots. As with the block root, the state root for a slot is stored at the start of the next slot to avoid a circular dependency
    * `historical_roots` -- a double batch merkle accumulator of the latest block and state roots defined by [`HistoricalBatch`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#historicalbatch). It's used to make historic merkle proofs against. Note that although this field grows unbounded, it grows at less than 10 KB per year
* Eth1
    * `eth1_data` -- the most recent [`Eth1Data`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#eth1data) that validators have come to consensus on and stored in state. Validator deposits from eth1 can be processed into eth2 through the latest deposit contained within the `eth1_data` root
    * `eth1_data_votes` -- a running list of votes on new `Eth1Data` to be stored in state. If any `Eth1Data` achieves `> 50%` of proposer votes in a voting period, this data is stored in state and new deposits can be processed
    * `eth1_deposit_index` -- the index of the next deposit to be processed. Deposits must be added to the next block and processed if `state.eth1_data.deposit_count > state.eth1_deposit_index`
* Registry
    * `validators` -- a list of `Validator` records, tracking the current full registry. Each validator contains relevant data such as pubkey, withdrawal credentials, effective balance, a slashed boolean, and status (pending, active, exited, etc)
    * `balances` -- a list mapping 1:1 with the `validator_registry`. The granular/frequently changing balances are pulled out of the `validators` list to reduce the amount of re-hashing (in a cache optimized SSZ implementation) that needs to be performed at each epoch transition
* Randomness
    * `randao_mixes` -- The randao mix from each epoch for the past `EPOCHS_PER_HISTORICAL_VECTOR` epochs. At the start of each epoch, the `randao_mix` from the previous epoch is copied over as the base of the current epoch. At each block, the `hash` of the `block.randao_reveal` is xor'd into the running mix of the current epoch
* Slashings
    * `slashings` -- a per-epoch store of past `EPOCHS_PER_SLASHINGS_VECTOR` epochs. It stores the total slashed GWEI during each epoch. The sum of this list at any time gives the *recent slashed balance* and is used to calculate the proportion of balance that should be slashed for slashable validators
* Attestations
    * `Attestation`s from blocks are converted to `PendingAttestation`s and stored in state for bulk accounting at epoch boundaries. We store two separate lists:
    * `previous_epoch_attesations` -- List of `PendingAttestation`s for slots from the previous epoch. *note*: these are attestations attesting to slots in the previous epoch, not necessarily those included in blocks during the previous epoch.
    * `current_epoch_attesations` -- List of `PendingAttestation`s for slots from the current epoch. Copied over to `previous_epoch_attestations` and then emptied at the end of the current epoch processing
* Finality
    * `justification_bits` -- 4 bits used to track justification during the last 4 epochs to aid in finality calculations
    * `previous_justified_checkpoint` -- the most recent justified `Checkpoint` as it was during the previous epoch. Used to validate attestations from the previous epoch
    * `current_justified_checkpoint` -- the most recent justified `Checkpoint` during the current epoch. Used to validate current epoch attestations and fork choice purposes
    * `finalized_checkpoint` -- the most recent finalized `Checkpoint`, prior to which blocks are never reverted.

#### `HistoricalBatch`

```python
class HistoricalBatch(Container):
    block_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    state_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
```

#### `Eth1Data`

```python
class Eth1Data(Container):
    deposit_root: Root
    deposit_count: uint64
    block_hash: Bytes32
```

#### `Fork`

```python
class Fork(Container):
    previous_version: Version
    current_version: Version
    epoch: Epoch  # Epoch of latest fork
```

#### `Validator`

```python
class Validator(Container):
    pubkey: BLSPubkey
    withdrawal_credentials: Bytes32  # Commitment to pubkey for withdrawals
    effective_balance: Gwei  # Balance at stake
    slashed: boolean
    # Status epochs
    activation_eligibility_epoch: Epoch  # When criteria for activation were met
    activation_epoch: Epoch
    exit_epoch: Epoch
    withdrawable_epoch: Epoch  # When validator can withdraw funds
```

#### `PendingAttestation`

```python
class PendingAttestation(Container):
    aggregation_bits: Bitlist[MAX_VALIDATORS_PER_COMMITTEE]
    data: AttestationData
    inclusion_delay: Slot
    proposer_index: ValidatorIndex
```

#### `Checkpoint`

```python
class Checkpoint(Container):
    epoch: Epoch
    root: Root
```

## State Transitions

The core of Phase 0 is the [beacon chain state transition function](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-chain-state-transition-function). The post-state corresponding to a pre-state `state` and a signed beacon-block `signed_block` is defined as `state_transition(state, signed_block)`.

>Note: state transitions that trigger an unhandled exception (e.g. a failed assert or an out-of-range list access) are considered invalid.

### `state_transition`

[`state_transition`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-chain-state-transition-function) is the top level function of the state transition. It accepts a pre-state and a signed beacon block, and outputs a post-state.

```python
def state_transition(state: BeaconState, signed_block: SignedBeaconBlock, validate_result: bool=True) -> BeaconState:
    block = signed_block.message
    # Process slots (including those with no blocks) since block
    process_slots(state, block.slot)
    # Verify signature
    if validate_result:
        assert verify_block_signature(state, signed_block)
    # Process block
    process_block(state, block)
    # Verify state root
    if validate_result:
        assert block.state_root == hash_tree_root(state)
    # Return post-state
    return state
```

During the production of a block, its state-root is computed by running the transition with the candidate block, while disabling its validation (i.e. `validate_result = False`). The state-root of the block is then updated, and the block is signed. Block production is explained in greater detail in the [Validator spec](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/validator.md).

### Slot processing

Slot processing occurs at the start of every slot.

>Definition: a slot is a period of time in which a block proposer proposes a block and a set of committees attest to it.

### `process_slots`

`process_slots` is the first component of the state transition function. It handles the set of state updates that happen at each slot.

>Note: in optimal operation, a single `BeaconBlock` is proposed each slot by the selected proposer from the current epoch's active validator set. However, the assigned proposer can fail to propose in a timely manner, resulting in a *skipped* slot.


```python
def process_slots(state: BeaconState, slot: Slot) -> None:
    assert state.slot <= slot
    while state.slot < slot:
        process_slot(state)
        # Process epoch on the start slot of the next epoch
        if (state.slot + 1) % SLOTS_PER_EPOCH == 0:
            process_epoch(state)
        state.slot += Slot(1)
```

If there are skipped slots in between the state transition input block and its parent, then multiple slots are transitioned during `process_slots` up to the `block.slot`.

Consensus forming clients will sometimes have to call this function directly to transition state through empty slots when attesting to, or producing, blocks.

### `process_slot`

`process_slot` is called once per slot to cache recent data from the previous slot.

```python
def process_slot(state: BeaconState) -> None:
    # Cache state root
    previous_state_root = hash_tree_root(state)
    state.state_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT] = previous_state_root
    # Cache latest block header state root
    if state.latest_block_header.state_root == Bytes32():
        state.latest_block_header.state_root = previous_state_root
    # Cache block root
    previous_block_root = hash_tree_root(state.latest_block_header)
    state.block_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT] = previous_block_root
```

> Note: when `process_slot` is called the `state` has not yet been modified, so `state` still refers to the post-state of the previous slot.

> This means that `previous_state_root` refers to the post-state root of the previous slot (the unmodified state root). As you can see, it's cached both in `state.state_roots`, and in `state.latest_block_header` if the previous slot contained a block.

> Also note that `state.slot` is still equal to the previous slot (since `state.slot += 1` happens after `process_slot` is called from within `process_slots`) so any updates to items that reference `state.slot` are accounting for the previous slot.

`process_slot` consists of the following state changes:

* Cache state root
    * The state root of the previous slot is cached into `state.state_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT]`. Remember that, as mentioned above, `hash_tree_root(state) == previous_state_root` because the `state` has not yet been modified from the post state of the previous slot
* Cache latest block header state root
    * If the previous slot contained a block (in other words, was not a skipped slot), then we insert the `previous_state_root` into the `state_root` of the cached `state.latest_block_header`. This block header remains in state through any skipped slots until the next block occurs
* Cache block root
    * We cache the `hash_tree_root` of the latest block into `state.block_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT]` at each slot. If the previous slot was skipped, then the block from the latest non-skipped slot is cached

### Epoch processing

[Epoch processing](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#epoch-processing) occurs at the _start_ of the 0th slot (`slot % EPOCH_LENGTH == 0`) of each epoch.

> Definition: an epoch consists of multiple slots (currently 32) after which validators are reshuffled into different committees.

### `process_epoch`

`process_epoch` is the primary container function that calls the rest of the epoch processing sub-functions. It's only called at epoch boundaries. 

>Note: because `state.slot` has not yet been incremented during the conditional check in `process_slots`, epoch transitions occur at the state transition defined by the `slot` at the start of the epoch (`slot % SLOTS_PER_EPOCH == 0`)

```python
def process_epoch(state: BeaconState) -> None:
    process_justification_and_finalization(state)
    process_rewards_and_penalties(state)
    process_registry_updates(state)
    # @process_reveal_deadlines
    # @process_challenge_deadlines
    process_slashings(state)
    # @update_period_committee
    process_final_updates(state)
    # @after_process_final_updates
```

>Note: when `process_epoch` is run inside `process_slot`, `state.slot` is still equal to the previous slot (`slot % EPOCH_LENGTH == EPOCH_LENGTH - 1`), since it is only incremented afterwards.

#### Justification and Finalization

Everything related to justification and finalisation is handled by `process_justification_and_finalization`.

#### `process_justification_and_finalization`


```python
def process_justification_and_finalization(state: BeaconState) -> None:
    if get_current_epoch(state) <= GENESIS_EPOCH + 1:
        return

    previous_epoch = get_previous_epoch(state)
    current_epoch = get_current_epoch(state)
    old_previous_justified_checkpoint = state.previous_justified_checkpoint
    old_current_justified_checkpoint = state.current_justified_checkpoint

    # Process justifications
    state.previous_justified_checkpoint = state.current_justified_checkpoint
    state.justification_bits[1:] = state.justification_bits[:-1]
    state.justification_bits[0] = 0b0
    matching_target_attestations = get_matching_target_attestations(state, previous_epoch)  # Previous epoch
    if get_attesting_balance(state, matching_target_attestations) * 3 >= get_total_active_balance(state) * 2:
        state.current_justified_checkpoint = Checkpoint(epoch=previous_epoch,
                                                        root=get_block_root(state, previous_epoch))
        state.justification_bits[1] = 0b1
    matching_target_attestations = get_matching_target_attestations(state, current_epoch)  # Current epoch
    if get_attesting_balance(state, matching_target_attestations) * 3 >= get_total_active_balance(state) * 2:
        state.current_justified_checkpoint = Checkpoint(epoch=current_epoch,
                                                        root=get_block_root(state, current_epoch))
        state.justification_bits[0] = 0b1

    # Process finalizations
    bits = state.justification_bits
    # The 2nd/3rd/4th most recent epochs are justified, the 2nd using the 4th as source
    if all(bits[1:4]) and old_previous_justified_checkpoint.epoch + 3 == current_epoch:
        state.finalized_checkpoint = old_previous_justified_checkpoint
    # The 2nd/3rd most recent epochs are justified, the 2nd using the 3rd as source
    if all(bits[1:3]) and old_previous_justified_checkpoint.epoch + 2 == current_epoch:
        state.finalized_checkpoint = old_previous_justified_checkpoint
    # The 1st/2nd/3rd most recent epochs are justified, the 1st using the 3rd as source
    if all(bits[0:3]) and old_current_justified_checkpoint.epoch + 2 == current_epoch:
        state.finalized_checkpoint = old_current_justified_checkpoint
    # The 1st/2nd most recent epochs are justified, the 1st using the 2nd as source
    if all(bits[0:2]) and old_current_justified_checkpoint.epoch + 1 == current_epoch:
        state.finalized_checkpoint = old_current_justified_checkpoint
```

##### Justification

Within [`process_justification_and_finalization`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#justification-and-finalization), we check for justification updates in both the previous and current epoch.

We utilize the `PendingAttestation`s contained within the `previous_epoch_attestations` and `current_epoch_attestations` lists to check if the previous and current epochs have been justified.

>Note: these lists are already pre-filtered to only contain attestations that voted on the expected `source` designated by the current `state`.

If at least 2/3 of the active balance attests to the previous epoch, the previous epoch is justified. Similarly, if at least 2/3 of the active balance attests to the current epoch, the current epoch is justified.

##### Finalization

Only the 2nd, 3rd, and 4th most recent epochs can be finalized. This is done by checking:

1. Which epochs (within the past 4) are used as the source of the most recent justifications.
2. That the epochs between source and target are also justified (if any exist).

If these conditions are satisfied, the source is finalized. Note that we only consider `k=1` and `k=2` finality rules discussed in [section 4.4.3 of the draft paper](https://github.com/ethereum/research/blob/master/papers/ffg%2Bghost/paper.pdf).

#### Rewards and Penalties

Everything related to rewards and penalties is handled by [`process_rewards_and_penalties`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#rewards-and-penalties-1). Specifically, this is where all of the rewards and penalties collected from the previous epoch are applied to validator balances.

#### `process_rewards_and_penalties`


```python
def process_rewards_and_penalties(state: BeaconState) -> None:
    if get_current_epoch(state) == GENESIS_EPOCH:
        return

    rewards, penalties = get_attestation_deltas(state)
    for index in range(len(state.validators)):
        increase_balance(state, ValidatorIndex(index), rewards[index])
        decrease_balance(state, ValidatorIndex(index), penalties[index])
```

Since attestations are given up to one epoch to be included on chain, rewards from attestations are not processed until the end of the following epoch. Due to this, rewards and penalties are skipped during the `GENESIS_EPOCH`.

In Phase 0, rewards and penalties are collected from `get_attestation_deltas` which returns the rewards and penalties for each validator as tuples of lists. Rewards and penalties are separated to avoid signed integers.

Individual rewards are scaled by both the validator's `effective_balance` and $\frac{1}{\sqrt{TotalBalance}}$. See [the design rationale](https://notes.ethereum.org/s/rkhCgQteN#Base-rewards) for an explanation of why this is the case. 

Note that we use a unit of account called `base_reward` to keep track of the rewards and/or penalties associated with each of the components of a validator's duties.

```python
def get_base_reward(state: BeaconState, index: ValidatorIndex) -> Gwei:
    total_balance = get_total_active_balance(state)
    effective_balance = state.validators[index].effective_balance
    return Gwei(effective_balance * BASE_REWARD_FACTOR // integer_squareroot(total_balance) // BASE_REWARDS_PER_EPOCH)
```

##### `get_attestation_deltas`

`get_attestation_deltas` determines how much each validator's balance changes as a function of their attestation behaviour in the previous epoch.

```python
def get_attestation_deltas(state: BeaconState) -> Tuple[Sequence[Gwei], Sequence[Gwei]]:
    previous_epoch = get_previous_epoch(state)
    total_balance = get_total_active_balance(state)
    rewards = [Gwei(0) for _ in range(len(state.validators))]
    penalties = [Gwei(0) for _ in range(len(state.validators))]
    eligible_validator_indices = [
        ValidatorIndex(index) for index, v in enumerate(state.validators)
        if is_active_validator(v, previous_epoch) or (v.slashed and previous_epoch + 1 < v.withdrawable_epoch)
    ]

    # Micro-incentives for matching FFG source, FFG target, and head
    matching_source_attestations = get_matching_source_attestations(state, previous_epoch)
    matching_target_attestations = get_matching_target_attestations(state, previous_epoch)
    matching_head_attestations = get_matching_head_attestations(state, previous_epoch)
    for attestations in (matching_source_attestations, matching_target_attestations, matching_head_attestations):
        unslashed_attesting_indices = get_unslashed_attesting_indices(state, attestations)
        attesting_balance = get_total_balance(state, unslashed_attesting_indices)
        for index in eligible_validator_indices:
            if index in unslashed_attesting_indices:
                rewards[index] += get_base_reward(state, index) * attesting_balance // total_balance
            else:
                penalties[index] += get_base_reward(state, index)

    # Proposer and inclusion delay micro-rewards
    for index in get_unslashed_attesting_indices(state, matching_source_attestations):
        attestation = min([
            a for a in matching_source_attestations
            if index in get_attesting_indices(state, a.data, a.aggregation_bits)
        ], key=lambda a: a.inclusion_delay)
        proposer_reward = Gwei(get_base_reward(state, index) // PROPOSER_REWARD_QUOTIENT)
        rewards[attestation.proposer_index] += proposer_reward
        max_attester_reward = get_base_reward(state, index) - proposer_reward
        rewards[index] += Gwei(max_attester_reward // attestation.inclusion_delay)

    # Inactivity penalty
    finality_delay = previous_epoch - state.finalized_checkpoint.epoch
    if finality_delay > MIN_EPOCHS_TO_INACTIVITY_PENALTY:
        matching_target_attesting_indices = get_unslashed_attesting_indices(state, matching_target_attestations)
        for index in eligible_validator_indices:
            penalties[index] += Gwei(BASE_REWARDS_PER_EPOCH * get_base_reward(state, index))
            if index not in matching_target_attesting_indices:
                effective_balance = state.validators[index].effective_balance
                penalties[index] += Gwei(effective_balance * finality_delay // INACTIVITY_PENALTY_QUOTIENT)

    return rewards, penalties
```

It does the following:


* For validators that are either active or slashed but not yet withdrawable:

    * **Reward for FFG Source, Target & Head**: If the validator's source, target, or head from any included attestation made in the previous epoch matches the current source, target, or head of the previous epoch, issue a `base_reward` scaled by the fraction of active validators also attesting to this correct value; otherwise issue a `base_reward` penalty. There is a maximum reward of `3 * base_reward` reward (one for each correct value).
    * **Reward for attestation inclusion speed**: Find the earliest included attestation for each validator. If such an attestation exists, give one `base_reward` split between the proposer and the included validator. Give the proposer a small reward --`base_reward` (relative to the included validator) divided by `PROPOSER_REWARD_QUOTIENT` -- for including the attestation. Give the remaining portion of the `base_reward` to the included validator but scale it in proportion to how quickly the attestation was included.
    * **Inactivity penalty**: if the chain isn't finalising, penalize everyone, but particularly those who are not participating. Specifically, add a `BASE_REWARD_PER_EPOCH * base_reward` penalty to each active validator, and add an `effective_balance * finality_delay // INACTIVITY_PENALTY_QUOTIENT` penalty to each validator that did not vote on the correct target. Note that validators participating optimally during an inactivity leak will not incur any penalties [Potential bug discussed [here](https://github.com/ethereum/eth2.0-specs/issues/1370)].


#### Registry updates

Updates to the validator registry are handled in a per-epoch basis in [`process_registry_updates`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#registry-updates).

#### `process_registry_updates`

```python
def process_registry_updates(state: BeaconState) -> None:
    # Process activation eligibility and ejections
    for index, validator in enumerate(state.validators):
        if is_eligible_for_activation_queue(validator):
            validator.activation_eligibility_epoch = get_current_epoch(state) + 1

        if is_active_validator(validator, get_current_epoch(state)) and validator.effective_balance <= EJECTION_BALANCE:
            initiate_validator_exit(state, ValidatorIndex(index))

    # Queue validators eligible for activation and not yet dequeued for activation
    activation_queue = sorted([
        index for index, validator in enumerate(state.validators)
        if is_eligible_for_activation(state, validator)
        # Order by the sequence of activation_eligibility_epoch setting and then index
    ], key=lambda index: (state.validators[index].activation_eligibility_epoch, index))
    # Dequeued validators for activation up to churn limit
    for index in activation_queue[:get_validator_churn_limit(state)]:
        validator = state.validators[index]
        validator.activation_epoch = compute_activation_exit_epoch(get_current_epoch(state))
```

It handles the following:

* Validators who have a sufficient `effective_balance` to be activated (`== MAX_EFFECTIVE_BALANCE`) but that have not yet been added to the queue (`activation_eligibility_epoch == FAR_FUTURE_EPOCH`), are assigned as eligible for activation starting from the next epoch
* Validators who have too low an `effective_balance` (`<= EJECTION_BALANCE`) are ejected (added to the withdrawal queue)
* Validators eligible for activation, but not yet activated, are considered queued if the chain has finalized since they were assigned as eligible. The queue is FIFO: sorted by time of eligibility epoch (with validator index serving as a tie breaker)
* Validators that are in the activation queue are activated within the churn limit


#### Slashings

Slashings are handled by `process_slashings`.

#### `process_slashings`

[`process_slashings`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#slashings) goes through the recent slashings and penalizes slashed validators proportionally.

```python
def process_slashings(state: BeaconState) -> None:
    epoch = get_current_epoch(state)
    total_balance = get_total_active_balance(state)
    for index, validator in enumerate(state.validators):
        if validator.slashed and epoch + EPOCHS_PER_SLASHINGS_VECTOR // 2 == validator.withdrawable_epoch:
            increment = EFFECTIVE_BALANCE_INCREMENT  # Factored out from penalty numerator to avoid uint64 overflow
            penalty_numerator = validator.effective_balance // increment * min(sum(state.slashings) * 3, total_balance)
            penalty = penalty_numerator // total_balance * increment
            decrease_balance(state, ValidatorIndex(index), penalty)
```

Validators that are slashed are done so in proportion to how many validators have been slashed within a recent time period. This occurs independently of the order in which validators are slashed. 

#### Final updates

Final (miscellaneous) updates are handled by `process_final_updates`.

#### `process_final_updates`

[`process_final_updates`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#final-updates) handles various functionality that needs to occur every epoch.

```python
def process_final_updates(state: BeaconState) -> None:
    current_epoch = get_current_epoch(state)
    next_epoch = Epoch(current_epoch + 1)
    # Reset eth1 data votes
    if (state.slot + 1) % SLOTS_PER_ETH1_VOTING_PERIOD == 0:
        state.eth1_data_votes = []
    # Update effective balances with hysteresis
    for index, validator in enumerate(state.validators):
        balance = state.balances[index]
        HALF_INCREMENT = EFFECTIVE_BALANCE_INCREMENT // 2
        if balance < validator.effective_balance or validator.effective_balance + 3 * HALF_INCREMENT < balance:
            validator.effective_balance = min(balance - balance % EFFECTIVE_BALANCE_INCREMENT, MAX_EFFECTIVE_BALANCE)
    # Reset slashings
    state.slashings[next_epoch % EPOCHS_PER_SLASHINGS_VECTOR] = Gwei(0)
    # Set randao mix
    state.randao_mixes[next_epoch % EPOCHS_PER_HISTORICAL_VECTOR] = get_randao_mix(state, current_epoch)
    # Set historical root accumulator
    if next_epoch % (SLOTS_PER_HISTORICAL_ROOT // SLOTS_PER_EPOCH) == 0:
        historical_batch = HistoricalBatch(block_roots=state.block_roots, state_roots=state.state_roots)
        state.historical_roots.append(hash_tree_root(historical_batch))
    # Rotate current/previous epoch attestations
    state.previous_epoch_attestations = state.current_epoch_attestations
    state.current_epoch_attestations = []
```

It takes care of the following:

* Reset eth1 data votes if we are at the start of a new voting period
* Recalculate effective balances of all validators (using hysteresis)
* Set slashed balances for next epoch to 0
* Set the resulting randao mix from the current epoch as the base randao mix in the next epoch
* Append historic batch to historical root if at the end of an accumulation period
* Move `current_epoch_attestations` to `previous` and empty  `current`

### Block Processing

Block processing occurs once every time the `state_transition` function is called, after processing all of the slots between the previous and current block.

### `process_block`

[`process_block`](https://github.com/ethereum/eth2.0-shttps://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#block-processing) is the main function that calls the sub-functions of block processing. 

```python
def process_block(state: BeaconState, block: BeaconBlock) -> None:
    process_block_header(state, block)
    process_randao(state, block.body)
    process_eth1_data(state, block.body)
    process_operations(state, block.body)
```

If any asserts fail, or if any exceptions are thrown during block processing, then the block is seen as invalid and should be discarded.

#### Block Header

#### `process_block_header`

[`process_block_header`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#block-header) determines whether, at a high level, a block is valid.

```python
def process_block_header(state: BeaconState, block: BeaconBlock) -> None:
    # Verify that the slots match
    assert block.slot == state.slot
    # Verify that the parent matches
    assert block.parent_root == hash_tree_root(state.latest_block_header)
    # Cache current block as the new latest block
    state.latest_block_header = BeaconBlockHeader(
        slot=block.slot,
        parent_root=block.parent_root,
        state_root=Bytes32(),  # Overwritten in the next process_slot call
        body_root=hash_tree_root(block.body),
    )

    # Verify proposer is not slashed
    proposer = state.validators[get_beacon_proposer_index(state)]
    assert not proposer.slashed
```

It does so by verifying that:

* The block is for the current `state.slot`
* The parent block hash is correct
* The proposer is not slashed

`process_block_header` also stores the block header in `state` (for later use in the state transition function).

> Note: the `state_root` is set to `Bytes32()` (empty bytes) to avoid the circular dependency that arises from the state root being embedded in the block, while the block is part of the state. This state root is added at the start of the next slot during `process_slot`.

#### RANDAO

#### `process_randao`

[`process_randao`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#randao) verifies that the block's RANDAO reveal is the valid signature of the current epoch by the proposer and if so, `xor`'s the `hash` of the `randao_reveal` into the epoch's `randao_mix`.

```python
def process_randao(state: BeaconState, body: BeaconBlockBody) -> None:
    epoch = get_current_epoch(state)
    # Verify RANDAO reveal
    proposer = state.validators[get_beacon_proposer_index(state)]
    signing_root = compute_signing_root(epoch, get_domain(state, DOMAIN_RANDAO))
    assert bls.Verify(proposer.pubkey, signing_root, body.randao_reveal)
    # Mix in RANDAO reveal
    mix = xor(get_randao_mix(state, epoch), hash(body.randao_reveal))
    state.randao_mixes[epoch % EPOCHS_PER_HISTORICAL_VECTOR] = mix
```

#### Eth1 data

#### `process_eth1_data`

[`process_eth1_data`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#eth1-data) adds the block's `eth1_data` to `state.eth1_data_votes`. 


```python
def process_eth1_data(state: BeaconState, body: BeaconBlockBody) -> None:
    state.eth1_data_votes.append(body.eth1_data)
    if state.eth1_data_votes.count(body.eth1_data) * 2 > SLOTS_PER_ETH1_VOTING_PERIOD:
        state.eth1_data = body.eth1_data
```

If more than half of the votes in the voting period are for the same `eth1_data`, it updates `state.eth1_data` with this winning `eth1_data`.

#### Operations

#### `process_operations`

```python
def process_operations(state: BeaconState, body: BeaconBlockBody) -> None:
    # Verify that outstanding deposits are processed up to the maximum number of deposits
    assert len(body.deposits) == min(MAX_DEPOSITS, state.eth1_data.deposit_count - state.eth1_deposit_index)

    for operations, function in (
        (body.proposer_slashings, process_proposer_slashing),
        (body.attester_slashings, process_attester_slashing),
        (body.attestations, process_attestation),
        (body.deposits, process_deposit),
        (body.voluntary_exits, process_voluntary_exit),
        # @process_shard_receipt_proofs
    ):
        for operation in operations:
            function(state, operation)
```

[`process_operations`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#operations) first verifies that the expected number (`min(MAX_DEPOSITS, state.eth1_data.deposit_count - state.eth1_deposit_index)`) of `Deposit` operations are included.

It then processes the included [beacon chain operations](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-operations) by order of type (`ProposerSlashing`, `AttesterSlashing`, `Attestation`, `Deposit`, `VoluntaryExit`) and applies each operation type's function to the state and the operation in question.

##### Proposer Slashings

#### `process_proposer_slashing`

[`process_proposer_slashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#proposer-slashings) defines the validation conditions that a `ProposerSlashing` must meet to be included on chain, as well as performing the resulting state updates.

```python
def process_proposer_slashing(state: BeaconState, proposer_slashing: ProposerSlashing) -> None:
    # Verify header slots match
    assert proposer_slashing.signed_header_1.message.slot == proposer_slashing.signed_header_2.message.slot
    # Verify the headers are different
    assert proposer_slashing.signed_header_1 != proposer_slashing.signed_header_2
    # Verify the proposer is slashable
    proposer = state.validators[proposer_slashing.proposer_index]
    assert is_slashable_validator(proposer, get_current_epoch(state))
    # Verify signatures
    for signed_header in (proposer_slashing.signed_header_1, proposer_slashing.signed_header_2):
        domain = get_domain(state, DOMAIN_BEACON_PROPOSER, compute_epoch_at_slot(signed_header.message.slot))
        signing_root = compute_signing_root(signed_header.message, domain)
        assert bls.Verify(proposer.pubkey, signing_root, signed_header.signature)

    slash_validator(state, proposer_slashing.proposer_index)
```

It verifies the following validation conditions:

* That the `slot` of `header_1` and `header_2` are equal
* That `header_1` and `header_2` are not equal
* That the proposer is slashable (`is_slashable_validator`). This means verifying that:
    * The validator has not yet been `slashed` 
    * The validator has been activated in the past -- i.e.`current_epoch` is greater than or equal to the validator `activation_epoch`
    * The validator is not yet withdrawable -- i.e. `current_epoch` is less than the validator `withdrawable_epoch`
* That both `header_1` and `header_2` have valid signatures from the validator defined by `proposer_index`

```python
def is_slashable_validator(validator: Validator, epoch: Epoch) -> bool:
    """
    Check if ``validator`` is slashable.
    """
    return (not validator.slashed) and (validator.activation_epoch <= epoch < validator.withdrawable_epoch)
```

If the above conditions are met, `process_proposer_slashing` performs state updates to slash the `proposer`. This happens in the `slash_validator(state, proposer_index)` function call.


```python
def slash_validator(state: BeaconState,
                    slashed_index: ValidatorIndex,
                    whistleblower_index: ValidatorIndex=None) -> None:
    """
    Slash the validator with index ``slashed_index``.
    """
    epoch = get_current_epoch(state)
    initiate_validator_exit(state, slashed_index)
    validator = state.validators[slashed_index]
    validator.slashed = True
    validator.withdrawable_epoch = max(validator.withdrawable_epoch, Epoch(epoch + EPOCHS_PER_SLASHINGS_VECTOR))
    state.slashings[epoch % EPOCHS_PER_SLASHINGS_VECTOR] += validator.effective_balance
    decrease_balance(state, slashed_index, validator.effective_balance // MIN_SLASHING_PENALTY_QUOTIENT)

    # Apply proposer and whistleblower rewards
    proposer_index = get_beacon_proposer_index(state)
    if whistleblower_index is None:
        whistleblower_index = proposer_index
    whistleblower_reward = Gwei(validator.effective_balance // WHISTLEBLOWER_REWARD_QUOTIENT)
    proposer_reward = Gwei(whistleblower_reward // PROPOSER_REWARD_QUOTIENT)
    increase_balance(state, proposer_index, proposer_reward)
    increase_balance(state, whistleblower_index, whistleblower_reward - proposer_reward)
```

##### Attester Slashings

#### `process_attester_slashing`

[`process_attester_slashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attester-slashings) defines the validation conditions that an `AttesterSlashing` must meet to be included on chain, as well as performing the resulting state updates.

```python
def process_attester_slashing(state: BeaconState, attester_slashing: AttesterSlashing) -> None:
    attestation_1 = attester_slashing.attestation_1
    attestation_2 = attester_slashing.attestation_2
    assert is_slashable_attestation_data(attestation_1.data, attestation_2.data)
    assert is_valid_indexed_attestation(state, attestation_1)
    assert is_valid_indexed_attestation(state, attestation_2)

    slashed_any = False
    indices = set(attestation_1.attesting_indices).intersection(attestation_2.attesting_indices)
    for index in sorted(indices):
        if is_slashable_validator(state.validators[index], get_current_epoch(state)):
            slash_validator(state, index)
            slashed_any = True
    assert slashed_any
```

It verifies the following validation conditions:

* That `attestation_1.data` and `attestation_2.data` are slashable if they turn out to be what we call either a *double* or *surround* vote (this is checked by `is_slashable_attestation_data(attestation_1.data, attestation_2.data`):

    * Where *Double* means that both of the following conditions hold:
        * `attestation_1.data` does not equal `attestation_2.data`
        * `attestation_1.data.target.epoch` equals `attestation_2.data.target.epoch`

    * And *Surround* means that both of the following hold:
        * `attestation_1.data.source.epoch` is less than `attestation_2.data.source.epoch`
        * `attestation_2.data.target.epoch` is less than `attestation_1.data.target.epoch`
* That both `attestation_1` and `attestation_2` are valid `IndexedAttestation`s (i.e. have valid indices and signatures, as checked by the `is_valid_indexed_attestation` function calls)

```python
def is_slashable_attestation_data(data_1: AttestationData, data_2: AttestationData) -> bool:
    """
    Check if ``data_1`` and ``data_2`` are slashable according to Casper FFG rules.
    """
    return (
        # Double vote
        (data_1 != data_2 and data_1.target.epoch == data_2.target.epoch) or
        # Surround vote
        (data_1.source.epoch < data_2.source.epoch and data_2.target.epoch < data_1.target.epoch)
    )
```

```python
def is_valid_indexed_attestation(state: BeaconState, indexed_attestation: IndexedAttestation) -> bool:
    """
    Check if ``indexed_attestation`` has valid indices and signature.
    """
    indices = indexed_attestation.attesting_indices

    # Verify max number of indices
    if not len(indices) <= MAX_VALIDATORS_PER_COMMITTEE:
        return False
    # Verify indices are sorted and unique
    if not indices == sorted(set(indices)):
        return False
    # Verify aggregate signature
    pubkeys = [state.validators[i].pubkey for i in indices]
    domain = get_domain(state, DOMAIN_BEACON_ATTESTER, indexed_attestation.data.target.epoch)
    signing_root = compute_signing_root(indexed_attestation.data, domain)
    return bls.FastAggregateVerify(pubkeys, signing_root, indexed_attestation.signature)
```

After verifying the above validation conditions, `process_attester_slashing` checks which of the validators in the intersecting set of indices from `attestation_1` and `attestation_2` are slashable during the current epoch (`is_slashable_validator`), and slashes the ones that are (`slash_validator`).

Finally, it verifies that at least one validator was slashed in the above operation (`slashed_any`).

##### Attestations

#### `process_attestation`

[`process_attestation`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestations) defines the validation conditions that an `Attestation` must meet to be included on chain, as well as performing the resulting state updates.

```python
def process_attestation(state: BeaconState, attestation: Attestation) -> None:
    data = attestation.data
    assert data.index < get_committee_count_at_slot(state, data.slot)
    assert data.target.epoch in (get_previous_epoch(state), get_current_epoch(state))
    assert data.target.epoch == compute_epoch_at_slot(data.slot)
    assert data.slot + MIN_ATTESTATION_INCLUSION_DELAY <= state.slot <= data.slot + SLOTS_PER_EPOCH

    committee = get_beacon_committee(state, data.slot, data.index)
    assert len(attestation.aggregation_bits) == len(committee)

    pending_attestation = PendingAttestation(
        data=data,
        aggregation_bits=attestation.aggregation_bits,
        inclusion_delay=state.slot - data.slot,
        proposer_index=get_beacon_proposer_index(state),
    )

    if data.target.epoch == get_current_epoch(state):
        assert data.source == state.current_justified_checkpoint
        state.current_epoch_attestations.append(pending_attestation)
    else:
        assert data.source == state.previous_justified_checkpoint
        state.previous_epoch_attestations.append(pending_attestation)

    # Verify signature
    assert is_valid_indexed_attestation(state, get_indexed_attestation(state, attestation))
```

It verifies the following validation conditions:

* That `data.index` is a valid committee index for the slot
* That `data.target.epoch` refers to either the previous or current epoch
* That the current slot (`state.slot`) is at least `MIN_ATTESTATION_INCLUSION_DELAY` slots away from the slot of the attestation
* That `state.slot` is no more than `SLOTS_PER_EPOCH` away from the slot of the attestation
* That `aggregation_bits` is the length of the assigned `committee` for the `slot`/`index`

And then:

* If `data.target.epoch` is the current epoch:
    * It verifies that `data.source` is equal to `state.current_justified_checkpoint`
    * It appends a `PendingAttestation` of the `attestation` to `state.current_epoch_attestations`
* If `data.target.epoch` is the previous epoch:
    * It verifies that `data.source` is equal to `state.previous_justified_checkpoint`
    * It appends a `PendingAttestation` of the `attestation` to `state.previous_epoch_attestations`

Finally, it validates the signature of the attestation using `is_valid_indexed_attestation`.

```python
def is_valid_indexed_attestation(state: BeaconState, indexed_attestation: IndexedAttestation) -> bool:
    """
    Check if ``indexed_attestation`` has valid indices and signature.
    """
    indices = indexed_attestation.attesting_indices

    # Verify max number of indices
    if not len(indices) <= MAX_VALIDATORS_PER_COMMITTEE:
        return False
    # Verify indices are sorted and unique
    if not indices == sorted(set(indices)):
        return False
    # Verify aggregate signature
    pubkeys = [state.validators[i].pubkey for i in indices]
    domain = get_domain(state, DOMAIN_BEACON_ATTESTER, indexed_attestation.data.target.epoch)
    signing_root = compute_signing_root(indexed_attestation.data, domain)
    return bls.FastAggregateVerify(pubkeys, signing_root, indexed_attestation.signature)
```

##### Deposits

#### `process_deposit`

[`process_deposit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#deposits) defines the validation conditions that a `Deposit` must meet to be included on chain, as well as performing the resulting state updates.

```python
def process_deposit(state: BeaconState, deposit: Deposit) -> None:
    # Verify the Merkle branch
    assert is_valid_merkle_branch(
        leaf=hash_tree_root(deposit.data),
        branch=deposit.proof,
        depth=DEPOSIT_CONTRACT_TREE_DEPTH + 1,  # Add 1 for the List length mix-in
        index=state.eth1_deposit_index,
        root=state.eth1_data.deposit_root,
    )

    # Deposits must be processed in order
    state.eth1_deposit_index += 1

    pubkey = deposit.data.pubkey
    amount = deposit.data.amount
    validator_pubkeys = [v.pubkey for v in state.validators]
    if pubkey not in validator_pubkeys:
        # Verify the deposit signature (proof of possession) which is not checked by the deposit contract
        deposit_message = DepositMessage(
            pubkey=deposit.data.pubkey,
            withdrawal_credentials=deposit.data.withdrawal_credentials,
            amount=deposit.data.amount,
        )
        domain = compute_domain(DOMAIN_DEPOSIT)  # Fork-agnostic domain since deposits are valid across forks
        signing_root = compute_signing_root(deposit_message, domain)
        if not bls.Verify(pubkey, signing_root, deposit.data.signature):
            return

        # Add validator and balance entries
        state.validators.append(Validator(
            pubkey=pubkey,
            withdrawal_credentials=deposit.data.withdrawal_credentials,
            activation_eligibility_epoch=FAR_FUTURE_EPOCH,
            activation_epoch=FAR_FUTURE_EPOCH,
            exit_epoch=FAR_FUTURE_EPOCH,
            withdrawable_epoch=FAR_FUTURE_EPOCH,
            effective_balance=min(amount - amount % EFFECTIVE_BALANCE_INCREMENT, MAX_EFFECTIVE_BALANCE),
        ))
        state.balances.append(amount)
    else:
        # Increase balance by deposit amount
        index = ValidatorIndex(validator_pubkeys.index(pubkey))
        increase_balance(state, index, amount)
```

For new validators, the signature is verified, since it must be valid for the funds to be awarded to a newly allocated `Validator` in the registry.

However, in this case, two things are different from all the other signature verifications:

- The signature domain only includes a deposit-domain-type. Not a fork-version. Deposits are made with the default -- the genesis fork version. Which means they are valid across fork transitions
- Even though the state transition may still be valid, the `Validator` is not allocated to the registry if the signature is invalid. Note that a user's signature may well be invalid, since it is not checked in the deposit contract

When handling deposits from validators that already exist (in other words, in the case where the `pubkey` specified in the `Deposit` already exists in the registry):

- Signatures are not validated (since proof of possession was already validated in the initial `Deposit` for the `pubkey` in question)
- No fields are updated other than to increase the validator balance by the deposit `amount`

##### Voluntary Exits

#### `process_voluntary_exit`

[`process_voluntary_exit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#voluntary-exits) defines the validation conditions that a `VoluntaryExit` must meet to be included on chain as well as performing the resulting state updates.

```python
def process_voluntary_exit(state: BeaconState, signed_voluntary_exit: SignedVoluntaryExit) -> None:
    voluntary_exit = signed_voluntary_exit.message
    validator = state.validators[voluntary_exit.validator_index]
    # Verify the validator is active
    assert is_active_validator(validator, get_current_epoch(state))
    # Verify exit has not been initiated
    assert validator.exit_epoch == FAR_FUTURE_EPOCH
    # Exits must specify an epoch when they become valid; they are not valid before then
    assert get_current_epoch(state) >= voluntary_exit.epoch
    # Verify the validator has been active long enough
    assert get_current_epoch(state) >= validator.activation_epoch + PERSISTENT_COMMITTEE_PERIOD
    # Verify signature
    domain = get_domain(state, DOMAIN_VOLUNTARY_EXIT, voluntary_exit.epoch)
    signing_root = compute_signing_root(voluntary_exit, domain)
    assert bls.Verify(validator.pubkey, signing_root, signed_voluntary_exit.signature)
    # Initiate exit
    initiate_validator_exit(state, voluntary_exit.validator_index)
```

Voluntary exits are the most straightforward operation: if a validator is active, hasn't exited yet, and has served for a sufficient amount of time, he or she can *opt in* to stop his or her duties at any time.

*Opt in* is verified with a signature. The validator will then be assigned two epochs: one for exit from its active duties (`exit_epoch`) and another for subsequent withdrawal (`withdrawable_epoch`).

> Note: as mentioned above, to be eligible to exit, a validator has to be active and have served for sufficient time. 

The assigned exit epoch may be delayed if other validators are already in queue for an exit (similar to activation, there is a churn to avoid large sudden changes in the validator set).

Finally, exited validators must wait a small period of time before withdrawing. During this period of time, they may be penalised for previously undetected bad behavior. This is why there is a separate assigned epoch for withdrawals (`withdrawable_epoch`).
