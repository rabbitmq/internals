# Uninterrupted cluster upgrade between minor releases

## Status as of November 2016

Currently, when you want to upgrade RabbitMQ from a minor version to
the next minor version (eg. 3.5.x to 3.6.x), you need to shutdown the
entire cluster. A RabbitMQ cluster with a mix of multiple minor versions
is unsupported: that's when we introduce incompatible changes, in
particular to the Mnesia schema.

The same rules apply for upgrades between major versions.

On rare occasions, like between 3.6.5 and 3.6.6, we need to import a
breaking change and again, you must shutdown the entire cluster to
upgrade.

## Plan to fix the situation

### Scope

This project targets the following goals:

1. Being able to run any minor versions from the same major branch mixed
   in the same cluster.
2. Being able to gradually upgrade all nodes of a cluster to a later
   minor version and still benefit from the new features and bugfixes at
   the end of the process.

The first item is a prerequisite to the second item.

This project does not try to make upgrades between major versions
possible without terminating a cluster: thus, breaking changes requiring
a cluster shutdown may happen even once this project is complete.

### Compatibility between brokers

#### Elements to consider

To be able to run different versions of RabbitMQ inside a single
cluster, all nodes must understand and emit data in a common format.

Here are the elements shared or exchanged by nodes:

* record definitions; They are the building blocks of inter-process
  communication and the Mnesia schema below;
* the Mnesia schema;
* messages exchanged between nodes;
* plugins ABI.

Today, we can't achieve compatibility because when we need to eg.
expand a record, we add a new field, which makes it incompatible with
a previous version of that record. A plugin must even be recompiled
against the new record definition to work again.

Therefore, records and general messages must be redesigned to be
extensible without breaking compatibility.

#### Extensible and backward-compatible records

We need to rethink how our records are designed to:

* allow modifications of a record without breaking a table schema
  (if there is one associated);
* permit the use of old and new records inside old and new code.

We can use an Erlang map inside a record to allow extensibility,
yet retaining backward compatibility: new keys would hold new
informations, while old key could still exist. And using maps
still allows pattern matching.

Here is an example with the `#amqqueue` record from the 3.6.x branch,
focused on the lists of queue slaves:

```erlang
#amqqueue{
  name,
  slaves = [],
  sync_slaves = []
}.
```

In 3.7.x, we add another list related to queue slaves:

```erlang
#amqqueue{
  name,
  slaves = [],
  sync_slaves = [],
  slaves_pending_shutdown = []
}.
```

And we already know we may need yet another list to track slaves pending
startup.

Instead, we could have used an extensible record in the 3.6.x branch of the form:

```erlang
#amqqueue{
  name,
  features = #{
    slaves_list => #{ % The value could be anything: a record, a map, ...
      slaves => [],
      sync_slaves => []
    }
  }
}.
```

And in 3.7.x, it could have been extended like this:

```erlang
#amqqueue{
  name,
  features = #{
    slaves_list_v2 => #{
      slaves => [],
      sync_slaves => [],
      slaves_pending_shutdown => []
    }
    % 'slaves_list' could exist, if the record was converted for
    % instance, but would have a lower precedence.
  }
}.
```

In 3.6.x, the code would look for `slaves_list` in the `features` map.
In 3.7.x, the code would look for `slaves_list_v2` and fallback on
`slaves_list` if the former key is missing (meaning it's an older
record):

```erlang
do_things(#amqqueue{features = #{slaves_list_v2 := Slaves}}) ->
    % Do things with the new format of slaves list.
    really_do_things(Slaves);
do_things(#amqqueue{features = #{slaves_list := Slaves}}) ->
    % Do things with the old format of slaves list.
    really_do_things(Slaves).
```

In the end, the record is unchanged and thus, the Mnesia table schema
remains the same.

#### Feature flags

Because we want to have different versions of RabbitMQ in the same
cluster, new nodes must not produce new records while older nodes are
still around.

Just looking at the version of running nodes is not enough either
because there could be stopped nodes. Furthermore, if new code is
backported to an older release for whatever reason, a new record could
be supported by a non-contiguous set of versions.

Other projects such as ZFS resolve that by using *feature flags*. A
given version of ZFS code has support for a certain list of features and
a filesystem has a list of features enabled. A ZFS implementation can
look at the features enabled on a particular filesystem:

* If a feature supported by the implementation is disabled, it continues
  to use the old format when writing data.
