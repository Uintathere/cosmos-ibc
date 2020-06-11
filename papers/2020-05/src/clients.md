The *client* abstraction encapsulates the properties that consensus algorithms of chains implementing the interblockchain
communication protocol are required to satisfy. These properties are necessary for efficient and safe
state verification in the higher-level protocol abstractions. The algorithm utilised in IBC to verify the 
consensus transcript and state sub-components of another machine is referred to as a "validity predicate",
and pairing it with a state that the verifier assumes to be correct forms a "light client" (colloquially shortened to "client").

\vspace{3mm}

### Motivation

&nbsp;

In the IBC protocol, an actor, which may be an end user, an off-chain process, or a chain,
needs to be able to verify updates to the state of another chain
which the other chain's consensus algorithm has agreed upon, and reject any possible updates
which the other chain's consensus algorithm has not agreed upon. A light client is the algorithm
with which an actor can do so. The client abstraction formalises this model's interface and requirements,
so that the IBC protocol can easily integrate with new chains which are running new consensus algorithms
as long as associated light client algorithms fulfilling the listed requirements are provided.

Beyond the properties described in this specification, IBC does not impose any requirements on
the internal operation of chains and their consensus algorithms. A chain may consist of a
single process signing operations with a private key, a quorum of processes signing in unison,
many processes operating a Byzantine fault-tolerant consensus algorithm, or other configurations yet to be invented
— from the perspective of IBC, a chain is defined entirely by its light client validation and equivocation detection logic.
Clients will generally not include validation of the state transition logic in general
(as that would be equivalent to simply executing the other state machine), but may 
elect to validate parts of state transitions in particular cases, and can validate
the entire state transition if doing so is asymptotically efficient, perhaps through compression using a SNARK [@coda_protocol].

Clients could also act as thresholding views of other clients. In the case where
modules utilising the IBC protocol to interact with probabilistic-finality consensus algorithms
which might require different finality thresholds for different applications, one write-only
client could be created to track headers and many read-only clients with different finality
thresholds (confirmation depths after which state roots are considered final) could use that same state.

The client protocol is designed to support third-party introduction. Consider the general example:
Alice, a module on a machine, wants to introduce Bob, a second module on a second machine who Alice knows (and who knows Alice),
to Carol, a third module on a third machine, who Alice knows but Bob does not. Alice must utilise
an existing channel to Bob to communicate the canonically-serialisable validity predicate for 
Carol, with which Bob can then open a connection and channel so that Bob and Carol can talk directly.
If necessary, Alice may also communicate to Carol the validity predicate for Bob, prior to Bob's
connection attempt, so that Carol knows to accept the incoming request.

Client interfaces are constructed so that custom validation logic can be provided safely
to define a custom client at runtime, as long as the underlying state machine can provide an
appropriate gas metering mechanism to charge for compute and storage. On a host state machine
which supports WASM execution, for example, the validity predicate and equivocation predicate
could be provided as executable WASM functions when the client instance is created.

\vspace{3mm}

### Definitions

&nbsp;

A *validity predicate* is an opaque function defined by a client type to verify headers depending on the current consensus state. Using the validity predicate should be far more computationally efficient than replaying the full consensus algorithm and state machine for the given parent header and the list of network messages.

A *consensus state* is an opaque type representing the state of a validity predicate.
The light client validity predicate algorithm in combination with a particular consensus state must be able to verify state updates agreed upon by the associated consensus algorithm.
The consensus state must also be serialisable in a canonical fashion so that third parties, such as counterparty chains,
can check that a particular machine has stored a particular state. It must also be
introspectable by the state machine which it is for, such that the state machine can look up its
own consensus state at a past height and compare it to a stored consensus state in another chain's client.

A *commitment root* is an inexpensive way for downstream logic to verify whether key/value
pairs are present or absent in a state at a particular height. Often this will be instantiated as the root of a Merkle tree.

A *header* is an opaque data structure defined by a client type which provides information to update a consensus state.
Headers can be submitted to an associated client to update the stored consensus state. They likely contain a height, a proof,
a new commitment root, and possibly updates to the validity predicate.

A *misbehaviour predicate* is an opaque function defined by a client type, used to check if data constitutes a violation of the consensus protocol. This might be two signed headers with different state roots but the same height, a signed header containing invalid state transitions, or other evidence of malfeasance as defined by the consensus algorithm.

