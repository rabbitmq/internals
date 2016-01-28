# Exchange Decorators #

Exchange decorators are modules implemented as behaviours that can let
you extend existing exchanges. For example, you might want to perform
some actions only when the exchange is created, or deleted, but leave
alone the whole routing logic to the underlying exchange.

Decorators are usually associated with exchanges via policies.

See the `active_for/1`
[callback](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_exchange_decorator.erl#L70)
to understand which functions on the exchange would be decorated.

Take a look at the
[Sharding Plugin](https://github.com/rabbitmq/rabbitmq-sharding/blob/master/src/rabbit_sharding_exchange_decorator.erl)
and the
[Federation Plugin](https://github.com/rabbitmq/rabbitmq-federation/blob/master/src/rabbit_federation_exchange.erl)
to see how exchange decorators are implemented.
