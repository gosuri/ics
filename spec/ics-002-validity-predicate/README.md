---
ics: 2
title: Validity Predicate
stage: draft
category: ibc-core
requires: 23, 24
required-by: 3
author: Juwoon Yun <joon@tendermint.com>, Christopher Goes <cwgoes@tendermint.com>
created: 2019-02-25
modified: 2019-08-15
---

## Synopsis

This standard specifies the properties that consensus algorithms of machines implementing the interblockchain
communication protocol are required to satisfy. These properties are necessary for efficient and safe
verification in the higher-level protocol abstractions. The algorithm utilised in IBC to verify the
consensus transcript & state sub-components of another machine is referred to as a "validity predicate",
and pairing it with a state that the verifier trusts forms a "light client" (often shortened to "client").

This standard also specifies how light clients will be stored, registered, and updated in the
canonical IBC handler. The stored client instances will be introspectable by a third party actor,
such as a user inspecting the state of the chain and deciding whether or not to send an IBC packet.

### Motivation

In the IBC protocol, a machine needs to be able to verify updates to the state of another machine
which the other machine's consensus algorithm has agreed upon, and reject any possible updates
which the other machine's consensus algorithm has not agreed upon. A light client is the algorithm
with which a machine can do so. This standard formalises the light client model and requirements,
so that the IBC protocol can easily integrate with new machines which are running new consensus algorithms
as long as associated light client algorithms fulfilling the listed requirements are provided.

Beyond the properties described in this specification, IBC does not impose any requirements on
the internal operation of machines and their consensus algorithms. A machine may consist of a
single process signing operations with a private key, a quorum of processes signing in unison,
many processes operating a Byzantine fault-tolerant consensus algorithm, or other configurations yet to be invented
— from the perspective of IBC, a machine is defined entirely by its light client validation & equivocation detection logic.

Clients could also act as thresholding views of other clients. In the case where
modules utilising the IBC protocol to interact with probabilistic-finality consensus algorithms
which might require different finality thresholds for different applications, one write-only
client could be created to track headers and many read-only clients with different finality
thresholds (confirmation depths after which state roots are considered final) could use that same state.

### Definitions

* `get`, `set`, `Key`, and `Identifier` are as defined in [ICS 24](../ics-024-host-requirements).

* `CommitmentRoot` is as defined in [ICS 23](../ics-023-vector-commitments). It must provide an inexpensive way for
  downstream logic to verify whether key-value pairs are present in state at a particular height.

* `ConsensusState` is an opaque type representing the state of a validity predicate.
  `ConsensusState` must be able to verify state updates agreed upon by the associated consensus algorithm.
  It must also be serialisable in a canonical fashion so that third parties, such as counterparty machines,
  can check that a particular machine has stored a particular `ConsensusState`.

* `ClientState` is an opaque type representing the state of a client.
  A `ClientState` must expose query functions to retrieve trusted state roots at previously
  verified heights and retrieve the current `ConsensusState`.

* `createClient`, `queryClient`, `updateClient`, `freezeClient`, and `deleteClient` function signatures are as defined in [ICS 25](../ics-025-handler-interface).
  The function implementations are defined in this specification.

### Desired Properties

Light clients must provide a secure algorithm to verify other chains' canonical headers,
using the existing `ConsensusState`. The higher level abstractions will then be able to verify
sub-components of the state with the `CommitmentRoot`s stored in the `ConsensusState`, which are
guaranteed to have been committed by the other chain's consensus algorithm.

`ValidityPredicate`s are expected to reflect the behaviour of the full nodes which are running the  
corresponding consensus algorithm. Given a `ConsensusState` and `[Message]`, if a full node
accepts the new `Header` generated with `Commit`, then the light client MUST also accept it,
and if a full node rejects it, then the light client MUST also reject it.

Light clients are not replaying the whole message transcript, so it is possible under cases of
consensus misbehaviour that the light clients' behaviour differs from the full nodes'.
In this case, an misbehaviour proof which proves the divergence between the `ValidityPredicate`
and the full node can be generated and submitted to the chain so that the chain can safely deactivate the
light client, invalidate past state roots, and await higher-level intervention.

## Technical Specification

### Data Structures

#### ConsensusState

`ConsensusState` is a opaque type defined by each consensus algorithm, used by the validity predicate to
verify new commits & state roots. Likely the structure will contain the last commit produced by
the consensus process, including signatures and validator set metadata.

