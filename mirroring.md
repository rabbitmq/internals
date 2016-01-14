The essay at the top of rabbit_mirror_queue_coordinator has quite a
decent overview of how mirroring works. In order to avoid repetition,
this will aim to be an even higher-level summary.

How mirroring works
-------------------

In very quick terms: the master is a rabbit_backing_queue (BQ)
implementation "between" the rabbit_amqqueue_process and
rabbit_variable_queue (VQ), while the slave is a full process
implementing most of a queue (again in terms of VQ/BQ). They
communicate via GM. Since the master can't receive messages in its own
right there is also an associated coordinator process.

See rabbit_mirror_queue_coordinator for much more.

How mirroring is controlled
---------------------------

Policies call into the queue to tell them their policy has changed,
which calls into rabbit_mirror_queue_misc to update mirrors. Each
mirroring mode is an implementation of the behaviour
rabbit_mirror_queue_mode - rmq_misc selects the appropriate rmq_mode,
asks it which nodes should have slaves, and starts and stops slaves as
appropriate.

Eager synchronisation
---------------------

The master and all slaves need to come to a halt while synchronising:
we assume that handling publishes, deliveries or acks while
synchronisation is ongoing is too hard. Therefore although the master
and slaves are gen_servers, they essentially go into
manually-implemented selective "receive" loops while syncing, only
responding to a small set of messages and letting others back up -
flow control will typically stop publishers from publishing too
much. While syncing, the processes do respond to info requests and
emit info messages periodically, so that rabbitmqctl and management do
not become unresponsive and outdated respectively, but otherwise they
are dead to the world.

Because of this, we need to take care not to interfere with the state
of the master too much - leaving it with a different flow control
state or set of monitors than it entered the sync process with would
lead to subtle bugs. The master therefore spawns a local "syncer"
process which handles communication with the slaves to sync them.

See rabbit_mirror_queue_sync for more details on how exactly the sync
protocol works.

GM
--

The gm.erl module contains an essay on how GM works. Unfortunately
it's not easy to understand; a property which it shares with GM in
general.

The overall principle is fairly clear: GM processes form a ring,
around which messages are broadcast. Each message goes round twice,
once to publish and once to acknowledge. New members enter the ring at
any point; if a member dies the ring needs to heal and ensure that
messages which might have been lost are sent again.

The last part is tricky. Much of the complexity in GM is around
knowing which members have what knowledge, and how to bring members up
to speed if one fails.

Additionally, the fact that ring information travels through two
routes (around the ring itself, and through Mnesia) makes it even
harder to reason about.

Finally, GM is not at all designed to cope with partial network
partitions: if A is partitioned from B, then B can remove it from
Mnesia, and that information can leak back to A via C. We currently
don't handle this situation well; this is the biggest unsolved
problem in RabbitMQ.

It might be worth replacing GM altogether with something new.

If modifying it, be very conservative about even small changes.
