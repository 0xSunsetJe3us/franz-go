Transactions
===

The `kgo` package supports transactional producing, consuming only committed
records, and the EOS consumer/producer.

For an example of the transactional producer and consuming committed offsets,
see [here](../examples/transactions/produce_and_consume). The only real catch
to worry about when producing is to make sure that you commit or abort
appropriately, and for consuming, make sure you use the
[`FetchIsolationLevel`][1] option with [`ReadCommitted`][2] option.

[1]: https://pkg.go.dev/github.com/twmb/franz-go/pkg/kgo#FetchIsolationLevel
[2]: https://pkg.go.dev/github.com/twmb/franz-go/pkg/kgo#ReadCommitted

For an example of the EOS consumer/producer, see
[here](../examples/transactions/eos). Because EOS requires much more care to
ensure things operate correctly, there exists a [`GroupTransactSession`][3] helper
type to manage everything for you. Basically, to help prevent any duplicate
processing, this helper type sets its internal state to abort whenever
necessary (specifically, when a group rebalance happens). This may occasionally
lead to extra work, but it should prevent consuming, modifying, producing, and
_committing_ a record twice.

[3]: https://pkg.go.dev/github.com/twmb/franz-go/pkg/kgo#GroupTransactSession

KIP-447?
===

## The problem
 
[KIP-447](https://cwiki.apache.org/confluence/display/KAFKA/KIP-447%3A+Producer+scalability+for+exactly+once+semantics)
bills itself as producer scalability for exactly once semantics. Mostly, this
is a KIP to add a bit more safety to EOS, and is mostly tailored for the Java
client due to how it is implemented.

Before KIP-447, Kafka Streams was implemented to consume from only one
partition, modify records it consumed, and produce back to a new topic. Streams
_could not_ consume from multiple partitions as a part of a consumer group,
because a rebalance could cause input partitions to move around unsafely.

As an example of the problem, let's say we have two EOS consumers, A and B,
both of which can consume partitions 1 and 2. Both partitions are currently
assigned to A, and A is consuming, modifying, and producing back as a part of
its EOS flow. A rebalance happens, and partition 2 moves to consumer B. At this
point, A may have processed some records and not yet issued a `TxnOffsetCommit`
request. B will see the old commit and begin consuming, which will reprocess
records. B will produce and eventually commit, and there will be duplicates.
At any point, A may eventually commit, but it is already too late. Duplicates
have been processed, and A never knew

Confluent released a [blog
post](https://www.confluent.io/blog/simplified-robust-exactly-one-semantics-in-kafka-2-5/#client-api-simplification)
describing how KIP-447 makes it possible to consume from multiple partitions as
a part of EOS, and walks through an example of how things were problematic
before. Specifically, it proposes the following scenario, which I copy here:

```
Two Kafka consumers C1 and C2 each integrate with transactional producers P1
and P2, each identified by transactional ID T1 and T2, respectively. They
process data from two input topic partitions tp-0 and tp-1. 

At the beginning, the consumer group has the following assignments: 
(C1, P1 [T1]): tp-0 
(C2, P2 [T2]): tp-1 

P2 commits one transaction that pushes the current offset of tp-1 to 5. Next,
P2 opens another transaction on tp-1, processes data up to offset 10, and
begins committing. Before the transaction completes, it crashes, and the group
rebalances the partition assignment: 

(C1, P1 [T1]): tp-0, tp-1 
(C2, P2 [T2]): None 

Since there is no such static mapping of T1 to partition tp-0 and T2 to
partition tp-1, P1 proceeds to start its transaction against tp-1 (using its
own transactional ID T1) without waiting for the pending transaction to
complete. It reads from last committed offset 5 instead of 10 on tp-1 while the
previous transaction associated with T0 is still ongoing completion and causes
duplicate processing.
```

Fundamentally, this example is missing one hidden detail: P2 did not complete
its transaction, so there actually are no duplicate records _at the end_ once
P1 reprocesses offsets 5 to 10. Duplicates only arise if P2 comes back alive
and finishes its commit before the transactional timeout. It's tough to imagine
this scenario truly happening; more realistic is if P1 loses connectivity for a
blip of time and then later reconnects to commit.

## The franz-go approach

The franz-go client supports KIP-447, but allows consuming multiple partitions as
an EOS consumer/producer even on older (pre 2.5.0) Kafka clusters.

To start, unlike the Java client, franz-go does not require a separate client
and producer. Instead, both are merged into one "client", and by merging them,
the producer knows the state of the consumer at all times. Importantly, this
means that **if the consumer is revoked, the producer knows it, and the
producer will only allow an abort at the end of the transaction**. This mostly
solves the problem, but more remains.

It is possible that the consumer has lost connectivity to the cluster, and so
the consumer does not actually know whether or not it is still in the group.
For example, if heartbeat requests are hanging, the consumer could have been
kicked from the group by Kafka. If we allow a transaction commit at this point,
then we will again recreate the problem: we will commit offsets when we should
not, and then we will commit the transaction. To work around this, the franz-go
client forces a successful heartbeat (and a successful response) immediately
before committing the transaction. **If a heartbeat immediately before
committing is successful, then we know we can commit within the session
timeout**. Only a little bit more remains.

Even if we commit immediately before ending a transaction, it is possible that
our commit will take so long that a rebalance happens before the commit
finishes. For example, say `EndTxn` is about to happen, and then every request
gets stuck in limbo. The consumer is booted, and then `EndTxn` completes. This
again recreates our problematic scenario. To work around this, the franz-go
client defaults the transactional timeout to be less than the group session
timeout. With this, then we have the following order of events:

1) we begin a transaction  
2) we know that we are still in the group  
3) either we end the transaction, or we hang long enough that the transaction timeout expires _before_ the member will be booted  

By having the transactional timeout strictly less than the session timeout,
we know that even if requests hang after our successful heartbeat, then
the transaction will be timed out before a rebalance happens.

If a rebalance happens while committing, the OnRevoked callback is blocked
until the `EndTxn` request completes, meaning either the `EndTxn` will complete
successfully before the member is allowed to rebalance, or the `EndTxn` will
hang long enough for the member to be booted. In either scenario, we avoid our
problem.

Thus, although we do support 2.5.0+ behavior, the client itself works around
duplicates in a pre-2.5.0 world with a lot of edge case handling.
