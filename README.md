# Fiber semantics

This document describes "Fiber semantics" a model for reasoning about a subset of "event driven architecture" where correctness, auditability and deletion policy are prioritized. The goal is to enable Event Carried State Transfer, ECST in a maintainable manner for complex domains.

## Audit log and data-products
In fiber semantics an audit log is optional and implemented same as the data-product, but typically with same or more strict constraints. This means they both use events, fibers and lines as described here, but are kept as separate storage items. 

## Assumptions
Key assumption in fiber semantics is that the producer can not know the needs of the consumer. This applies to functional and operational needs. Therefore, it is important to choose constraints carefully such that enough freedom is left to implementation, while being clear on semantics.

## Fiber
A fiber is an ordered series of events about a single and unique DomainId forming a singly linked list starting with the most resent event. The DomainId may represent a single thing in categories such as an entity, activity or a property of an entity. Different categories of DomainIds allows for different concepts:
- If the domainId identifies an entity we can refer to the fiber as the history of the entity with DomainId
- If the domainId identifies an activity like a transaction ID we can refer to the fiber as a saga
- If the domainId identifies a property of an entity we can refer to the fiber as a time-series. This use case is currently not central to the design, but might work well.

DomainIds are native to the domain and has a namespace for the DomainId. Multiple fibers within the same namespace can be stored and shared through a datastructure called a line. In fiber semantics a line can only support one namespace, but a namespace can be sharded by domainId across multiple lines, keeping the fibers intact. Between migrations fibers can only be read, created, updated, detached (soft delete) or rescued (undeleted). Mutating operations happen in an append-only manner.

The most fundamental building block in fiber semantics is the event. Event is a message containing a fact, something that has happened. Events are distinct from commands which may require a response of success or failure because they command something that should happen in the future. Note: In Fiber semantics query is a command which only does read type operations, query can also fail and the response is not optional. Events have a header with the following core concepts:

- Timestamp is the time in nanoseconds since the epoch. Timestamp is set when the event is successfully appended to the fiber and is considered immutable.
- DomainId is the unique identifier of an entity, activity or property in the domain. DomainId is immutable. 
- Detached marks the fiber as soft deleted. Migrations can keep, purge or lock a detached fiber of a domainId.
- Precursor forms fibers by chaining events with the same DomainId in a singly linked list.
- DomainEvent is the domain specific payload of the event.

Immutable fields in the event header still may get purged or pruned together with the complete event during migrations. The other header fields will get updated during migrations. The DomainEvent payload is only mutated during migrations with schema upgrades.

## Line datastructure

A line is an array created from one or more interleaved fibers. Events have header fields and a payload called DomainEvent. Events also has an implicit index which is their position on the array. Between migrations the array is locked in an append-only-log mode of operation. This guarantees full preservation of history between migrations. The singly linked list of each DomainId is intended to help identifying a fiber in the line without a full scan for the DomainId. This is intended to help any type of read operation for any DomainId to be near optimal in terms of speed and resource consumption, assuming a power law distribution of fiber lengths.

![LineDatastructure.png](Images%2FLineDatastructure.png)

## Auditable line migrations

"Domains can be messy"

The purpose of migrations is to simultaneously enable auditability, schema upgrades and deletion policies. Line migrations do however not take responsibility for how the audit log is maintained, but rather provides for an audit log to have a lossless view of any line version by writing the new version of the line as a separate datastructure with all kept events from the previous version. A consumer of the latest version of the line can choose to forget old schemas and deleted data by reading the latest version and doing a compare or a full reload of the state carried by the line. An audit log would typically only contain events in their original version, causing it to contain all versions of events across the full audit log. If audit log is enabled it should be appended simultaneously with the current line version, and hence old line versions could be deleted once the migration is complete since the audit log retains all events in their original schema. Note that schema migrations will cause the line to diverge from the audit log. Be careful about how domain information is changed during migrations with schema upgrades. Doing schema upgrades may be subject to law and regulations in the domain. A migration should never touch the audit log. Between migrations a system with audit log enabled should only do appends.

## Migration types
For each detached fiber the following logic is possible during migrations:
- Migrate(Keep): Fiber is unchanged. The state is retained as soft deleted. 
- Migrate(LockAndPrune) Fiber is pruned back to only having its last event. Key can not be created, but rescue is possible, however the history will be lost. Line reindexing needed.
- Migrate(Purge)

For each fiber that is not detached the default migration is "Keep". TODO: PruneAttached policy which only keeps the last n, n>0 events of each fiber. This could be implemented based on timestamp, history depth or line index.

Migrations that exclusively does keep-migrations do not require reindexing of the line. These may still do schema upgrades. All other migrations will require line reindexing, which means moving events up the line as free indexes are created and updating all precursors to the new indexes of the previous events in the fibers.

## Statemachine

![Statemachine.png](Images%2FStatemachine.png)

## Example implementation: Pardosa

Pardosa is an in-memory key-value storage layer which implements fiber semantics. Pardosa maintains pointers to the most resent event in every fiber in a hashmap. Also, the concept of an anchor is there to improve some worst case read operations, but the current implementation with anchoring at the start must be replaced with anchoring at (Fiber.len modulo n) to be effective. Doubly linked lists could also be maintained, but the gain would be questionable with better anchoring. Pardosa is TBD.
![PardosaExample.png](Images%2FPardosaExample.png)

## Foundational concepts
Fiber semantics is relying on language and model concepts form Domain Driven Design and Actor Model, not to be confused with any actor model framework. It´s a work in progress to link these models.