# Networking and Connections

This guide provides an overview of how networking (TCP socket acceptors, listeners) and connections
(for multiple protocols) are implemented in RabbitMQ.

## Boot Step(s)

When RabbitMQ starts, it executes a directed graph of boot steps, which depend on each other.
One of the steps is responsible for starting the networking-related machinery:

``` erlang
-rabbit_boot_step({networking,
                   [{mfa,         {rabbit_networking, boot, []}},
                    {requires,    log_relay}]}).
```

As you can see above, the function that kicks things off is `rabbit_networking:boot/0`.

### Listener Tracking

Before we dive into `rabbit_networking:boot/0`, it should be explained
that listeners are tracked using a Mnesia table, `rabbit_listener`.

The purpose of tracking listeners is two-fold:

 * to make it possible to stop active listeners during shutdown
 * to make it possible to list them e.g. in the management UI

The table is updated by `rabbit_networking:tcp_listener_started/3` and
`rabbit_networking:tcp_listener_stopped/3`.

### Distribution Listener

Erlang distribution TCP listener is also tracked: `rabbit_networking:boot/0`
uses node name and `erl_epmd:port_please/2` to determine distribution port.

### Messaging Protocol Listeners

Every protocol supported usually has 1 or 2 listeners:

 * Plain ("X over TCP")
 * TLS ("X over TLS")

Listeners are collected from config file sections of RabbitMQ core
and plugins that provide protocol support (e.g. STOMP).

`rabbit_networking:boot_tcp/0` and `rabbit_networking:boot_tcp/0` start plain TCP and
TLS listeners, respectively.


## Listener Process Tree

RabbitMQ as of 3.6.0 uses [Ranch](https://github.com/ninenines/ranch) in [embedded mode](https://github.com/ninenines/ranch/blob/master/doc/src/guide/embedded.asciidoc)
to accept TCP connections.

A listener is represented by two processes, which are
started under the `tcp_listener_sup` supervisor:

 * `tcp_listener`
 * `ranch_listener_sup`

The former handles listener tracking (see above), the latter is
a [Ranch listener](https://github.com/ninenines/ranch/blob/master/doc/src/guide/listeners.asciidoc) process.

`tcp_listener_sup` itself is a child of `rabbit_sup`, the top-level
RabbitMQ supervisor.

Every listener has one or more acceptors (under `ranch_acceptors_sup`)
and a supervisor for accepted client connections (under `ranch_conns_sup`).

`ranch_conns_sup` supervises client connections, which will be covered in more
details in the following section.


## AMQP 0-9-1 Connection Process Tree

Every AMQP 0-9-1 connection has a supervisor, `rabbit_connection_sup`, which is placed under
`ranch_conns_sup` in the process tree. It supervises two processes:

 * `rabbit_reader`: an important module in the protocol, see below
 * `rabbit_connection_helper_sup`: supervises helper processes

So the hierarchy of processes looks like this:

``` org-mode
 * rabbit_connection_sup
 ** rabbit_reader
 ** rabbit_connection_helper_sup
 *** rabbit_channel_sup_sup
 *** heartbeat_receiver
 *** heartbeat_sender
 *** rabbit_queue_collector
```

### rabbit_reader

`rabbit_reader` is one of the key modules in the AMQP 0-9-1 implementation.

This is a `gen_server`-like process that handles binary data parsing,
authentication, connection negotiation state machine, and keeps track
of channel processes.

This module also handles protocol "hand-offs" such as that to AMQP 1.0 reader
when an AMQP 1.0 client connects (despite being a completely different protocol,
it uses the same port as AMQP 0-9-1).

### Auxiliary processes

Every connection has several auxiliary processes supervised by
`rabbit_connection_helper_sup`:

 * `rabbit_channel_sup_sup`
 * `heartbeat_receiver` (module: `rabbit_heartbeat`)
 * `heartbeat_sender` (module: `rabbit_heartbeat`)
 * `rabbit_queue_collector`

#### rabbit_channel_sup_sup

Top-level supervisor for channels. Every channel is represented by
a group of processes under a supervisor (`rabbit_channel_sup`).

#### Heartbeat Processes

Heartbeat implementation uses two processes, one for sending heartbeats
and another handling client heartbeats.

See `rabbit_heartbeat:start_heartbeat_sender/4` and `rabbit_heartbeat:stop_heartbeat_sender/4`.

#### Queue Collector

Queue collector keeps track of exclusive queues, whose lifecycle
is tied to that of their declaring connection. Whenever a connection
is closed or lost, all exclusive queues that belonged to it must be
cleaned up.

`rabbit_queue_collector` is a `gen_server` which handles the above.


## STOMP Connetion Process Tree

