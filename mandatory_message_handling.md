# Mandatory Message Handling #

When we publish a message with the mandatory flag on, this means that
the broker must notify the publisher if the message is not routed to
any queue via the `basic.return` AMQP command. In this guide we will
see how the channel handles mandatory messages.

## Tracking Mandatory Messages ##

Mandatory messages are tracked in the `mandatory` field of the
channel's state record. Messages are tracked using our own
[dtree](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/dtree.erl)
data structure. As explained in that module documentation, entries on
the _dual-index tree_ are stored using a primary key, a set of
secondary keys, and a value. In the case of tracking mandatory
messages we have:

- primary key: the `MsgSeqNo` assigned to the message by the channel
- secondary keys: the list of queue pids where the message was routed
- value: the actual message

Keep in mind that delivering the message to queues is an asynchronous
operation, this means that in the `deliver_to_queues/2` function
inside the channel, we just know to which queues the message was
routed to, but this doesn't mean it had arrived there. Therefore only
when a queue has accepted the message, we can forget about it and
consider it as routed.

## Forgetting About Mandatory Messages ##

Once a queue has received a message, in other words, the message was
successfully routed to the queue, the queue process will cast the
following message back to the channel pid:

```
{mandatory_received, MsgSeqNo}
```

 The channel will then proceed to use the `MsgSeqNo` to forget about
the mandatory message it was tracking, by deleting it from the
`mandatory` dtree from the channel state.

Keep in mind that mandatory messages only require that a return is
sent in case the message is unroutable, so it's safe to forget about
it once the message has been routed to a queue.

## Sending Returns ##

If a mandatory message cannot be routed to any queue then we need to
send `basic.return`s back to the publisher. This is done in two
different places, responding to different situations.

The first one is the obvious one, if the message is not routed to any
queue, then we can safely send a `basic.return` back. Take a look at
the function `rabbit_channel:process_routing_mandatory/5` for more details.

The other situation arises when a queue where we had just publishes
messages crashes before it is able to receive the message. As
explained in the [deliver to queues](./deliver_to_queues.md) guide, we
monitor the QPids where the message was delivered. If the monitor
reports that the queue has crashed, then we will send `basic.return`
for all the messages that were delivered to the queues that
crashed. Take a look at the function
`rabbit_channel:handle_publishing_queue_down/3` for more information.

## Related guides ##

- [basic publish](./basic_publish.md)
- [deliver to queues](./deliver_to_queues.md)
