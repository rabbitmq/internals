# Credit Flow #

In order to prevent fast publishers from overflowing the broker with
more messages than it can handle at any particular moment, RabbitMQ
implements an internal mechanism called credit flow that will be used
by the various systems inside RabbitMQ to throttle down publishers,
while allowing the message consumers to catch up. In this blog post we
are going to see how credit flow works, and what we can do to tune its
configuration for an optimal behaviour.

Since version 3.5.5, RabbitMQ includes a couple of new configuration
values that let users fiddle with the internal credit flow
settings. Understanding how these work according to your particular
workload can help you get the most out of RabbitMQ in terms of
performance, but beware, increasing these values just to see what
happens can have adverse effects on how RabbitMQ is able to respond to
message bursts, affecting the internal strategies that RabbitMQ has in
order to deal with memory pressure. Handle with care.

To understand the new credit flow settings first we need to understand
how the internals of RabbitMQ work with regards to message publishing
and paging messages to disk. Let’s see first how message publishing
works in RabbitMQ.

## Message Publishing ##

To see how credit_flow and its settings affect publishing, let’s see
how internal messages flow in RabbitMQ. Keep in mind that RabbitMQ is
implemented in Erlang, where processes communicate by sending messages
to each other.

Whenever a RabbitMQ instance is running, there are probably hundreds
of Erlang processes exchanging messages to communicate with each
other. We have for example a reader process that reads AMQP frames
from the network. Those frames are transformed into AMQP commands that
are forwarded to the AMQP channel process. If this channel is handling
a publish, it needs to ask a particular exchange for the list of
queues where this message should end up going, which means the channel
will deliver the message to each of those queues. Finally if the AMQP
message needs to be persisted, the msg_store process will receive it
and write it to disk. So whenever we publish an AMQP message to
RabbitMQ we have the following Erlang message flow[1]:

```
reader -> channel -> queue process -> message store.
```

In order to prevent any of those processes from overflowing the next
one down the chain, we have a credit flow mechanism in place. Each
process initially grants certain amount of credits to the process that
is sending them messages. Once a process is able to handle N of
those messages, it will grant more credit to the process that sent
them. Under default credit flow settings (`credit_flow_default_credit`
under `rabbitmq.config`) these values are 200 messages of initial
credit, and after 50 messages processed by the receiving process, the
process that sent the messages will be granted 50 more credits.

Say we are publishing messages to RabbitMQ, this means the reader will
be sending one erlang message to the channel process per AMQP
basic.publish received. Each of those messages will consume one of
these credits from the channel. Once the channel is able to process 50
of those messages, it will grant more credit to the reader. So far so
good.

In turn the channel will send the message to the queue process that
matched the message routing rules. This will consume one credit from
the credit granted by the queue process to the channel. After the
queue process manages to handle 50 deliveries, it will grant 50 more
credits to the channel.

Finally if a message is deemed to be persistent (it’s persistent and
published to a durable queue), it will be sent to the message store,
which in this case will also consume credits from the ones granted by
the message store to the queue process. In this case the initial
values are different and handled by the `msg_store_credit_disc_bound`
setting: 2000 messages of initial credit and 500 more credits after
500 messages are processed by the message store.

So we know how internal messages flow inside RabbitMQ and when credit
is granted to a process that’s above in the msg stream. The tricky
part comes when credit is granted between processes. Under normal
conditions a channel will process 50 messages from the reader, and
then grant the reader 50 more credits, but keep in mind that a channel
is not just handling publishes, it’s also sending messages to
consumers, routing messages to queues and so on.

What happens if the reader is sending messages to the channel at a
higher speed of what the channel is able to process? If we reach this
situation, then the channel will block the reader process, which will
result in producers being throttled down by RabbitMQ. Under default
settings, the reader will be blocked once it sends 200 messages to the
channel, but the channel is not able to process at least 50 of them,
in order to grant credit back to the reader.

Again, under normal conditions, once the channel manages to go through
the message backlog, it will grant more credit to the reader, but
there’s a catch. What if the channel process is being blocked by the
queue process, due to similar reasons? Then the new credit that was
supposed to go to the reader process will be deferred. The reader
process will remain blocked.

