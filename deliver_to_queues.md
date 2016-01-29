# Deliver To Queues #

In this document we will be going over quite a few RabbitMQ modules,
since a message crosses all of these once it enters the broker:

```
rabbit_reader -> rabbit_channel -> rabbit_amqqueue -> delegate -> rabbit_amqqueue_process -> rabbit_backing_queue
```

Let's see this process in more detail.

The process of delivering messages to queues start during
`basic.publish`, right after the channel receives the result from
calling `rabbit_exchange:route/2`.

First we need to lookup the list of `#amqqueue` records based on the
destinations obtained from `route/2`. These records will be passed to
the function `rabbit_amqqueue:deliver/2` where they will be used to
obtain the _pids_ of the queue process where the message is going to
be delivered. Once the master and slave pids have been obtained, then
the message can start its way to be delivered to a queue process,
which consists of two parts: accounting for credit flow, and casting
the message into the queue process.

If the message delivery arrived with `flow = true`, then `credit_flow`
must be accounted for this message. One credit for each master Pid
where the message should arrive, plus one credit for each slave pid
that receives the message.

Then the message delivery will be sent to master pids and slave pids,
via the `delegate` framework. The Erlang message will have this shape:

```erlang
{deliver,            %% message tag
 Delivery,           %% The Delivery record
 SlaveWhenPublished} %% The Pid that received the message, was it a
                     %% slave when the deliver was published? This is
                     %% used in case of slave promotion
```

