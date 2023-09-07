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
and restart is a special case of crash recovery where there is no
output beyond the checkpoint and thus no input to be replayed
(although there might be unprocessed input).

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

If convenient, the output implementation can expose step numbering to
the clients consuming the output.

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
  database with `commit()`.

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
require recording it.  Furthermore, for a given step, the complete
division of input must become durable before or at the same time as
any part of the step's output (otherwise, we could have output that we
don't know how to reproduce).  There are a few places we could record
the division:

- We could record the division in the Input implementation (e.g. write
  it to a Kafka topic).  We have a few options for making sure that it
  commits before the output does:

    - Input could block until recording the division of input commits
      before returning.  This would add latency.

    - Input could provide a function to block until recording the
      division of input commits.  The worker could call this after
      running the circuit and before writing the output.  This would
      help to hide the latency.

    - Input and Output could use a Kafka transaction to atomically
      commit their changes together.  This will work if the input and
      output are in the same Kafka broker.

- We could attach the division of input to the output that corresponds
  to it.  It's possible for some but not all of the output to become
  durable (e.g. some Kafka partitions but not others), so the division
  of input information would have to be redundantly attached to each
  part that might independently become durable.  It would add the need
  for the system to read back the tail of the output on restart or
  recovery to discover how the input was divided.  This would in turn
  mean that the system would add its own output durability
  requirements (currently there are none).  We would also have to
  consider writing output records even in the case where a given step
  yields no output updates.

- It's difficult to record the division of input in the state
  database, because we don't require durability for the state database
  on a step-by-step basis (a checkpoint spans an arbitrary number of
  steps).

- We can add a new database or log that just records the division of
  input.

- We can adopt an entirely different design approach that avoid the
  need to record the boundaries of input steps.  To do so, we need to
  reconsider our requirement that a step has well-defined input and
  output, with the output always corresponding to the input.  Suppose
  we relax our definitions to allow a crash to move input from one
  step to another (but not across checkpoints).  Then, recovery would
  read the output that was produced beyond the checkpoint, run the
  circuit with the input produced beyond the checkpoint, and then
  write as output the difference between the new output and the
  previously produced output.  (If the output implementation allows
  output to be deleted or marked obsolete, then that would work too.)
  Then, it might be wise to produce a new checkpoint.

  With such an approach, there would be a one-to-one correspondence
  between input and output steps in ordinary operation.  Following
  recovery, the output following the checkpoint would correspond to
  the input following the checkpoint, but a more detailed
  correspondence would not be possible until the next checkpoint.

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

`start()` sets its local variable `state` to `Run` when it starts and
to `Stopped` just before it returns.  At the top of its internal loop,
instead of just getting input, it consults `state`:

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

# Scale in and out

The computational resources in our distributed system are hosts and
workers.  Distributed DBSP should support "scale out" to increase
workers and hosts and "scale in" to decrease them.  For performance,
input and state should be divided more or less evenly across workers
regardless of the current scale.

