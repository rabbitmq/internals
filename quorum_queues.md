## Quorum Queues Implementation Notes

#### Pre-Requisites

At a minimum read the [Raft paper](https://Raft.github.io/Raft.pdf).

And the [Ra docs](https://github.com/rabbitmq/ra/blob/master/doc/INTERNALS.md)

A familiarity of RabbitMQ queue behaviour is also beneficial.

### Overview

Quorum queues implement a persistent, replicated queue on top of
[Ra](https://github.com/rabbitmq/ra).

One queue uses one Ra cluster and thus each queue has its own set of processes
and its own logical Raft log. Quorum queues are persistent by virtue of all
commands and messages being appended to the disk log and
thus do not make use of the RabbitMQ message store.


The queue logic is implemented as a [Ra state machine](https://github.com/rabbitmq/ra/blob/master/doc/STATE_MACHINE_TUTORIAL.md)
 in pure erlang code without side-effects.
The queue is an AMQP-like FIFO queue supporting publish (enqueue),
subscribe (checkout), ack (settle) and return operations (as well as few others).
The queue machine logic is implemented in the `rabbit_fifo` module.

Raft clusters are leader/follower systems and thus for a quorum queue all operations
need to flow through the leader and then be replicated to the followers.

All replicas in a Ra cluster/quorum queue execute the same set of operations and maintain
the same state. Only the leader executes side effects such as sending deliveries
to expecting channels.

### Design Properties / Constraints

Several constraints have influenced the current design:

* Scale-out
    * Supporting many (thousands) of quorum queues within a RabbitMQ cluster.
* Resource efficiency
    * Avoid unnecessary writes of data to disk.
* Throughput requirements
    * RabbitMQ queues typically need to support good sustained throughput
scenarios.
* Reliable and safe fail-over (HA)
    * No data loss on fail-over
    * Fast (no blocking "queue sync")
* Stability
    * No flapping leaders

### State Machine Implementation

The quorum queue implementation consists of two primary modules:

 * `rabbit_fifo`: the `ra_machine` behaviour implementation for
the queue logic.

 * `rabbit_fifo_client`: a "client" module providing a high level API for
interacting with quorum queues.
It is primarily used by channels to perform stateful, reliable,
session-based interactions with the `rabbit_fifo` state machine.


#### RabbitMQ Integration

The RabbitMQ integration use the following modules:

 * `rabbit_quorum_queue`: quorum queue specific behaviour as well as  thin wrapper
API for calls into `rabbit_fifo_client`.
 * `rabbit_amqqueue`: handles dispatch to logic for the appropriate queue type.
 * `rabbit_channel`: mostly delegates to `rabbit_amqqueue` but also handles
`ra_events` emitted from the Ra server.

#### Session Implementation

Session state is maintained by both the client (channel) and server (queue).
It is used to detect lost deliveries and enqueues, flow control and to ensure
enqueue ordering invariants are maintained even as leaders crash and are replaced.

`rabbit_fifo_client` supports a single outgoing session (enqueues) and
multiple incoming sessions (consumers).

Sessions are automatically created when a channel either enqueues or consumes
from a queue.

Contrary to classic queues channels do _not_ monitor the current leader. The
reason for this is that quorum queues allow new leaders to be elected very quickly
when a failure happens and the consumer state is also persisted to the log so
a follower can take over and resume sending deliveries to a consuming channel
even if a leader fails. The quorum queue cluster should appear as a single entity
to the channel.

#### Enqueuers

In `rabbit_fifo` a process that enqueues messages is called an "enqueuer". This
would typically be a channel.

With every enqueue that is sent to the quorum queue the `rabbit_fifo_client`
attaches a monotonically incrementing sequence number as a correlation token
that Ra will return once a command has been successfully replicated and applied
to the state machine. This is also the point when
it can be confirmed using the publisher confirm mechanism.

If the `rabbit_fifo_client` receives a correlation sequence number that
is higher than
the lowest unconfirmed number it knows the enqueue command must have been lost
and resends the enqueues for the missing sequence numbers.

The `rabbit_fifo` state machine also uses these sequence number to re-sequence
enqueues. That is if it receives a higher sequence number than the next expected
it will cache the message in a holding queue until it can ensure messages
are only made available to the queue in sequence order.

Thus the ordering invariant between the channel and the queue is maintained even
when the leader changes and it takes some time for the channel to re-discover
the current leader.

#### Consumers

A channel can have multiple consumers and thus `rabbit_fifo` supports multiple
consumer sessions from a single channel. The reliability protocol is reversed
compared with enqueuers:

`rabbit_fifo` includes a monotonically incrementing sequence
number for every delivery to the channel. If the channel detects a missing
sequence number or numbers it will perform a read-only query of the quorum
queue to fill in the blanks before passing the deliveries on to the protocol.

#### Session State Clean Up

Session state cleanup is a tricky subject in general and there is no exception
to that rule for quorum queues.

`rabbit_fifo` monitors enqueuer and consumer processes but will only
clean up session state if it receives
a `'DOWN'` notification for the pid with the reason `noproc`.

If the reason is `noconnection` as would happen if the RabbitMQ node was
disconnected for any reason, `rabbit_fifo` will mark every enqueuer and consumer
on this node as "suspected". It will then ask Ra to monitor this node. When the
erlang node comes back `rabbit_fifo` will re-issue all monitors for all sessions
for that node. If any of the channels are no longer around it will then process
'DOWN' with `noproc` commands for these and remove the session state accordingly.

When a quorum queue is deleted an `eol` even is sent to every active channel
process that can then handle this and clean up all state held for the
quorum queue in question.

### Snapshotting

Quorum queues implement a distributed queue as a Raft state machine.
The main challenge is how to efficiently [snapshot](https://raft.github.io/raft.pdf)
(§7 of the Raft paper)
the Raft state machine state. Snapshotting allows Raft to reclaim disk space
used by the log by replacing it with a serialised version of the state
machine state. Raft can then truncate the log below the raft index of the snapshot.
A RabbitMQ queue could contain thousands of message payloads at any one time.
Writing such a state to disk would cause a lot unnecessary write amplification
as the message payloads are already persisted in the log.

Messages on a queue often do not remain for a long time and thus may be
consumed very shortly after they were included in a snapshot.

Any queue that removes messages in FIFO-ish order can be reasoned
about as a sliding
window of enqueue and dequeue commands in the Raft log. For every message that
is settled (acked) there is an earlier index corresponding to the enqueue command
of the message that was settled that now no longer contribute to the current
state of the queue. If no other messages with enqueue commands with a lower
raft index exist in the queue then the log can be truncated at this point and
replaced with a version of the state with no messages.
`rabbit_fifo` can then ask Ra to snapshot at the index of the now settled
enqueue with a "dehydrated" version of the state that does not include any
payload data.

This may be best described using an example.
This simplified quorum queue state machine has 3 commands:
* ENQ (enqueue)
* CHK (checkout or "consume")
* SET (settle /ack)

Every ENQ command results in a compacted version of the state at that time
being cached to be used as a snapshot at some future point. Message payloads of
the compacted (or "dehydrated") state are replaced by a simple prefix message count.

Ra uses a mechanism called the "Release Cursor Effect" that allows the state
machine to inform Ra of a suitable snapshot index and the state to use for this.

| Raft Index | Command| Snapshot Info | Note |
| :---: | :------  | ---------- | ---- |
| 1     | ENQ[A]   | Caching StateA with {prefix_msg_count = 1} | cache a state for every n enqueue with message payloads replaced by a simple message count |
| 2     | ENQ[B]   | Caching StateB with {prefix_msg_count = 2} | _ |
| 3     | CHK[C1,2]| A and B are out for delivery to the consumer | A consumer C1 checking out messages with a prefetch of 2
| 4     | SET[B]   | _ | Settling B does not increment the release cursor as A is still active |
| 5     | ENQ[C]   | Caching StateC with {prefix_msg_count = 2} | NB: B has been settled by this point and should not be part of the prefix count |
| 6     | SET[A]   | {release_cursor, index: 2, StateB} | At this point no log entries before and incl. index 2 contribute to the current state so we can use the cached state of index 2 to emit a release cursor at index 2 using the cached, dehydrated state.

In short when applying index 6 we emit a snapshot point at index 2, i.e.
"in the past".

In the above example should we have to replay the log from index 3 using the
snapshot taken after index 6 the CHK[C1,2] command would be
satisfied from the `prefetch_msg_count`
rather than the actual message queue. This is OK as we'll
replay up to index 6 at which point the messages that the prefix count replaced
are all consumed anyway.

### Avoiding Unbounded Log Growth

The show must go on. FIFO (ish) processing must continue or the log will grow and
grow and grow.

Things that could cause the Raft log to grow in an unbounded fashion:

* Stuck messages:
    * Either during enqueue re-sequencing (an enqueue arriving out of order would
be held in a pre-queue and not be made available for consumption until prior messages
have arrived) or in consumer checkout.
* Continuous use of `return` / `nack` with `requeue=true`.
    * This degenerate anti-pattern will cause the queue to grow in an unbounded fashion. Messages need to be consumed. For
this kind of pattern we need a different kind of queue _or_ a peek operation
that doesn't affect queue state.


#### Channel Implementation

The channel is mostly unaware of the existence of the quorum queue type apart
from needing to handle the `ra_event` info messages. All erlang messages sent
from the quorum queue are wrapped inside `ra_event` tuples (`{ra_event, LeaderId, Event}`).
The channel simply forwards all Ra events on to `rabbit_fifo_client:handle_event/2`
and handles the return value of this function which can include new deliveries
or confirmed enqueue notifications.

#### Leader Tracking

Every message sent back from any Ra server includes the current leader if known.
`rabbit_fifo_client` tracks this. The leader is also persisted into the
`rabbit_amqqueue` record (`pid` field) every time it changes.

#### Flow Control

Integration with [credit flow](./credit_flow.md) is done at the channel level.

Flow control between the channel and the queue is done by tracking the status
of commands sent to the Raft cluster.
If the channel has not yet received applied notifications for a certain window
of commands it will call `credit_flow:block/1` to begin to push back on the
connection and reader processes. If the window of unapplied commands fall below
the configured maximum it will unblock the process again.

#### Message Store

No message store integration, Ra is already persistent. That's all there is
to be said about that.


### Lifecycle of a Message

#### Publish

A channel receives a `basic.publish` that is routed to a quorum queue.
It calls `rabbit_amqqueue:deliver/3` which in turn inspects the queue type
of the queue and calls `rabbit_quorum_queue:deliver/3`.
`rabbit_quorum_queue` then calls `rabbit_fifo_client:enqueue/3` with the
channel scoped msg sequence number.
`rabbit_fifo_client` formats an `enqueue` message containing it's own ra
command scoped sequence number and stashes that together with the
msg and the sequence number provided by the channel.

The enqueue command is dispatched to the Ra leader known by the channel using
`ra:pipeline_command/4`.

At some point later a `ra_event` will be received by the channel that is
forwarded on to the `rabbit_fifo_client:handle_event/2` function. This event
contains the new sequence numbers that have been applied to the state machine.
`rabbit_fifo_client`  removes the message from the un-applied cache and returns
the channel provided sequence number to the channel. This means the message
has been safely enqueued by the quorum queue and can now be part of publisher
confirm processing.

#### Consume

The channel initiates a new consumer by calling `rabbit_quorum_queue:basic_consume/9`
which in turn calls `rabbit_fifo_client:checkout/3`. This is done using `ra:process_command/2`
which blocks until a reply is received.

After this point messages will begin to be sent to the channel wrapped as `ra_event`s
so they are handled and accounted for by `rabbit_fifo_client`.
`rabbit_fifo_client:handle_event/2` return type can return a list of newly delivered
messages which can be sent by the channel on to the network as a `basic.deliver`.

Acks and nacks are eventually forwarded on to `rabbit_fifo_client` which formats
them as `rabbit_fifo` commands. `rabbit_fifo_client` implements the same reliability
tracking as for messages to avoid acks and nacks getting lost in transit only there
is no visibility of this process to the channel code.