Once the queue process manages to go through the deliveries backlog
from the channel, it will grant more credit to the channel, unblocking
it, which will result in the channel granting more credit to the
reader, unblocking it. Once again, that’s under normal conditions,
but, you guessed it, what if the message store is blocking the queue
process? Then credit to the channel will be deferred, which will
remain blocked, deferring credit to the reader, leaving the reader
blocked. At some point, the message store will grant messages to the
queue process, which will grant messages back to the channel, and then
the channel will finally grant messages to the reader and unblock the
reader:

```
reader <--[grant]-- channel <--[grant]-- queue process <--[grant]--message store
```

Having one channel and one queue process makes things easier to
understand but it might not reflect reality. It’s common for RabbitMQ
users to have more than one channel publishing messages on the same
connection. Even more common is to have one message being routed to
more than one queue. What happens with the credit flow scheme we’ve
just explained is that if one of those queues blocks the channel, then
the reader will be blocked as well.

The problem is that from a reader standpoint, when we read a frame
from the network, we don’t even know to which channel it belongs
to. Keep in mind that channels are a logical concept on top of AMQP
connections. So even if a new AMQP command will end up in a channel
that is not blocking the reader, the reader has no way of knowing
it. Note that we only block publishing connections, consumers
connections are unaffected since we want consumers to drain messages
from queues. This is a good reason why it might be better to have
connections dedicated to publishing messages, and connections
dedicated for consumers only.

On a similar fashion, whenever a channel is processing message
publishes, it doesn’t know where messages will end up going, until it
performs routing. So a channel might be receiving a message that
should end up in a queue that is not blocking the channel. Since at
ingress time we don’t know any of this, then the credit flow strategy
in place is to block the reader until processes down the chain are
able to handle new messages.

One of the new settings introduced in RabbitMQ 3.5.5 is the ability to
modify the values for `credit_flow_default_credit`. This setting takes
a tuple of the form `{InitialCredit,
MoreCreditAfter}`. `InitialCredit` is set to 200 by default, and
`MoreCreditAfter` is set to 50. Depending on your particular workflow,
you need to decide if it’s worth bumping those values. Let’s see the
message flow scheme again:

```
reader -> channel -> queue process -> message store.
```

Bumping the values for `{InitialCredit, MoreCreditAfter}` will mean
that at any point in that chain we could end up with more messages
than those that can be handled by the broker at that particular point
in time. More messages means more RAM usage. The same can be said
about `msg_store_credit_disc_bound`, but keep in mind that there’s
only one message store[2] per RabbitMQ instance, and there can be many
channels sending messages to the same queue process. So while a queue
process has a value of 2000 as InitialCredit from the message store,
that queue can be ingesting many times that value from different
channel/connection sources. So 200 credits as initial
`credit_flow_default_credit` value could be seen as too conservative,
but you need to understand if according to your workflow that’s still
good enough or not.

## Message Paging ##

Let’s take a look at how RabbitMQ queues store messages. When a
message enters the queue, the queue needs to determine if the message
should be persisted or not. If the message has to be persisted, then
RabbitMQ will do so right away[3]. Now even if a message was persisted
to disk, this doesn’t mean the message got removed from RAM, since
RabbitMQ keeps a cache of messages in RAM for fast access when
delivering messages to consumers. Whenever we are talking about paging
messages out to disk, we are talking about what RabbitMQ does when it
has to send messages from this cache to the file system.

When RabbitMQ decides it needs to page messages to disk it will call
the function `reduce_memory_use` on the internal queue implementation in
order to send messages to the file system. Messages are going to be
paged out in batches; how big are those batches depends on the current
memory pressure status. It basically works like this:

The function `reduce_memory_use` will receive a number called target
ram count which tells RabbitMQ that it should try to page out messages
until only that many remain in RAM. Keep in mind that whether messages
are persistent or not, they are still kept in RAM for fast delivery to
consumers. Only when memory pressure kicks in, is when messages in
memory are paged out to disk. Quoting from our code comments: “The
question of whether a message is in RAM and whether it is persistent
are orthogonal”.

The number of messages that are accounted for during this chunk
calculation are those messages that are in RAM (in the aforementioned
cache), plus the number of pending acks that are kept in RAM (i.e.:
messages that were delivered to consumers and are pending
acknowledgment). If we have 20000 messages in RAM (cache + pending
acks) and then target ram count is set to 8000, we will have to page
out 12000 messages. This means paging will receive a quota of 12000
messages. Each message paged out to disk will consume one unit from
that quota, whether it’s a pending ack, or an actual message from the
cache.