For [input/output synchronization](#inputoutput-synchronization),
scale must be fixed at a given checkpoint.  That is, if a checkpoint
requires recovery, then recovery must take place with the same scale
that was initially used.  Thus, the state database must include scale
information, and changing scale requires committing a checkpoint then
updating the scale data without processing any input and immediately
committing again.

> 💡 The [alternate design approach](#alternate-design-approach) would
allow recovery at different scale.

## Scaling input

[Input](#input) provides input to each worker.  As the number of
workers increases or decreases, it must adapt to provide data to the
new workers.  The distribution of data among workers does not affect
correctness, because DBSP does not assume anything about the
distribution of input.  For performance, data should be fairly well
balanced among workers.

The distribution of data among workers only affects performance up to
the first time it is re-sharded with the "exchange" operator.  SQL
programs are likely to re-shard data early on, so input scaling might
affect performance in only a limited way.  We do not want it to be a
bottleneck, but it also seems unwise to focus much effort on it.

We currently expect DBSP input to arrive using the Kafka protocol.
Kafka allows data to be divided into several partitions.  The number
of partitions is essentially fixed (the admin can increase it but not
decrease it).  Each worker reads data from some number of partitions.
As scale changes, the assignment also changes to balance input across
workers.

## Scaling state

When the number of workers changes, DBSP must also change how the
"exchange" operator (described in [Distribution of
computation](#distribution-of-computation)) re-shards data among
workers.  Exchange occurs entirely inside DBSP, so the restrictions
of, e.g., Kafka do not constrain it.

However, computations that use an exchange operator also maintain
state associated with keys, which must reside in the same worker as
the key.  When scaling adjust exchanges so that a given key is sent to
a different worker, the state associated with that key must also be
moved.  To speed up scaling operations, we want to move as little
state as possible.

> 💡 If DBSP maintains state in a distributed database, instead of a
> per-worker local database, then DBSP itself doesn't need to
> explicitly move state data around, but the database still needs to
> do it.

DBSP currently divides data in exchange by hashing each key and taking
`hash % W`, where `W` is the number of workers, as a worker index.
This is a worst case for moving state: when the number of workers
increases from `W` to `W+1`, `W/(W+1)` of the state moves.

<details><summary>Some better approaches</summary>
Let `S` be the number of "shards" for the exchange operator to divide
data into.  Then the following algorithms seem like reasonable ones:

* Fixed sharding: Choose a relatively large number `S`, e.g. 1024, of
  "shards", numbered `0..S`.  Initially, for `W` workers, assign
  approximately `S/W` shards to each worker, e.g. `0..333`,
  `333..666`, and `666..1000` for `S = 1000` and `W = 3`.  To add a
  new worker, assign it approximately `1/S` shards, taking them from
  the existing workers so that the new division is approximately even,
  e.g. a fourth worker might receive shards `0..83`, `333..416`, and
  `666..750`.

  This algorithm reassigns a minimum amount of traffic when `W`
  changes.  As `W` approaches `S`, it divides data more and more
  unevenly.  It requires a `S`-sized array to express the assignments.

  This is essentially the same as the Kafka partitioning algorithm,
  but we can afford for `S` to be larger than `P`.

* Dynamic sharding: Choose `0 < k < 1` as the maximum inaccuracy in
  sharding to tolerate, e.g. `k = 0.1` for 10% inaccuracy.  Let the
  number of shards `S` be `(W/k).next_power_of_two()`.  Assign
  approximately `S/W` shards to each worker.  To increase or decrease
  the number of workers by 1, first recalculate `S`.  If this doubles
  `S`, then double the bounds of every existing shard slice, so that,
  e.g., `333..416` becomes `666..832`.  If this halves `S`, then halve
  the bounds (and adjust assignments, if necessary, for a minimum
  amount of disruption).  After adjusting `S`, assign workers to
  shards such that the division is approximately even.

  With this algorithm, shard data using the shard assigned to the data
  hash's `S.ilog2()` most-significant bits.

  This algorithm reassigns a minimum amount of traffic when `W`
  changes.  It requires a `S`-sized array to express the assignments
  (but the array size adapts appropriately for `W` and `k`).

Fixed partitioning requires one to choose `S` appropriately, which
might be hard.  Dynamic partitioning seems like a good approach.  It
might take some work to implement it exactly correctly (it is possibly
a novel algorithm), so it might not be worth implementing initially.

The choice of algorithm only matters for adding or removing a few
hosts at a time.  If the number of workers changes by more than 2× up
or down, then most of the data needs to move anyway, which means that
the choice of algorithm does not matter.
</details>

## Layout changes

We call the assignment of partitions and shards to workers, and
workers to hosts a "layout".  Scale-in and scale-out are both examples
of layout changes.

Given fault tolerance, we can change the layout in a "cold" fashion
by:

1. Terminate DBSP workers for the old layout.
2. Redistribute state as necessary.
3. Start DBSP workers with the new layout.

We can add or remove hosts in a "hot" fashion by:

1. Start the hosts to be added, if any.
2. Start copying state as necessary, while the circuit continues
   executing.
3. Stop the workers that are to be removed, initialize the workers
   that are to be added, and pause the others.
4. Finish copying the state.
5. Update the assignment of partitions and shards to workers.
6. Start the workers to be added, and un-pause the others.
7. Stop the hosts to be removed.

# Rolling upgrade

It's desirable to be able to upgrade the DBSP software without
disrupting an ongoing computation.  This section describes two
approaches.

Rolling upgrade does not support changing the computation or the
dataflow graph that implements it.  This procedure is for upgrades
that, e.g., fix a security vulnerability.

## Via scale-in and scale-out

Rolling upgrade of DBSP can leverage scale-out and scale-in as
primitive operations.  This approach works if a spare host is
available for the upgrade:

1. Add a new host to the computation, using the new version of DBSP.
2. Remove a host using the old version of DBSP.
3. If any hosts remain using the old version, start over from step 1.

If no spare host is available, swap steps 1 and 2.

## Via layout change

Iterative rolling upgrade, as described in the previous section, moves
all DBSP state at least twice: step 1 moves `1/n` of the data to the
added host, step 2 moves `1/n` of the data from the removed host, and
the process is repeated `n` times.

However, the layout change process is more general.  In a single step,
one can add host A and remove host B and ensure that the data assigned
to B is now assigned to A, so that only `1/n` of the data needs to
move.  Over `n` iteration, only half as much data moves.  One could
also do a single step that removes all of the old hosts and adds all
of the new ones.

Furthermore, if a given host has enough memory and other resources,
one could start the new DBSP processes on the same hosts as the old
ones and use a layout change that avoids moving any data at all.
