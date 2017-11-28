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

#### Serving Reads at a Timestamp

Each replica keeps track of the safe time (the last time at which everything was up to date).
This ends up being the minimum of both the paxos safe time and the transaction manager's safe time.
Transactions in the middle of a two-phase commit are given a safe time of infinity.

#### Assigning Timestamps to Read-Only Transactions

There are two phases:

1. Assign a timestamp to the read.
2. Then, execute the reads as snapshot reads.

These can occur at any replica that is up to date.

### Details

#### Read-Write Transactions

Writes that occur in a transaction are buffered at the client until commit.
Therefore, reads in a transaction do not see the effects of the transaction's writes.

Reads in read-write transactions are wound-wait (references to data of future transactions are rolled back) to avoid deadlocks.

#### Read-Only Transactions

Assigning a timestamp requires a negotiation phase between all the Paxos groups that are involved in the reads.
Therefore, scope must be provided for each one of these.

For single-site reads, Spanner does better than `TT.now().latest`, referencing the last transacted commit write time.
This satisfies external consistency.

#### Schema Change Transactions

TrueTime allows Spanner to perform atomic schema changes.
For a schema change of t, all reads and writes will succeed if they precede t, but will be blocked otherwise.

#### Refinements

There are few very minor adjustments that have to be made for various manners of timestamping.

## Evaluation

### Benchmarks

Two-phase commits get a bit too slow around 100 participants.

### Availability

Throughput rises on leader election.

### TrueTime

Epsilon is actually a fairly small value.
The 99.9 percentile peaks around 10ms, and that includes tail-latency.

## F1

Failover has been invisible to them.
They chose Spanner because MySQL wasn't cutting it, and they needed consistency (so BigTable wasn't an option).

## Related Work

DynamoDB uses a key-value interface, but it only replicates within a region.
Megastore allows for replication outside a region but does not achieve high performance.

## Future Work

Spanner is used for Google's advertising.

They are implementing automatic maintenance of secondary indices, automatic load-based sharding, and a few other things.

## Conclusions

TrueTime is the linch pin.
