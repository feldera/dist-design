# Fault Tolerance Approaches

Multiple approaches satisfy the [requirements for fault
tolerance](README.md#fault-tolerance).  The following sections
describe three of them.  The [persistent state](#persistent-state)
approach is the one adopted for DBSP, so it is covered in detail in
[our main document](README.md) instead of here.

We assume circuits are deterministic, regardless of the number of
workers.

## Input logging

Summary: Just log the input.  Reproduce state and output by replaying
the input from the start.

Suppose we have `W` workers, `H` input handles, and `K` output
handles.  We use the following Kafka topics, each with `W` partitions:

* `input[0]` through `input[H-1]`.  Whatever is providing input to DBSP
  writes to these topics.  Worker `w` reads from partition `w` in each
  of these topics.

* `output[0]` through `output[K-1]`.  Worker `w` writes to partition `w`
  in each of these topics.  Whatever is reading output from DBSP reads
  from these topics.

* `input-step`.  Each event is an `H`-element array of event offsets.
  If `offsets[]` is the array in the event in partition `w` with event
  offset `seq`, then worker `w` included `input[h]` up to `offsets[h]`
  in its computations for sequence number `seq`.

  There is a required invariant: if `e` is the least number of events
  in any partition in `output`, then every partition in `input-step`
  has at least `e` events.
  
On a given host, take the following variables:

* `workers`, the set of workers on the host.
* `dbsp`, the `DBSPHandle` for the host's multithreaded circuit runtime.
* `input_handles`, the vector of `H` `CollectionHandle` input handles.
* `output_handles`, the vector of `K` `OutputHandle` output handles.
* `consumer`, a Kafka consumer, subscribed to partitions `workers` in
  `input[*]` and to all partitions in `input-step`, all initially
  positioned at the start of the partition.
* `producer`, a Kafka producer for the same broker as `consumer`.

For `seq` in `0..`:

1. Put data into the circuit for this step.  First, attempt to read
   one event from each partition in `workers` from `input-step`:

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
     partition in `input-step` or until we can read any number of events
     from partitions `workers` in `input[*]`.

     If we read any events from `input-step` in any partition in `workers`,
     that's a fatal error.  It means that some other process is
     running as one of our workers.

     For `h` in `0..H`, for `w` in `workers`, feed what we got (if
     anything) from `input[h]` in partition `w` to input handle `h`
     for worker `w`.

     For `w` in `workers`, write to `input-step` what we just fed to
     the circuit.  (We need to commit this write immediately, so that
     the other hosts can read it before they commit their `output`
     changes.  Otherwise, we could violate the invariant.)
   
   This step can mostly be parallelized into the individual workers,
   but some care is needed for the blocking part in the "normal" path.

4. Run the DBSP circuit for one step.

5. Start a Kafka transaction.

6. For `k` in `0..K`, for `w` in `workers`, write data from output
   handle `k` in worker `w` to partition `w` of `output[k]`, unless
   that output was already produced in a previous run.

7. Read one event for `input-step` for each partition where we didn't
   already read one in step 1 above, blocking as necessary.  Also
   block until our own write or writes to `input-step` commit.
   
   This is necessary because we need all of `input-step` to commit
   before any of `output[*]`.  Otherwise, we could violate the
   invariant.

8. Commit the Kafka transaction.

## Checkpoint and restore

Summary: Periodically, log a delta to the circuit's state.  If the
state log grows too big, log a full snapshot of the circuit's state.

Suppose we have `W` workers, `H` input handles, and `K` output
handles.  In addition to those for input logging, we use the following
Kafka topics, each with `W` partitions:

* `snapshots`.  Groups of events form snapshots of the circuit's
  state.  A single event would suffice but Kafka isn't good at
  messages bigger than about 1 MB so we will need to divide it up.

  The exact grouping structure isn't obvious.  We need to be able to
  find the first event in the most recent snapshot.  Kafka
  transactions should ensure that only complete snapshots are
  written.

* `deltas`.  Events that describe state updates from one sequence
  number to another.  Each delta must refer to its base sequence
  number.  It should be possible to apply deltas starting from the
  most recent snapshot or from older snapshots.

Each host runs something like this:

- Load the most recent state snapshot that exists across all workers.

- Apply all of the state deltas up to the most recent delta that
  exists across all workers.
  
- Follow a loop similar to input logging, but also write deltas
  sometimes and snapshots occasionally.
  
### Operator state

With this approach to fault tolerance, stateful operators need a way
to save and restore their state and report and apply deltas.  Here's a
first stab at an approach by adding to `trait Operator`.  This should
be enough of an API to implement checkpoint and restore:

```rust
pub trait Operator: 'static {
    /// ...

    /// For an operator that contains internal state, this method must report
    /// that state by calling `_f` one or more times for each partition that it
    /// holds.
    ///
    /// Operators that don't contain internal state can use the default
    /// implementation.
    fn get_state<F>(&mut self, _f: F)
    where
        F: FnMut(usize, Vec<u8>),
    {
    }

    /// For an operator that contains internal state, this method must set that
    /// state to `_state` previously obtained from `get_state()`.  The caller
    /// may move partitions between workers.
    ///
    /// Operators that don't contain internal state can use the default
    /// implementation.
    fn set_state<'a>(&mut self, _state: impl Iterator<Item = &'a (usize, Vec<u8>)>) {}

    /// For an operator that contains internal state, this method must report
    /// the changes in that state since the "base state", which is the last time
    /// `get_state()`, `set_state()`, or `apply_delta()` was called.  For each
    /// state partition that has changed inside the operator, it must call `f`
    /// one or more times to report the changes.
    ///
    /// Operators that don't contain internal state can use the default
    /// implementation.
    fn get_delta<F>(&mut self, _f: F)
    where
        F: FnMut(usize, Vec<u8>),
    {
    }

    /// For an operator that contains internal state, this method must apply a
    /// delta previously obtained from `get_delta()`.  The state before the call
    /// must be the same as the base state at the time `get_delta()` was called,
    /// on a per-partition basis.  The caller may move partitions between
    /// workers.
    ///
    /// Operators that don't contain internal state can use the default
    /// implementation.
    fn apply_delta<'a>(&mut self, _delta: impl Iterator<Item = &'a (usize, Vec<u8>)>) {}
}
```

## Persistent state

Summary: Each worker maintains a local database of the circuit's
state.  Each step updates the local database in an open transaction.
Occasionally, a coordinator tells all the workers to commit their
databases.

This approach uses the same Kafka topics as for input logging.

Each host runs something like this:

- Open the state database, which is at some consistent step across all
  the workers.
  
- Follow a loop similar to input logging, but use and update state in
  the state database, and occasionally commit the database in a
  coordinated way, [using some care](rocksdb.md#coordinated-commit).

# Trade-offs

Each approach require stateful DBSP operators to support the
additional APIs listed as "yes" below:

| API                             | Input logging | Checkpoint & restore | Persistent state |
| ------------------------------- | ------------- | -------------------- | ---------------- |
| Save state snapshots            |               |                  yes |                  |
| Save state deltas               |               |                  yes |                  |
| Apply state snapshots           |               |                  yes |                  |
| Apply state deltas              |               |                  yes |                  |
| Use and update persistent state |               |                      |              yes |

Storage requirements for each approach:

| Storage requirement             | Input logging | Checkpoint & restore | Persistent state |
| ------------------------------- | ------------- | -------------------- | ---------------- |
| Input log                       |           yes |                  yes |              yes |
| State delta logs                |               |                  yes |                  |
| State snapshot logs             |               |                  yes |                  |
| Persistent state                |               |                      |              yes |
| Output log                      |           yes |                  yes |              yes |

Features offered by each approach:

| Feature                         | Input logging | Checkpoint & restore | Persistent state |
| ------------------------------- | ------------- | -------------------- | ---------------- |
| Larger than memory state        |               |                      |              yes |
| Efficient resume                |               |                  yes |              yes |

Input logging is probably a bad approach in the general case, because
the cost of resuming grows without bound over time, as does the size
of the input log.  It might make sense in some special case where it
is known that the input log can be compacted because its integral is
bounded in size.

Persistent state offers the unique built-in ability to support larger
than memory state.  Unlike the other approaches, it doesn't need to
rerun any computation or reconstruct any state.  However, the other
approaches can add support for larger than memory state by putting
that state into a database, which doesn't have to be durable and thus
can possibly be faster than a persistent state database.