`ConsensusState` MUST be generated from an instance of `Consensus`, which assigns unique heights
for each `ConsensusState`. Two `ConsensusState`s on the same chain SHOULD NOT have the same height
if they do not have equal commitment roots. Such an event is called an "equivocation", and should one occur,
a proof should be generated and submitted so that the client can be frozen.

The `ConsensusState` of a chain MUST have a canonical serialization, so that other chains can check
that a stored consensus state is equal to another.

```typescript
type ConsensusState = bytes
```

#### Header

A `Header` is a blockchain header which provides information to update a `ConsensusState`.
Headers can be submitted to an associated client to update the stored `ConsensusState`.

Headers are representation-opaque to the IBC protocol. They likely contain a height, a proof,
a commitment root, and possibly updates to the validity predicate.

```typescript
type Header = bytes
```

#### ValidityPredicate

A `ValidityPredicate` is a light client function to verify `Header`s depending on the current `ConsensusState`.
Using the ValidityPredicate SHOULD be far more computationally efficient than replaying the full consensus algorithm
for the given parent `Header` and the list of network messages.

The `ValidityPredicate` type is defined as

```typescript
type ValidityPredicate = (Header) => (bool)
```

The boolean returned indicates whether the provided header was valid.

If the provided header was valid, the client MUST also mutate internal state to store
now-finalised consensus roots and update any necessary signature authority tracking (e.g.
changes to the validator set) for future calls to the validity predicate.

The detailed specification of `ValidityPredicate` can be found in [CONSENSUS.md](./CONSENSUS.md).

#### MisbehaviourPredicate

An `MisbehaviourPredicate` is a light client function used to check if data
constitutes a violation of the consensus protocol. This might be two headers
with different state roots but the same height, a signed header containing invalid
state transitions, or other evidence as defined by the consensus algorithm.

The `MisbehaviourPredicate` type is defined as

```typescript
type MisbehaviourPredicate = (bytes) => (bool)
```

The boolean returned indicates whether the evidence of misbehaviour was valid.
If misbehaviour was valid, the client MUST also mutate internal state to mark appropriate heights which
were previously considered valid invalid, according to the nature of the misbehaviour.

More details about `MisbehaviourPredicate`s can be found in [CONSENSUS.md](./CONSENSUS.md)

#### ClientState

`ClientState` is the light client state, which must expose the latest `ConsensusState`, a validity predicate,
a misbehaviour predicate, and finally a function to lookup verified `CommitmentRoot`s,
which are then used to verify presence or absence of particular key-value pairs in state at particular heights.
It may keep arbitrary internal state to track verified roots and past misbehaviours.

Light clients are representation-opaque — different consensus algorithms can define different light client update algorithms —
but they must expose this common set of query functions to the IBC handler.

```typescript
interface ClientState {
  initialize: (ConsensusState) => void
  getVerifiedRoot: (uint64) => CommitmentRoot
  validityPredicate: ValidityPredicate
  misbehaviourPredicate: MisbehaviourPredicate
}
```

where
  * `initialize` must set an initial `ConsensusState` provided when the light client is created, and initialize any internal state accordingly
  * `verifiedRoot` is a function mapping heights to previously verified `CommitmentRoot`s
  * `validityPredicate` is a validity predicate as described above
  * `misbehaviourPredicate` is a misbehaviour predicate as described above

The `consensusState` MUST be stored under a particular key, defined below, so that other chains can verify that a particular consensus state has been stored.

### Sub-protocols

IBC handlers MUST implement the functions defined below.

#### Preliminaries

Clients are stored under a unique `Identifier` prefix.
This ICS does not require that client identifiers be generated in a particular manner, only that they be unique.

`clientKey` takes an `Identifier` and returns a `Key` under which to store a particular client.

```typescript
function clientKey(id: Identifier): Key {
  return "clients/{id}"
}
```

Consensus states MUST be stored separately so that they can be independently verified.

`consensusStateKey` takes an `Identifier` and returns a `Key` under which to store the consensus state of a client.

```typescript
function consensusStateKey(id: Identifier): Key {
  return "clients/{id}/consensusState"
}
```

##### Utilising past roots

