# Notes on Spanner: Google's Globally Distributed Database

## Foreword

This [paper] was published for the 2012 Operating System Design and Implementation.

[paper]: ../papers/spanner-osdi-2012.pdf

## Introduction

Spanner's primary benefits are that it is:

* designed to be globally replicated
* uses a SQL-like language for querying
* able to support general purpose transactions
* a key-value store with multi-versioning, where old versions can be garbage collected
* able to allow for dynamically controlled replication configurations
* able to control data locality
  * for read latency
  * for write latency
* able to control how many replicas are maintained
* able to re-balance data across regions
* able to provide externally-consistent reads and writes
* able to provide globally-consistent reads across the database at a timestamp

The consistent reads allow for consistent MapReduce executions and atomic schema update.

Probably the biggest gain this has over other options (e.g. Cassandra) is it's ACID-like transaction consistency.

## Implementation

Each "universe" has a universe master, which allows for global debugging, a placement driver, which handles automated movements of data across zones.
Each zone has a zone master, which assigns data to span servers.
There are between one hundred and several thousand span servers per zone.
Each zone also has a location proxy which allows for clients to locate the data assigned to serve their data.

### Spans Server Software

Each span server has between one hundred and one thousand instances of tablets.
Each tablet implements mappings of:

```
(key:string, timestamp:int64) => string
```

The tablet's state is stored in a set of B-tree-like files and a write-ahead log.
Colossus (successor to Google File System) is used.

Paxos state machines are used to allow for replication.
There are replica leaders and slaves.

### Directories

Directories (or buckets) are sets of contiguous keys that share a given prefix.

Movedir is the background task used to move directories between Paxos groups.
These moves can occur while client operations are outstanding.

### Data Model

Each row must have a name, which leads to the requirement that each table have a primary key.

For relational kinds of data, the `INTERLEAVE IN` declaration is critical.
It not only allows for lookups based on relation, but it also specifies that the data locality should be closer, helping make distribution more efficient.

## True Time

True time is used in Spanner.
The API of true time allows for `TT.now()`, which returns a `TTInterval`.
This function ensures that `earliest` and `latest` values of the interval precede and follow the times of invoked events.

Both GPS and atomic clocks are used to establish the time.
Using both allows for a kind of hedging, as the allowances of both are unrelated to the other approach's allowances.

Each daemon polls a variety of masters to minimize risk.

Epsilon is used to define the instantaneous error bound, where it is half the length of the `TTInterval`.

## Concurrency Control

True Time is used in to achieve externally-consistent transaction, lock-free read-only transactions, and non-blocking reads in the past.

### Timestamp Management

#### Paxos Leader Leases

Time leases (10 seconds by default) are used to make leadership long-lived.
Once a quorum of votes is received as to who should be the leader, the leader assumes the position.

The lease interval starts when the quorum is discovered and ends when it no longer has the majority of votes.

For each Paxos group, the leader's lease interval is disjointed from every other leader's.

#### Assigning Timestamps to Read-Write Transactions

Transactional reads and writes use two-phase locking.
This introduces some complexity in regards to the timestamps.

A leader must only assign timestamps within the interval of its lease.

Also, if transaction T2 occurs after T1, then the timestamp of the start for T2 must be greater than the commit of T1.
