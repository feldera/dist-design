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

For DBSP, fault tolerance means that the system can recover from the
crash of one or more workers while providing "exactly once" semantics.
We implement fault tolerance by storing state in a database that is
periodically checkpointed.  To recover, all of the workers load the
most recent checkpoint and replay the input starting from that point,
discarding any duplicate output that this produces.  Graceful shutdown
and restart is a special case of crash recovery in which no input
needs to be replayed.

This form of fault tolerance has the following requirements, each of
which may be implemented in a variety of fashions:

* A per-worker database that supports checkpointing.  See
  [Database](#database).

* A durable log of input.  See [Input](#input).

* An output log, whose durability requirements arise from the needs of
  the system that consumes the output.  See [Output](#output).

## Database

Each worker maintains its circuit's state in a database.  We describe
the database as per-worker.  It could also be implemented as a single
distributed database.

Running the circuit queries and updates each database.  Occasionally,
a coordinator tells all of the workers to commit their databases.  If
all of them succeed, this forms a checkpoint that becomes the new
basis for recovery.

This section specifies the abstract interface that the database must
implement.  It is a key-value database that must support two versions.
Let `Version` be a `u64` type representing a database version serial
number:

  * A new database has only one version, numbered 0, which is empty.

  * Opening a database at version `v` causes the other version, if
    any, to be discarded.

  * Committing a database that was opened at version `v` creates a new
    version `v + 1`, either of which may be opened with `open`.
    
  * A database closed in any way other than `commit` retains the
    version that was passed to `open`.  If a crash happens during
    `commit` then it's OK for both versions or just the version passed
    to `open` to be available on recovery.

A database implementation presents the following interface to workers:

* `fn versions() -> Range<Version>`

  Reports the versions that the database can open.  The returned
  `Range` has length 1 or 2.
  
* `fn open(v: Version) -> Result<Database>`

  Opens the database.  Reads and writes will occur in version `v + 1`,
  which is initially a copy of version `v`.  The database discards
  versions other than `v`.
  
* `fn commit(db: Database) -> Result<()>`

  Commits the current version of `db` and closes the database.  After
  a successful return, if `v` was the version passed to `open()`, then
  versions `v` and `v + 1` may now be opened, where `v` has the same
  contents as before and `v + 1` reflects any changes made while the
  database was open.
  
* `fn get(db: &mut Database, table, key) -> Result<value>`  
  `fn put(db: &mut Database, table, key, value) -> Result<()>`

  Key-value pair read and write.
  
* `fn get_step(db: &Database) -> Result<Step>`  
  `fn set_step(db: &mut Database, step: Step) -> Result<()>`

  Helpers for reading and writing the step number for the next
  step to be applied.

## Input

A **step** is one full execution of the distributed circuit.  We
number steps sequentially as `Step` (an alias for `u64`).  A step
includes the circuit's input and output during that step.

To execute a step in the distributed circuit, all of the workers need
to obtain input for the step.  To ensure that recovery always produces
the same output as previous runs, the input for a given step must be
fixed.  That means that input must be logged durably.

The division of input between steps must also be durable (see
[Input/output synchronization](#inputoutput-synchronization)).

Input only need be durable until its step has become committed to the
earliest version in its local database.

Consider a worker to have input for steps `first..last`.  This might
be an empty range if no new input has arrived since the earliest
version of the local database.

An input implementation presents the following interface to workers:

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

An output implementation presents the following interface to workers:

* `fn write(step: Step, output: Vec<ZSet> -> Result<()>`

  Writes `output` as the output for step number `step`:
  
    * If `step` is in the range `0..first`, then `output` has already
      been produced before and discarded or deleted.  Discard it.
      
    * If `step` is in the range `first..last`, then `output` has
      already been produced before.  Now it has been produced
      redundantly during recovery.  Optionally, check that the new
      output matches the old output and signal an error if it differs.
      Discard `output`.

    * If `step == last`, then append it to the output stream and
      increment `last`.
    
    * `step > last` is an error.
    
## Worker

A worker presents the following interface to the coordinator:

* `fn versions() -> Range<Version>`

  Delegates to the local database.
  
* `fn start(version: Version)`

  Opens the local database with `open(version)`.  Sets a local
  variable `step` using `get_step()` from the database.
  
  Repeats the following loop until the coordinator calls `stop()`:
  
  - Gets input by calling `read(step)`.
  
  - Runs the circuit for one step, obtaining `output`.  The DBSP
    circuit coordinates across all of the workers on all of the hosts,
    exchanging data as necessary.

  - Writes output by calling `write(step, output)` (but see
    [Input/output synchronization](#inputoutput-synchronization)).
  
  - Increments `step`.
  
  Stores `step` into the database using `set_step()`.  Commits the
  database with `commit()`.  Sets `state` to `Stopped`.
  
* `fn stop()`

  Stops the loop in `start()`.
  
## Coordinator

In a loop:

- Calls `versions()` on each worker to get its database version range,
  intersects the results, and takes the maximum value `v`.  (If the
  intersection is the empty set, that's a fatal error.)
  
  If this is not the first iteration of the loop, `v` should be one
  more than in the previous iteration.
  
- Calls `start(v)` on each worker.

- Waits for an appropriate interval between checkpoints.

- Calls `stop()` on each worker.

## Synchronization

### Input/output synchronization

[Input](#input) mentioned that the division of input between steps
must be durable.  That is, a crash must not move input from one step
to another, even though such a change would ordinarily not change the
integral of the distributed circuit output following the steps that
changed.  This is because output may already have been produced for
one (or more, or even all) workers for steps being replayed, and
moving input from one step to another would potentially require some
of that already-produced output to change, which cannot be done.

In practice, making the division of input into steps durable seems to
require recording it, in the [Input](#input) implementation.
Furthermore, for a given step, it must become durable before or at the
same time as the step's output (otherwise, we could have output that
we don't know how to reproduce).  There are a few ways:

- The Input implementation could block until the division of input
  commits before returning.  This would add latency.

- Input could provide a function to block until the division of input
  commits.  The worker could call this after running the circuit.
  This would help to hide the latency.

- If the Input and Output implementations use underlying tech that
  allows it, they could cooperate to use a transaction to atomically
  commit the division of input and output.

<details><summary>Alternate design approach</summary>

In the system as described, a step has well-defined input and output,
with the output always corresponding to the input.  Suppose we relax
our definitions to allow a crash to input to move from one step to
another (but not across checkpoints).  Then, recovery would read the
output that was produced beyond the checkpoint, run the circuit with
the input produced beyond the checkpoint, and then write as output the
difference between the new output and the previously produced output.

With such an approach, there would be a one-to-one correspondence
between input and output steps in ordinary operation.  Following
recovery, the output following the checkpoint would correspond to the
input following the checkpoint, but a more detailed correspondence
would not be possible until the next checkpoint.
</details>

### Coordinator/worker synchronization

We need some synchronization to allow all the workers to pause at the
same step without blocking.  This isn't a big design consideration.

<details><summary>Coordinator/worker synchronization details</summary>

Here's one possible approach.

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

`start()` initially sets its local variable `state` to `Run`.  Then,
at the top of its loop, instead of just getting input, it consults
`state`:

* If `state` is `Run`, which is the common case, gets input by
  calling `read(step, &token)`.

* If `state` is `Pause`, sets it to `Paused(step)` and blocks until it
  becomes `Stop(_)`.

* If `state` is `Stop(x)`:

  - If `step < x`, gets input by calling `read(step, &token)`.

  - If `step == x`, breaks out of the loop.

  - If `step > x`, fatal error.

Refine the worker interface with:

* `fn pause() -> Step`

  Sets `state` to `Pause`.  Cancels `token`.  Blocks until `state`
  becomes `Paused(step)` and returns `step`.

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

