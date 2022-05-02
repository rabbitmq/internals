This file attempts to document the overall structure of a queue, and
how persistence works.

Each queue is a [gen_server2 Erlang process](https://learnyousomeerlang.com/clients-and-servers). The usual pattern of the API and
implementation being in one file is not applied; `rabbit_amqqueue` is
the API (a module) and `rabbit_amqqueue_process` is the implementation (a `gen_server2`).

Startup
-------

The queue's supervisor initially starts the process as
rabbit_prequeue. This is a gen_server which determines whether the
process is an HA slave or a regular queue or master (see HA
documentation), and if so whether it is starting afresh or needs to
recover. This then uses the gen_server2 "become" mechanism to become
the correct sort of process - for this document we'll deal with
rabbit_amqqueue_process for regular queues.

The queue process decides for itself what it should be (rather than
having some library function that starts different types of processes)
so that it can do the right thing if it crashes and is restarted by
the supervisor - it might have been started as a master but need to
restart as a slave after crashing, for example. Or vice-versa.

Sub-modules
-----------

The queue process probably has the most code running in it of any
process; the rabbit_amqqueue_process has had various subsystems broken
out of it into separate modules over the years. The most major such
break-out is the queue implementation API, described by the
rabbit_backing_queue behaviour.

The aim of the code within rabbit_amqqueue_process is therefore mainly
to take the abstract queue implementation and make it support AMQPish
features, by handling consumers, implementing features like TTL and max
length in terms of lower level APIs, and coordinating everything.

Recently all the consumer-handling code was moved into
rabbit_queue_consumers.

rabbit_backing_queue
--------------------

The behaviour rabbit_backing_queue (BQ) implements a Rabbit-ish queue
with persistence and so on. The module rabbit_variable_queue (VQ) is
the major implementation of this behaviour.

This split was introduced with the "new" persister in 2.0.0. At the
time this was done so the old persister could be offered as a backup
(rabbit_invariable_queue) if serious bugs were found in the new
implementation. rabbit_invariable_queue is long gone but the mechanism
to configure an alternate module is still there. At various times
there have been proposals to provide alternate queue implementations
(using Mnesia, SQL etc) but this never came to anything. (One
rationale for optional e.g. SQL-based queues is that they would make
queue-browsing, atomic transactions and so on trivial, at the cost of
performance.)

The BQ behaviour had a secondary use that has turned out to be
important - it provides an API where we can insert a proxy to modify
how the queue behaves by intercepting calls and deferring to
VQ. Currently there are two such proxies: rabbit_mirror_queue_master
(see HA documentation) and rabbit_priority_queue (which implements
priority queues by providing one BQ implemented in terms of several
BQs.

rabbit_variable_queue
---------------------

So this is the meat of the queue implementation. This implements a
queue in terms of various sub-queues, with various degrees of
paged-out-ness.

publish -> [q1 -> q2 -> delta -> q3 -> q4] -> consumer

q1 and q4 contain "alpha" messages, meaning messages are entirely
within RAM. q2 and q3 contain "beta" and "gamma" messages, meaning
they have metadata in RAM (message ID, position etc) and contents on
disk. Finally, delta messages are on disk only. Many of the subqueues
can be empty so that messages do not need to pass through all states
if the queue is short.

The essay at the top of rabbit_variable_queue goes into a great deal
more detail on this.

Most of the complexity of VQ deals with moving messages between the
various queues in an optimised way. The actual persistence is handled
by rabbit_queue_index (QI) and rabbit_msg_store.

rabbit_queue_index
------------------

QI contains metadata that needs to be held per queue even if one
message is published to multiple queues - publication records with a
small amount of metadata, and delivery / acknowledgement record. In
3.5.0 the QI was extended to directly handle persistence of tiny
messages to improve performance by reducing the number of I/O ops we
do. The QI exists as "segment" files containing a log of the actions
which have taken place for an ordered segment (i.e. part) of the
queue, and an out of order journal which we write to any time anything
happens. Again, see the module for much more detail.

Note that everything as far as this part is within the main queue
process.

rabbit_msg_store
----------------

#### The following note applies to versions prior to 3.7

-----------------

There are also two msg_store processes per broker - one for transient
messages and one for persistent ones (the transient one can be deleted at startup).

-----------------

Since version 3.7 message stores are organised according to
[per-vhost message store](#per-vhost-message-store)

-----------------

The msg_store is a disk-based reference-counting key-value store,
storing messages in log-structured files. Again, see its module for
more details.

If one message is published to multiple queues, they will all submit
it to the message store, and the store will detect the non-first
requests to record the message and just increment the reference count.

The message store is designed to allow clients (i.e. queues) to read
from store files directly without calling into the message store
process. Only writes go via the process. There are a number of shared
ETS tables to coordinate what's going on.

We go to some effort to avoid unnecessary work. For example, the
message store maintains a table of "flying writes" - writes which have
been submitted by queues but not yet actioned. If a request to delete
a message is enqueued before the message is actually written, the
write is cancelled.

The message store needs an index, from message-id to {file, offset,
etc}. This is also pluggable. The default index is implemented in ETS
and so each message has an in-memory cost.

Message store index also contains reference-counters for messages
and serves as a synchronization point between queues, message store process
and GC process. Message store inserts new entries to the index and updates
reference-counters, GC prcess updates file locations and removes entries
using `delete_object`, queue processes only read entries.

Reference-counter updates, file location updates and deletes from the index
should be atomic.

Message store logic assumes that lookup operations for non-existent message
locations (if message is not yet written to file) are cheap.

See the [message store index behaviour module](https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit_common/src/rabbit_msg_store_index.erl) for more details.

The message store also needs to be garbage collected. There's an extra
process for GC (so that GC can lock some files and the message store
can concurrently serve from the rest). Within the message store, "GC"
boils down to combining together two files, both of which are known to
have over 50% messages where the ref count has gone to 0. See the
`rabbit_msg_store_gc` module for more details on how that works.


Per-vhost message store
------------------------

*Per-vhost message store was introduced in version 3.7*

### Process structure

Since version 3.7 queues and message stores processes are grouped in
supervision trees per-vhost.

The goal here is to isolate processes managing data (like queues and message stores)
on different vhosts from each other.
So when there is an issue in one vhost, others can function without interruptions.
Vhosts that experienced errors can restart and recover their data or stay "down"
for some time until an operator intervene and fix the error.

The data directories are also isolated per-vhost. Each vhost has its own data
directory with all the queues and message stores in it.

The supervision tree for two vhosts and two queues per vhost would look like:

```

rabbit_sup
|
|
--- ...
|
|
--- rabbit_vhost_sup_sup
    |
    |
    --- <rabbit_vhost_sup_wrapper> - supervision tree for vhost_1
    |   |
    |   |
    |   --- <rabbit_vhost_process>
    |   |
    |   |
    |   --- <rabbit_vhost_sup>
    |       |
    |       |
    |       --- <rabbit_recovery_terms>
    |       |
    |       |
    |       --- <rabbit_msg_store> - persistent message store for vhost_1
    |       |
    |       |
    |       --- <rabbit_msg_store> - transient message store for vhost_1
    |       |
    |       |
    |       --- <rabbit_amqqueue_sup_sup> - supervisor to contain queues for vhost_1
    |           |
    |           |
    |           --- <rabbit_amqqueue_sup> - vhost_1/queue_1 supervisor
    |           |   |
    |           |   |
    |           |   <rabbit_amqqueue_process/rabbit_mirror_queue_slave> - vhost_1/queue_1 process
    |           |
    |           |
    |           --- <rabbit_amqqueue_sup> - vhost_1/queue_2 supervisor
    |               |
    |               |
    |               <rabbit_amqqueue_process/rabbit_mirror_queue_slave> - vhost_1/queue_2 process
    |
    |
    --- <rabbit_vhost_sup_wrapper> - supervision tree for vhost_2
        |
        |
        --- <rabbit_vhost_process>
        |
        |
        --- <rabbit_vhost_sup>
            |
            |
            --- <rabbit_recovery_terms>
            |
            |
            --- <rabbit_msg_store> - persistent message store for vhost_2
            |
            |
            --- <rabbit_msg_store> - transient message store for vhost_2
            |
            |
            --- <rabbit_amqqueue_sup_sup> - supervisor to contain queues for vhost_2
                |
                |
                --- <rabbit_amqqueue_sup> - vhost_1/queue_1 supervisor
                |   |
                |   |
                |   <rabbit_amqqueue_process/rabbit_mirror_queue_slave> - vhost_1/queue_1 process
                |
                |
                --- <rabbit_amqqueue_sup> - vhost_1/queue_2 supervisor
                    |
                    |
                    <rabbit_amqqueue_process/rabbit_mirror_queue_slave> - vhost_1/queue_2 process

```
Processes given in `<angle brackets>` are not registered. Names represent controlling modules.

As you can see, each vhost has it's own pair of message stores and all the vhost
queue processes are grouped in the vhost queues supervisor (`rabbit_amqqueue_sup_sup`).

#### Recovery

If a queue process fails, it can be restored without impacting other queues.

If a message store fails, the entire vhost message store will be restarted,
including both message stores and all the vhost queues.
This is because of callback based publish acknowledgements, if a message store
restarts and queue processes keep going, some messages can never
be acknowledged.

Vhost restart process follows same recovery steps as when a node starts.

#### More about vhost processes and modules

##### rabbit_vhost_sup_sup
--------------------------

A `simple_one_for_one` supervisor. Serves as a container for vhosts.
Has an API for starting and stopping vhost supervisors, retrieving a vhost supervisor
by name, and checking if a vhost is alive.

Also manages an ETS table, containing an index of vhost processes.

The module is aware of the `vhost_restart_strategy` setting, which controls if a single
vhost failure and inability to restart should take down the entire node.

If the `rabbit_vhost_sup_sup` supervisor crashes - the node will be shut down.


##### rabbit_vhost_sup_wrapper
------------------------------

An intermediate supervisor to control vhost restarts.
It allows several restarts (3 in 5 minutes).
3 restarts - to handle failures in both message stores,
5 minutes - so if there is a data corruption error, there is enough time to get
the error during recover, so the supervisor will not retry recoveries forever.

After max restarts it gives up with `shutdown` message, which can be interpreted
by the `rabbit_vhost_sup_sup` supervisor according to configured `vhost_restart_strategy`.

The wrapper makes sure that `rabbit_vhost_sup` is started before recovery process
and is empty, because recovery process will dynamically add children to `rabbit_vhost_sup`.

Should this process fail, the vhost will not be restarted. If an exit signal is
not `normal` or `shutdown`, the `rabbit_vhost_sup_sup` process will crash
which will take down the node.


##### rabbit_vhost_process
--------------------------

An entity process for a vhost. It manages the vhost recovery process on start and
notifies that vhost is down on terminate.

The aliveness status of this process is used to check that the vhost is "alive".

This process will also terminate the vhost supervision tree if the vhost is deleted
from the database.


##### rabbit_vhost_sup
----------------------

A container supervisor for a vhost data store processes, such as message stores,
queues and recovery terms.

The restart strategy is `one_for_all`, which will restart the vhost should any
message store process fail. This will restart all the vhost queues.

Should this process crash, the vhost will be restarted (up to 3 times in 5 minutes)
using recovery process.

### Data storage

Each vhost data is stored in a separate directory.
The directory name for a vhost is `<mnesia_dir>/msg_stores/vhosts/<vhost_hash>`,
where `<mnesia_dir>` is a configured RabbitMQ data directory (`RABBITMQ_MNESIA_DIR` variable)
and `<vhost_hash>` is a hash of the vhost name. The hash is used to comply with
file name restrictions.

A vhost name hash can be generated using the `rabbit_vhost:dir/1` function.

A vhost directory path can be generated using the `rabbit_vhost:msg_store_dir_path/1` function.

Each vhost directory contains all its message stores and queues directories.

Example directory structure of a message store (with one vhost for simplicity):

```
mnesia_dir
|
|
--- ...
|
|
--- msg_stores
    |
    |
    --- vhosts
        |
        |
        --- <vhost_hash>
            |
            |
            --- .vhost - a file, containing the vhost name
            |
            |
            --- recovery.dets
            |
            |
            --- msg_store_persistent - persistent message store
            |   |
            |   |
            |   --- ... - the message store data files
            |
            |
            --- msg_store_transient - transient message store
            |   |
            |   |
            |   --- ...
            |
            |
            --- queues
                |
                |
                --- <queue_name_hash>
                |   |
                |   |
                |   --- .queue_name - a file, containing the vhost and the queue name
                |   |
                |   |
                |   --- ... - the queue data files
                |
                |
                --- <queue_name_hash>
                    |
                    |
                    --- .queue_name
                    |
                    |
                    --- ...
```

Each vhost directory contains `.vhost` file, with a name of the vhost. The file
can be used for troubleshooting, when the RabbitMQ node cannot be used to
generate the vhost directory name.

Each vhost has it's own recovery DETS table.

Queue directory names are also generated using a hash function.

Each queue directory contains a `.queue_name` file with the queue and the vhost names.
