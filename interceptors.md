# Interceptors #

Interceptors are modules implemented as behaviours that allow plugin
authors to intercept and modify AMQP methods before they are handled
by the channel process. They were originally created for the
development of the
[Sharding Plugin](https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbitmq_sharding/README.extra.md#intercepted-channel-behaviour)
to facilitate mapping queue names as specified by users vs. the actual
names used by sharded queues. Another plugin using interceptors is the
[Message Timestamp Plugin](https://github.com/rabbitmq/rabbitmq-message-timestamp)
which injects timestamps into message properties during
`basic.publish`.

An interceptor must implement the `rabbit_channel_interceptor`
behaviour. The most important callback is `intercept/3` where an
interceptor will be provided with the original AMQP method record that
the channel should process, the AMQP method content, if any, and the
interceptor state (see
[init/1](https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/src/rabbit_channel_interceptor.erl#L36)). This
callback should take the AMQP method that was passed to it, and the
content, and modify it accordingly. For example, the Sharding Plugin
will receive a `basic.consume` method, with a sharded queue called
`my_queue` and it will map that name, to the appropriate shard for the
client that issued the `basic.consume` command, for example:
`sharding: my_queue - 1`. This means that if the channel received the
following record:

```erlang
#'basic.consume'{queue = <<"my_queue">>}
```

Then the interceptor will pass back the following transformed method:

```erlang
#'basic.consume'{queue = <<"sharding: my_queue - 1">>}
```

This process is transparent to the user and to the channel
code. There's no need by RabbitMQ core developers to add extra
functionality to the `rabbit_channel` in order to support sharding,
since the interceptors take care of that. For example, if we need to
inject a timestamp into each message that crossed the broker, instead
of modifying the `rabbit_channel` code to do that, we can just create
a new interceptor for the `basic.publish` method, and there inject the
desired timestamp.

Interceptors can do more than just modifying AMQP methods, they can
also forbid its access. A good example is again the Sharding
Plugin. If we have a sharded queue called `my_queue` then it won't
make much sense to allow users to declare queues with that name, so
the sharding interceptor also intercepts the `queue.declare` method,
but in this case if the queue name provided matches that of a sharded
queue, then a channel error is produced.

Keep in mind that while we can enable several interceptors, only one
interceptor can intercept a particular AMQP method, otherwise we would
need to define interceptors priorities, plus a way to merge the
results of their invocations.

## Enabling Interceptors ##

To enable interceptors, they have to be registered into the
[rabbit_registry](./rabbit_registry.md), via a
[boot step](./boot_steps.md):

```erlang
-rabbit_boot_step({?MODULE,
                   [{description, "sharding interceptor"},
                    {mfa, {rabbit_registry, register,
                           [channel_interceptor,
                            <<"sharding interceptor">>, ?MODULE]}},
                    {cleanup, {rabbit_registry, unregister,
                               [channel_interceptor,
                                <<"sharding interceptor">>]}},
                    {requires, rabbit_registry},
                    {enables, recovery}]}).
```

Once the interceptor is registered, only new channels will use it to
intercept AMQP methods. Channels that were already running won't load
the interceptor. In a similar fashion, if a plugin that provides
interceptors is disabled, then only new channels will stop using these
interceptors.