\vspace{3mm}

### Desired properties

&nbsp;

Light clients must provide a secure algorithm to verify other chains' canonical headers,
using the existing consensus state. The higher level abstractions will then be able to verify
sub-components of the state with the commitment roots stored in the consensus state, which are
guaranteed to have been committed by the other chain's consensus algorithm.

Validity predicates are expected to reflect the behaviour of the full nodes which are running the
corresponding consensus algorithm. Given a consensus state and a list of messages, if a full node
accepts a new header, then the light client must also accept it, and if a full node rejects it,
then the light client must also reject it.

Light clients are not replaying the whole message transcript, so it is possible under cases of
consensus misbehaviour that the light clients' behaviour differs from the full nodes'.
In this case, a misbehaviour proof which proves the divergence between the validity predicate
and the full node can be generated and submitted to the chain so that the chain can safely deactivate the
light client, invalidate past state roots, and await higher-level intervention.

The validity of the validity predicate is dependent on the security model of the
consensus algorithm. For example, the consensus algorithm could be BFT proof-of-authority
with a trusted operator set, or BFT proof-of-stake with a tokenholder set, each of which
have a defined threshold above which Byzantine behaviour may result in divergence.

Clients may have time-sensitive validity predicates, such that if no header is provided for a period of time
(e.g. an unbonding period of three weeks in a proof-of-stake system) it will no longer be possible to update the client.

\vspace{3mm}

### State verification

&nbsp;

Client types must define functions to authenticate internal state of the state machine which the client tracks.
Internal implementation details may differ (for example, a loopback client could simply read directly from the state and require no proofs).
Externally-facing clients will likely verify signature or vector commitment proofs.

\vspace{3mm}

### Example client instantiations

&nbsp;

#### Loopback

A loopback client of a local chain merely reads from the local state, to which it must have access. This is analogous to `localhost` or `127.0.0.1` in TCP/IP.

\vspace{3mm}

#### Simple signatures

A client of a solo machine with a known public key checks signatures on messages sent by that local machine.
Multi-signature or threshold signature schemes can also be used in such a fashion.

\vspace{3mm}

#### Proxy clients

Proxy clients verify another (proxy) chain's verification of the target chain, by including in the
proof first a proof of the client state on the proxy chain, and then a secondary proof of the sub-state of
the target chain with respect to the client state on the proxy chain. This allows the proxy client to
avoid storing and tracking the consensus state of the target chain itself, at the cost of adding
security assumptions of proxy chain correctness.

\vspace{3mm}

#### BFT consensus and verifiable state

The most common and most useful client type will be light clients for instances of BFT consensus algorithms such as Tendermint [@tendermint_consensus_without_mining], GRANDPA [@grandpa_consensus], or HotStuff [@hotstuff_consensus], with state machines utilising Merklized state trees such as an IAVL+ tree [@iavl_plus_tree] or a Merkle Patricia tree [@patricia_tree]. The client algorithm for such instances will utilise the BFT consensus algorithm's light client validity predicate and treat at minimum consensus equivocation (double-signing) as misbehaviour, along with other possible misbehaviour types specific to the proof-of-authority or proof-of-stake system involved.

\vspace{3mm}

### Client lifecycle

&nbsp;

#### Creation

Clients can be created permissionlessly by anyone at any time by specifying an identifier,
client type, and initial consensus state.

\vspace{3mm}

#### Update

Updating a client is done by submitting a new header. When a new header is verified
with the stored client state's validity predicate and consensus state, the client will
update its internal state accordingly, possibly finalising commitment roots and
updating the signature authority logic in the stored consensus state.

If a client can no longer be updated (if, for example, the unbonding period has passed),
it will no longer be possible to send any packets over connections and channels associated
with that client, or timeout any packets in-flight (since the height and timestamp on the
destination chain can no longer be verified). Manual intervention must take place to
reset the client state or migrate the connections and channels to another client. This
cannot safely be done automatically, but chains implementing IBC could elect
to allow governance mechanisms to perform these actions
(perhaps even per-client/connection/channel with a controlling multi-signature or contract).

\vspace{3mm}

#### Misbehaviour

If the client detects evidence of misbehaviour, the client can be take appropriate action, possibly invalidating
previously valid commitment roots and preventing future updates. What precisely constitutes misbehaviour will
depend on the consensus algorithm which the validity predicate is validating the output of.