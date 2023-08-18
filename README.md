# Distributed DBSP design

This document proposes a design for supporting the following features
in DBSP:

- [Distribution of computations](#distribution-of-computation) across multiple hosts.
- [Fault tolerance](#fault-tolerance) and fault detection.
- [Scale out and in](#scale-in-and-out) of a running computation.
- [Rolling upgrade](#rolling-upgrade) of DBSP itself.

The following features are out of scope:

- Upgrade of computation, or other changes to a computation or the
  dataflow graph that implements it in DBSP.
- Multiple connected pipelines.

We aim to satisfy the following base requirements:

* Allow any number of producers to submit input data to any number of
  input handles.  Producers must be able to submit data in
  variable-sized batches.  Producers must be able to determine whether
  their input was received and recorded, to allow for exactly-once
  semantics.

* Allow any number of consumers to obtain output data from any number
  of output handles.  Consumers must be able to determine whether they
  have already received a given piece of output, to allow for
  exactly-once semantics.  Consumers must able to determine the input
  data offsets that correspond to given output.

# Distribution of computation

DBSP implements computation in the form of a circuit.  A circuit runs
in a single thread, passing data from "input handles" through
"operators" to "output handles":

```
[Figure 1]

         ┌────────────┐   ┌────────────┐   ┌────────────┐
input───>│ operator 1 │──>│ operator 2 │──>│ operator 3 │───>output
         └────────────┘   └────────────┘   └────────────┘
```

To run a computation across multiple worker threads, DBSP instantiates
multiple copies of the circuit, all identical.  Each copy performs the
same computation in parallel across a disjoint part of the data.  In a
multi-worker computation, DBSP input handles automatically spread the
input evenly across the workers, and DBSP output handles allow for
merging the output from multiple workers for the most common kinds of
data.

For multi-worker computation, operators require individual care.  Some
operators, such as filter and map, work unchanged multi-worker.
Others require data to be distributed appropriately across workers;
for example, equi-join requires data with equal keys to be processed
in the same worker.  These operators internally use a special
"exchange" operator to "re-shard" data across workers, using a hash
function to obtain an equal distribution of keys.  The diagram below
shows data being re-sharded before operators 2 and 3 execute:

```
[Figure 2]

         ┌────────────┐   ┌────────────┐   ┌────────────┐
      ┌─>│ operator 1 │──>│ operator 2 │──>│ operator 3 │───>output
      │  └────────────┘\ /└────────────┘\ /└────────────┘
input─┤                 X                X
      │  ┌────────────┐/ \┌────────────┐/ \┌────────────┐
      └─>│ operator 1 │──>│ operator 2 │──>│ operator 3 │───>output
         └────────────┘   └────────────┘   └────────────┘
```

To extend multi-worker computation to multi-host computation, we
instantiate the circuit multiple times on different hosts.  This
simply requires extending the "exchange" operator to work across hosts
using RPC:

```
[Figure 3]

         ╔══host 1═══════════════════════════════════╗
         ║   ┌────────────┐           ┌────────────┐ ║
      ┌─────>│ operator 1 │──┐     ┌─>│ operator 2 │───>output
      │  ║   └────────────┘  │     │  └────────────┘ ║
      │  ║                   ├─┐ ┌─┤                 ║
      │  ║   ┌────────────┐  │ │ │ │  ┌────────────┐ ║
      ├─────>│ operator 1 │──┘ │ │ └─>│ operator 2 │───>output
      │  ║   └────────────┘    │ │    └────────────┘ ║
      │  ╚═════════════════════│ │═══════════════════╝
input─┤                        ├─┤
      │  ╔══host 2═════════════│ │═══════════════════╗
      │  ║   ┌────────────┐    │ │    ┌────────────┐ ║
      ├─────>│ operator 1 │──┐ │ │ ┌─>│ operator 2 │───>output
      │  ║   └────────────┘  │ │ │ │  └────────────┘ ║
      │  ║                   ├─┘ └─┤                 ║
      │  ║   ┌────────────┐  │     │  ┌────────────┐ ║
      └─────>│ operator 1 │──┘     └─>│ operator 2 │───>output
         ║   └────────────┘           └────────────┘ ║
         ╚═══════════════════════════════════════════╝
```

# Fault tolerance

For fault tolerance, each worker maintains a database with the
circuit's state.  Occasionally, a coordinator tells all of the workers
to commit their databases way.  To recover from a fault, all of the
workers load the most recently committed state snapshot and replay the
input starting from that point, discarding any duplicate output that
this produces.

Input must be stored durably at least the latest snapshot has gone
past it.

## Database

Each worker has a database with a key-value interface.  The database
must support up to two versions of data.  Let `Version` be a `u64`
type representing a database version serial number:

  * A new database has only one version, numbered 0, which is empty.

  * Opening a database at any given version causes the other version,
    if any, to be discarded.

  * Committing a database that was opened with `open(version)` creates
    a new version `version + 1`, either of which may be opened with
    `open`.
    
  * A database closed by a crash or in any way other than `commit`
    retains the version that was passed to `open()`.  If a crash
    happens during `commit` then it's OK for both versions or just the
    vesion passed to `open()` to be available on recovery.

* `fn versions() -> Range<Version>`

  Reports the versions that the database can open.  The returned
  `Range` has length 1 or 2.
  
* `fn open(version: Version) -> Result<Database>`

  Opens the database at the specified `version`.  The database
  discards versions other than `version`.
  
* `fn commit(db: Database) -> Result<()>`

  Commits the current version of `db` and closes the database.
  
* `fn get(db: &mut Database, table, key) -> Result<value>`  
  `fn put(db: &mut Database, table, key, value) -> Result<()>`

  Key-value pair read and write.
  
* `fn get_step(db: &Database) -> Result<Step>`  
  `fn set_step(db: &mut Database, step: Step) -> Result<()>`

  Helpers for reading and writing the step number for the next
  step to be applied.

## Input

Input and output are divided into numbered steps.  Let `Step` be a
`u64` type representing an input or output step.

Each worker needs to obtain input for each step.  Input must be logged
durably.  The assignment of input to a particular step must also be
durable; for example, a crash must not cause additional data (that
arrived post-crash, for example) to become part of a step.

Input only need be durable until its step has become committed to the
earliest version in its local database.

Consider a worker to have input for steps `first..last`.  This might
be an empty range if no new input has arrived since the earliest
version of the local database.

* `fn read(step: Step) -> Result<Vec<ZSet>>`

  Reads input for step number `step`, which must be in `first..=last`,
  that is, either a step currently in the log or just after.  In the
  latter case, the function blocks until input arrives.

  Returns a Z-set per input handle.  If a single step batches input
  from multiple event arrivals for a given input handle, they are
  added together into a Z-set.
  
## Output

Each worker needs to produce output for each step.  DBSP can reproduce
output for steps since the earliest version in any of the workers'
databases.

The need for output durability depends on the requirements of the
systems that consume DBSP's output.  Those systems can also apply
their own expiration or deletion policies to output.

Consider a worker to have output for steps `first..last`, which might
be an empty range.

* `fn write(step: Step, output: Vec<ZSet> -> Result<()>`

  Writes `output` as the output for step number `step`:
  
    * If `step` is in the range `0..first`, then `output` has already
      been produced before and discarded or deleted.  Discard it.
      
    * If `step` is in the range `first..last`, then `output` has
      already been produced before.  Now it has been produced
      redundantly during recovery.  Optionally, check that the new
      output matches the old output and warn if it differs.  Discard
      `output`.

    * If `step == last`, then append it to the output stream and
      increment `last`.
    
    * `step > last` is an error.
    
## Worker

A worker has the following interface:

* `fn versions() -> Range<Version>`

  Delegates to the local database.
  
* `fn start(version: Version)`

  Opens the local database with `open(version)`.  Sets a local
  variable `step` using `get_step()` from the database.  Set `state`
  to `Run`.
  
  Repeat the following loop until the coordinator calls `stop()`:
  
  - Get input by calling `read(step)`.
  
  - Run the circuit for one step, obtaining `output`.  The DBSP
    circuit coordinates across all of the workers on all of the hosts,
    exchanging data as necessary.

  - Write output by calling `write(step, output)`.
  
  - Increment `step`.
  
  Store `step` into the database using `set_step()`.  Commit the
  database with `commit()`.  Set `state` to `Stopped`.
  
* `fn stop()`

  Stops the loop in `start()`.
  
## Coordinator

In a loop:

- Call `versions()` on each worker to get its database version range,
  intersects the results, and takes the maximum value `v`[^1].
  
  If this is not the first iteration of the loop, `v` should be one
  more than in the previous iteration.
  
  [^1]: If the intersection is the empty set, that's a fatal error.

- Call `start(v)` on each worker.

- Wait for an appropriate interval between checkpoints.

- Call `stop()` on each worker.

## Synchronization details

<details><summary>Synchronization details</summary>

We need some synchronization to allow all the workers to pause at the
same step without blocking.  Here's one possible approach.

Give each worker a cancellation token `token`, something like
[`tokio_util::sync::CancellationToken`][1], plus a `state`:

```
enum State {
    Run,
    Pause,
    Paused(Step),
    Stop(Step),
    Stopped,
}
```

At the top of the loop in `start()`, instead of just getting input,
consult `state`:

* If `state` is `Run`, which is the common case, get input by
  calling `read(step, &token)`.

* If `state` is `Pause`, set it to `Paused(step)` and block until it
  becomes `Stop(_)`.

* If `state` is `Stop(x)`:

  - If `step < x`, get input by calling `read(step, &token)`.

  - If `step == x`, break out of the loop.

  - If `step > x`, fatal error.

Refine the worker interface with:

* `fn pause() -> Step`

  Sets `state` to `Pause`.  Cancel `token`.  Block until `state`
  becomes `Paused(step)` and return `step`.

* `fn stop(step: Step)`

  Sets `state` to `Stop(step)`.  Blocks until `state` becomes
  `Stopped`.

Refine the coordinator by, instead of just calling `stop()` on each
worker:

- Call `pause()` on each worker and take `step` as the maximum
  return value.
  
- Call `stop(step)` on each worker.

Refine the input interface by making `read()` take the cancellation
token and block until input arrives or `token` is cancelled, whichever
comes first.
</details>

[1]: https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html

