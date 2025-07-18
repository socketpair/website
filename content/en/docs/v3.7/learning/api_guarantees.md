---
title: etcd API guarantees
weight: 2750
description: API guarantees made by etcd
---

etcd is a consistent and durable key value store.
The key value store is exposed through [gRPC Services].
etcd ensures the strongest consistency and durability guarantees for a distributed system.
This specification enumerates the API guarantees made by etcd.

### APIs to consider

* KV APIs
  * [Range](../api/#range)
  * [Put](../api/#put)
  * [Delete](../api/#delete-range)
  * [Transaction](../api/#transaction)
* Watch APIs
  * [Watch](../api/#watch-api)
* Lease APIs
  * [Grant](../api/#obtaining-leases)
  * [Revoke]
  * [Keep alive](../api/#keep-alives)

KV API allows for direct reading and manipulation of key value store.
Watch API allows subscribing to key value store changes.
Lease API allows assigning a time to live to a key.

Both KV and Watch APIs allow access to not only the latest versions of keys, but
also previous versions are accessible within a continuous history window, limited
by a compaction operation.

Calling KV API will take an immediate effect, while Watch API will return with some unbounded delay.
In correctly working etcd cluster you should expect to see watch events to appear with 10ms delay after them happening.
However, there is no limit and events in unhealthy clusters might never arrive.

## KV APIs

etcd ensures durability and strict serializability for all KV api calls.
Those are the strongest isolation guarantee of distributed transactional database systems.

### Durability

Any completed operations are durable. All accessible data is also durable data.
A read will never return data that has not been made durable.

### Strict serializability

KV Service operations are atomic and occur in a total order, consistent with
real-time order of those operations. Total order is implied through [revision].
Read more about [strict serializability].

For transactions without nested TXNs, the order of execution of operations
is guaranteed to be the same as in its list of operations, which means stable
GET responses within the transaction.
For transactions with nested TXNs, the order of execution is not specified.

Strict serializability implies other weaker guarantees that might be easier to understand:

#### Atomicity

All API requests are atomic; an operation either completes entirely or not at
all. For watch requests, all events generated by one operation will be in one
watch response. Watch never observes partial events for a single operation.

#### Linearizability

From the perspective of client, linearizability provides useful properties which
make reasoning easily. This is a clean description quoted from
[the original paper][linearizability]: `Linearizability provides the illusion
that each operation applied by concurrent processes takes effect instantaneously
at some point between its invocation and its response.`

For example, consider a client completing a write at time point 1 (*t1*). A
client issuing a read at *t2* (for *t2* > *t1*) should receive a value at least
as recent as the previous write, completed at *t1*. However, the read might
actually complete only by *t3*. Linearizability guarantees the read returns the
most current value. Without linearizability guarantee, the returned value,
current at *t2* when the read began, might be "stale" by *t3* because a
concurrent write might happen between *t2* and *t3*.

etcd ensures linearizability for all other operations by default.
Linearizability comes with a cost, however, because linearized requests must go
through the Raft consensus process. To obtain lower latencies and higher
throughput for read requests, clients can configure a request’s consistency mode
to `serializable`, which may access stale data with respect to quorum, but
removes the performance penalty of linearized accesses' reliance on live consensus.

## Watch APIs

Watches make guarantees about events:
* Ordered - events are ordered by revision.
  An event will never appear on a watch if it precedes an event in time that
  has already been posted. For transactions without nested TXNs, the order of
  generated events is guaranteed to be the same as in its list of operations.
  For transactions with nested TXNs, the order of generated events is not
  specified.
* Unique - an event will never appear on a watch twice.
* Reliable - a sequence of events will never drop any subsequence of events
  within the available history window. If there are events ordered in time as
  a < b < c, then if the watch receives events a and c, it is guaranteed to
  receive b as long b is in the available history window.
* Atomic - a list of events is guaranteed to encompass complete revisions.
  Updates in the same revision over multiple keys will not be split over several
  lists of events.
* Resumable - A broken watch can be resumed by establishing a new watch starting
  after the last revision received in a watch event before the break, so long as
  the revision is in the history window.
* Bookmarkable - Progress notification events guarantee that all events up to a
  revision have been already delivered.

etcd does not ensure linearizability for watch operations. Users are expected
to verify the revision of watch events to ensure correct ordering with other operations.

## Lease APIs

etcd provides [a lease mechanism][lease]. The primary use case of a lease is
implementing distributed coordination mechanisms like distributed locks. The
lease mechanism itself is simple: a lease can be created with the grant API,
attached to a key with the put API, revoked with the revoke API, and will be
expired by the wall clock time to live (TTL). However, users need to be aware
about [the important properties of the APIs and usage][why] for implementing
correct distributed coordination mechanisms.

## etcd specific definitions

### Operation completed

An etcd operation is considered complete when it is committed through consensus,
and therefore “executed” -- permanently stored -- by the etcd storage engine.
The client knows an operation is completed when it receives a response from the
etcd server. Note that the client may be uncertain about the status of an
operation if it times out, or there is a network disruption between the client
and the etcd member. etcd may also abort operations when there is a leader
election. etcd does not send `abort` responses to  clients’ outstanding requests
in this event.

### Revision

An etcd operation that modifies the key value store is assigned a single
increasing revision. A transaction operation might modify the key value store
multiple times, but only one revision is assigned. The revision attribute of a
key value pair that was modified by the operation has the same value as the
revision of the operation. The revision can be used as a logical clock for key
value store. A key value pair that has a larger revision is modified after a key
value pair with a smaller revision. Two key value pairs that have the same
revision are modified by an operation "concurrently".

[grpc Services]: ../api/#grpc-services
[lease]: https://web.stanford.edu/class/cs240/readings/leases.pdf
[linearizability]: https://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf
[serializable_isolation]: https://en.wikipedia.org/wiki/Isolation_(database_systems)#Serializable
[strict serializability]: http://jepsen.io/consistency/models/strict-serializable
[txn]: ../api/#transaction
[why]: ../why/#notes-on-the-usage-of-lock-and-lease
[revision]: #revision
