# Queue Decorators #

Queue decorators are modules implemented as behaviours that can let
you extend queues. A decorator will have a set of callbacks that will
be called from the queue process in response to some events that might
happen at the queue, like when messages are delivered to consumers,
which might cause the list of active consumers to be updated.

They were added to the broker as a way to handle
[consumer priorities](https://www.rabbitmq.com/consumer-priority.html)
or by the federation plugin, to know when to move messages across
[federated queues](https://www.rabbitmq.com/federated-queues.html).

Decorators need to implement the `rabbit_queue_decorator`
[behaviour](https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/src/rabbit_queue_decorator.erl)
and are usually associated with queues via policies.

A Queue decorator can receive notifications of the following events:

- Queue Startup
- Queue Shutdown
- Consumer State Changed (active consumers)
- Queue Policy Changed
