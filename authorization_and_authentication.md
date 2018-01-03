# Authorization and Authentication Backends

This document describes authentication and authorization machinery that
implements [access control](https://www.rabbitmq.com/access-control.html).

Authentication backends should not be confused with authentication mechanisms,
which are defined in some protocols supported by RabbitMQ.
For AMQP 0-9-1 authentication mechanisms, see [documentation](http://www.rabbitmq.com/authentication.html).

## Definitions

Authentication and authorization are often confused or used interchangeably. That's
wrong and RabbitMQ separates the two. For the sake of simplicity, we'll define
authentication as "identifying who the user is" and authorization as
"determining what the user is and isn't allowed to do."


## Authentication Mechanisms

AMQP 0-9-1 supports multiple authentication **mechanisms**. The mechanisms decide how a
client connection authenticates, for example, what should be considered a
set of credentials.

In practice in 99% of cases only two mechanisms are used:

 * `PLAIN` (a set of credentials such as username and password)
 * `EXTERNAL`, which assumes authentication happens out of band (not performed
   by RabbitMQ authN backends), usually [using x509 (TLS) certificates](https://github.com/rabbitmq/rabbitmq-auth-mechanism-ssl).
   This mechanism ignores client-provided credentials and relies on TLS [peer certificate chain
   verification](https://tools.ietf.org/html/rfc6818).

When a client connection reaches [authentication stage](https://github.com/rabbitmq/rabbitmq-server/blob/v3.7.2/src/rabbit_reader.erl#L1304), a mechanism requested by the client
and supported by the server is selected. The mechanism module then checks whether it can
be applied to a connection (e.g. the TLS-based mechanism will reject non-TLS connections).

An authentication mechanism is a module that implements the [rabbit_auth_mechanism](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_auth_mechanism.erl) behaviour, which includes
3 functions:

 * `init/1`: self-explanatory
 * `should_offer/1`: if this mechanism is enabled, should it be offered for a given socket?
 * `handle_response/2`: the core authentication logic of the mechanism

The `PLAIN` mechanism extracts client credentials and passes them to
a chain of authentication and authorization backends.


## Authentication and Authorization Backends

Authentication (authN) and authorization (authZ) backend(s) use
client-provided credentials to decide whether the client passes
authentication and should be granted access to the target virtual
host.

The above sentence implies that the `PLAIN` (or similar)
authentication mechanism is used and already validated the presence of
client credentials.

Authentication and authorization backends for a chain of
responsibility: a set of backends is applied to the same set of client
credentials and as soon as one of them reports success, the entire
operation is considered to be successful.

Authentication and authorization backends can be provided by
plugins. They are modules that provide implementations must implement
the following behaviours:

 * `rabbit_authn_backend` for authentication ("authn") backends
 * `rabbit_authz_backend` for authorization ("authz") backends

It is possible to implement both in a single module.
For example `internal`, `ldap` and `http` backends do so.

It is possible to use multiple backends for authn or authz. Then the first
positive result returned by a backend in the chain is considered to be final.

### AuthN Backend

The `rabbit_authn_backend` behaviour defines authentication process with single function: 

``` erlang
user_login_authentication(UserName, AuthProps) -> {ok, #auth_user{}} | {refused, Format, Args} | {error, Reason}
``` 
Where `UserName` is the name of the user which is trying to authorize,
`AuthProps` is an authorization context (proplist) (e.g. it can be `[]` for x509 certificate-based
authentication or `[{password, Password}]` for password-based one).

This function returns

 * `{ok, #auth_user{}}` in case of successfull authentication. The `#auth_user{}` record is then passed
    on to other modules, associated with the connection, etc.
 * `{refused, Format, Args}` when user authentication fails. `Format` and `Args` are meant to be used
    with `io:format/2` and similar functions.
 * `{error, Reason}` when an unexpected error occurs.

### AuthZ Backend

The `rabbit_authz_backend` behaviour defines functions that authorize access
to RabbitMQ resources, such as `vhost`, `exchange`, `queue` or `topic`.

It contains following functions:

``` erlang
% if user is allowed to access broker.
user_login_authorization(UserName) -> {ok, Impl} | {ok, Impl, Tags} | {refused, Format, Args} | {error, Reason}.
% if user have access to specific vhost.
check_vhost_access(#auth_user{}, Vhost, Socket) -> boolean() | {error, Reason}.
% if user have access to specific resource
check_resource_access(#auth_user{}, #resource{}, Permission) -> boolean() | {error, Reason}.
```

Where 

 * `UserName`, `Format`, `Args`: see above.
 * `Impl` is internal state of authorization backend. It will vary between backends and can be thought of
   as backend's `State`.
 * `Tags` is user tags. Those are used by features such as policies, plugins such as management, and so on. Tags can be an empty list.
 * `Vhost` is self-explanatory
 * `Permission` currently one of `configure`, `read`, or `write`

The `#auth_user{}` record represents a user whenever we need to
check access to vhosts and resources.

This record has the following structure:
`#auth_user{ username :: binary(), impl :: any(), tags :: [any()] }`,

where `impl` is internal backend state, `tags` is user tags (see above).

`impl` can be used to check resource access by querying an external data source or performing
a check solely on the provided state (local data).

`#resource{ virtual_host :: binary(), kind :: query|exchange|topic, name :: binary() }`
represents a resource (a queue, exchange, or topic) access to which is restricted.

### Configuring Backends

Backends are configured the usual way and can have multiple "syntaxes"
(recognised forms):

``` erlang
% To enable single backend:
{rabbit, [{auth_backends, [my_auth_backend]}]}.
% To check several backends. If one is refused - check next.
{rabbit, [{auth_backends, [my_auth_backend, my_other_auth_backend]}]}.
% To use different modules as AuthN and AuthZ backends
{rabbit, [{auth_backends, [{my_authn_backend, my_authz_backend}]}]}.
% You can still fallback if using different modules
{rabbit, [{auth_backends, [{my_authn_backend, my_authz_backend}, my_other_auth_backend]}]}.
```

If backend is defined by a tuple,
the first element will be used as an `AuthN` module and the second as the `AuthZ` one.
If it is defined by an atom, it willbe used for both `AuthN` and `AuthZ`.

When a backend is defined by a list, the server will use modules in the chain in order
until one of them returns a positive result or the list is exhausted (the Chain of Responsibility
pattern in object-oriented parlance).

If authentication is successfull then the `AuthZ` backend from the same tuple ("chain element")
will be used for authorization checks later.

### Example Backends:

 * `rabbit_auth_backend_dummy`: a dummy no-op backend, only used as the most trivial example
 * `rabbit_auth_backend_internal`: internal data store backend. See https://www.rabbitmq.com/access-control.html for more info
 * `rabbit_auth_backend_ldap`: provides `AuthN` and `AuthZ` backends in a single module, backed by LDAP
 * `rabbit_auth_backend_http`: provides `AuthN` and `AuthZ` backends in a single module, backed by an HTTP service
 * `rabbit_auth_backend_amqp`: provides `AuthN` and `AuthZ` backends in a single module, backed by an AMQP 0-9-1 service that uses request/response ("RPC")
 * `rabbit_quth_backend_uaa`: provides `AuthN` and `AuthZ` backends in a single module, backed by [Cloud Foundry UAA](https://github.com/cloudfoundry/uaa).





