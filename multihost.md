# Multihost FT design

Multihost Feldera needs a membership service to tell it the network
name of each member and its own index within the list of members.  A
StatefulSet in Kubernetes is sufficient.

Multihost Feldera runs the pipeline process on every host.  It also
runs a single [coordinator](#coordinator) process, either on one of
the pipeline hosts (e.g. member 0) or a separate host.

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
  can scan all of the cluster members.

- `stats`, `metrics`: Pulls stats/metrics from all of the cluster
  members and aggregates them.

- `metadata`: I don't know what this is for.

- `heap_profile`, `dump_profile`: Profiles one cluster member
  specified in the call (default 0, I guess).

- `input_endpoint`: Forward to an arbitrary cluster member.

- `output_endpoint`: Connect to all of the cluster members and
  aggregate their output.

- `pause_input_endpoint`, `start_input_endpoint`: Forward to all of
  the cluster members.

> [1] and [2] are starting points but this new design is simpler
> because the pipeline only starts a step when the coordinator tells
> it.

[1]: README.md#Coordinator
[2]: README.md#Coordinator%2Fworker%20synchronization

## Protocol between coordinator and pipeline

The pipeline provides the following interface to the coordinator
(probably through extra web service calls):

* `fn checkpoints() -> Vec<Step>`: Returns the step numbers that the
  pipeline has checkpointed.  A fresh pipeline without any checkpoints
  will return an empty vector, otherwise it will have length 1 or 2.

* `fn open(step: Option<Step>)`: Opens the pipeline.  If `step` is
  specified, the pipeline reads the checkpoint for that step;
  otherwise, it starts fresh.  The pipeline may delete checkpoints
  other than the one opened.

* `fn step()`: Runs the next step.  The step number for the step is
  that provided to `open` plus the number of prior calls to `step`.

* `fn checkpoint()`: Checkpoint at the current step.  If there were
  already more than one checkpoint before this call, the pipeline may
  discard all but the newest of them before creating the new one.

The coordinator provides the following interface to each pipeline:

* `fn received(step: Step, n: u64)`: Notifies the coordinator that the
  pipeline received `n` input records for the given `step`.  If the
  pipeline sends this message more than once for the same `step`, then
  the coordinator adds the `n`s together to get the total number of
  records.  This allows the coordinator to trigger steps based on
  received data.

* `fn replay(step: Step)`: Notifies the coordinator that the pipeline
  has input log data for the given `step`.  This allows the
  coordinator to trigger steps based on log replay.

  A pipeline should not call both `received` and `replay` for the same
  `step`.

