# Distributed DBSP design

This document proposes a design for supporting the following features
in DBSP:

- [Distribution of computations](#distribution-of-computation) across multiple hosts.
- [Fault tolerance](#fault-tolerance) and fault detection.
- [Scale out and in](#scale-in-and-out) of a running computation.
- [Rolling upgrade](#rolling-upgrade) of DBSP itself.

The following features are out of scope:

- Upgrade of computation, or other changes to a computation implemented in DBSP.
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

To gracefully resume a computation, DBSP requires a consistent
snapshot of the following between two steps of the computation:

1. **Input** obtained from a data source but not yet processed.
2. **State** inside the circuit across all the workers.
3. **Output** produced by the circuit in a previous step but not yet
   acknowledged by its destination.
   
We need the input, output, and state captured at the same step, or to
be able to recompute them.  There are multiple ways to achieve this,
as summarized in [Fault Tolerance Approaches](approaches.md).  We
adopt the approach called "persistent state" there.

In the persistent state approach, each worker maintains a local
database of the circuit's state.  Each step updates the local
database.  Occasionally, a coordinator tells all of the workers to
commit their databases in a coordinated way.

In this approach, to recover from a fault, all of the workers load the
most recently committed state snapshot and replay the input starting
from that point, discarding any duplicate output that this produces.

The following sections describe some details for input and output and
state.
   
## Input and output

We adopt Apache Kafka for input and output.  Kafka is already widely
used for data streaming, which makes it an appropriate and familiar
interface for users.  Please refer to [Understanding Kafka](kafka.md)
for relevant information about Kafka.

In particular, for DBSP, suppose we have `W` workers, `H` input
handles, and `K` output handles.  We use the following Kafka topics,
each with `W` partitions[^1]:

* `input[0]` through `input[H-1]`.  Whatever is providing input to DBSP
  writes to these topics.  Worker `w` reads from partition `w` in each
  of these topics.

* `output[0]` through `output[K-1]`.  Worker `w` writes to partition `w`
  in each of these topics.  Whatever is reading output from DBSP reads
  from these topics.

We also need to keep track of how the input was divided up into
batches for the computation.  We use another Kafka topic with `W`
partitions for this:

* `step`.  Each event is an `H`-element array of event offsets.  If
  partition `w` with event offset `seq` holds such an array
  `offsets[]`, then worker `w` included `input[h]` up to `offsets[h]`
  in its computations for sequence number `seq`.

  There is a required invariant: if `e` is the least number of events
  in any partition in `output`, then every partition in `step` has at
  least `e` events.

[^1]: Section [Scale in and out](#scale-in-and-out) generalizes to the
    case where the number of Kafka partitions differs from the number
    of workers.

## State

To store operator state, we adopt RocksDB, a key-value store library
that supports data larger than memory.  RocksDB offers atomic
transactions with write-ahead logging.  RocksDB does not have a
built-in way to implement atomic transactions across a DBSP cluster,
but we can layer them on top.  See [Understanding RocksDB](rocksdb.md)
for details.

RocksDB works on local storage.  Each worker needs its own durable
local storage, and loss or corruption of any worker's storage corrupts
the whole computation.  We could instead use a distributed database
that includes fault tolerance of its own.  FoundationDB is a good
candidate.

## Algorithm

On a given host, take the following variables:

* `workers`, the set of workers on the host.
* `dbsp`, the `DBSPHandle` for the host's multithreaded circuit runtime.
* `input_handles`, the vector of `H` `CollectionHandle` input handles.
* `output_handles`, the vector of `K` `OutputHandle` output handles.
* `consumer`, a Kafka consumer, subscribed to partitions `workers` in
  `input[*]` and to all partitions in `step`, all initially positioned
  at the start of the partition.
* `producer`, a Kafka producer for the same broker as `consumer`.

Then:

- Open the state database.  Follow the [procedure](rocksdb.md) to get
  this worker's state in sync with all of the other workers.  Call
  that step `start_seq`.
  
- Open a transaction on the state database (if one isn't already
  open).
  
For `seq` in `start_seq..`:

1. Put data into the circuit for this step.  First, attempt to read
   one event from each partition in `workers` from `step`:

   * If we get an event for one or more of the workers, then we're
     replaying the log after restarting.  In this case, there will
     normally be an event for all of the workers but, if there was a
     failure, we might have an event in only some of them.  To handle
     that, we write an event to each partition that lacked one.  It's
     easiest to just fill them in as "empty", indicating that no data
     is to be included in those workers for this step, and it seems
     like we might as well do that because this case occurs at most
     once per program run.

     For `h` in `0..H`, for `w` in `workers`, read as much data from
     `input[h]` in partition `w` as `step` said (or nothing, if we
     filled it in as empty), and feed it to input handle `h` for
     worker `w`.

   * Otherwise, if we didn't get any events, we're running normally
     instead of replaying.  Block until we read one event from any
     partition in `step` or until we can read any number of events
     from partitions `workers` in `input[*]`.

     If we read any events from `step` in any partition in `workers`,
     that's a fatal error.  It means that some other process is
     running as one of our workers.

     For `h` in `0..H`, for `w` in `workers`, feed what we got (if
     anything) from `input[h]` in partition `w` to input handle `h`
     for worker `w`.

     For `w` in `workers`, write to `step` what we just fed to the
     circuit.  (We need to write this immediately, because the other
     hosts cannot continue before they read it.)
   
   This step can mostly be parallelized into the individual workers,
   but some care is needed for the blocking part in the "normal" path.

4. Run the DBSP circuit for one step, using and updating state from
   the state database.  The DBSP circuit coordinates across all of the
   workers on all of the hosts, exchanging data as necessary.

5. Start a Kafka transaction.

6. For `k` in `0..K`, for `w` in `workers`, write data from output
   handle `k` in worker `w` to partition `w` of `output[k]`, unless
   that output was already produced in a previous run.

   This can be run in parallel across the individual workers.

7. Read one event from `step` for each partition where we didn't
   already read one in step 1 above, blocking as necessary.  Also
   block until our own write or writes to `step` commit.
   
   This is necessary because we need all of `step` to commit before
   any of `output[*]`.  Otherwise, we could violate `step`'s
   invariant.

8. Commit the Kafka transaction.

9. If there's a coordinated commit request from the coordinator, then
   commit the state database and report that it's complete.  (The host
   may still commit in the absence of a commit request, as long as the
   database implementation can still roll back to the most recently
   coordinated commit.)

# Scale in and out

The computational resources in our distributed system are hosts and
workers.  Distributed DBSP should support "scale out" to increase
workers and hosts and "scale in" to decrease them.

## Data partitioning for scale-in/out

Given a particular number of workers `W`, the leading question for
scale-in/out is how to distribute and re-distribute data among the
workers for efficient processing.  The following sections describe
strategies for DBSP input and output and for the exchange operator.

### Input and output 

We have described distributed DBSP such that the number of workers `W`
equals the number of partitions `P` in each Kafka topic for input and
output.  It is impractical to maintain this invariant as `W` increases
and decreases over time, especially since Kafka allows the admin to
increase, but not decrease, the number of partitions in a topic.

Thus, we propose for a distributed DBSP deployment to fix `P` at
creation time.  `P` should be greater than or equal to the anticipated
maximum number of workers, but not so large as to waste disk space on
the Kafka brokers.  `P = 64` seems like a reasonable initial choice.

We must assign each partition to a worker.  If `P > W`, then at least
some workers will accept input data from multiple partitions, and if
`P < W`, then some workers will not accept data directly from any
partitions.  In the latter case, or if `P` is not a multiple of `W`,
the distribution of input data across workers will be unbalanced.

> 💡 This discussion assumes that every input and output uses the same
> number of partitions and same partition assignment.  That seems like
> a reasonable place to start.  However, we could support separate
> partition counts for inputs and outputs, or for each input and
> output.

DBSP does not assume anything about how producers distribute data
among input partitions.  For example, it does not assume that related
data are produced into the same partition.

### Exchange

The "exchange" operator described in [Distribution of
computation](#distribution-of-computation) rebalances data by
redistributing keys across workers.  However, computations that use an
exchange also maintain state about data streams that must reside in
the same worker that the stream flows through.  When scale-in/out
adjusts exchanges to send data to different workers, the associated
state must also move and, to make scale-in/out faster, we want to move
as little as possible.

> 💡 If DBSP maintains state in a distributed database, instead of a
> per-worker local database, then DBSP itself doesn't need to
> explicitly move state data around, but the database still needs to
> do it.

DBSP currently divides data in exchange by hashing each key and taking
`hash % W` as a worker index.  This is a worst case for moving state:
when the number of workers increases from `W` to `W+1`, `W/(W+1)` of
the state moves.

DBSP could reuse the mapping from Kafka partitions to workers, by
taking `hash % P` as a partition number and then using the same
partition-to-worker assignment as for input and output.  If `P < W` or
if `P` is not a multiple of `W`, this causes the same kind of
unbalanced distribution as already described for input and output.  We
can do better, because exchange lacks Kafka's disk space and scaling
constraints.

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