To avoid race conditions between client updates (which change the state root) and proof-carrying
transactions in handshakes or packet receipt, many IBC handler functions allow the caller to specify
a particular past root to reference, which is looked up by height. IBC handler functions which do this
must ensure that they also perform any requisite checks on the height passed in by the caller to ensure
logical correctness.

#### Create

Calling `createClient` with the specified identifier & initial consensus state creates a new client.

```typescript
function createClient(id: Identifier, clientState: ClientState, consensusState: ConsensusState) {
  assert(get(clientKey(id)) === null)
  set(clientKey(id), clientState)
  clientState.initialize(consensusState)
}
```

#### Query

Client consensus state and previously verified roots can be queried by identifier.

```typescript
function queryClientConsensusState(id: Identifier): ConsensusState {
  return get(consensusStateKey(id))
}
```

```typescript
function queryClientRoot(id: Identifier, height: uint64): CommitmentRoot {
  return get(clientKey(id)).getVerifiedRoot(height)
}
```

#### Update

Updating a client is done by submitting a new `Header`. The `Identifier` is used to point to the
stored `ClientState` that the logic will update. When a new `Header` is verified with
the stored `ClientState`'s `ValidityPredicate` and `ConsensusState`, the client MUST
update its internal state accordingly, possibly finalising commitment roots and
updating the signature authority logic in the stored consensus state.

```typescript
function updateClient(id: Identifier, header: Header) {
  client = get(clientKey(id))
  assert(client !== null)
  assert(client.validityPredicate(header))
}
```

#### Misbehaviour

If the client detects evidence of misbehaviour, the client can be alerted, possibly invalidating
previously valid state roots & preventing future updates.

```typescript
function submitMisbehaviour(id: Identifier, evidence: bytes) {
  client = get(clientKey(id))
  assert(client !== null)
  assert(client.misbehaviourPredicate(evidence))
}
```

### Example Implementation

An example validity predicate is constructed for a chain running a single-operator consensus algorithm,
where the valid blocks are signed by the operator. The operator signing Key
can be changed while the chain is running.

The client-specific types are then defined as follows:
- `ConsensusState` stores the latest height and latest public key
- `Header`s contain a height, a new commitment root, a signature by the operator, and possibly a new public key
- `ValidityPredicate` checks that the submitted height is monotonically increasing and that the signature is correct
- `MisbehaviourPredicate` checks for two headers with the same height & different commitment roots

```typescript
interface ConsensusState {
  height: uint64
  publicKey: PublicKey
}

interface ClientState {
  frozen: bool
  verifiedRoots: Map<uint64, CommitmentRoot>
}

interface Header {
  height: uint64
  commitmentRoot: CommitmentRoot
  signature: Signature
  newPublicKey: Maybe<PublicKey>
}

interface Evidence {
  h1: Header
  h2: Header
}

function commit(root: CommitmentRoot, height: uint64, newPublicKey: Maybe<PublicKey>): Header {
  signature = privateKey.sign(root, height, newPublicKey)
  header = Header{height, root, signature}
  return header
}

function validityPredicate(clientState: ClientState, header: Header): bool {
  if (!consensusState.publicKey.verify(header.signature))
    return false
  if (header.newPublicKey !== null)
    consensusState.publicKey = header.newPublicKey
  client.verifiedRoots[height] = header.commitmentRoot
  return true
}

function getVerifiedRoot(clientState: ClientState, height: uint64) {
  assert(!client.frozen)
  return client.verifiedRoots[height]
}

function misbehaviourPredicate(consensusState: ConsensusState, evidence: Evidence): bool {
  h1 = evidence.h1
  h2 = evidence.h2
  if (h1.height === h2.height &&
      h1.commitmentRoot !== h2.commitmentRoot &&
      consensusState.publicKey.verify(h1.signature) &&
      consensusState.publicKey.verify(h2.signature)) {
    client.frozen = true
    return true
  }
  return false
}
```

### Properties & Invariants

- Client identifiers are first-come-first-serve: once a client identifier has been allocated, all future headers & roots-of-trust stored under that identifier will have satisfied the client's validity predicate.

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

In a future version, this ICS will define a new function `unfreezeClient` that can be called 
when the application logic resolves an misbehaviour event.

## Example Implementation

Coming soon.

## Other Implementations

Coming soon.

## History

March 5th 2019: Initial draft finished and submitted as a PR
May 29 2019: Various revisions, notably multiple commitment-roots
Aug 15 2019: Major rework for clarity around client interface

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).