# Fiber semantics

This document describes "Fiber semantics" a model for reasoning about "event driven architecture".

## Fiber
A fiber is an ordered series of events about a domainId. The domainId may represent a single thing in categories such as an entity, activity or a property of an entity. Different categories of domainId allows for different concepts:
- If the domainId identifies an entity we can refer to the fiber as a history
- If the domainId identifies an activity like a transaction we can refer to the fiber as a saga
- If the domainId identifies a property of an entity we can refer to the fiber as a timeseries. This use case is currently not sentral to the design, but might work well.

DomainIds are native to the domain and has a kind. Multiple fibers with the same kind can be stored and shared through a datastructure called a line. Between migrations fibers can only be read, created, updated, detached (soft delete) and rescued (undeleted).

Important concepts in fibers:
- Event is a message containing a fact, something that has happened. Events are distinct from commands which may require a response of success or failure. Note: query is a command which only reads.
- Timestamp is the time of the event in nanoseconds since the epoch
- DomainId is the unique identifier of an entity, activity or property in the domain. Migrations can keep, purge or lock a detached the fiber of a domainId.
- Detached marks the fiber as soft deleted
- Precursor forms fibers by chaining events eith the same domainId in a singly linked list.
- DomainEvent is the domain spesific payload of the event.

## Line datastructure

A line is an array created from one or more interleaved fibers which are singly linked lists of events with the same key. Events have header fields and a payload called domainevent. Events also has an implicit index which is their position on the array. Between migrations the array is locked in an append-only-log mode of operation. This guarantees full preservation of history between migrations. The singly linked list of each key is intended to help identifying a fiber in the line without a full scan for the key. This is intended to help any type of read operation for any key to be near optimal in terms of speed and resource consumption, assuming a power law distribution of keys.

## Auditable line migrations

"Domains can be messy"

The purpose of migrations is to enable auditability, schema upgrades and deletion policies. Line migrations do however not take responsibility for how the auditlog is maintained, but rather provides for an auditlog to have a lossless view of any line version by simply writing the new version of the line as a separate datastructure. Anyone consuming only the latest version of the line will be able to forget old schemas and deleted data by reading the latest version and doing a compare or full reload of its state.

## Migration types
For each detached fiber the following logic is possible during migrations:
- Migrate(Keep): Fiber is unchanged. The state is retained as soft deleted. 
- Migrate(LockAndPrune) Fiber is pruned back to only having its last event. Key can not be created, but rescue is possible, however the history will be lost. Line reindexing needed.
- Migrate(Purge)

For each fiber that is not detached the default migration is "Keep". TODO: KeepAndPrune poliy which only keeps the last n, n>0 events can be implemented based on timestamp, history depth or line index.

Migrations that exclusively does keep-migrations does not require reindexing of the line. These may still do schema upgrades. All other migrations will require line reindexing, which means updating the precursors to the new indexes of the previous events in the fibers.