You can learn more about the delegate framework
[here](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/delegate.erl#L19).

## AMQQueue Process Message Handling ##

At this point the message delivery will finally arrive at the queue
process, implemented as a `gen_server2` callback inside the
`rabbit_amqqueue_process` module. The message from the delegate
framework will be received by the `handle_cast/2` callback. This
callback will ack the `credit_flow` issued in above, and it will
monitor the message sender. The message sender is usually the
`rabbit_channel` that received the process. This pid is tracked using
the
[pmon module](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/pmon.erl). The
state is kept as part of the `senders` filed in the gen_server state
record. Once the message sender is accounted for the delivery is
passed to the function `deliver_or_enqueue/3`. There is where the
message will either be sent to a consumer or enqueued into the backing
queue.

### Mandatory Message Handling ###

The first thing `deliver_or_enqueue/3` does is to account for the
mandatory flag of the delivery. If the message was published as
mandatory, then at this point the queue process will consider the
message as routed to the queue. To that effect, the queue process will
cast the message `{mandatory_received, MsgSeqNo}` to the channel pid
that received the delivery. The channel process will the proceed to
forget the message, since from the point of view mandatory message
handling, there isn't anything left to do for that particular
delivery.

Take a look at the
[mandatory message handling guide](./mandatory_message_handling.md) for
more info.

### Message Confirm Handling ###

When handling confirms we need to take into account two things: is the
queue durable, and was the message published as persistent. If that's
the case, then the queue process will keep track of the `MsgId` in
order to confirm the message back later to the channel that received
it from a producer. To achieve that, the queue process keeps track of
a dictionary in the process state, using `msg_id_to_channel` record
field to hold it. As the name of the field implies, this dictionary
maps _msg ids_ to _channels_. When a message is finally persisted to
disk by the backing queue, then the BQ will notify the queue process,
which will send the confirm back to the channel using the
`msg_id_to_channel` dictionary just mentioned.

If the queue was non durable, or the message was published as
transient, then the queue process will proceed to issue a confirm back
to the channel that sent the message in.

The function `rabbit_misc:confirm_to_sender/2` is the one taking care
of sending confirms back to channels.

Take a look at the
[publisher confirm handling guide](./publisher_confirms.md) for more info.

### Check for Message Duplicates ###

The next step is to check if the message has been seen by the queue
before. If the backing queue responds that the message is a duplicate,
then processing stops right here, since there's anything left to do
for this delivery, so `deliver_or_enqueue/3` simply returns.

### Attempt to Deliver the Message to a Consumer ###

To try to send the message delivery to a consumer, the function
`attempt_delivery/4` is called. This function will in turn call
`rabbit_queue_consumers:delivery/3` which takes a `FetchFun`, the
`QueueName`, and the `Consumers State` for this particular queue. The
Fetch Fun will return the message that will be delivered to the
consumer (if a consumer is available). This function deals with
message acknowledgment from the point of view of the queue. If the
consumer is in `ackmode = true`, then the message will be
`publish_delivered` into the backing queue, otherwise the message will
be discarded.

Discarding a message involves confirming the message, in case that's
required for this particular delivery, and telling the backing queue
to discard it as well.

Once the queue attempted to deliver the message straight to a
consumer, it will call the function `maybe_notify_decorators/2` which
takes care of telling the queue decorators that the consumer state
might have changed. See the [queue decorators](./queue_decorators.md)
guide for more information on how decorators work.

The `attempt_delivery/4` will return back to the
`deliver_or_enqueue/3` function telling it if the message was
`delivered` or if it is still `undelivered`. If the message was
delivered to a consumer, then there's nothing else to do, and
`deliver_or_enqueue/3` will simply return. Otherwise there's still
more to do.

### Handling Undelivered Messages ###

When handling undelivering messages, there's a special case that can
be considered an optimization. If the queue has a
[TTL](https://www.rabbitmq.com/ttl.html) of 0, and no
[DLX](https://www.rabbitmq.com/dlx.html) has been set up, then there
is no point in queueing this message, so it can be discarded in the
same way as explained above.

If a message cannot be discarded, then it has to be enqueued, so the
queue process will `publish` the message into the backing queue. After
the message has been published, we need to enforce the various
policies that might apply to this queue, like `max--length` for
example. This means we need to see if the queue head has to be
dropped. Once that's enforced, then we also have to check if we need
to drop expired messages. Both these functions work in conjunction
with the DLX feature mentioned above. At this point
`deliver_or_enqueue/3` returns.

## Bookkeeping ##

Even if we are done with the delivery after this was handled by the
respective queue processes where it was sent, we still need to perform
some bookkeeping on the channel side. The `rabbit_amqqueue:deliver/2`
function will return a list of `QPids` that received the
messages. This list of pids will be used now for bookkeeping.

### Queue Monitoring ###

The first thing to do is to monitor the queue pids to which the
message was delivered. This is done among other things, to account for
credit flow in case the queue goes down. We don't want to block the
channel forever if a queue that's blocking it is actually down.

Take a look at the `handle_info` channel callback for the case when a
`DOWN` message is
[received](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_channel.erl#L578).

### Process Mandatory Messages ###

Here if the message wasn't delivered to any queue, then it's time to
issue `basic.return`s back to the publisher that sent them. If the
message was delivered to queues, then those `QPids` will be kept into
a dictionary for later processing.

As explained above, once the queue process receives the message
delivery, then it will take care of updating the `mandatory`
dictionary on the channel's state.

### Process Confirms ###

Similar as with mandatory messages, if the message wasn't routed to
any queue, then it's time to record the message as confirmed. If the
message was delivered to some queues, then it will be tracked as
unconfirmed until the queue updates the message status.

### Stats Update ###

The final step for the channel is to account for stats, so it will
update the exchange stats, indicating that a message has been routed,
and then it will also update the queue stats, to indicate that a
message was delivered to this or that queue.

## Summary ##

Delivering a message to a RabbitMQ queue is quite an involved process,
and we didn't even touch on queue mirroring! The main things to
account for when handling a delivery are mandatory messages and
message confirms. Both have to be handled accordingly, and the whole
process is coordinated between the channel process and the queue
process that receives the message. Other than that, the queue needs to
see if the message can be delivered to a consumer or if it has to be
enqueued for later. Once this is handled, the queue needs to enforce
the various policies that can be applied to it, like TTLs, or
max-lengths.

To understand what happens once a message arrives to a queue, take
look at the [variable queue](./variable_queue.md). guide.
