Structure:

* We need a durable endpoint and a factory for one.  Initially, they
  might just have a single implementation, for Kafka.
  
```
type Step = u64;

/// Just like `InputFactory`.
trait DurableInputFactory {
    fn name(&self) -> Cow<'static, str>;

    fn new_endpoint(
        &self,
        name: &str,
        config: &YamlValue
    ) -> AnyResult<Box<dyn DurableInputEndpoint>>;
}

trait DurableInputEndpoint {
    /// Requests that the endpoint retrieves `step`, feeds it to `consumer`,
    /// and then signals completion by calling `consumer.eoi()`.  When the
    /// step has been durably recorded, the endpoint should also call the
    /// (new) function `committed()` on `consumer`.
    ///
    /// The requested `step` must either be in the range returned by `steps()`
    /// or just beyond it.  In the latter case, the endpoint should wait until
    /// `commit()` is called before it signals completion.
    fn read(step: Step, consumer: Box<dyn InputConsumer>);

    /// Requests that the endpoint commits `step`, which must be the step
    /// previously passed to `read()`.
    fn commit(step: Step);

    /// The range of steps that the endpoint has available without blocking.
    fn steps() -> Range<Step>;

    /// Notifies the endpoint that steps less than `step` aren't needed
    /// anymore.  It may optionally discard them.
    fn expire(step: Step);
}
```

* What kinds of fault tolerance do we support?
* Upgrade for single-node case
  - How do we distinguish compatible and incompatible upgrade?
  - What about Kafka/RedPanda upgrade?



Leonid:

* Can we do this entirely from the database? without adding more state
  to the Kafka input adapter?
  
* Kafka transactions: do we need them?

```rust
type Step = u64;

trait OutputEndpoint {
    /// Notifies the output endpoint that data subsequently written by `push_buffer`
    /// belong to the given `step`.  If data for the given step has been written
    /// before, the endpoint should discard it up to the previously written length.
    /// Optionally, the endpoint may compare the previous version to the new version
    /// and log a warning if it differs.  Beyond the previously written length,
    /// the endpoint should append to it if `step` is the last step in the output,
    /// or log a warning if the output already contains steps beyond `step`.
    fn start_step(step: Step) -> AnyResult<()>;

    /// Notifies the output endpoint that it no longer needs to track the positions
    /// of steps before `step`.  It may optionally discard that tracking metadata.
    fn expire_step(step: Step);
}
```