Once we know how many messages need to be paged out, we need to decide
from where we should page them first: pending acks, or the message
cache. If pending acks is growing faster than messages the cache, ie:
more messages are being delivered to consumers than those being
ingested, this means the algorithm will try to page out pending acks
first, and then try to push messages from the cache to the file
system. If the cache is growing faster than pending acks, then
messages from the cache will be pushed out first.

The catch here is that paging messages from pending acks (or the cache
if that comes first) might result in the first part of the process
consuming all the quota of messages that need to be pushed to disk. So
if pending acks pushes 12000 acks to disk as in our example, this
means we won’t page out messages from the cache, and vice versa.

This first part of the paging process sent to disk certain amount of
messages (between acks + messages paged from the cache). The messages
that were paged out just had their contents paged out, but their
position in the queue is still in RAM. Now the queue needs to decide
if this extra information that’s kept in RAM needs to be paged out as
well, to further reduce memory usage. Here is where
`msg_store_io_batch_size` finally enters into play (coupled with
`msg_store_credit_disc_bound` as well). Let’s try to understand how
they work.

The settings for `msg_store_credit_disc_bound` affect how internal
credit flow is handled when sending message to disk. The
`rabbitmq_msg_store` module implements a database that takes care of
persisting messages to disk. Some details about the why’s of this
implementation can be found here: RabbitMQ, backing stores, databases
and disks.

The message store has a credit system for each of the clients that
send writes to it. Every RabbitMQ queue would be a read/write client
for this store. The message store has a credits mechanism to prevent a
particular writer to overflow its inbox with messages. Assuming
current default values, when a writer starts talking to the message
store, it receives an initial credit of 2000 messages, and it will
receive more credit once 500 messages are processed. When is this
credit consumed then? Credit is consumed whenever we write to the
message store, but that doesn’t happen for every message. The plot
thickens.

Since version 3.5.0 it’s possible to embed small messages into the
queue index, instead of having to reach the message store for
that. Messages that are smaller than a configurable setting (currently
4096 bytes) will go to the queue index when persisted, so those
messages won’t consume this credit. Now, let’s see what happens with
messages that do need to go to the message store.

Whenever we publish a message that’s determined to be persistent
(persistent messages published to a durable queue), then that message
will consume one of these credits. If a message has to paged out to
disk from the cache mentioned above, it will also consume one
credit. So if during message paging we consume more credits than those
currently available for our queue, the first half of the paging
process might stop, since there’s no point in sending writes to the
message store when it won’t accept them. This means that from the
initial quota of 12000 that we would have had to page out, we only
managed to process 2000 of them (assuming all of them need to go to
the message store).

So we managed to page out 2000 messages, but we still keep their
position in the queue in RAM. Now the paging process will determine if
it needs to also page out any of these messages positions to disk as
well. RabbitMQ will calculate how many of them can stay in RAM, and
then it will try to page out the remaining of them to disk. For this
second paging to happen, the amount of messages that has to be paged
to disk must be greater than `msg_store_io_batch_size`. The bigger
this number is, the more message positions RabbitMQ will keep in RAM,
so again, depending on your particular workload, you need to tune this
parameter as well.

Another thing we improved significantly in 3.5.5 is the performance of
paging queue index contents to disk. If your messages are generally
smaller than `queue_index_embed_msgs_below`, then you’ll see the
benefit of these changes. These changes also affect how message
positions are paged out to disk, so you should see improvements in
this area as well. So while having a low `msg_store_io_batch_size`
might mean the queue index will have more work paging to disk, keep in
mind this process has been optimized.

## Queue Mirroring ##

To keep the descriptions above a bit simpler, we avoided bringing
queue mirroring into the picture. Credit flows also affects mirroring
from a channel point of view. When a channel delivers AMQP messages to
queues, it sends the message to each mirror, consuming one credit from
each mirror process. If any of the mirrors is slow processing the
message then that particular mirror might be responsible for the
channel being blocked. If the channel is being blocked by a mirror,
and that queue mirror gets partitioned from the network, then the
channel will be unblocked only after RabbitMQ detects the mirror
death.

Credit flow also takes part when synchronising mirrored queues, but
this is something you shouldn’t care too much about, mostly because
there’s nothing you could do about it, since mirror synchronisation is
handled entirely by RabbitMQ.

## Footnotes ##

1. A message can be delivered to more than one queue process.
2. There are two message stores, one for transient messages and one for persistent messages.
3. RabbitMQ will call fsync every 200 ms.
