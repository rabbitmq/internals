# Variable Queue #

## Publishing messages ##

When a message is published to the queue, the first thing we have to
do is to determine if the message is persistent. We track this
information in the `msg_status` record:

```erlang
-record(msg_status,
        { seq_id,
          msg_id,
          msg,
          is_persistent,
          is_delivered,
          msg_in_store,
          index_on_disk,
          persist_to,
          msg_props
        }).
```

Message statuses are kept in a record where the `is_persistent` field
is set to `true` if the queue is durable and the message was published
as persistent:

```erlang
is_persistent = IsDurable andalso IsPersistent
```

If it was determined that the message needs persistence, then it will
be immediately written to disk, either to the message store or the 
queue index, depending on the message size (see
`queue_index_embed_msgs_below`).

Internally the `variable_queue` keeps messages on four `queue` data
structures. They are a variation of erlang's _queue_ module, but which
some extensions that allow getting the queue length in constant
time. These four queues are identified on the variable queue state as
`q1`, `q2`, `q3` and `q4`. The need for these four queues becomes
apparent once disk paging is taken into account.

`q4` keeps track of the oldest messages, that is, those at the front
of the queue, those that will be delivered earlier to consumers.

`q3` only has messages when there has been some disk paging due to
memory pressure, or if we have a queue that has recovered contents
from disk, due to a broker restart for instance. This means that
messages that once were in `q4` only, now have had their content
pushed to disk, and their references are now kept in `q3`. So when a
message arrives into the variable queue, we need to determine if the
message needs to be inserted at the back of `q4`, or somewhere else.

If `q3` is empty, this means we haven't paged queue contents to disk,
so the messages at the front of the queue are still in `q4`, and the
last message arriving to the queue is still in `q4` as well. So new
messages can be inserted at the back of `q4`. Now, if `q3` has
messages in it, this means at some point we have paged to disk, so
some messages that were at the rear of `q4` are in `q3` now. This
means a new message _can't_ be inserted into `q4`, otherwise we will
lose message ordering; therefore, if `q3` has messages, new messages
go into `q1`.

```erlang
case ?QUEUE:is_empty(Q3) of
    false -> State1 #vqstate { q1 = ?QUEUE:in(m(MsgStatus1), Q1) };
    true  -> State1 #vqstate { q4 = ?QUEUE:in(m(MsgStatus1), Q4) }
end,
```

## Fetching Messages ##

Messages are fetched by calling `queue_out/1`, which retrieves
messages from `q4` or, if `q4` is empty then they are retrieved from
`q3`.

For `q3` to have messages, it means that at some point messages were
paged to disk due to memory pressure which led to
`push_alphas_to_betas/2` being called. Another way for `q3` to get
messages is when we load messages from disk into memory, for example
when `maybe_deltas_to_betas/1` is called; this can happen when we are
recovering queue contents during queue initialization, or also when we
try to load more messages from disk so they can be delivered to
clients.

When there are no more messages in `q4`, `fetch_from_q3/1` is called
trying to obtain messages from `q3`. If `q3` is empty, then the queue
must be empty. Remember that if `q3` wasn't empty, then new messages
arriving into the queue were put into `q1`.

If `q3` wasn't empty, but the message fetched was the last one there,
we must see if we need to load more messages from disk, or if we need
to migrate `q1` messages into `q4`.

So let's say we fetched the last message from `q3` and we know there
are no more messages on disk (delta count = 0). If there were new
publishes while `q3` had messages, those messages are in `q4`, so we
need to move messages from `q1` into `q4`. Why? Remember that during
publishing messages are queued into `q4` when `q3` is empty, otherwise
they go into `q1`. Imagine that we were publishing messages into `q4`,
then at some point we had to `push_alphas_to_betas`, which means some
`q4` messages were moved into `q3`. Now that `q3` has some messages,
_new_ messages are put into `q1`, but from the point of view for an
external user of the backing queue, `q3` messages come first than
those in `q1`, i.e.: they are at the front of the queue. So when there
are no more messages in `q3`, we can start consuming those in `q1`,
but since `queue_out/1` only fetches messages from `q4`, we move `q1`
contents there.

Now let's say there were remaining messages on disk, instead of moving
messages from `q1` into `q4`, we have to load messages from disk into
`q3`. This is accomplished by calling `maybe_deltas_to_betas/1`.

`maybe_deltas_to_betas/1` reads the index and then depending on what
it finds there, it loads messages at the rear of `q3`. If there are no
more messages on disk, then those messages that are in `q2` are moved
to the rear of `q3`. When we look into message paging we will see why
only when there are no more paged messages, we move `q2` into `q3`,
and why `q2` messages go to the back of `q3`. For now keep in mind
that all this message shuffling is done to ensure message ordering
from an external observer's point of view. `q2` only has messages if
`q1` had messages before, and `q1` pushes messages to `q2` only when
some messages have been paged before, so whatever is on disk, comes
before than whatever is on `q2`.

Remember, messages in `q1` are recently published messages that went
there because `q3` had messages, so those are the last ones we should
deliver.

