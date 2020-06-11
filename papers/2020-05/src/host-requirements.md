### Module system

&nbsp;

The host state machine must support a module system, whereby self-contained, potentially mutually distrusted packages of code can safely execute on the same ledger, control how and when they allow other modules to communicate with them, and be identified and manipulated by a "master module" or execution environment.

\vspace{3mm}

### Key/value Store

&nbsp;

The host state machine must provide a key/value store interface allowing values to be read, written, and deleted.

These functions must be permissioned to the IBC handler module only, so only the IBC handler module can write or delete a certain subset of paths.
This will likely be implemented as a sub-store (prefixed key-space) of a larger key/value store used by the entire state machine.

Host state machines must provide an instance of this interface which is "provable", such that the light client algorithm for the host chain
can verify presence or absence of particular key-value pairs which have been written to it.

This interface does not necessitate any particular storage backend or backend data layout. State machines may elect to use a storage backend configured in accordance with their needs, as long as the store on top fulfils the specified interface and provides commitment proofs.

\vspace{3mm}

### Consensus state introspection

&nbsp;

Host state machines must provide the ability to introspect their current height, current
consensus state (as utilised by the host machine's light client algorithm), and a bounded
number of recent consensus states (e.g. past headers). These are used to prevent man-in-the-middle
attacks during handshakes to set up connections with other chains — each chain checks that the other
chain is in fact authenticating data using its consensus state.

\vspace{3mm}

### Timestamp access

&nbsp;

In order to support timestamp-based timeouts, host machines must provide a current Unix-style timestamp.
Timeouts in subsequent headers must be non-decreasing.

\vspace{3mm}

### Port system

&nbsp;

Host state machines must implement a port system, where the IBC handler can allow different modules in the host state machine to bind to uniquely named ports. Ports are identified by an identifier, and must be permissioned so that:

- Once a module has bound to a port, no other modules can use that port until the module releases it
- A single module can bind to multiple ports
- Ports are allocated first-come first-serve
- "Reserved" ports for known modules can be bound when the state machine is first started

This permissioning can be implemented with unique references (object capabilities [@object_capabilities]) for each port, with source-based authentication(a la `msg.sender` in Ethereum contracts), or with some other method of access control, in any case enforced by the host state machine.

\vspace{3mm}

### Exception/rollback system

&nbsp;

Host state machines must support an exception or rollback system, whereby a transaction can abort execution and revert any previously made state changes (including state changes in other modules happening within the same transaction), excluding gas consumed and fee payments as appropriate.

\vspace{3mm}

### Data availability

&nbsp;

For deliver-or-timeout safety, host state machines must have eventual data availability, such that any key/value pairs in state can be eventually retrieved by relayers. For exactly-once safety, data availability is not required.

For liveness of packet relay, host state machines must have bounded transactional liveness, such that incoming transactions are confirmed within a block height or timestamp bound (in particular, less than the timeouts assigned to the packets).

IBC packet data, and other data which is not directly stored in the Merklized state but is relied upon by relayers, must be available to and efficiently computable by relayer processes.