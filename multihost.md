# Multihost FT design

Multihost Feldera needs a membership service to tell it the network
name of each pipeline node and its own index within the list.  A
StatefulSet in Kubernetes is sufficient.

Multihost Feldera runs the pipeline process on every host.  It also
runs a single [coordinator](#Coordinator) process, either on one of
the pipeline nodes (e.g. node 0) or a separate host.

## Load-balancing input and output across hosts

Some connectors can't easily load-balance across multiple hosts.  For
input, the URL adapter is a good example.  For multihost use, the
coordinator can instantiate each of these adapters on a single host.
This should be a safe place to start for every kind of input and
output adapter, with some consideration:

- For output adapters, we must insert a gather operator just before
  the output operator.

- For input adapters, we should insert a shard operator just after the
  input operator.

As an advanced step, some input and output adapters should be able to
load-balance.  For example, the Kafka input and output adapters could
assign different partitions to workers on different hosts.

## Triggering steps

In multihost instances, the hosts need to all trigger a step in
lockstep.  (If they do not, the hosts that trigger early will get
stuck waiting for the others in the first exchange operator.)

### Trigger reasons

A single-host instance can trigger a step for many reasons:

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

> Optionally, a host that can locally conclude that reason 1 or 2 is
> fulfilled can start a step without the coordinator telling it.  This
> might be a worthwhile optimization with single-host input adapters,
> since it would give that host an early start on sending its records
> to the other hosts.

## Coordinator

Inside the Feldera cluster, the coordinator takes responsibility for
telling the workers when to start and stop and run.  The coordinator
also provides services outside the Feldera cluster.  It provides the
HTTP API to the pipeline manager, implementing each API in terms of
the Feldera processes on each of the hosts:

- `start`, `pause`, `shutdown`: Controls when the coordinator executes
  steps.

- `checkpoint`: Controls when the coordinator checkpoints the
  pipeline.

- `query`: Requires creating a new `datafusion` `TableProvider` that
  can scan all of the pipeline nodes' tables.

- `stats`, `metrics`: Pulls stats/metrics from all of the pipeline
  nodes and aggregates them.

- `metadata`: I don't know what this is for.

- `heap_profile`, `dump_profile`: Profiles one pipeline node
  specified in the call (default 0, I guess).

- `input_endpoint`: Forward to an arbitrary pipeline node.

- `output_endpoint`: Connect to all of the pipeline nodes and
  aggregate their output.

- `pause_input_endpoint`, `start_input_endpoint`: Forward to all of
  the pipeline nodes.

> [1] and [2] are starting points but this new design is simpler
> because the pipeline only starts a step when the coordinator tells
> it.

[1]: README.md#coordinator
[2]: README.md#coordinatorworker-synchronization

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
  must be one of the steps returned by `checkpoints()`.  The pipeline
  may delete checkpoints other than `step`.

* `fn create(config)`: Starts a fresh pipeline without any history,
  with the provided configuration.  The pipeline may discard any
  existing checkpoints.

* `fn step(step)`: Runs the next step, `step`, which must be the step
  number provided to `open` plus the number of prior calls to `step`.

* `fn checkpoint()`: Checkpoint at the current step.  If there was
  already more than one checkpoint before this call, the pipeline may
  discard all but the newest of them before creating the new one.

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

## Runtime

### Pipeline startup

The pipelines should be invoked with a new command-line option that
keeps them from instantiating the circuit immediately on startup,
because the coordinator needs to find out what checkpoints they have
in common first.

> Alternatively, this could become the new behavior for all pipelines.
The pipeline manager can always tell it to start the circuit.

### Coordinator startup

1. When the coordinator starts up, it first calls `state()` on all of
   the pipeline nodes.  If all of them return `Open(step)` or
   `Running(step)` for the same `step`, then the coordinator crashed
   and restarted (but the pipelines remained up), and initialization
   is now complete.

2. Otherwise, if any of the pipeline nodes in the previous step
   returned anything other than `Closed`, then the coordinator calls
   `shutdown` (the existing HTTP API) on all of them.

3. The coordinator calls `checkpoints` on all of the pipeline nodes
   and takes the largest step number, `step`, that is in all of the
   result vectors, if any.  Then:

   - If `step` exists, the coordinator calls `open(step)` on all of
     the pipeline nodes.

   - Otherwise, the coordinator uses the configuration that the
     pipeline manager provided to create a configuration for each
     pipeline node.  For example, input and output adapters need to be
     assigned to a single node rather than to all of them (until
     multihost adapters are implemented).  The coordinator calls
     `create()` on each node with the appropriate configuration.

### Coordinator main loop

Let `step` be the current step.

Normally, we start from step 1 below, but if coordinator startup
detected that all of the pipelines have the state `Running(step)`,
then jump to step 3.

1. Wait until:

  - Any pipeline node calls `replay(step)`.

  - Any pipeline nodes call `received(step, n)` and either the total
    `n` is greater than the threshold, or the timer expires.

  - The clock tick timer expires.

  If the checkpoint timer expires, trigger a checkpoint (see
  [checkpoint procedure](#Checkpoint-procedure)) and start over.

2. Call `step(step)` on each pipeline node.

3. Wait for `step` to return successfully on all of the pipeline
   nodes.

4. Start over from step 1.

In parallel, the coordinator runs a liveness check on each of the
pipelines: periodically, it calls `state()` on each one of them.  If
it does not return the expected result, then the coordinator restarts
itself (see [Coordinator startup](#Coordinator-startup).  If it does
not return within a deadline, then the coordinator calls `shutdown` on
the pipeline and then restarts itself.

## Crashes and recovery

When the coordinator and pipelines start up together, they naturally
recover and replay starting from the newest checkpoint.

If the coordinator fails and restarts but the pipelines stay up, then
the coordinator startup procedure detects this and resumes gracefully.

If one (or more) of the pipelines crashes and restarts, the liveness
check detects this and restarts from the newest checkpoint.  (Pipeline
nodes cannot restart and replay individually, only in lockstep,
because of data exchanges with a pipeline.)
