# Rabbitmq Authorization and Authentication backends.

This document describes authorization and authentication for access control.

For protocol level authorization mechanisms see http://www.rabbitmq.com/authentication.html

For more info about access control see https://www.rabbitmq.com/access-control.html

## Backend modules

To authorize users to access RabbitMQ resource authorization, backend modules is used.
Backend module is any Erlang module, implementing following behaviours:
- `rabbit_authn_backend` - authentication backend
- `rabbit_authz_backend` - authorization backend

Implementing both behaviours in single module is common behaviour. 
For example `internal`, `ldap` and `http` backends do so.

### AuthN backend

`rabbit_authn_backend` behaviour defines authentication process with single function: 

```
user_login_authentication(UserName, AuthProps) -> {ok, #auth_user{}} | {refused, Format, Args} | {error, Reason}
``` 
Where `UserName` - user name, which is trying to authorize, `AuthProps` - authorization properties proplist (currently can be `[]` or `[{password, Password}]`).

This function returns:
- `{ok, #auth_user{}}` - successfull authorization. `#auth_user{}` record is used further.
- `{refused, Format, Args}` - user authorization is invalid. Error message can be formatted from Format and Args.
- `{error, Reason}` - internal error occured.

### AuthZ backend

`rabbit_authz_backend` behaviour defines authorization of user acces to RabbitMQ resources, such as `vhost`, `exchange`, `queue` or `topic`.

It defines following functions:
```
% if user is allowed to access broker.
user_login_authorization(UserName) -> {ok, Impl} | {ok, Impl, Tags} | {refused, Format, Args} | {error, Reason}.
% if user have access to specific vhost.
check_vhost_access(#auth_user{}, Vhost, Socket) -> boolean() | {error, Reason}.
% if user have access to specific resource
check_resource_access(#auth_user{}, #resource{}, Permission) -> boolean() | {error, Reason}.
```

Where 

- `UserName`, `Format`, `Args` - same as above.
- `Impl` - internal state of authorization backend
- `Tags` - user tags. Used by management plugin to define role of user.
- `Vhost` - vhost name
- Permission - `configure | read | write`

There are `#auth_user{}` record used to check access to vhosts and resources. 

This record has following structure: `#auth_user{ username :: binary(), impl :: any(), tags :: [any()] }`, where `impl` - internal state for backend, `tags` - user tags, used by management plugin.

`impl` can be used to check resource access without querying user data or perform some specific actions on every check.

`#resource{ virtual_host :: binary(), kind :: query|exchange|topic, name :: binary() }` - resource record used by resource access checks.

### Configuring backends

Backends are configured with application env like this:
```
% To enable single backend:
{rabbit, [{auth_backends, [my_auth_backend]}]}.
% To check several backends. If one is refused - check next.
{rabbit, [{auth_backends, [my_auth_backend, my_other_auth_backend]}]}.
% To use different modules as AuthN and AuthZ backends
{rabbit, [{auth_backends, [{my_authn_backend, my_authz_backend}]}]}.
% You can still fallback if using different modules
{rabbit, [{auth_backends, [{my_authn_backend, my_authz_backend}, my_other_auth_backend]}]}.
```

If backend is defined by tuple - first element will be chosen for AuthN and second for AuthZ backend, if it is defined by atom - it will be chosen for both AuthN and AuthZ backends.

Server will try to authenticate using first backend in list, if refused - will try nex one, until it reach end of the backend list.

If authentication (AuthN) is successfull - AuthZ backend from this step will be used in further checks.

### Current available backends:

- `rabbit_auth_backend_internal` - internal data store backend. See https://www.rabbitmq.com/access-control.html for more info
- `rabbit_auth_backend_dummy` - dummy backend, CANNOT AUTHORIZE OR AUTHENTICATE USERS
- `rabbit_auth_backend_ldap` - plugin backend, used to auth users using LDAP database. See https://www.rabbitmq.com/ldap.html and https://github.com/rabbitmq/rabbitmq-auth-backend-ldap for more info
- `rabbit_auth_backend_http` - plugin backend to auth using external http server. See https://github.com/rabbitmq/rabbitmq-auth-backend-http
- `rabbit_auth_backend_amqp` - plugin backend to auth by connecting to an authorisation server over RPC-over-AMQP. See https://github.com/rabbitmq/rabbitmq-auth-backend-amqp
- `rabbit_quth_backend_uaa` - Plugin backend to auth with UAA authorization service. See https://github.com/rabbitmq/rabbitmq-auth-backend-uaa





