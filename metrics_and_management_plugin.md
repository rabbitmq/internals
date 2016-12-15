## Management Plugin internal architecture 3.6.7+

Since 3.6.7 the management plugin has been re-designed to spread the memory
used for statistics across the entire rabbit cluster instead of aggregating
it all in a single node. Doing this isn't free. There is a trade-off in
metric latency and processing for memory stability.


### Components

There are three main components:

* core metrics
* rabbitmq-management-agent
* rabbitmq-management



### Core metrics

Core metrics are implemented in the rabbitmq server itself consisting of
a set of of ETS tables storing either counters or proplists containing details
or metrics of various entities. The schema of each table is documented in
[rabbit_core_metrics.hrl](https://github.com/rabbitmq/rabbitmq-common/blob/master/include/rabbit_core_metrics.hrl)
in `rabbitmq-common`.

Mostly counters that are incremented in real-time as message interactions occur
in queues, channels, exchanges etc.

This replaces the previous approach of emitting events containing metrics
at regular intervals. `created` and `deleted` events are still emitted,
however `stats` events have been removed.

Because no unbounded queues are involved this approach should have fixed
memory overhead in relation to the number of active entities in the system.



### Management Agent


`rabbitmq-managment-agent` is responsible for turning core metrics into
data structures suitable for `rabbitmq-management` consumption.  This is
done on a per node basis. There are no inter-node communications involved.

The management agent runs a set of metrics collector processes. There is one
process per core metrics table. Each collector periodically read its associated
core metrics table and performs some table-specific processing which produces
new data points to be inserted into the management metrics tables (defined in
[rabbitmq_mgmt_metrics.hrl](https://github.com/rabbitmq/rabbitmq-management-agent/blob/master/include/rabbit_mgmt_metrics.hrl)).
The collection interval is determined by the smallest configured retention intervals.

In addition to the collector processes there is a garbage collection event
handler that handles the `delete` events emitted by the various processes to ensure
stats are completely cleared up. To make this efficient there is also a set of
index tables (see `rabbitmq_mgmt_metrics.hrl`) that allow the gc process to
remove all stats for a particular entity.

The management agent plugin also hosts the `rabbitmq_mgmt_external_stats` process
that periodically updates the core metrics tables with node specific stats.
Arguably this should be moved to the core at some point.

It is worth noting that the latency of metrics is now related to the retention
interval and is typically higher than the previous version.


#### exometer_slide

The [exometer_slide](https://github.com/rabbitmq/rabbitmq-management-agent/blob/master/src/exometer_slide.erl)
module is a key part of the management stats processing.
It allows us to reasonably efficiently store a sliding window of incoming metrics
and also perform various processing on this window. It was extracted from the
[exometer_core](https://github.com/Feuerlabs/exometer_core) project but has
since been heavily modified to fit our specific needs.

One notable addition is the "incremental" slide type that is used to aggregate
data from multiple sources. A typical example would be vhost message rates.


### Management API

The `rabbitmq-management` plugin is now mostly a fairly thin HTTP querying layer.

It also handles the distributed querying and stats merging logic. When a stats
request comes in the plugin contacts each node in parallel for a set of "raw"
stats (typically `exometer_slide` instances). It uses the [delegate](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/delegate.erl)
module for this and has it's own `delegate` supervision tree to avoid affecting
the one used for core rabbit delegations. Once stats for
each node has been collected it merges the data then proceeds with processing
this (for example turn sliding window data points into rates) for API
consumption. Most of this logic is implemented in the `rabbit_mgmt_db` module.

This distributed querying/merging is arguably the most complex part of the stats
system.


Because the information returned by the http API is fairly heavily augmented (e.g.
a request for a queue would also contain channel details) we often have to
perform multiple distributed queries in response to a stats request.
For example, to get the channel details for a queue we first have to fetch the
queue stats, inspect the consumers attached to that queue then query for the
channel details based on the consumer channel).


There are also inefficiencies when listing entities whose number could
be unbounded (queues, channels, exchanges and connections).
As we are able to sort on almost any stats including rates we always
need to fetch _all_ entity stats from each node, merge, sort then typically
return a smaller page of items to the API. For systems with lots of such
entities this can become very inefficient as potentially large amounts of data
need to travel between nodes for each request. Therefore all requests that can
return large numbers of entities go through an adaptive cache processes that adjusts
its cache expiry time based on how long it took to fetch all that data. This
should provide some degree of protection against excessive entity listings. It
would be prudent to reduce the frequency of these queries if at all possible
in heavily loaded systems.