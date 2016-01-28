# Publisher Confirms #

Publisher Confirms are a way to tell publishers that messages have
been accepted by the broker an that the broker now takes full
responsibility for the message (ie: it was written to disk if it was
persistent, replicated if mirroring was enabled, and so on). Take a
look at the
[publisher confirms](https://www.rabbitmq.com/confirms.html)
documentation in order to understand the feature.

## Tracking Confirms ##

Confirms work a bit differently than mandatory messages and
`basic.return`. In the case of mandatory messages we only need to send
a `basic.return` if the message can't be routed, but for publisher
confirms we need to send an `ack` if the message was accepted by the
broker and a `nack` if that's not the case. This means we need two
fields in the channel's state record in order to track this
information.

The first one is a
[dtree](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/dtree.erl)
stored in the field `unconfirmed`, which keeps track of the `MsgSeqNo`
associated with the QPids to to which the message was delivered and
the Exchange Name used to publish the message. As explained in the
_dtree_ documentation, entries on the _dual-index tree_ are stored
using a primary key, a set of secondary keys, and a value. In the case
of tracking unconfirmed messages we have:

- primary key: the `MsgSeqNo` assigned to the message by the channel
- secondary keys: the list of queue pids where the message was routed
- value: the exchange where the message was published

The second field used when dealing with confirms is called `confirmed`
which keeps a list of the messages that were delivered to queues and
for which the queues have taken responsibility. The only thing left to
do with these messages is for the channel to send the `acks` back to
the publisher. This list tracks pairs of `{MsgSeqNo, XName}`.

## Marking Messages as Confirmed ##

Once a queue has dealt with a message (for example persisted it to
disk, in the case of persistent messages), then it will send confirms
back to the channel. This done by the QPid by casting the following
message back to the channel:

```erlang
{confirm, MsgSeqNos, QPid}`
```

The channel will deal with this message in the proper `handle_cast/2`
callback where it will remove the `MsgSeqNos` from the `unconfirmed`
dtree and then call `record_confirms/2`, with the passing in those
`MsgSeqNos` and related exchange names that were obtained form the
dtree. Keep in mind that at this point the function `record_confirms/2`
is only adding the messages to the `confirmed` list. They channel
still needs to send the `acks` back to the publisher.

Another place where confirms are recorded is in the function
`process_routing_confirm/5`. If the message wasn't routed to any
queue, then it will be immediately marked as confirmed by being added
to the `confirmed` list, otherwise it will track them in the
`unconfirmed` dtree until a queue confirms the message.

Finally as explained in the
[deliver to queues](./deliver_to_queues.md) guide, we monitor the
QPids where the message was delivered. If the monitor reports that the
queue has crashed the function `handle_publishing_queue_down/3` will
be called. In the case of confirms there are two cases: if the queue
had an abnormal exit, then `nacks` will be sent to messages that were
routed to that particular queue that just went down. If the queue had
a normal exit, then messages that went to that queue will be marked as
confirmed.

## Sending Out Confirms ##

Confirms or `acks` are sent back to the publisher whenever the channel
processes a request. For example, once it dealt with an AMQP method
like `basic.publish`, the channel will then send confirms out, if any.

For mode details, take a look at the functions `reply/2`, `noreply/3`
and `next_state/1` inside the `rabbit_channel` module.

## Sending Out Nacks ##

Nacks are sent to publishers when a queue that should have handled the
message has exited with an abnormal reason. Check the function
`handle_publishing_queue_down/3` for more information.

## Related guides and documentation ##

- [publisher confirms](https://www.rabbitmq.com/confirms.html)
- [basic publish](./basic_publish.md)
- [deliver to queues](./deliver_to_queues.md)
