# Understanding RocksDB

RocksDB is a key-value store library that operates on data larger than
memory.  It supports multiple "column families" or namespaces for
key-value pairs, which in other system would be called tables or
databases.

RocksDB supports transactions, which may span multiple column
families.  Multiple transactions may be open at once (e.g. from
different threads).  RocksDB supports both optimistic and pessimistic
locking.

RocksDB supports snapshots, which do not persist on disk.  An
application can read a snapshot the same way it would the live
database.  Snapshots are read-only.

RocksDB keys should be limited to about 8 MB and values to about 3 GB.
RocksDB column families perform best if there are no more than a few
hundred.

## Coordinated commit

For fault tolerance, we need to periodically take a consistent state
snapshot across all of the workers in a computation.  The snapshot
needs to represent the same step at every worker, because there is no
way to advance the state of workers individually.

Per-worker transactions are almost sufficient.  To use them, each
worker keeps a transaction open from one snapshot to the next.  When
it is time for the next snapshot, the coordinator tells all of the
workers to commit their transactions.  If all of them commit (or all
of them abort), then there is no problem.  If some of them commit and
some of them abort, then the ones that committed need to roll back to
their previous states.

The ability to roll back a committed transactions isn't a common
feature.  RocksDB doesn't have it.

Failure during commit should be a rare problem.  This may mean that
it's OK if recovery is relatively expensive.

RocksDB has many features that seem like they could be components of
solutions:

* "Snapshots", but these are in-memory only and do not persist.

* "Checkpoints", which are on-disk copies of a database at a
  particular point in time.  Making a checkpoint does not require
  copying all of the files because many of them can be hard-linked.
  It is still relatively expensive and requires manual management of
  the file system.

* Read-only and secondary instances, which allow a database to be
  opened in a second process.

* Two-phase commit.  There is barely any information on how this
  works.  What is there talks about it as a way to speed up commit on
  a particular database.  It doesn't make it clear that it's meant to
  be used as a building block for distributed systems.

* Keep a separate undo log or redo log (e.g. in a separate column
  family in RocksDB, or in Kafka).
  
* Maintain multiple versions of each column family within the column
  family itself.
  
The latter two approaches seem most attractive.  They are described in
more detail below.

### Maintaining an undo log in RocksDB

We introduce database version numbers.  At any given time, a given
database has records for two versions: `V-1`, which is the version
that the undo log rolls back to, and `V`, the current version.

At distributed DBSP startup, the coordinator asks each worker's `V`.
If all of them report the same value, then operation begins normally.
If some of them report `V = x` and others `V = x-1`, then the
coordinator tells the ones reporting the larger value to roll back by
applying their undo logs.  (If their values of `V` differ by more than
one, that is a fatal error.)  All of the workers then clear their undo
logs; optionally, they may commit and open a new transaction.  Thus,
now all the workers agree on a particular `V` and have an empty undo
log.  If a crash occurs during this process, then it repeats on the
next restart.

While running, each worker keeps a transaction open.  While running,
for each operation that it applies to the current version of the
database, it writes a compensating entry to the undo log.  Each worker
may commit its local transaction and open a new one as necessary.  To
commit across all the workers, each worker commits its local
transaction.  If all the workers commit successfully, then operations
continues normally, with each worker opening a new transaction and,
within that transaction, clearing the undo log.  Otherwise, the
coordinator tells them all to restart.

### Multiversioning within a column family

We introduce database version numbers.  At any given time, version `V`
is the version that will be restored on failure and `V+1` is the next
version that will be committed.  The database also likely contains
numerous records with versions `0..V`.

In each key-value pair in the database, the value is a map from
`Version` to `Option<Data>`.  A given `(version,Some(data))` pair
means that the key had the given `data` starting at `version`, until
the next higher version appearing in the map.  A `(version,None)` pair
indicates a deletion.

To solve our problem, we maintain multiversioned records.  We
introduce a Version column family that reports the range of versions
represented in the database.  At distributed DBSP startup, the
coordinator takes the intersection of these ranges across all of the
workers.  If the intersection is empty, that is a fatal error.
Otherwise, let `V` be the latest version within the intersection.

The coordinator reports `V` to all of the workers.  If a particular
worker has records with a version greater than `V`, then it iterates
through all such records[^1], removes data for later versions, updates
the Version column family to indicate that no version greater than `V`
exists in the database, and commits the changes.  Thus, at this point
every worker has no records with version greater than `V`.  If a crash
occurs during this process, then it repeats on the next restart.

While running, each worker keeps a transaction open in its local
RocksDB.  The transaction updates the Version column family to
indicate that the database includes data for versions `V..=V+1`, and
database operations occur on version `V+1`.  Any previous data in a
record must be retained.  However, if modifying a record would result
in the record containing more than two versions, then data for the
oldest verion may be discarded.  This is because recovery will never
require moving back more than one version.

To commit across all the workers, each worker commits its local
transaction.  Then each one opens a new transaction and increments
`V`.  If all of the workers commit successfully, then operation
continues normally.  Otherwise, the coordinator tells them all to
restart.

[^1]: RocksDB has transaction log iterators that can make this
    efficient.  If this doesn't work, then it would be necessary to
    iterate across all records.  If this is going to be a common
    failure mode, then it might make sense to use the undo-log
    approach from the previous section, instead.
