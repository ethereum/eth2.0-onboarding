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

> Note: while there are several possible states a validator can be in, only those marked as **active** can take part in the Ethereum 2.0 protocol. This is why it's important to keep track of which state validators are in.

The linking in of shard chains has evolved over time from including a data root, to linking in more elaborate transition data at a per-block pace for all shards.

Improving the UX of cross-shard communication, has required a reduction in the initial shard count (from 1024 to 64). However this has been offset by an increase in other parameters -- such as increasing the maximum number of shards per slot from 16 to 64.

You can find a discussion of the new shard chain/crosslink structure [here](https://notes.ethereum.org/@vbuterin/HkiULaluS).

> Note: the attested inclusion of a shard transition is called a "crosslink".

To enable state transitions, the beacon chain has a `state_transition` function which takes as input a `BeaconState` (pre state) and a `BeaconBlock` and returns a new beacon state (what we call a post state).

Beginning with the genesis state, the post state of a block is considered valid if it passes all of the guards within the `state_transition` function.

> Note: the pre state of a block is defined as being a valid post state of the previous block. This definition extends recursively all the way back to the genesis state. 

### Fork Choice Rule

Given a block tree, the [fork choice rule](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/fork-choice.md) provides a single chain (the canonical chain) and resulting state based upon recent messages from validators.

The fork choice rule consumes the block-tree along with the set of most recent attestations from active validators, and identifies a block as the current head of the chain.

> Note: an attestation is a vote for a block proposal

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

> Note: the committed to `attestation.data.source` serves as the FFG source pair discussed in depth in ["Combining GHOST and Casper"](https://github.com/ethereum/research/blob/master/papers/ffg%2Bghost/paper.pdf), while the committed to `attestation.data.target` serves as the FFG target pair.

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

## Eth2 parameters

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

The main benefit of SSZ-tree-hashing is its support for different tree depths when merkleizing the underlying data. This allows for any of the contents of complex data structures to be summarized in-place as a merkle root.

The full expansion can be proven for this root. Compose this, and partial datastructures can be proven as well.

The type based tree-structure enables efficient proof navigation, and sophisticated caching techniques to be used on consensus objects (e.g. `BeaconState`). **Techniques like these greatly reduce the amount of hashing work that needs to be done at each slot, without compromising the data-type expressiveness of the consensus types.**

The serialization of SSZ is focused on determinism and efficient lookups. It is not a streaming encoding, but generally very efficient. And more standard than previous approaches such as RLP. Additionaly the complete coverage of serialization and hash-tree-root on the same type system avoids gaps in functionality and ambiguity.


### Beacon operations

[Beacon operations](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-operations) are datastructures that can be added to a `BeaconBlock` by a block proposer.

They are used to transmit messages concerning:

- Proposer slashings
- Attester slashings
- Attestations
- Deposits
- Voluntary exits

They are the primary vehicle through which messages related to the validation/construction of the chain are communicated. You can think of them as validator-level transactions to the beacon chain state machine.

There is a maximum number of beacon operations allowed per block. And different operations may have different maximum values associated with them. These numbers are defined in the constants in the [max operations per block](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#max-operations-per-block) subsection of the Beacon chain specification.

#### `ProposerSlashing`

A [`ProposerSlashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#proposerslashing) operation is used to police potentially malicious validator block proposal activity.

>Note: slashings are major penalties given for malicious operations.

 Validators can be slashed if they sign two different beacon blocks for the same slot. This makes duplicate block proposals expensive. The idea is to disincentivize activity that might lead to forking and conflicting views of the canonical chain.
 
 Some important points:
 
* `ProposerSlashing` contains proof that the slashable offense has occurred.
* It contains three fields:
    * `proposer_index` - The validator index of the validator to be slashed for double proposing
    * `signed_header_1` - The signed header of the first of the two slashable beacon blocks
    * `signed_header_2` - The signed header of the second of the two slashable beacon blocks
* Since `hash_tree_root(signed_block.message) == hash_tree_root(signed_block_header.message)`, a single signature is valid for both data structures. This means `SignedBeaconBlockHeader` can be used as a proof to reduce data size.

<!insert_class `ProposerSlashing`>

<!insert_class `SignedBeaconBlockHeader`>

<!insert_class `BeaconBlockHeader`>

#### `AttesterSlashing`

An [`AttesterSlashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attesterslashing) operation is used to police potentially malicious validator attestation activity that might lead to finalizing two conflicting chains.

Validators can be slashed if they sign two conflicting attestations -- where conflicting is defined by [`is_slashable_attestation_data`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#is_slashable_attestation_data) which checks if the attestations are slashable according to Casper FFG rules (in particular the "double" and "surround" vote conditions).

It contains two fields:

* `attestation_1` - The first of the two slashable attestations (in [`IndexedAttestation`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#indexedattestation) form).
* `attestation_2` - The second of the two slashable attestations (in `IndexedAttestation` form).

>Note that we use an `IndexedAttestation`, as opposed to a bitfield based `Attestation`. This allows us to check if the attestations are slashable without recomputing (historical) committee indices.

<!insert_class `AttesterSlashing`>

<!insert_class `IndexedAttestation`>

#### `Attestation`

An [`Attestation`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestation) is the primary message type that validators create for consensus.

The core of the message is the [`AttestationData`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestationdata) which contains fields signalling support for the head of the chain and the FFG vote.

> Note that in phase 1, `AttestationData` will have an additional shard data field.

Although beacon blocks are only created by one validator per slot, _all_ validators have a chance to create one attestation per epoch (through their assignment to a Beacon Committee). In the optimal case, all active validators create and have an attestation included into a block during each epoch.

`AttestationData` is the primary component committed to by each validator. It has five fields:

   * `slot` - the slot that the validator/committee is assigned to attest to
   * `index` - the index of the committee making the attestation (committee indices are mapped to shards in Phase 1)
   * `beacon_block_root` - block root of the beacon block seen as the head of the chain during the assigned slot
   * `source` - the most recent justified `Checkpoint` in the `BeaconState` during the assigned slot
   * `target` - the `Checkpoint` the attesters are attempting to justify (the current epoch and epoch boundary block)

<!insert_class `AttestationData`>

<!insert_class `Checkpoint`>
    

The outer datastructure -- `Attestation` -- contains the aggregate signature and the participation bitfield required for verification of the signature. It has three fields:

   * `aggregation_bits` - a list of bits containing a single bit for each member of the committee. Each validator that participated in this aggregate signature is assigned a value of `1`. These bits are ordered by the sort of the associated crosslink committee.
   * `data` - the `AttestationData` that was signed by the validator or collection of validators.
   * `signature` - the aggregate BLS signature of the attestation.
   
   <!insert_class `Attestation`>

#### `Deposit`

A [`Deposit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#deposit) represents incoming validator deposits from the eth1 [deposit contract](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/deposit-contract.md).

* Fields
    * `proof` - the merkle proof against the current `state.eth1_data.root` in the `BeaconState`. Note that the `+ 1` in the vector length is due to the SSZ length mixed into the root.
    * `data` - the [`DepositData`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#depositdata) submit to the deposit contract to be verified using the `proof` against the `state.eth1_data.root`.
        * `pubkey` - BLS12-381 pubkey to be used by the validator to sign messages
        * `withdrawal_credentials` - `BLS_WITHDRAWAL_PREFIX` concatenated with the last 31 bytes of the hash of an offline pubkey to be used to withdraw staked funds after exiting. This key is not used actively in validation and can be kept in cold storage.
        * `amount` - amount in Gwei that was deposited
        * `signature` - signature of the `DepositMessage(pubkey, amount, withdrawal_credentials)` using the `privkey` pair of the `pubkey`. This is used as a one-time "proof of possession" required for securely using BLS keys. But also signs over the withdrawal_credentials, essential to avoid injection of other withdrawal credentials.

No deposit index is explicitly defined, as it is already verified through the merkle inclusion proof. (And can easily be derived from the mix-in node in the proof).

<!insert_class `Deposit`>

<!insert_class `DepositData`>

<!insert_class `DepositMessage`>

#### `VoluntaryExit`

A [`VoluntaryExit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#voluntaryexit) allows a validator to voluntarily exit validation duties. This object is wrapped into a [`SignedVoluntaryExit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#signedvoluntaryexit) which is included on chain.

* `VoluntaryExit`
    * Fields
        * `epoch` - minimum epoch at which this exit can be included on chain. Helps prevent accidental/nefarious use in chain reorgs/forks.
        * `validator_index` - the exiting validator's index within the `BeaconState` validator registry.
* `SignedVoluntaryExit`
    * Fields
        * `message` - The `VoluntaryExit` signed over in this object.
        * `signature` - signature of the `VoluntaryExit.message` included in this object. Signed by the pubkey associated with the `Validator` defined by `validator_index`.

<!insert_class `VoluntaryExit`>

<!insert_class `SignedVoluntaryExit`>

### `BeaconBlock`

[`BeaconBlock`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beaconblock) is the primary component of the beacon chain. Each block contains a reference (`parent_root`) to the block root of its parent forming a chain of ancestors ending with the parent-less genesis block. A `BeaconBlock` is composed of an outer container with a limited set of "header" fields and a [`BeaconBlockBody`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beaconblockbody) which contains fields specific to the action of the block on state. In optimal operation, a single `BeaconBlock` is proposed during each slot by the selected proposer from the current epoch's active validator set.

* `BeaconBlock`
    * This is the main container/header for a beacon block.
    * Note that `hash_tree_root(BeaconBlock) == hash_tree_root(BeaconBlockHeader)` and thus signatures of each are equivalent.
    * Fields
        * `slot` - the slot for which this block is created. Must be greater than the slot of the block defined by `parent_root`
        * `parent_root` - the block root of the parent block, forming a block chain
        * `state_root` - the hash root of the post state of running the state transition through this block
        * `body` - a `BeaconBlockBody` which contains fields for each of the [beacon operation](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-operations) objects as well as a few proposer input fields

<!insert_class `BeaconBlock`>

<!insert_class `BeaconBlockHeader`>

* `BeaconBlockBody`
    * Fields
        * `randao_reveal` - the `BLSSignature` of the current epoch (by the block proposer) and, when mixed in with the other proposers' reveals, constitutes the seed for randomness
        * `eth1_data` - a vote on recent data from the Eth1 chain consisting of the following fields:
            * `deposit_root` - the SSZ List `hash_tree_root` of all of the deposits in the deposit contract
            * `deposit_count` - the number of successful validator deposits that have been made into the deposit contract thusfar
            * `block_hash` - the eth1 block hash that contained the `deposit_root`. This `block_hash` is intended to be used for finalization of the Eth1 chain in the future
        * `graffiti` - this is 32 bytes for validators to decorate as they choose with no defined in-protocol use
        * `proposer_slashings`, `attester_slashings`, `attestations`, `deposits`, `voluntary_exits`
            * Lists (with max lengths) of each operation type that can be included into the `BeaconBlockBody`

<!insert_class `BeaconBlockBody`>

<!insert_class `Eth1Data`>

* `SignedBeaconBlock`
    * Signed wrapper of `BeaconBlock`
    * Fields
        * `message` - The `BeaconBlock` signed over in this object.
        * `signature` - signature of the `BeaconBlock` `message` included in this object. Sign by the pubkey of the proposer for the given slot.

<!insert_class `SignedBeaconBlock`>

### Beacon state

The [`BeaconState`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beaconstate) is the resulting state of running the state transition function on a chain of `BeaconBlock`s starting from the genesis state. It contains the information about fork versioning, historical caches, data about the eth1 chain/validator deposits, the validator registry, information about randomness, information about past slashings, recent attestations, recent crosslinks, and information about finality.

#### `BeaconState`

The following are descriptions of the fields contained within the `BeaconState`, grouped into logical units:

* Fork Versioning
    * `genesis_time` -- tracks the Unix timestamp during which the genesis slot occurred. This allows a client to calculate what the current slot should be according to wall clock time
    * `slot` -- time is divided into "slots" of fixed length (`SECONDS_PER_SLOT`) at which actions occur and state transitions happen. This field tracks the slot of the containing state, not necessarily the slot according to the local wall clock
    * `fork` -- a mechanism for handling forking (upgrading) the beacon chain. The main purpose here is to handle versioning of signatures and handle objects of different signatures across fork boundaries
* History
    * `latest_block_header` -- cache of the latest block header seen in the chain defining this state. During the slot transition of the block, the header is stored _without_ the real state root but instead with a stub of `Root()` (empty `0x00` bytes). At the start of the _next slot_ transition before anything has been modified within state, the state root is calculated and added to the `latest_block_header`. This is done to eliminate the circular dependency of the state root being embedded in the block header while still allowing state roots to be referenced within the beacon state
    * `block_roots` -- per-slot store of the recent block roots. The block root for a slot is added at the start of the next slot to avoid the circular dependency due to the state root being embedded in the block. For slots that are skipped (no block in the chain for the given slot), the most recent block root in the chain prior to the current slot is stored for the skipped slot. When validators attest to a given slot, they use this store of block roots as an information source to cast their vote
    * `state_roots` -- per-slot store of the recent state roots. The state root for a slot is stored at the start of the next slot to avoid a circular dependency
    * `historical_roots` -- a double batch merkle accumulator of the latest block and state roots defined by [`HistoricalBatch`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#historicalbatch) to make historic merkle proofs against. Note that although this field grows unbounded, it grows at less than 10 KB per year
* Eth1
    * `eth1_data` -- the most recent [`Eth1Data`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#eth1data) that validators have come to consensus upon and stored in state. Validator deposits from eth1 can be processed up through the latest deposit contained within the `eth1_data` root
    * `eth1_data_votes` -- running list of votes on new `Eth1Data` to be stored in state. If any `Eth1Data` achieves `> 50%` proposer votes in a voting period, this majority data is stored in state and new deposits can be processed
    * `eth1_deposit_index` -- the index of the next deposit to be processed. Deposits must be added to the next block and processed if `state.eth1_data.deposit_count > state.eth1_deposit_index`
* Registry
    * `validators` -- a list of `Validator` records, tracking the current full registry. Each validator contains relevant data such as pubkey, withdrawal credentials, effective balance, a slashed boolean, and status (pending, active, exited, etc)
    * `balances` -- a list mapping 1:1 with the `validator_registry`. The granular/frequently changing balances are pulled out of the `validators` list to reduce the amount of re-hashing (in a cache optimized SSZ implementation) that needs to be performed at each epoch transition
* Randomness
    * `randao_mixes` -- The randao mix from each epoch for the past `EPOCHS_PER_HISTORICAL_VECTOR` epochs. At the start of each epoch, the `randao_mix` from the previous epoch is copied over as the base of the current epoch. At each block, the `hash` of the `block.randao_reveal` is xor'd into the running mix of the current epoch
* Slashings
    * `slashings` -- per-epoch store of past `EPOCHS_PER_SLASHINGS_VECTOR` epochs of the total slashed GWEI during that epoch. The sum of this list at any time gives the "recent slashed balance" and is used to calculate the proportion of balance that should be slashed for slashable validators
* Attestations
    * `Attestation`s from blocks are converted to `PendingAttestation`s and stored in state for bulk accounting at epoch boundaries. We store two separate lists:
    * `previous_epoch_attesations` -- List of `PendingAttestation`s for slots from the previous epoch. *note*: these are attestations attesting to slots in the previous epoch, not necessarily those included in blocks during the previous epoch.
    * `current_epoch_attesations` -- List of `PendingAttestation`s for slots from the current epoch. Copied over to `previous_epoch_attestations` and then emptied at the end of the current epoch processing
* Finality
    * `justification_bits` -- 4 bits used to track justification during the last 4 epochs to aid in finality calculations
    * `previous_justified_checkpoint` -- the most recent justified `Checkpoint` as it was during the previous epoch. Used to validate attestations from the previous epoch
    * `current_justified_checkpoint` -- the most recent justified `Checkpoint` during the current epoch. Used to validate current epoch attestations and for fork choice purposes
    * `finalized_checkpoint` -- the most recent finalized `Checkpoint`, prior to which blocks are never reverted.

<!insert_class `BeaconState`>

#### `Fork`

<!insert_class `Fork`>

#### `Validator`

<!insert_class `Validator`>

#### `PendingAttestation`

<!insert_class `PendingAttestation`>

#### `Checkpoint`

<!insert_class `Checkpoint`>

## State Transition

The core of Phase 0 is the [beacon chain state transition function](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-chain-state-transition-function). The post-state corresponding to a pre-state `state` and a signed beacon-block `signed_block` is defined as `state_transition(state, signed_block)`. State transitions that trigger an unhandled exception (e.g. a failed assert or an out-of-range list access) are considered invalid.

### `state_transition`

[`state_transition`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-chain-state-transition-function) is the top level function of the state transition. It accepts a pre-state and a signed beacon block, and outputs a post-state.

<!insert_function `state_transition`>

During the production of a block, its state-root is computed by running the transition with the candidate block, while disabling validation. The state-root of the block is then updated, and the block is signed. Block production is explained in greater detail in the [Validator spec](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/validator.md).

### `process_slots`

`process_slots` is the first component of the state transition function. It handles all of the state updates that happen at each slot regardless of any block. If there are "skipped" slots in between the state transition input block and its parent, then multiple slots are transitioned during `process_slots` up to the `block.slot`.

Consensus forming clients must sometimes call this function to transition state through empty slots when attesting to or producing blocks.

<!insert_function `process_slots`>

### `process_slot`

`process_slot` is called once per slot to cache recent data from the previous slot. Note that the `state` has not yet been modified so still represents the post state of the previous slot. Also `state.slot` is still equal to the previous slot (`state.slot += 1` happens after `process_slot` from within `process_slots`) so any updates to items wrt `state.slot` are accounting for the previous slot.

The unmodified state root, `previous_state_root`, is the post-state root of the previous slot. `previous_state_root` is cached in `state.state_roots`, and cached in `state.latest_block_header` if the previous slot contained a block.

`process_slot` consists of the following state changes:
* Cache state root
    * The state root of the previous slot is cached into `state.state_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT]`. Note that `hash_tree_root(state) == previous_state_root` because the `state` has not yet been modified from the post state of the previous slot
* Cache latest block header state root
    * If the previous slot contained a block (was not a skipped slot), then we insert the `previous_state_root` into the `state_root` of the cached `state.latest_block_header`. This block header remains in state through any skipped slots until the next block occurs
* Cache block root
    * We cache the `hash_tree_root` of the latest block into `state.block_roots[state.slot % SLOTS_PER_HISTORICAL_ROOT]` at each slot. If the previous slot was skipped, then the block from the latest non-skipped slot is cached

<!insert_function `process_slot`>

### `process_epoch`

`process_epoch` is only called at epoch boundaries. Note that because `state.slot` has not yet been incremented during the conditional check in `process_slots`, epoch transitions occur at the state transition defined by the `slot` that is the start of `slot % SLOTS_PER_EPOCH == 0`

### Epoch Processing

[Epoch processing](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#epoch-processing) occurs at the _start_ of the 0th slot (`slot % EPOCH_LENGTH == 0`) of each epoch. Note that `state.slot` is still equal to the previous slot (`slot % EPOCH_LENGTH == EPOCH_LENGTH - 1`) when `process_epoch` is run inside `process_slot` and only incremented after.

`process_epoch` is the primary container function that calls the rest of the epoch processing sub-functions.

<!insert_function `process_epoch`>

#### Justification and Finalization

##### Justification

Within [`process_justification_and_finalization`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#justification-and-finalization), justification updates are checked for both the previous and current epoch. We utilize the `PendingAttestation`s contained within `previous_epoch_attestations` and `current_epoch_attestations` to check if the previous and current epoch have been justified, respectively. Note that these lists are already pre-filtered to only contain attestations that voted on the expected `source` designated by the current `state`.

If at least 2/3 of the active balance attests to the previous epoch, the previous epoch is justified. If at least 2/3 of the active balance attests to current epoch, the current epoch is justified.

##### Finalization

Only the 2nd, 3rd, and 4th most recent epochs can be finalized. This is done by checking which epochs (if within the past 4) is used as the source of the most recent justifications and if the epochs between source and target (if any) are also justified. If these conditions are satisfied, the source is finalized. We only consider `k=1` and `k=2` finality rules discussed in [section 4.4.3 in the draft paper](https://github.com/ethereum/research/blob/master/papers/ffg%2Bghost/paper.pdf).

<!insert_function `process_justification_and_finalization`>

#### Rewards and Penalties

All of the rewards and penalties collected from the previous epoch are applied to validator balances in [`process_rewards_and_penalties`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#rewards-and-penalties-1). Due to attestations being allowed 1 epoch for inclusion, rewards from attestations are not processed until the end of the following epoch. Due to this, rewards and penalties are skipped during the `GENESIS_EPOCH`.

In Phase 0, rewards and penalties are collected from `get_attestation_deltas` which return the rewards and penalties for each validator as tuples of lists. Rewards and penalties are separated to avoid signed integers.

<!insert_function `process_rewards_and_penalties`>

All rewards scale with the individual validator `effective_balance` and $\frac{1}{\sqrt{TotalBalance}}$. See [the design rationale](https://notes.ethereum.org/s/rkhCgQteN#Base-rewards) for explanation. We deal in units `base_reward` rewards and/or penalties for each of the components of a validator's duty.

<!insert_function `get_base_reward`>

##### `get_attestation_deltas`

`get_attestation_deltas` determines how much each validator's balance changes as a function of their attestation behaviour in the previous epoch.

* For each of the active validators and any slashed but not yet withdrawable validator:
    * Reward for FFG Source, Target & Head - If the validator's source, target, or head from any included attestation made in the previous epoch match the current source, target, or head of the previous epoch, issue a `base_reward` scaled by fraction of active validators also attesting to this correct value; else issue a `base_reward` penalty. This is a maximum of `3 * base_reward` reward (one for each correct value).
    * Reward for attestation inclusion speed - Find the attestation that was included the soonest for each validator. If such an attestation exists, give one `base_reward` split between the proposer and the included validator. Give the proposer a small reward --`base_reward` relative to the included validator divided by `PROPOSER_REWARD_QUOTIENT` -- for including the attestation. Give the remaining portion of the `base_reward` to the included validator but scale it proportional to how quickly the attestation was included.
    * Inactivity penalty - if finalization is not occurring, penalize everyone, but particularly those who are not participating. Specifically, add a `BASE_REWARD_PER_EPOCH * base_reward` penalty to each active validator, and add a `effective_balance * finality_delay // INACTIVITY_PENALTY_QUOTIENT` penalty to each validator that did not vote on the correct target. Note that if a validator is optimally participating during an inactivity leak, their balance remains unchanged [Potential bug being discussed [here](https://github.com/ethereum/eth2.0-specs/issues/1370)]

<!insert_function `get_attestation_deltas`>

#### Registry updates

Updates to the validator registry are handled in a per-epoch basis in [`process_registry_updates`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#registry-updates)

* Validators whose `effective_balance` are sufficient (`== MAX_EFFECTIVE_BALANCE`) to be activated but have not yet been added to the queue (`activation_eligibility_epoch == FAR_FUTURE_EPOCH`), are assigned as eligible for activation starting from next epoch.
* Validators whose `effective_balance` are too low (`<= EJECTION_BALANCE`) are "ejected" by being added to the withdrawal queue.
* Validators that are eligible for activation, and not yet activated, are considered queued if the chain has finalized since their eligibility aknowledgement. The queue is FIFO: sorted by time of eligibility epoch (and tie break on validator index).
* Validators that are in the activation queue are activated within the churn limit.

<!insert_function `process_registry_updates`>

#### Slashings

Validators that are slashed are done so in proportion to how many validators have been slashed within a recent time period. This occurs independently of the order in which validators are slashed. [`process_slashings`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#slashings) goes through the recent slashings and penalizes slashed validators proportionally.

<!insert_function `process_slashings`>

#### Final updates

[`process_final_updates`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#final-updates) handles sundry functionality that needs to occur with epoch-frequency, namely:
* Reset eth1 data votes if the start of a new voting period
* Recalculate effective balances of all validators (using hysteresis)
* Set slashed balances for next epoch to 0
* Set the resulting randao mix from the current epoch as the base randao mix in the next epoch.
* (Potentially) append historic batch to historical root.
* Move `current_epoch_attestations` to `previous` and empty the `current`

<!insert_function `process_final_updates`>

### Block Processing

[`process_block`](https://github.com/ethereum/eth2.0-shttps://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#block-processing) is the main function that calls the sub-functions of block processing. If any asserts or exceptions are thrown in block processing, then the block is seen as invalid and should be discarded.

<!insert_function `process_block`>

#### Block Header

[`process_block_header`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#block-header) determines whether, at a high level, a block is valid. It does so by verifying that:
* The block is for the current `state.slot`
* The parent block hash is correct
* The proposer is not slashed

`process_block_header` also stores the block header in `state` for later uses in the state transition function. _Note_ that the `state_root` is set to `Bytes32()` (empty bytes) to avoid a circular dependency. This state root is added at the start of the next slot during `process_slot`.

<!insert_function `process_block_header`>

#### RANDAO

[`process_randao`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#randao) verifies that the block's RANDAO reveal is the valid signature of the current epoch by the proposer and if so, `xor`'s the `hash` of the `randao_reveal` into the epoch's `randao_mix`.

<!insert_function `process_randao`>

#### Eth1 data

[`process_eth1_data`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#eth1-data) adds the block's `eth1_data` to `state.eth1_data_votes`. If more than half of the votes in the voting period are for the same `eth1_data`, update `state.eth1_data` with this winning `eth1_data`.

<!insert_function `process_eth1_data`>

#### Operations

[`process_operations`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#operations) first performs the following operation size check:

* Verify that the expected number (`min(MAX_DEPOSITS, state.eth1_data.deposit_count - state.eth1_deposit_index)`) of `Deposit` operations are included

`process_operations` then processes the included [beacon chain operations](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#beacon-operations) by order of type (`ProposerSlashing`, `AttesterSlashing`, `Attestation`, `Deposit`, `VoluntaryExit`) and list order within each type against the operation function specific to each operation type.

<!insert_function `process_operations`>

##### Proposer Slashings

[`process_proposer_slashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#proposer-slashings) defines the validation conditions that a `ProposerSlashing` must meet to be included on chain and the resulting state updates. A brief discussion of these conditions and updates below:

* Verify that the `slot` of `header_1` and `header_2` are equal
* Verify that `header_1` and `header_2` are not equal
* Verify that the proposer is slashable (`is_slashable_validator`)
    * Verify that the validator is not yet `slashed` 
    * Verify that the validator has been activated in the past -- i.e.`current_epoch` is greater than or equal to the validator `activation_epoch`
    * Verify that the validator is not yet withdrawable -- i.e. `current_epoch` is less than the validator `withdrawable_epoch`
* Verify that both `header_1` and `header_2` have valid signatures by the validator defined by `proposer_index`

`process_proposer_slashing` performs state updates to slash the `proposer` by calling `slash_validator(state, proposer_index)`.

<!insert_function `process_proposer_slashing`>

<!insert_function `is_slashable_validator`>

##### Attester Slashings

[`process_attester_slashing`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attester-slashings) defines the validation conditions that an `AttesterSlashing` must meet to be included on chain and the resulting state updates. A brief discussion of these conditions and updates below:

* Verify that `attestation_1.data` and `attestation_2.data` are slashable by _either_ "double" or "surround" vote (i.e. `is_slashable_attestation_data`):
    * "Double" if both of the following hold:
        * `attestation_1.data` does not equal `attestation_2.data`
        * `attestation_1.data.target.epoch` equals `attestation_2.data.target.epoch`
    * "Surround" if both of the following hold:
        * `attestation_1.data.source.epoch` less than `attestation_2.data.source.epoch`
        * `attestation_2.data.target.epoch` less than `attestation_1.data.target.epoch`
* Verify that both `attestation_1` and `attestation_2` are valid `IndexedAttestation`s (`is_valid_indexed_attestation`) (including signature verification)

For each validator in the intersecting set of indices from `attestation_1` and `attestation_2`, if the validator is slashable during the current epoch (`is_slashable_validator`), then slash the validator (`slash_validator`).

Verify that at least one validator was slashed in the above operation.

<!insert_function `process_attester_slashing`>

<!insert_function `is_slashable_attestation_data`>

<!insert_function `is_valid_indexed_attestation`>

##### Attestations

[`process_attestation`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#attestations) defines the validation conditions that an `Attestation` must meet to be included on chain and the resulting state updates. A brief discussion of these conditions and updates below:

* Verify that `data.index` is a valid committee index for the slot
* Verify that `data.target.epoch` is for either the previous or current epoch
* Verify that `state.slot` is least `MIN_ATTESTATION_INCLUSION_DELAY` slots since the slot of the attestation
* Verify that `state.slot` is no more than `SLOTS_PER_EPOCH` since the slot of the attestation
* Verify that `aggregation_bits` is the length of the assigned `committee` for the `slot`/`index`
* If `data.target.epoch` is the current epoch:
    * Verify that `data.source` is equal to `state.current_justified_checkpoint`
    * Append a `PendingAttestation` of the `attestation` to `state.current_epoch_attestations`
* If `data.target.epoch` is the previous epoch:
    * Verify that `data.source` is equal to `state.previous_justified_checkpoint`
    * Append a `PendingAttestation` of the `attestation` to `state.previous_epoch_attestations`
* Validate the signature of the attestation using `is_valid_indexed_attestation`

<!insert_function `process_attestation`>

<!insert_function `is_valid_indexed_attestation`>

##### Deposits

[`process_deposit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#deposits) defines the validation conditions that a `Deposit` must meet to be included on chain and the resulting state updates.

In case of the new validator case, the signature is verified, and must be valid for the funds to be awarded to a newly allocated `Validator` in the registry. However, two things are different from all other signature verifications:
- The signature domain only includes a deposit-domain-type. Not a fork-version. Deposits are made with the default, the genesis fork version. And thus valid across fork transitions
- The state transition is valid, but does not allocate the `Validator`, if the signature is invalid. Deposit processing must progress, even if a user makes a mistake. Signatures are not checked in the deposit contract, and can thus be invalid

In the case of an already existing validator for the `pubkey` specified in the `Deposit`:
- Signatures are not validated (proof of possession was already validated in initial `Deposit` for `pubkey`)
- No fields are updated other than increase the validator balance by the deposit `amount`

<!insert_function `process_deposit`>

##### Voluntary Exits

[`process_voluntary_exit`](https://github.com/ethereum/eth2.0-specs/blob/v0.10.0/specs/phase0/beacon-chain.md#voluntary-exits) defines the validation conditions that a `VoluntaryExit` must meet to be included on chain and the resulting state updates.

Voluntary exits are the most straightforward operation: a validator can opt in to stop its duties. Opt in is verified with a signature. The validator will then be scheduled for exit from its active duties `exit_epoch` and subsequent withdrawal `withdrawable_epoch`.

To be eligible to exit, a validator has to be active and have served for sufficient time. Scheduling assigns an exit epoch, which may be delayed if other validators are already in queue for an exit. Similar to activation, there is a churn to avoid large sudden changes in the validator set.

After exit, the validator can not directly remove funds, as the system may still penalize previously undetected bad behavior. After a minimal delay, the validator can then withdraw.

<!insert_function `process_voluntary_exit`>
