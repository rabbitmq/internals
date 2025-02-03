Original: https://github.com/videlalvaro/rabbit-internals/blob/master/rabbit_boot_process.md

## RabbitMQ Boot Process ##

RabbitMQ is designed as an Erlang/OTP application which means that during start up it will be initialized as such. The function `rabbit:start/2` will be called which lives in the file `rabbit.erl` where the [application behaviour](http://erlang.org/doc/apps/kernel/application.html#Module:start-2) is implemented.

When RabbitMQ starts running it goes through a series of what are called __boot steps__ that take care of initializing all the core components of the broker in a specific order. The whole boot step concept is –as far as I can tell– something unique to RabbitMQ. The idea behind it is that each subsystem that forms part of RabbitMQ as a whole will declare on which other systems it depends on and if it's successfully started, which other systems it will enable. For example, there's no point in accepting client connections if the layer that routes messages to queues is not enabled.

The implementation is very elegant, it relies on adding custom attributes to erlang modules that declare how to start a boot step, in which boot steps it depends on and which boot steps it will enable, here's an example:

    -rabbit_boot_step({recovery,
                       [{description, "exchange, queue and binding recovery"},
                        {mfa,         {rabbit, recover, []}},
                        {requires,    empty_db_check},
                        {enables,     routing_ready}]}).

Here the step name is `recovery`, which as the description says it manages _"exchange, queue and binding recovery"_. It requires the `empty_db_check` boot step and enables the `routing_ready` boot step. As you can see, there's a `mfa` argument which specifies the `Module` `Function` and `Arguments` to call in order to start this boot step.

So far this seems very simple and even can make us doubt of the usefulness of such approach: why is there a need for boot steps at all? Why there isn't just a call to functions one after the other and that's it? Well, it is not that simple.

Boot steps can be separated into groups. A group of boot steps will enabled certain other group. For example `routing_ready` is actually enabled by many others boot steps, not just `recovery`. One of such steps is the `empty_db_check` that ensures that the Mnesia, Erlang's built-in distributed database, has the default data, like the default `guest` user for example. Also we can see that the `recovery` boot step also depends on `empty_db_check` so this logic takes care of running them in the right order that will satisfy the interdependencies they have.

There are boot steps that don't enable nor require others to be run. They are used to signal that a group of boot steps have happened as a whole, so the next group can start running. For example we have the external infrastructure step:

    {external_infrastructure,
        [{description,"external infrastructure ready"}]}

As you see it lacks the `requires` and the `enables` properties. But since many steps declare that their enable it, then `external_infrastructure` won't be run until those steps are run. Also many steps that come after in the chain require `external_infrastructure` to have run before, so they won't be started either until it had been processed.

But the story doesn't ends here. RabbitMQ can be extended with plugins that add new exchanges or authentication methods to the broker. Taking the exchanges as an example, each exchange type is registered into the broker via the `rabbit_registry` module, that means the `rabbit_registry` has to be started __before__ we can register a plugin. If we want to add new exchanges we don't have to worry about when they will be started by the broker, neither we have to care of managing the functional dependencies of our exchange. We just add a `-rabbit_boot_step` declaration to our exchange module where we say that our custom exchange depends on `rabbit_registry` et voilà, the exchange will be ready to use.

There's more to it too. In the same way your custom exchange can add their own boot steps to hook up into the server boot process, you can add extra boot steps that perform some stuff in between of RabbitMQ's predefined boot steps. Keep in mind that you have to know what you are doing if you are going to plug into RabbitMQ boot process.

Now, if you have been doing some Erlang programming you may be wondering at this point how does this even work at all. Erlang modules can have attributes, like the list of exported functions, or the declaration of which behaviour is implemented by the module, but there's no where a mention in the Erlang documentation about `boot_steps` and of course there's nothing about `-rabbit_boot_steps`. How do they work then?

When the broker is starting it builds a list of all the modules defined in the loaded applications. Once the list of modules is ready it's scanned for attributes called `rabbit_boot_steps`. If there are any, they are added to a new list. This list is further processed and converted into an [directed acyclic graph](http://en.wikipedia.org/wiki/Directed_acyclic_graph) which is used to maintain an order between the boot steps, that is the boot steps are ordered according to their dependencies. Here is where I think relies the elegance of this solution: add declarations to modules in the form of custom module attributes, scan for them and do something smart with the information. This speaks about the flexibility of Erlang as a language.

## Individual boot steps in detail ##

Here's a graphic that shows the boot steps and their interconnections. An arrow from boot step __A__ to boot step __B__ means that __A__ enables __B__. A line with no arrows on both ends from __A__ to __B__ means that __A__ is required by __B__. You can open the image file in a separate window to see it [full size](http://github.com/videlalvaro/rabbit-internals/raw/master/images/boot_steps.png).

![demo](http://github.com/videlalvaro/rabbit-internals/raw/master/images/boot_steps.png "Rabbit Boot Steps")

As we can see there the boot steps are somehow grouped. All starts at the `pre_boot` step continues at the `external_infrastructure` step and so on. Between `pre_boot` and `external_infrastructure` other steps occur that contribute to enable `external_infrastructure`. Now let's give a brief description of what happens on each of them.

### pre_boot ###

The `pre_boot` signals the start of the boot process. After it happens RabbitMQ will start processing the other boot steps like `file_handle_cache`.

### external_infrastructure ###

The `file_handle_cache` is used to manage file handles to synchronize reads and writes to them. See `file_handle_cache.erl` for an in depth explanation of its purpose.

The next step that starts is the `worker_pool`. The worker pool process manages a pool of up to `N` number of workers where `N` is the return of `erlang:system_info(schedulers)`. It's used to parallelize function calls across the pool.

Then the turn goes to the `database` step. This one is used to prepare the [Mnesia](http://www.erlang.org/doc/man/mnesia.html) database which is used by RabbitMQ to track exchanges meta information, users, vhosts, bindings, etc.

The `codec_correctness_check` is used to ensure that the AMQP binary generator is working properly, that is, that it will generate the right protocol frames.

Once all the previous steps have run then the `external_infrastructure` step will be processed signaling the boot process that it can continue with the following steps.

### kernel_ready ###

Once the external infrastructure is ready RabbitMQ will proceed with booting its own kernel. The first step will be the `rabbit_registry` which keeps a registry of plugins and their modules. For example it maps authentication mechanisms to modules with the actual implementation. The same thing is done from _exchange type_ to _exchange type implementation_. This means that if a message is published to an exchange of type _direct_ the registry will be responsible of telling the broker where the routing logic for the direct exchange resides, returning the module name.

After the `rabbit_registry` is ready, it's time to start the authentication modules. RabbitMQ will go through each of them, starting them and making them available. Some steps here are `rabbit_auth_mechanism_amqplain`, `rabbit_auth_mechanism_plain` and so on. If there's a plugin implementing an authentication mechanism, then it will be started at this point.

The next step is the `rabbit_event` which handles event notification for statistics collection. For example when a new channel is created, then a notification like `rabbit_event:notify(channel_created, infos(?CREATION_EVENT_KEYS, State))` is fired.

Then is time for the `rabbit_log` to start which manages the logging inside RabbitMQ. This process will delegate logging calls to the native error_logger module.

The same procedure used to enable the authentication mechanism is now repeated for the exchanges. Steps like `rabbit_exchange_type_direct` or `rabbit_exchange_type_fanout` are executed here. If you installed plugins with custom exchange types, they will be registered at this point.

Now is time to run the `kernel_ready` step in order to continue initializing the core of RabbitMQ.

### core_initialized ###

The first step of this group is the `rabbit_alarm` which starts the memory alarm handler. It will perform alarm management for different events that may happen during the broker life. For example if the memory is about to surpass the `memory_high_watermar` setting, then this module will fire an event.

Next is the `rabbit_node_monitor` which notifies other nodes in the cluster about its own node presence. It also takes cares of dealing with the situation of other node dying.

Then is the turn of the `delegate_sup` step. This supervisor will start a pool of children that will be used to parallelize calls to processes. For example when routing messages, the delegates take care of sending the messages to each of the queues that ought to receive the message.

The next step to be started is the `guid_generator` which as its name implies is used as a _Globally Unique Identifier Server_. This process is called for example when the server needs to generate random queue names, or consumer tags, etc.

Next on the list is the `rabbit_memory_monitor` which monitors queues memory usage. It will take care of flushing messages to disk when a queue reaches certain level of memory.

Finally the `core_initialized` step will be run and the boot step process will continue with the routing infrastructure.

### routing_ready ###

At this stage RabbitMQ will start to fill up the Mnesia tables with information regarding the exchanges, routing and bindings. In order to do so first the step `empty_db_check` is run. This step will check that the database has the required information inside else it will insert it. At this point the default `guest` user will be created.

Once the database is properly setup the `recovery` step is run. This step will restart recover the bindings between queues and exchanges. At this point is where the actual queue processes are started.

After the queues are running the new boot steps that involve the mirrored queues will be called. Once the mirrored queues are ready the `routing_ready` step will take part and the boot step procedure will continue.

### log_relay ###

Before RabbitMQ is ready to start accepting clients is time to start the `rabbit_error_logger` which is done during the `log_relay` boot step and from here the `networking` will be ready to run.

### networking ###

The `networking` will start all the supervisors that are concerned with the different listeners specified in the application configuration. A `tcp_listener_sup` will be started for each interface/port combination in which RabbitMQ is listening to. The SSL listeners will be started and the tcp client will be ready to accept connections.

### direct_client ###

RabbitMQ is nearly done with the boot process. The `direct_client` step is used to start the supervisor tree that takes cares of accepting _direct client connections_. The direct client is used for AMQP connections that use the Erlang distribution protocol. Once this is finished is time to proceed to the final step.

### notify_cluster ###

At this point RabbitMQ is ready to start munching messages. The only thing that remains to do is to notify other nodes in the cluster of it's own presence. That is accomplished via the `notify_cluster` step.

## Summary ##

If you read this far you can see that starting an application like RabbitMQ is not an easy task. Thanks to the __boot steps__ technique the process can be managed in such a way that the interdependencies between processes can be satisfied without sacrificing sanity. What's even more impressive is that this technique can be used to extend the broker in a way that goes beyond what the original developers planed for the server.
