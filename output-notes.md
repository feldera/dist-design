We need to produce output reliably and make sure that if it's produced
again then it gets discarded.

We have to assign a partition ourselves.

One way is to add a key to each Kafka event that identifies the step
number and the sequence number within the step.  At startup, we can
read the most recent key and suppress output that comes before that.

> Another way would be to add an "output-index" topic, similar to the
"input-index".  Each entry would identify the range of output events
corresponding to an output step.  This seems like more work than
necessary for now.

We can use transactions to avoid writing partial output.  We need a
transaction ID for that.  Since only one producer should be writing to
a given partition at a time, we can use the topic and partition number
as the transaction id.

