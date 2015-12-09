# Backing Queue #

RabbitMQ supports plugable backing queues by modules implementing the
`rabbit_backing_queue` behaviour.

The backing queue `init/3` callback expects an `async_callback()`
parameter which is a `fun` callback which takes the backing queue
state, and returns a new state. Keep reading to understand what all
this callback mumbo-jumbo means.

**TL;DR:** due to the two problems explained below, this callback
  takes care of executing certain functions in the context of a
  particular Erlang process.

## Process Dictionary Problem ##

Understanding how this callback works is vital since the persistence
layer of the backing queue does heavy use of the _process dictionary_
and the use of `self()` to track who opened which file handle. What
this means is that even tho the the backing queue behaviour callbacks
seem to have the referential transparent property, they do not. Behind
the scenes, some of the backing queue behaviour callbacks will `put/get`
values to/from the process dictionary, but if one of said callbacks is
executed in a different process context, then those values won't be
found on the process dictionary, and everything else breaks havoc.

## File Handle Cache Problem ##

The same applies for the `file_handle_cache` tracking who owns which
file handle by calling `self()` inside its functions implementations
instead of expecting a `Pid` as parameter for example. The call of
`self()` again violates referential transparency. The function
behaviour now depends on the process context on which it's
called. This means that closing file handles must be done from the
same caller that issued the file open.

## How Things Work ##

The function `rabbit_amqqueue_process:bq_init/3` takes care of
initializing the backing queue implementation, whether it is the
`rabbit_variable_queue`, the `rabbit_mirror_queue_master`, the
`rabbit_priority_queue` or your own backing queue behaviour
implementation.

The async callback passed into `BQ:init` is defined as:

```erlang
fun (Mod, Fun) ->
        rabbit_amqqueue:run_backing_queue(Self, Mod, Fun)
end
```

This `fun` will take a module argument, which is usually an atom
referring to the backing queue module being used, for example
`rabbit_variable_queue`, or `rabbit_mirror_queue_master`. The second
argument expected by this callback is a `fun` that will be passed
along to `rabbit_amqqueue:run_backing_queue/3`. Now lets see what
`rabbit_amqqueue:run_backing_queue` does.

### rabbit_amqqueue:run_backing_queue ###

The function body is like this:

```erlang
run_backing_queue(QPid, Mod, Fun) ->
    gen_server2:cast(QPid, {run_backing_queue, Mod, Fun}).
```

It sends a `{run_backing_queue, Mod, Fun}` message to whatever process
was provided as `QPid`. **This is important**, since that process'
context is the one which will get its process dictionary modified
indirectly, and at the same time will own file handles when they are
opened by the _msg\_store_ for example.

Back to `rabbit_amqqueue_process` we will see that this module has a
callback for the message mentioned above:

```erlang
handle_cast({run_backing_queue, Mod, Fun},
            State = #q{backing_queue = BQ, backing_queue_state = BQS}) ->
    noreply(State#q{backing_queue_state = BQ:invoke(Mod, Fun, BQS)});
```

This function takes care of extracting the current `backing_queue`
module and `backing_queue_state` from its own process state, and then
calling `BQ:invoke(Mod, Fun, BQS)`.

This is what `BQ:invoke/3` does:


```erlang
invoke(?MODULE, Fun, State) -> Fun(?MODULE, State);
invoke(      _,   _, State) -> State.
```

Invoke's implementation is pretty simple, if the `Mod` argument
provided to it matches the current module, in this example
`rabbit_variable_queue`, then the `Fun` will be executed with
`rabbit_variable_queue` as first parameter and the backing queue
`State` as the second argument. To reiterate, what's important to
understand is that `Fun` will be executed in the context of whatever
`QPid` was referring to above. In the case we are analyzing so far,
this is the `rabbit_amqqueue_process` pid.

## What Fun ##

