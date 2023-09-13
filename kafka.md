# Understanding Kafka

Kafka is a distributed system to store and buffer message.  In Kafka
parlance, clusters of "brokers" (servers) accept "events" (messages)
from "producers", store them, and make them available to "consumers".

There is something called Kafka Streams.  This is a Java-only API.  It
is not what we are using.

There's a fair amount of stuff in the Java kafka library and in
librdkafka that is not exposed in Rust yet, e.g. for deleting records.

The documentation at <https://kafka.apache.org/documentation/> is
helpful.

## Events

An event (aka message) has the following fields:

  * "Topic", a Unicode string.  Events in different topics are
    independent.

  * "Partition", a small integer that designates a subdivision of a
    topic.

    A topic has an admin-specified fixed number of partitions.  Each
    partition has a cost in disk space (a directory and at least two
    files) and in broker memory (`replica.fetch.max.bytes`, by default
    1 MB, per partition) and file descriptors (see [1] and [2]).

    Partition data is stored in "segments" (files) that have an
    admin-specified maximum size (1 GB by default).  Segments also
    have a maximum age (7 days by default).  At any given time, a
    partition has one active segment, where data will be added, and
    zero or more inactive segments.  When the active segment reaches
    its maximum size or age, Kafka creates a new active segment and
    the previously active segment becomes inactive.

    Partitions are append-only.

    A producer may assign an event to a partition.  If it does not,
    then Kafka chooses the partition by hashing the key, if the event
    has one, or round-robin, if it does not.

    A consumer specifies one or more partitions to read.
    
    Kafka allows the admin to increase (but not decrease) the number
    of partitions in a topic.  This doesn't redistribute existing
    messages, so the new partitions start out empty.  Kafka will
    rebalance topic-based consumer groups to include the new
    partitions (see [Consumer Groups](#consumer-groups)).

  * "Offset" within the event's partition.  The first event in a
    partition has offset 0, the second one offset 1, and so on.
    Applications can see gaps in event offsets because of event
    deletion or compaction and because transactions insert commit and
    abort markers.

    A consumer can seek to particular offsets in a partition and to
    the first and last offset in a partition.  The first offset in a
    partition can be greater than 0 for all the reasons given for gaps
    plus the ones listed in [Event Lifetime](#event-lifetime) below.

    Producers can't control offsets.  The broker assigns offsets
    sequentially as it appends events to partitions.

  * "Key", an optional `Vec<u8>`.  Kafka uses keys in two related
    ways:

    1. Log compaction (see [Event Lifetime](#event-lifetime)).

    2. Kafka chooses the default partition (see below) by hashing the
       key.

    The Kafka API doesn't give consumers a way to seek or filter data
    based on key.

  * "Timestamp", as an `i64` in milliseconds since the Unix epoch.

    The producer may provide a timestamp.  If not, or if Kafka has
    been configured to override producer timestamps, Kafka fills it
    in.

    A consumer can query the least offset of an event in a partition
    with a timestamp greater than or equal to a given timestamp.
    (This might be slow, because Kafka's Java API accepts a timeout.)
    This allows a consumer to seek by timestamp.

    Old versions of Kafka (before 0.10, released in 2016) didn't have
    timestamps.

  * Application data, which Kafka passes from producer to consumer but
    does not otherwise interpret:

    - "Payload": `Vec<u8>`.

    - "Headers": A map from `Vec<u8>` from `Vec<u8>`.

    The Kafka API doesn't give consumers a way to seek or filter data
    based on application data.

[1]: https://stackoverflow.com/questions/59740832/cost-of-an-unused-kafka-topic-partition
[2]: https://stackoverflow.com/questions/40694188/kafka-number-of-topics-vs-number-of-partitions

## Event lifetime

Events in Kafka are immutable, meaning that the event with a given
topic, partition, and offset topic will always have the same key,
timestamp, and application data, as long as it is not deleted.

Events can be deleted in at least the following ways:

* Time limit.  Kafka can automatically delete data from a partition
  when it reaches a given age, by default 7 days.

* Size limit.  Kafka can discard data from the beginning of a
  partition when the partition as a whole exceeds a given size.  Kafka
  does not limit the size of a partition by default.

* Log compaction[3].  When multiple data items in the same partition
  have the same key, Kafka assumes that the latest one supplants the
  earlier ones.  When Kafka compacts the log, it deletes the earlier
  instances of the key.  If the latest version of a given key has an
  empty payload then, after a configurable additional time, Kafka will
  delete the event entirely.  Kafka enables compaction by default.

* API.  The [`deleteRecords`][4] API can delete all the records in a
  partition earlier than a specified offset.
  
Kafka applies time and size limits at the segment level, by deleting
inactive segments from a partition.
  
[3]: https://kafka.apache.org/documentation/#compaction
[4]: https://kafka.apache.org/35/javadoc/org/apache/kafka/clients/admin/Admin.html#deleteRecords(java.util.Map)

## Consumer groups

A "consumer group" is a set of Kafka consumers that use the same
`group.id` in their consumer configurations.  Consumer group
membership comes and goes as consumers come up and down with a given
`group.id`.

A consumer group is associated with topics and, within each of those
topics, a position per partition (as the offset of an event).  Kafka
can commit (update) the stored positions automatically or manually.
With the manual update mode and Kafka [Transactions](#transactions),
position updates can be atomic to other Kafka activity.

Consumer groups can be used two ways:

* Topic subscription, for any number of consumers in a group.

  Each consumer uses `subscribe` and related APis to specify just the
  topics to read.  Kafka then automatically divides the partitions
  among the consumers currently subscribed to those topics.  As
  consumers come and go, Kafka automatically rebalances the partitions
  among the consumers.

* Manual partition assignment, only for a group with one consumer at a
  time, because Kafka does not coordinate multiple consumers and they
  will conflict.

  The consumer uses `assign` and related consumer APIs to specify
  particular topics and partitions to read.

Kafka documents that a consumer doesn't have to be in a consumer group
if it doesn't need either of the above features.  In practice,
sometimes it seems to be required anyway.  The Kafka broker durably
stores the current offset and membership of each consumer group, so
it's best to reuse consumer group names rather than generating a fresh
random one on every connection.

## Transactions

Kafka supports transactions.  A transaction includes reading from a
consumer and writing to a producer within a single broker.  Committing
the transaction does two things atomically:

1. Updates the consumer's positions in its consumer group.

2. Appends records to the producer.

Transactions take place within a given broker and do not span brokers.

A program opens and commits (or aborts) a transaction on a producer.
The producer automatically includes the records appended to it while
the transaction is open.  The program adds consumer position updates
to the transaction manually, before commit, with
`send_offsets_to_transaction`.

A transaction spans only one consumer and one producer within a single
broker.  The restriction to one consumer and one producer might not be
a significant restriction because a single consumer or producer can
read or write any number of topics and partitions.  (The API might
actually support multiple consumers; the documentation does not say.)

A consumer in a transaction must be part of a consumer group.

Before it starts its first transaction, the program using a producer
must call `init_transactions` on it.  This checks the
`transactional.id` configured on the producer, which is required, and
aborts any open transaction from that producer.  See "How to pick a
transactional.id" at [Transactions in Apache Kafka][3].

Kafka does not mark transaction boundaries for consumers.

See also:
  * <https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/>
  * <https://kafka.apache.org/35/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html>
  * <https://kafka.apache.org/documentation/#semantics>

[3]: https://www.confluent.io/blog/transactions-apache-kafka/

## Pitfalls

Consumer groups are required in cases where it seems the documentation
says they are not needed.  See [Consumer groups](#consumer-groups)
above.

Doing a seek within a consumer before polling to read data causes
strange errors, see [seek before poll].  Instead, specify the desired
seek offset when assigning the partition to the consumer.

[seek before poll]: https://github.com/confluentinc/confluent-kafka-go/issues/121#issuecomment-362308376
