# RabbitMQ Internals #

This project aims to explain how RabbitMQ works internally. The goal
is to make it easier to contribute for newcomers to the project, and
at the same time have a common repository of knowledge to be shared
across the project contributors.

## Purpose ##

Most interesting modules in RabbitMQ projects have documentation
essays, sometimes quite extensive, at the top. The aim here is not to
duplicate what's there, but to provide the highest-level overview as
to the overall architecture.

## Guides ##

In order to understand how RabbitMQ's internals work, it's better to
try to follow the logic of how a message progresses through
RabbitMQ as it is handled by the broker, otherwise, you would end up
navigating through many guides without a clear context of what's going
on, or without knowing what to read next. Therefore we have prepared
the following guides to help you understand how RabbitMQ works:

### Basic Publish Guide ###

Here we follow the life of a message since it's received from the
network until it has been routed by the exchanges. We take a look at
the various processing steps that happen to a message right until it
is delivered to one or perhaps many queues.

[Basic Publish](./basic_publish.md)

### Deliver To Queues Guide ###

After the message has been routed, the broker needs to deliver that
message to the respective queues. Here not only the message has to be
sent to queues, but also mandatory messages and publisher confirms
need to be taken into account. Also, the queue needs to try to deliver
the message to prospective consumer, otherwise the message ends up
queued.

[Deliver To Queues](./deliver_to_queues.md)

### Queues and Message Store

Provides an overview of the Erlang processes that back queues
and how they interact with the message store, message index and so on.

[Queues and Message Store](./queues_and_message_store.md)

### Variable Queue Guide ###

Ultimately, messages end up queued at the
[backing queue](https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/src/rabbit_backing_queue.erl). From
here they can be retrieved, acked, purged, and so on. The most common
implementation of the backing queue behaviour is the
`rabbit_variable_queue`
[module](https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/src/rabbit_variable_queue.erl),
explained in the following guide:

[Variable Queue](./variable_queue.md)

### Mandatory Messages and Publisher Confirm Guides ###

As explained on the [Deliver To Queues](./deliver_to_queues.md) guide,
a channel has to handle messages published as mandatory and also take
care of publisher confirms. These processes are explained in the
following guides:

- [Mandatory Message Handling](./mandatory_message_handling.md)
- [Publisher Confirms](./publisher_confirms.md)

### Authentication and Authorization ###

As explained in the [Basic Publish](./basic_publish.md), there are
some rules to see if a message can be accepted by the broker from a
certain publisher. This is explained in the following guide:

[Authorization and Authentication Backends](./authorization_and_authentication.md)

### Internal Event Subsystem

In some cases components in a running node communicate via events.
Some events are consumed by other nodes.

[Internal Events](./internal_events.md)

### Management Plugin ###

An architectural overview of the v3.6.7+ version of the management plugin.

[Metrics and Management Plugin](./metrics_and_management_plugin.md)

## Maturity and Completeness

These guides are not complete, haven't been edited, and are work in
progress in general.

So if you find yourself wanting more detail, check the code first!

## License

(c) Pivotal Software Inc, 2015-2016

Released under the
[Creative Commons Attribution-ShareAlike 3.0 Unported](https://creativecommons.org/licenses/by-sa/3.0/)
license.