## Paging messages to disk ##

Disk paging starts when the function `reduce_memory_use/1` is
called. This function calls `push_alphas_to_betas/2` to start sending
messages to disk. _Alpha_ messages, are those messages where the
contents and the queue position are held on RAM, and we are trying to
convert them to _betas_, i.e.: messages where we still keep the
position of the message in RAM, but we send the contents to disk.

When paging to disk we try to first page those messages that are going
to be delivered later, those that from a client point of view are at
the rear of the queue, so we start with `q1`'s contents. If there are
messages on disk (because we have paged out queue contents already, or
because we didn't load all the queue contents in memory), then
messages are moved from `q1` into `q2`, otherwise they go into
`q3`. Then we move messages from `q4` into `q3`.

Keep in mind that we move messages based on a _quota_ that's
calculated taking into account how messages are in ram vs how many
messages `set_ram_duration_target/2` decided that need to be paged out
to disk. If after moving messages from _alpha_ into _beta_ this quota
hasn't been consumed entirely, we have to then push messages from
_beta_ to _delta_. _Delta_ messages are those messages where the
message content and their position are only held on disk.

To page to disk those messages whose position in the queue was still
on RAM we call `push_betas_to_deltas/2`. We first page messages from
`q3` into disk, and then we page messages from `q2`, but there's a
catch. Keep in mind that we might not want to page every single
message out to disk. `q3` holds messages that are in front of the
queue compared to those in `q2`, so `q3` messages are paged in reverse
order, that is, those messages at the rear of `q3` are sent to disk
first, with the idea that if the quota of messages that need paging is
reached, then we will keep in RAM messages that will be sent soon to
clients.

## Example ##

**publish msgs `[1, 2, 3, 4, 5, 6, 7, 8, 9]`**:

```
Q4: [1, 2, 3, 4, 5, 6, 7, 8, 9]

Q3: []

Q2: []

Q1: []

Delta: []
```

**publish msgs `[10]`**:

```
q4: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Q3: []

Q2: []

Q1: []

Delta: []
```

**push_alphas_to_betas**:

```
Q4: [1, 2, 3, 4, 5, 6, 7]

Q3: [8, 9, 10]

Q2: []

Q1: []

Delta: []
```

**publish msgs `[11, 12, 13, 14, 15]`**:

```
Q4: [1, 2, 3, 4, 5, 6, 7]

Q3: [8, 9, 10]

Q2: []

Q1: [11, 12, 13, 14, 15]

Delta: []
```

**push_alphas_to_betas**:

```
Q4: [1, 2, 3, 4]

Q3: [5, 6, 7, 8, 9, 10, 11, 12, 13]

Q2: []

Q1: [14, 15]

Delta: []
```

**push_betas_to_deltas**:

```
Q4: [1, 2, 3, 4]

Q3: [5, 6, 7, 8, 9]

Q2: []

Q1: [14, 15]

Delta: [10, 11, 12, 13]
```

**publish msgs `[16, 17, 18, 19, 20]`**:

```
Q4: [1, 2, 3, 4]

Q3: [5, 6, 7, 8, 9]

Q2: []

Q1: [14, 15, 16, 17, 18, 19, 20]

Delta: [10, 11, 12, 13]
```

**push_alphas_to_betas**:

```
Q4: [1]

Q3: [2, 3, 4, 5, 6, 7, 8, 9]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**fetch 3 messages**:

```
Q4: []

Q3: [4, 5, 6, 7, 8, 9]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**fetch 5 messages**:

```
Q4: []

Q3: [9]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**fetch 1 message**:

```
Q4: []

Q3: []

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**Q3 became empty, but we have msgs on disk/delta, so we call
maybe_deltas_to_betas to load messages from delta into Q3**:

```
Q4: []

Q3: [10, 11, 12, 13]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: []
```

**maybe_deltas_to_betas saw that now there are no more messages on
disk, so it joins Q3 with Q2, Q2 messages going to the rear of Q3**:

```
Q4: []

Q3: [10, 11, 12, 13, 14, 15, 16, 17]

Q2: []

Q1: [18, 19, 20]

Delta: []
```

**publish msgs `[21, 22, 23, 24, 25]`**:

```
Q4: []

Q3: [10, 11, 12, 13, 14, 15, 16, 17]

Q2: []

Q1: [18, 19, 20, 21, 22, 23, 24, 25]

Delta: []
```

**fetch 8 messages**:

```
Q4: []

Q3: []

Q2: []

Q1: [18, 19, 20, 21, 22, 23, 24, 25]

Delta: []
```

**Q3 is empty and Delta is empty as well, time to move Q1 messages into
Q4**:

```
Q4: [18, 19, 20, 21, 22, 23, 24, 25]

Q3: []

Q2: []

Q1: []

Delta: []
```

**fetch 1 message**:

```
Q4: [19, 20, 21, 22, 23, 24, 25]

Q3: []

Q2: []

Q1: []

Delta: []
```

and so on.