Now let's try to find out what `Fun` actually is. To get to this we
need to see how `rabbit_variable_queue` is initialized`.

`rabbit_variable_queue:init/6` will call into
`msg_store_client_init/3` passing our initial callback as the third
parameter (`msg_store_client_init/3` then expands into
`msg_store_client_init/4`). Let's refresh what that callback was:

```erlang
fun (Mod, Fun) ->
        rabbit_amqqueue:run_backing_queue(Self, Mod, Fun)
end
```

That callback will be now wrapped into yet another `fun` like this:

```erlang
fun () -> Callback(?MODULE, CloseFDsFun) end
```

To see that in context:

```erlang
msg_store_client_init(MsgStore, Ref, MsgOnDiskFun, Callback) ->
    CloseFDsFun = msg_store_close_fds_fun(MsgStore =:= ?PERSISTENT_MSG_STORE),
    rabbit_msg_store:client_init(MsgStore, Ref, MsgOnDiskFun,
                                 fun () -> Callback(?MODULE, CloseFDsFun) end).
```

So now we have a clue of what the `Fun` passed into our callback might
be. It is whatever `msg_store_close_fds_fun` returned as
`CloseFDsFun`. Let's check:

```erlang
msg_store_close_fds_fun(IsPersistent) ->
    fun (?MODULE, State = #vqstate { msg_store_clients = MSCState }) ->
            {ok, MSCState1} = msg_store_close_fds(MSCState, IsPersistent),
            State #vqstate { msg_store_clients = MSCState1 }
    end.
```

We get a `fun` that will only be executed if the `Mod` argument
matches, in this case `rabbit_variable_queue`. That `fun` takes as
second argument our `rabbit_variable_queue` state.

On `msg_store_client_init/4` above we said that our initial callback
gets wrapped like this:

```erlang
fun () -> Callback(?MODULE, CloseFDsFun) end
```

This means inside the msg_store, at various places, that `fun` closure
gets called without arguments which in turn calls our callback with
the `CloseFDsFun`. We end up with something like what's below after
some expansions:

```erlang
fun (rabbit_variable_queue, Fun) ->
        rabbit_amqqueue:run_backing_queue(QPid, rabbit_variable_queue,
            fun (?MODULE, State = #vqstate { msg_store_clients = MSCState }) ->
                {ok, MSCState1} = msg_store_close_fds(MSCState, IsPersistent),
                State #vqstate { msg_store_clients = MSCState1 }
            end)
end
```

So our `rabbit_amqqueue_process` will ask the backing queue module
to invoke that expanded fun in the context of the
`rabbit_amqqueue_process` Pid:

```erlang
handle_cast({run_backing_queue, Mod, Fun},
            State = #q{backing_queue = BQ, backing_queue_state = BQS}) ->
    noreply(State#q{backing_queue_state = BQ:invoke(Mod, Fun, BQS)});
```

This very same technique is used on `rabbit_variable_queue:init/3` to
setup the functions that will write messages to disk (see
`rabbit_variable_queue:msgs_written_to_disk/3`) and the ones that will
write the message indexes to disk (see
`rabbit_variable_queue:msg_indices_written_to_disk/2).

## It's About Context ##

From all these layers of indirection, what's important to understand
is that the `Pid` passed into `rabbit_amqqueue:run_backing_queue/3`
determines the context on which all the functions implementing message
persistence will be run. Unless your `rabbit_backing_queue` behaviour
implementation is just a proxy like that of `rabbit_priority_queue`,
you must take that `Pid` context into account, since it will hold file
handles references and its process dictionary will be the one where
the `file_handle_cache` will store its information.

If you want a second example of what we outlined above, take a look at
`rabbit_mirror_queue_slave:bq_init/3` where the Pid provided to
`run_backing_queue/3` in this case is the _slave_ Pid. The slave
process implements it's own `handle_cast({run_backing_queue, Mod,
Fun}, State)` function clause, on which funs from
`rabbit_variable_queue` like `msg_store_close_fds_fun`,
`msgs_written_to_disk`, `msg_indices_written_to_disk` and
`msgs_and_indices_written_to_disk` will be run.
