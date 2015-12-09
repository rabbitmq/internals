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
 