* If a feature enabled on the filesystem is not supported by the
  implementation, it refuses to mount the filesystem.

The user is responsible for enabling features when he is sure he won't
have to mount a filesystem with an older implementation.

We can use the same principle with RabbitMQ. Each version comes with a
list of supported "features" and the list of enabled features is stored
in Mnesia.

When a new node starts, it looks at the enabled features. If it doesn't
support one of them, it refuses to boot. If it supports more features
than enabled, it makes sure to never produce data which would rely on
disabled features because other nodes might not support that or the user
may want to rollback to an older version.

The user can enable new features when the cluster is ready. All nodes
must support the new feature to allow it to be enabled. If that is not
the case, the feature is not enabled.

Once new features are enabled, nodes can produce newer data. At this
point, it means old and new data are in flight (eg. in queues in
memory). That's why the code must still support both formats/messages.

If we take the same `#amqqueue` record example:

* RabbitMQ 3.6.x would have the following feature flags:

 ```erlang
SuportedFeatures = [
      amqqueue_slave_list
].
```

* RabbitMQ 3.7.x would have the following feature flags:

 ```erlang
SuppoertedFeatures = [
      amqqueue_slave_list,
      amqqueue_slave_list_v2
].
```

* Initially, only the `amqqueue_slave_list` feature would be enabled,
  while the cluster is running RabbitMQ 3.6.x.

After RabbitMQ is upgraded from 3.6.x to 3.7.0, it would not produce
`#amqqueue` records with the `slave_list_v2` map entry yet. It would
continue to use the old `slave_list` map entry. However, once the
feature is enabled, it would produce the new entry, while still keeping
support for the old entry which might still be in flight.

#### When to get rid of old data support

The RabbitMQ code must keep old code around for the entire life of the
major branch.

In the next major branch, we may remove old code because mixing major
versions remains unsupported.

However, all features must remain in the list of supported features,
even if the code to handle them disappeared. This is because enabled
features in an existing cluster must be present in the list of supported
features.

#### Note about the performance

Checking feature flags in Mnesia for every operations would be
expensive. A possible solution is to cache the information in the
process, either in the state or the dictionary. Then once an operator
decides to enable features, processes could be notified so they refresh
their cached list of enabled features.

### Upgrading a cluster

With backward-compatible code, extensible records and feature flags in
place, upgrading a cluster would consist of:

1. Installing the new version of RabbitMQ on all nodes. This means:

 * stop the broker
 * install the new version
 * restart the broker

 At this point, no new feature flag is enabled. The user benefits from
 the changes which do not depend on new features only. He can decide to
 rollback to a previous version of RabbitMQ.

2. Once all nodes are running the latest code, the user can enable new
 features.

 He can decide to enable all of them at once or just one. This is useful
 if he hits a problem fixed by a particular feature and wants to verify
 the issue is actually solved. This can be useful to us too during
 development.

 Brokers produce and exchange new records, while still handling old
 ones. Rollback is not possible anymore.

### Working with breaking changes

#### From a developer point of view

When working on the core of RabbitMQ or a plugin, wether it is a tier-1
plugin or not, a developer must be rigourous about changing shared and
exchanged data format. When he wants to introduce a breaking change, he
will have to:

* add and document a new feature flag;
* update the code so old and new formats are both handled.

The code snippets above give an overview of that.

Specifically for plugins, they may come with a list of feature flags
they require to run. This could be in addition to or instead of the
RabbitMQ version check.

> **TODO:** The implementation details remain to be designed.

#### From an operator point of view

An operator will need commands to manage features flags during a cluster
upgrade:

* list feature flags;
* get information about a feature flag;
* enable one, many or all feature flags.

When listing feature flags or querying informations about them, the
following elements will be of interest:
* the name of the feature;
* a description of the change;
* is the feature is enabled;
* (optional) a list of RabbitMQ versions where the feature flag was
  introduced;

> **TODO:** The implementation details remain to be designed.

## Future out-of-scope ideas

### Downgrading a node

Downgrading a node means that features not supported by the targetted old
version must be disabled. Disabling a feature means that:

* Nodes needs to produce old record again.
* New records still in flight need to be converted back to their old
  version.

The latter point would need code to find in-flight data and convert it.
Once this is done, the old version of RabbitMQ can be deployed exactly
like the new one was installed. Thus the order of operations is simply
reversed.

Obivously the difficulty is in the "find and convert data" part. This
would be impossible for certain features.
