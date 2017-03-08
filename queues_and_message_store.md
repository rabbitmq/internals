This file attempts to document the overall structure of a queue, and
how persistence works.

Each queue is a [gen_server2 Erlang process](http://learnyousomeerlang.com/clients-and-servers). The usual pattern of the API and
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

There are also two msg_store processes per broker - one for transient
messages and one for persistent ones (mainly so the transient one can
just be deleted at startup).

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

See message store index behaviour module for more details.

The message store also needs to be garbage collected. There's an extra
process for GC (so that GC can lock some files and the message store
can concurrently serve from the rest). Within the message store, "GC"
boils down to combining together two files, both of which are known to
have over 50% messages where the ref count has gone to 0. See
rabbit_message_store_gc for more details on how that works.
