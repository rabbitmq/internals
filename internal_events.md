# Internal Events

This document describes a mechanism RabbitMQ components use to notify
each other of events. Note that this mechanism is not used to transfer
messages between components or nodes. It is also entirely transient.

## Overview

Client connection, channels, queues, consumers, and other parts of the system
naturally generate events. Other parts of the system can be interested
in observing those events. RabbitMQ has a very minimalistic mechanism that
is used for internal event notifications both within a single node and
across nodes in a cluster.

For example, when a policy is modified, RabbitMQ needs to apply it
to matching queues and notify the queues that no longer match.
These events are irrelevant to clients and have no relation to messaging
protocols.

## Events, Metrics, Stats

Perhaps the heaviest user of this notification subsystem, known
as `rabbit_event`, is the [management plugin](./metrics_and_management_plugin.md).
Management plugin's metrics are collected in a variety of ways but often
transferred over the internal event subsystem.

For example, when a connection is accepted, authenticated and access
to the target virtual host is authorised, it will emit an event of type
`connection_created`. When a connection is closed or fails for any reason,
a `connection_closed` event is deleted. Events from connections, channels, queues, consumers,
and so on are processed and stored as metrics
to be later served over HTTP to the management plugin UI application.


## rabbit_event

Both internal event publishers and consumers interact with the notification
subsystem using a single module, `rabbit_event`. Publishers typically
use the `rabbit_event:notify/2` function, consumers register
[gen_event](http://learnyousomeerlang.com/event-handlers) event handlers.

Every event is an instance of the `#event` record.
An event has a name (e.g. `connection_created` or `queue_deleted`), a timestamp and an
dictionary-like data structure (a proplist) for payload.

The mechanism is very minimalistic: every event handler receives a copy
of every event and ignores those that are irrelevant to it.


## Acting User Details

Starting with RabbitMQ 3.7.0, internal components try to associate an
acting user with each emitted event, where possible. For example,
if a channel is opened on a connection, the acting user is the user
of that connection.

In some cases there is no acting user or it cannot be known, for example,
when a connection is forcefully closed via CLI tools. In such cases
dummy usernames are used, e.g. `rmq-internal` or `rmq-cli`.


## rabbitmq-event-exchange Plugin

[rabbitmq-event-exchange](https://github.com/rabbitmq/rabbitmq-event-exchange) is a plugin that consumes internal events
and re-publishes them to a topic exchange, thus exposing the events
to clients (applications).

This can be used by monitoring an audit systems.
