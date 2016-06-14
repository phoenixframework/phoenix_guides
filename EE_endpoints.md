### Introduction
Each request comes in through an endpoint. Endpoints handle the request up to the
point of passing it off the a [Router](http://www.phoenixframework.org/docs/routing).

Each request begins and ends it's lifecycle inside your application in an endpoint.

A Phoenix application starts up our HelloPhoenix.Endpoint as a supervised process.
This is where Supervision from Erlang and Elixir come into this framework.

Below is from `lib/hello_phoenix.ex`:
```elixir
 # ...
 children = [¬
   # Start the endpoint when the application starts¬
   supervisor(HelloPhoenix.Endpoint, []),¬
   # ... Other supervised processes like Ecto Repo or
   # worker processes
   supervisor(HelloPhoenix.Repo, []),
   supervisor(HelloPhoenix.QueueWorker, [])
  ]
  # ...
```
### Endpoint Contents
An endpoint often includes plugs that are useful throughout all routes of
your application. At it's end an endpoint can define multiple routers that would
handle the actual routes.

Let's cover some of the elements of an Endpoint layer by layer from outside
moving inward.

An Endpoint is an Elixir module like the Controllers and Router.

```elixir
defmodule HelloPhoenix.Endpoint do¬
  ...
end
```
Frequently the first directive inside of our Endpoint module will be an inclusion
of the Phoenix.Endpoint and a declaration that this is an otp_app.TODO:Reference to what the otp_app: :hello_phoenix actually does
```elixir
  use Phoenix.Endpoint, otp_app: :hello_phoenix
```
We call a socket to exist on the /socket URI. /socket requests will be handled by the
HelloPhoenix.UserSocket module.

```elixir
  socket "/socket", HelloPhoenix.UserSocket
```
Next a series of plugs which would have to relevant to every route in your application.
You may want to customize some of the features, enabling gzip:true when deploying to production
with digested static files.

Static files are served from priv/static before any part of our request makes it to a router.

```elixir
  plug Plug.Static,¬
    at: "/", from: :hello_phoenix, gzip: false,
    only: ~w(css fonts images js favicon.ico robots.txt)
```
A code reloading feature is included into our application which uses a socket
to communicate that code can be reloaded for development preview.

```elixir
if code_reloading? do¬                                                                                            a
      socket "/phoenix/live_reload/socket", Phoenix.LiveReloader.Socket¬                                              ),¬
      plug Phoenix.LiveReloader¬
      plug Phoenix.CodeReloader¬
end¬
```
The next section defines a list of separate applications that perform useful
operations on our request. A logger is enabled. A request ID is generated.

```elixir
  plug Plug.RequestId¬
  plug Plug.Logger
```

For example a session cookie is created and signed to ensure request security
and prevent XSS. TODO: Here include links to detailed documentation for each
of the default plugs.

```elixir
 plug Plug.Session,
     store: :cookie,
     key: "_hello_phoenix_key",
     signing_salt: "change_me"
```
Finally by convention the router for our application is included where the request
will be routed to the appropriate place within your code.
```elixir
  plug HelloPhoenix.Router
```

In these succinct lines all of the common functionality among our HTTP requests
is defined and can be customized in this one endpoint definition.

TODO: cover any use case that calls for multiple endpoints :)






