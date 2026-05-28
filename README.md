# Fiber semantics
This document describes "Fiber semantics" a model for reasoning about a subset of "event driven architecture" where correctness, auditability and deletion policy are prioritized. The goal is to enable Event Carried State Transfer, ECST in a maintainable manner for complex domains.

## Audit log and data-products
In fiber semantics an audit log is optional and implemented same as the data-product, but typically with same or more strict constraints during migrations. This means they both use events, fibers and Draglines as described here, but are kept as separate storage items. Audit logs are typically derived from the dataproduct by a separate component or service.

## Assumptions
Key assumption in fiber semantics is that the producer can not know the needs of the consumer. This applies to functional and operational needs. Therefore, it is important to choose constraints carefully such that enough freedom is left to implementation, while being clear on semantics.

## Fiber
A **fiber** is an ordered replay scope within a log. It is the sequence of events the runtime treats as one local continuity for append, checked replay, precursor validation, and application-level reconstruction.

A fiber is not the same thing as a domain identifier. Its identity is log-local runtime identity: a handle meaningful inside the Dragline or log that allocated it.

Applications may use fibers to model entity histories, sagas, causal workflows, time-series streams, or other replayable continuities. The mapping from domain concepts to fibers is application policy.

Domain identity such as `OrderId`, `AccountId`, `DeviceId`, `SagaId`, or `TraceId` belongs in the event payload or application schema.

Multiple fibers can be stored and shared through a datastructure called a `Dragline`. Between migrations fibers can only be created, read, updated, detached (soft delete) or rescued (undeleted). Dragline and fibers are append-only between migrations.

The most fundamental building block in fiber semantics is the event. Event is a message containing a fact, something that has happened. Events are distinct from commands which may require a response of success or failure because they command something that should happen in the future. Note: In Fiber semantics query is a command which only does read type operations, query can also fail and the response is not optional. Events have a header with the following core concepts:

- ``FiberId`` is the unique identifier of a series of connected events conserning an entity, activity or property in the domain. Application must choose mapping.
- Detached marks the fiber as soft deleted. Migrations can keep, purge or lock a detached fiber of a `FiberId`.
- Precursor forms fibers by chaining events with the same `FiberId` in a singly linked list.
- DomainEvent is the domain specific payload of the event.

Immutable fields in the event header still may get purged or pruned together with the complete event during migrations. The other header fields will get updated during migrations. The DomainEvent payload is only mutated during migrations with schema upgrades.

## Application causality keys
An application may choose a **causality key**: a domain value used to decide which fiber or fibers are relevant for a business concept.

Examples:
- `OrderId` for an order history,
- `PaymentId` for a payment saga,
- `AccountId` for account-related activity,
- `DeviceId` for device telemetry,
- `TraceId` or `CorrelationId` for workflow causality.

The application owns the meaning of this key. Pardosa does not treat the key as substrate identity and does not require every application to choose the same kind of key.

A causality key may map to:
- one current fiber,
- many historical fibers,
- a set of related saga fibers,
- a projection state,
- or no fiber yet.

## Fiber indexes
Pardosa may provide an optional typed mapping from application-owned causality keys to log-local fiber handles.
The application owns the key semantics. Pardosa owns index consistency, rebuildability, and lookup performance.

Conceptually:
```rust
CausalityKey<K> -> FiberHandle(s)
```

Examples:
```rust
OrderId -> current order fiber
AccountId -> many account activity fibers
SagaId -> saga fiber
DeviceId -> current telemetry fiber
```

The index is not an independent source of truth. It is derived from, or checked against, the append-only log. If an index becomes inconsistent, the log wins and the index must be rebuilt or rejected.

## Dragline datastructure
A Dragline is an array created from one or more interleaved fibers. Events have header fields and a payload called DomainEvent. Events also has an implicit index which is their position on the array. Between migrations the array is locked in an append-only-log mode of operation. This guarantees full preservation of history between migrations. The singly linked list of each `FiberId` is intended to help identifying a fiber in the Dragline without a full scan for the `FiberId`. This is intended to help any type of read operation for any `FiberId` to be near optimal in terms of speed and resource consumption, assuming a power law distribution of fiber lengths.

![LineDatastructure.png](Images%2FLineDatastructure.png)

## Auditable Dragline migrations
"Domains can be messy"

The purpose of migrations is to simultaneously enable auditability, schema upgrades and deletion policies. Dragline migrations do however not take responsibility for how the audit log is maintained, but rather provides for an audit log to have a lossless view of any Dragline version by writing the new version of the Dragline as a separate datastructure with all kept events from the previous version. A consumer of the latest version of the Dragline can choose to forget old schemas and deleted data by reading the latest version and doing a compare or a full reload of the state carried by the Dragline. An audit log would typically only contain events in their original version, causing it to contain all versions of events across the full audit log. If audit log is enabled it should be appended simultaneously with the current Dragline version, and hence old Dragline versions could be deleted once the migration is complete since the audit log retains all events in their original schema. Note that schema migrations will cause the Dragline to diverge from the audit log. Be careful about how domain information is changed during migrations with schema upgrades. Doing schema upgrades may be subject to law and regulations in the domain. A migration should never touch the audit log. Between migrations a system with audit log enabled should only do appends.

## Migration types
For each detached fiber the following logic is possible during migrations:
- Migrate(Keep): Fiber is unchanged. The state is retained as soft deleted. 
- Migrate(LockAndPrune) Fiber is pruned back to only having its last event. Key can not be created, but rescue is possible, however the history will be lost. Dragline reindexing needed.
- Migrate(Purge)

For each fiber that is not detached the default migration is "Keep". TODO: PruneAttached policy which only keeps the last n, n>0 events of each fiber. This could be implemented based on timestamp, history depth or Dragline index.

Migrations that exclusively does keep-migrations do not require reindexing of the Dragline. These may still do schema upgrades. All other migrations will require Dragline reindexing, which means moving events up the Dragline as free indexes are created and updating all precursors to the new indexes of the previous events in the fibers.

## Statemachine
NOTE: state `PURGED` can be eliminated by consollidating into state `UNDEFINED`. Transition `Rescue()` from state `LOCKED` is no longer needed as we have properly decoupled DomainID from `FiberId` in new design.

![Statemachine.png](Images%2FStatemachine.png)

## Example implementation: Pardosa

Pardosa is an in-memory key-value storage layer which implements fiber semantics. Pardosa maintains pointers to the most resent event in every fiber in a hashmap. Also, the concept of an anchor is there to improve some worst case read operations, but the current implementation with anchoring at the start must be replaced with anchoring at (Fiber.len modulo n) to be effective. Doubly linked lists could also be maintained, but the gain would be questionable with better anchoring. Pardosa is TBD.
![PardosaExample.png](Images%2FPardosaExample.png)

## Foundational concepts
Fiber semantics is relying on language and model concepts form Domain Driven Design and Actor Model, not to be confused with any actor model framework. It´s a work in progress to link these models.
