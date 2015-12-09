# What are those transactions inside the exchange callback modules? #

Many callbacks inside the `rabbit_exchange_type` behaviour expect a
`tx()` parameter which is defined as follows:

```erlang
-type(tx() :: 'transaction' | 'none').
```

Then for example create is defined like:

```erlang
 %% called after declaration and recovery
-callback create(tx(), rabbit_types:exchange()) -> 'ok'.
```

The question is, what's the purpose of that transaction parameter?

This is related to how RabbitMQ runs Mnesia transactions for its
internal bookkeeping:

[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_misc.erl#L546](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_misc.erl#L546)

As you can see in that code there's this PrePostCommitFun which is
called in Mnesia transaction context, and after the transaction has
run.

So here for example:
[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange.erl#L176](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange.erl#L176)
the create callback from the exchange is called inside a Mnesia
transaction, and outside of afterwards.

You can see this in action/understand the usefulness of it when
considering an exchange like the topic exchange which keeps track of
its own data structures:

[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L54](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L54)
[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L64](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L64)
[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L69](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L69)

Deleting the exchange, adding or removing bindings, are all done
inside a Mnesia transaction for consistency reasons.
