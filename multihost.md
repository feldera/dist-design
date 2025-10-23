# Multihost FT design

Multihost Feldera needs a membership service to tell it the network
name of each pipeline node and its own index within the list.  A
StatefulSet in Kubernetes is sufficient.  Each member of the set runs
a pipeline process.

Multihost Feldera also runs a single [coordinator](#Coordinator)
process in a separate host.  The pipeline manager talks to the
coordinator instead of to the pipeline processes.

## Coordinator

The coordinator is a process separate from the the pipeline process.
It provides the HTTPS API to the pipeline manager, implementing each
API in terms of the Feldera processes on each of the hosts.

Inside the Feldera cluster, the coordinator takes responsibility for
telling the workers when to start and stop and run.  The coordinator
also provides services outside the Feldera cluster.

> [1] and [2] are starting points but this new design is simpler
> because the pipeline only starts a step when the coordinator tells
> it.

[1]: README.md#coordinator
[2]: README.md#coordinatorworker-synchronization

### HTTPS API implementations

- `start`, `pause`, `shutdown`, `checkpoint`: Control the coordinator
  main loop.

- `query`: Requires creating a new `datafusion` `TableProvider` that
  can scan all of the pipeline nodes' tables.

- `stats`, `metrics`: Pulls stats/metrics from all of the pipeline
  nodes and aggregates them.

- `metadata`: Forwards to an arbitrary pipeline node.

- `heap_profile`, `dump_profile`, `dump_json_profile`: Profiles one
  pipeline node specified in the call (default 0, I guess).

- `input_endpoint`: Forward to an arbitrary pipeline node.

- `output_endpoint`: Connect to all of the pipeline nodes and
  aggregate their output.

- `pause_input_endpoint`, `start_input_endpoint`: Forward to all of
  the pipeline nodes.

- TBD:

  `activate`  
  `approve`  
  `status`  
  `suspendable`  
  `time_series`  
  `time_series_stream`  
  `lir`  
  `checkpoint/sync`  
  `checkpoint_status`  
  `checkpoint/sync_status`  
  `suspend`  
  `start_transaction`  
  `commit_transaction`  
  `ingress/{table_name}`  
  `egress/{table_name}`  
  `completion_status`  

## Runtime

### Pipeline startup

The pipelines need a new startup option that makes them expect
connections from the coordinator rather than the pipeline manager.
This option keeps them from instantiating the circuit immediately on
startup, because the coordinator needs to list their checkpoints and
find out which ones they have in common first.

> Alternatively, this could become the new behavior for all pipelines.
The pipeline manager can always tell it to instantiate the circuit.

### Coordinator startup

See [Protocol between coordinator and pipeline] for definitions of the
calls mentioned below.

1. Check for coordinator crash: Call `state()` on all of the pipeline
   nodes.  If all of them return `Open(step)` or `Running(step)` for
   the same `step`, then the coordinator crashed and restarted (but
   the pipelines remained up), and initialization is now complete.

2. Reset following pipeline crash: If any of the pipeline nodes in the
   previous step returned anything other than `Closed`, then the
   coordinator calls `shutdown` (the existing HTTP API) on all of
   them.

3. Resume pipelines from a common point: The coordinator calls
   `checkpoints` on all of the pipeline nodes.  If there exists at
   least one step number, in common all of the result vectors, then
   take the largest `step` among those and call `open(step)` on all
   the pipeline nodes, and initialization is complete.

4. Start pipelines from the beginning: The coordinator uses the
   configuration that the pipeline manager provided to create a
   configuration for each pipeline node.  For example, input and
   output connectors need to be assigned to a single node rather than to
   all of them (until multihost connectors are implemented).  The
   coordinator calls `create()` on each node with the appropriate
   configuration.

[Protocol between coordinator and pipeline]: #protocol-between-coordinator-and-pipeline

### Coordinator main loop

Let `step` be the current step.

Normally, we start from step 1 below, but if coordinator startup
detected that all of the pipelines have the state `Running(step)`,
then jump to step 3.

1. Wait until one of the following occurs, to trigger a step:

  - Any pipeline node calls `replay(step)`.

  - Any pipeline nodes call `received(step, n)` and either the total
    `n` is greater than the threshold, or the timer expires.

  - The clock tick timer expires.

  Alternatively, if the checkpoint timer expires, trigger a checkpoint
  instead.

2. If a step was triggered, call `step(step)` on each pipeline node;
   otherwise, a checkpoint was triggered, and call `checkpoint(step)`
   on each pipeline node.

3. Wait for `step` or `checkpoint` to return on all of the pipeline
   nodes.

4. If all of the calls were successful, start over from step 1.  If
   any of them failed, restart the coordinator.

In parallel, the coordinator runs a liveness check on each of the
pipelines: periodically, it calls `state()` on each one of them.  If
it does not return the expected result, then the coordinator restarts
itself (see [Coordinator startup](#Coordinator-startup)).  If it does
not return within a deadline, then the coordinator calls `shutdown` on
the pipeline and then restarts itself.

### Crashes and recovery

When the coordinator and pipelines start up together, they naturally
recover and replay starting from the newest checkpoint.

If the coordinator fails and restarts but the pipelines stay up, then
the coordinator startup procedure detects this and resumes gracefully.

If one (or more) of the pipelines crashes and restarts, the liveness
check detects this and restarts from the newest checkpoint.  (Pipeline
nodes cannot restart and replay individually, only in lockstep,
because of data exchanges with a pipeline.)

## Protocol between coordinator and pipeline

The pipeline provides the following interface to the coordinator
(probably through extra web service calls):

* `fn state() -> (Closed, Open(step), Running(step)`: Returns the
  processing state:

  - `Closed`: The pipeline isn't open yet.

  - `Open(step)`: The pipeline is open and stopped at the given step.

  - `Running(step)`: The pipeline is running a step and when it is
    complete, it will have the given step number.

* `fn checkpoints() -> Vec<Step>`: Returns the step numbers that the
  pipeline has checkpointed.  A fresh pipeline without any checkpoints
  will return an empty vector, otherwise it will have length 1 or 2.

* `fn open(step: Step)`: Opens the pipeline at the given `step`, which
  must be one of the steps returned by `checkpoints()`.

* `fn create(config)`: Starts a fresh pipeline without any history,
  with the provided configuration.

* `fn step(step)`: Runs the next step, `step`, which must be the step
  number provided to `open` plus the number of prior calls to `step`.

* `fn checkpoint(step)`: Checkpoint at the current step, which must be
  `step`.

The coordinator provides the following interface to each pipeline:

* `fn received(step: Step, n: u64)`: Notifies the coordinator that the
  pipeline received `n` input records for the given `step`.  The
  pipeline may send this message more than once for the same `step`
  with `n` increasing each time.  This allows the coordinator to
  trigger steps based on received data.

* `fn replay(step: Step)`: Notifies the coordinator that the pipeline
  has input log data for the given `step`.  This allows the
  coordinator to trigger steps based on log replay.

A pipeline should not call both `received` and `replay` for the same
`step`.  The pipeline should periodically repeat these calls for the
latest `step` to allow for coordinator crashes.

## Connectors

Initially, all the connectors could run in a single host, e.g. host 0.
Later, they could be spread across the hosts.  Eventually, individual
connectors might be able to load-balance across multiple hosts.  For
example, the Kafka input and output connectors could assign different
partitions to workers on different hosts.  (Others, such as the URL
connector, cannot  easily load-balance across multiple hosts.)

For output connectors, just before the output operator, we would need to
insert an operator to gather to a worker on the host where the
connector resides.

For input connectors, we could insert a shard operator just after the
input operator.  It might not be necessary, since most operators start
out by sharding.

## Triggering steps

In multihost instances, the hosts need to all trigger a step in
lockstep.  (If they do not, the hosts that trigger early will get
stuck waiting for the others in the first exchange operator.)

### Trigger reasons

A single-host instance can trigger a step for many reasons, including:

1. `N` records arrived.
2. At least 1 record arrived and a timer expired.
3. Clock tick (if enabled).
4. Checkpoint timer expired (if checkpointing is enabled).
5. Replaying is ongoing.

Reasons 1 and 2 still apply to multihost instances, but `N` should
count across the hosts.  (However, the default `N` of 1 makes adding
up counts across hosts unnecessary.)

For reasons 3 and 4, probably the coordinator should measure clock
ticks and checkpoint intervals.

### Design

Each host waits in its controller for records to arrive.  If records
arrive, it tells the coordinator how many.  The coordinator tells each
host when to trigger a step.

> Optionally, a host that can locally conclude that a trigger has
> occurred can start a step without the coordinator telling it.  This
> might be a worthwhile optimization with single-host input
> connectors, since it would give that host an early start on sending
> its records to the other hosts.

## Scaling

"Scaling" is changing the number of workers without restarting the
pipeline.  "Scale out" increases the number of workers, "scale in"
decreases them.

We consider only "cold" scaling, that is, scaling a suspended
pipeline, by starting it up with more or fewer workers than previously
ran it.

The challenge for scaling is workers' access to data on storage[^1].
Stored data is sharded among the workers.  Workers must be able to
access the data for the new layout:

- If storage is on a local file system, we will have to implement an
  access method, such as an RPC interface in each pipeline process for
  workers to transfer data.

- If storage is on an object store, the pipeline processes can access
  data for other workers.

- If storage is synchronized to an object store, the pipeline
  processes can synchronize from other workers' data.

A further challenge is that scaling requires moving part, not all, of
other workers' data, e.g. adding a fifth worker requires moving 1/4 of
the data from each of the existing workers.  Consider a single batch
within a trace, which is stored as a single file on each worker:

- One option is to copy the existing workers' files to the new worker,
  scan them for the data to be used by the new worker, and discard the
  rest.  This is inefficient because it will copy 5x as much data as
  the new worker retains.

  Each existing worker also needs to scan its data and discard the 25%
  that it no longer needs.  This requires fully reading the both index
  and data for the batch's keys, since sharding is done based on the
  hash of the key but the index is a B-tree and therefore every key
  must be read and hashed.  This can be done in a single pass with the
  scan needed to copy data to the new worker.  It might also be
  possible, by adding hashes to the storage file format, to
  efficiently filter out the keys to be removed when the read the file
  later.

- Suppose the new worker can instead scan the trace for the data that
  it actually needs (for example, using an RPC interface provided by
  the existing worker, or accessing it through an object store),
  instead of having to copy the whole thing.  This is less inefficient
  because it does not copy all the data, but it still requires the new
  worker to read the entire index and data for the batch's keys.

  The existing workers need to discard or filter 25% of their data,
  same as the previous case.

These options mean that resharding data requires reading and hashing
all the keys for all the batches, which is expensive.

Some ideas to increase efficiency:

- Add a hash index to the batch files.  This would still require
  reading all of the data blocks for the keys, since almost all of
  them would contain a key whose hash is sharded to the new worker,
  but it would not require hashing any of the data.

- Instead of a single shard and a single file, use many "microshards"
  with one file each.  Each microshard is assigned to a worker.  Then
  adding a fifth worker would move only 1/4 of the microshards from
  each of the existing workers, and those workers could simply delete
  their copies, which is perfectly efficient.

  The problem is, how does a worker work using many microshards?  By
  searching and stepping through all of them in parallel, so that
  using them is much more expensive.  So, to make scaling faster, we
  makes operations slower, which is not a good tradeoff, since we're
  running a lot more often than we're scaling.

- Divide scaling into two phases:

  * Phase 1 runs while the pipeline continues executing.  This phase
    reads all of the data in the existing workers, batch by batch, and
    passes an appropriate subset of it to the new worker.

    > The data that will remain with the existing worker still has to be
    discarded or filtered, same as before.

    This phase works on the largest batches first.  These batches are
    the least likely to be merged, so they are a good target for
    redistribution.  (It might make sense to disable starting new merges
    on large batches in phase 1, or to only process batches that are not
    currently being merged and then disable merging on them.  It is even
    possible to make the merger produce multiple output batches based on
    hash.)

  * Phase 2 runs when we have decided that phase 1 has reached a kind of
    steady state, where new data coming in is yielding batches that need
    to be divided by hash as quickly as we can divide them.  It seems
    likely that this is when all the largest batches have been divided.

    In phase 2, we halt the pipeline, divide the remaining batches by
    hash (only small batches should be left), suspend, and then resume
    with the new layout.

- "Micro-workers": Fix the number of workers at a high number, such as
  64 or 256 (8× or 32× the current default, respectively), and then
  scale by migrating whole workers.  This is efficient for scaling.
  For 64 workers assigned to 8 cores (plus 8 for background mergers),
  the operational changes are:

  * Exchange scales up from 8×8 to 64×64, a 64× increase.  Before the
    result can be read, each worker waits for 8× as many messages to
    come in, and each worker must do a 64-way merge instead of an
    8-way merge.

  * Scheduling 64 threads over 8 cores is more challenging than
    scheduling 8 threads over 8 cores.  Maybe it makes sense to do
    some kind of cooperative scheduling to keep the workers in
    lockstep, although exchange will keep them somewhat synced anyhow.

    > How do we interpret and implement CPU pinning?

  * We have a few options regarding background merger threads.  We
    could maintain the 1:1 background:foreground ratio, in which case
    the mergers would compete with each other.  That could be a
    positive effect if merger threads find themselves blocking on I/O.
    Or we could use only a single background thread for multiple
    foreground threads, a 1:8 ratio in this case.  This would prevent
    competing on CPU although they might waste time blocking on I/O
    more often.

    Batches being merged would be one-eighth the size.  Our mergers
    *should* adapt to this OK because of the approach they use now.

- Micro-workers without migration: Scale out and in by adjusting the
  amount of CPU available to a pipeline, without migrating anything at
  all and without changing the number of workers.

- Micro-workers with per-host scale-in/out: Scale out and in by
  adjusting the amount of CPU available to a pipeline, without
  migrating anything at all.  However, we do change the number of
  workers on each host: we can "scale in" N workers into 1 worker by
  merging their spines, initially by just adding all of their batches
  into a single spine and then afterward the merger would fix it up so
  it performed better.  "Scale out", which isn't as important, would
  split them somehow.

[^1]: For cold scaling, there is no in-memory data.

### Storage aspects for migrating data

In AWS EC2, we use EBS, which is expensive and local-only.  S3 is too
high-latency for production use, and NVMe is not durable.

Directions we could go for migrating data given the storage options:

* Avoid the need to migrate data at all, using one of the above
  scaling techniques that don't (i.e. "micro-workers without
  migration" or "micro-workers with per-host scale-in/out").

* Stick with our current sync-checkpoint-to-s3 and
  sync-checkpoint-from-s3 approach.  This starts copying from EBS to
  s3 after a checkpoint is complete, and it copies from s3 to EBS
  before starting from a checkpoint.  It will be slow for a full
  migration.

* More closely integrate s3 sync rather than using rclone, by copying
  to and from s3 in the background.  We could start running before
  sync from s3 was complete, and we could start copying data to s3
  before a checkpoint.  This would have less delay than currently.

* Directly operate on s3 but use a local NVMe "instance store" as an
  ephemeral cache.  Only a completed checkpoint to s3 would ensure
  durability, so the ephemeral NVMe would simply serve as a high-speed
  cache.  Exactly once configuration could use EBS for journaling.
