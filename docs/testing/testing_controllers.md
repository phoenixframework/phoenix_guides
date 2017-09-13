# Testing Controllers

We're going to take a look at how we might test drive a controller which has endpoints for a JSON api.

### Set up

First create a blank project by running

```console
$ mix phx.new hello_phoenix -y
```

Change into the newly-created `hello_phoenix` directory, configure
your database in `config/dev.exs` and then run

```console
$ mix ecto.create
```

If you have any questions about this process, now is a good time to
jump over to the [Up and Running Guide](up_and_running.html).


At this point, in a real project, you might reach for the Phoenix
generator for creating a JSON resource which looks like this:

```console
$ mix phx.gen.json  AllTheThings Thing things some_attr:string another_attr:string
```

In this command, AllTheThings is the Context; Thing is the Schema;
things is the plural name of the schema (which is used as the table
name).  Then some_attr and another_attr are the database columns on
table `things` of type string.

However, *don't* actually run this command right now.  Instead, we're
going to explore test driving out a similar result to what a generator
would give us.

Let's create an `Accounts` context for this example.. Since context
creation is not in scope of this guide, we will use the generator.
For more in-depth coverage of what a context is, read [the Contexts
Guide](contexts.html#content)

```console
$ mix phx.gen.context Accounts User users name:string email:string:unique password:string
```

Ordinarily we would spend time tweaking the generated migration file
(`priv/repo/migrations/<datetime>_create_users.exs`) to add things
like non-null constraints and so on, but we don't care about that for
this example.  Just run the migration:

```console
$ mix ecto.migrate
```

### Test driving

We will be using Test Driven Development (TDD) to develop our code.
The first step with TDD is to make sure the tests pass as is!  To do
this run:

```console
$ mix test
```

The tests created by the code generators out of the box should
pass. On the other hand, if you've done anything interesting with your
database setup in `config/dev.exs`, you might run into an error here.
To fix this error, take a look at `config/test.exs` and see if any
settings from `dev.exs` need to be copied over.

For example, if the database port is at some non-standard setting,
then running `mix test` will generate this kind of error:

```console
** (Mix) The database for HelloPhoenix.Repo couldn't be created: FATAL 3D000 (invalid_catalog_name): database "postgres" does not exist

16:04:56.704 [error] GenServer #PID<0.3290.0> terminating
** (Postgrex.Error) FATAL 3D000 (invalid_catalog_name): database "postgres" does not exist
    (db_connection) lib/db_connection/connection.ex:148: DBConnection.Connection.connect/2
    (connection) lib/connection.ex:622: Connection.enter_connect/5
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: nil
```

These sorts of issues can usually be fixed by just examining
both `dev.exs` and `test.exs` and making the necessary edits to
`test.exs`.  Also, it is always best to handle these issues *before*
creating tests that you expect to fail!

Once the baseline tests are passing without any issues, it is time to
move on to actual development.  What we are going for is a controller
with the standard CRUD actions. We'll start with our test since we're
TDDing this. Create a `user_controller_test.exs` file in
`test/hello_phoenix_web/controllers`

```elixir
# test/hello_phoenix_web/controllers/user_controller_test.exs

defmodule HelloPhoenixWeb.UserControllerTest do
  use HelloPhoenixWeb.ConnCase

end
```

There are many ways to approach TDD. Here, we will think about each action we want to perform, and handle the "happy path" where things go as planned, and the error case where something goes wrong, if applicable.

```elixir
# test/hello_phoenix_web/controllers/user_controller_test.exs

defmodule HelloPhoenixWeb.UserControllerTest do
  use HelloPhoenixWeb.ConnCase

  test "index/2 responds with all Users"

  describe "create/2" do
    test "Creates, and responds with a newly created user if attributes are valid"
    test "Returns an error and does not create a user if attributes are invalid"
  end

  describe "show/2" do
    test "Responds with user details if the user is found"
    test "Responds with a message indicating user not found"
  end

  describe "update/2" do
    test "Edits, and responds with the user if attributes are valid"
    test "Returns an error and does not edit the user if attributes are invalid"
  end

  test "delete/2 and responds with :ok if the user was deleted"

end
```

Here we have tests around the 5 controller CRUD actions we need to implement for a typical JSON API. In 2 cases, index and delete, we are only testing the happy path, because in our case they generally won't fail because of domain rules (or lack thereof). In practical application, our delete could fail easily once we have associated resources that cannot leave orphaned resources behind, or number of other situations. On index, we could have filtering and searching to test. Also, both could require authorization.

Create, show and update have more typical ways to fail because they need a way to find the resource, which could be non existent, or invalid data was supplied in the params. Since we have multiple tests for each of these endpoints, putting them in a `describe` block is good way to organize our tests.

Let's run the test:

```console
$ mix test test/hello_phoenix_web/controllers/user_controller_test.exs
```

We get 8 failures that say "Not implemented" which is good. Our tests don't have blocks yet.

Let's add our first test. We'll start with `index/2`.

```elixir
# test/hello_phoenix_web/controllers/user_controller_test.exs

defmodule HelloPhoenixWeb.UserControllerTest do
  use HelloPhoenixWeb.ConnCase

  alias HelloPhoenix.Accounts

  @user1_attrs %{email: "grobblefruit@example.org", name: "grobblefruit", password: "surf and skate"}

  # setup creates user1 for all tests, and generates the conn
  setup do
    {:ok, user} = Accounts.create_user(@user1_attrs)
    conn = build_conn()
    {:ok, conn: conn, user: user }
  end

  test "index/2 responds with all Users", %{conn: conn, user: user} do

    response = conn
    |> get(user_path(conn, :index))
    |> json_response(200)

    expected = %{
      "data" => [
        %{ "email" => user.email, "name" => user.name }
      ]
    }

    assert response == expected

  end
```

Let's take a look at what's going on here. First, some attributes for
a valid user are defined, and then a [`setup`
block](https://hexdocs.pm/ex_unit/1.5.1/ExUnit.Callbacks.html#content)
is used to create that user before each test.  The `setup` block also
creates a `conn` variable to be used in each test.  The `conn` and
`user` are then returned as a map from the `setup` block, and so are
merged into the context that is passed to each test.

The index test then hooks into the context to extract the contents of
the `conn:` and `user:` keys.  The `conn` is piped to a `get` function
to make a `GET` request to our `UserController` index action, which is
in turn piped into `json_response/2` along with the expected HTTP status code. This will return the JSON from the response body, when everything is wired up properly. We represent the JSON we want the controller action to return with the variable `expected`, and assert that the `response` and `expected` are the same.

Our expected data is a JSON response with a top level key of `"data"`
containing an array of users that have `"name"` and `"email"`
properties that should match those of the `user` object loaded into
the context by the `setup` function.  Even though the user also has a
"password" property, we don't want our JSON API spitting out the
passwords, and so we do not expect to see the password in the returned
JSON.

When we run the test we get an error that we have no `user_path` function.

In our router, we'll uncomment the `api` scope at the bottom of the
auto-generated file, and then add a resource for `User` in the API.
Because we aren't going to be generating HTML forms to create and edit
users, we add the `except: [:new, :edit]` to skip those endpoints.

```elixir
# lib/hello_phoenix_web/router.ex

defmodule HelloPhoenixWeb.Router do
  use HelloPhoenixWeb, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/", HelloPhoenixWeb do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
  end

  # Other scopes may use custom stacks.
  scope "/api", HelloPhoenixWeb do
    pipe_through :api
    resources "/users", UserController, except: [:new, :edit]
  end
end

```

Before running the test again, check out our new paths by running

```console
$ mix phx.routes
```

You should see six new routes in addition to the default '/' route:

```console
page_path  GET     /               HelloPhoenixWeb.PageController :index
user_path  GET     /api/users      HelloPhoenixWeb.UserController :index
user_path  GET     /api/users/:id  HelloPhoenixWeb.UserController :show
user_path  POST    /api/users      HelloPhoenixWeb.UserController :create
user_path  PATCH   /api/users/:id  HelloPhoenixWeb.UserController :update
           PUT     /api/users/:id  HelloPhoenixWeb.UserController :update
user_path  DELETE  /api/users/:id  HelloPhoenixWeb.UserController :delete
```

Note that all of these routes use `HelloPhoenixWeb.UserController`.
If you run the test now, you'll get a new error informing us that we
don't yet have a `UserController`. Let's add it, along with the `index/2` action we're testing. Our test description has us returning all users:

```elixir
# lib/hello_phoenix_web/controllers/user_controller.ex

defmodule HelloPhoenixWeb.UserController do
  use HelloPhoenixWeb, :controller
  alias HelloPhoenix.Accounts

  def index(conn, _params) do
    users = Accounts.list_users()
    render(conn, "index.json", data: users)
  end

end
```

The controller uses the `Accounts` context module's functions to
access the underlying data store.  Here all users are retrieved by
calling `Accounts.list_users()`.

When we run the test again, our failing test tells us we have no view. Let's add it. Our test specifies a JSON format with a top key of `"data"`, containing an array of users with attributes `"name"` and `"email"`.

```elixir
# lib/hello_phoenix_web/views/user_view.ex

defmodule HelloPhoenixWeb.UserView do
  use HelloPhoenixWeb, :view

  def render("index.json", %{data: users}) do
    %{data:
      render_many( users, HelloPhoenixWeb.UserView, "user.json", as: :data)
      }
  end

  def render("user.json", %{data: user}) do
    %{
      name: user.name,
      email: user.email
      # inserted_at: user.inserted_at,
      # updated_at: user.updated_at
    }
  end
end
```

The view module for the index uses the `render_many/4` function.
According to the
[documentation](https://hexdocs.pm/phoenix/Phoenix.View.html#render_many/4),
using `render_many/4` is "roughly equivalent" to using `Enum.map/2`,
and in fact `Enum.map` is called under the hood.  The main difference
is that the rendering step is kept within the `render` function, which
means the code benefits from library-quality error checking and so on.


And with that, one of the eight tests passes when we run it!

### Time for the Show

We'll also cover the `show/2` action here so we can see how to handle an error case.

Our show tests currently look like this:

```elixir
  describe "show/2" do
    test "Responds with user details if the user is found"
    test "Responds with a message indicating user not found"
  end
```

Run this test only by running the following command: (if your show tests don't start on line 45, change the line number accordingly)

```console
$ mix test test/hello_phoenix_web/controllers/user_controller_test.exs:45
```

Our first `show/2` test result is, as expected, not yet implemented.
Let's build a test around what we think a successful `show/2` should look like.

```elixir
# test/hello_phoenix_web/controllers/user_controller_test.exs line 45

test "Responds with user details if the user is found", %{conn: conn, user: user} do
  response = conn
  |> get(user_path(conn, :show, user.id))
  |> json_response(200)

  expected = %{
    "data" =>
    %{ "email" => user.email, "name" => user.name }

  }

  assert response == expected
end
```

This is very similar to our `index/2` test, except `show/2` requires a user id, and our data is a single JSON object instead of an array.

When we run our test the error tells us we need a `show/2` action.

```elixir
# lib/hello_phoenix_web/controllers/user_controller.ex

defmodule HelloPhoenixWeb.UserController do
  use HelloPhoenixWeb, :controller
  require Logger

  alias HelloPhoenix.Accounts

  def index(conn, _params) do
    users = Accounts.list_users()
    render(conn, "index.json", data: users)
  end

  def show(conn, %{"id" => id}) do
    # try do
      user = Accounts.get_user!(id)
      render conn, "show.json", data: user
    # catch
    # end

  end

end
```

You'll notice we only handle the case where we successfully find a
user. When we TDD we only want to write enough code to make the test
pass. We'll add more code when we get to the error handling test for
`show/2`.  However, because the standard `get_user!/1` function throws
an error if it can't find the specified id, we've wrapped the call in
a commented out "try/catch" block.  Sure this is borderline
schizophrenic, but sometimes life intervenes and interrupts the
programming flow. Often it is wise to do small things like this as
little reminders of things that need to be done.

Running the test tells us we need a `render/2` function that can pattern match on `"show.json"`:

```elixir
# lib/hello_phoenix_web/views/user_view.ex

defmodule HelloPhoenixWeb.UserView do
  use HelloPhoenixWeb, :view

  def render("index.json", %{data: users}) do
    %{data:
      render_many( users, HelloPhoenixWeb.UserView, "user.json", as: :data)
      }
  end

  def render("show.json", %{data: user}) do
    %{data:
      render_one( user, HelloPhoenixWeb.UserView, "user.json", as: :data)
      }
  end

  def render("user.json", %{data: user}) do
    %{
      name: user.name,
      email: user.email
      # inserted_at: user.inserted_at,
      # updated_at: user.updated_at
    }
  end
end
```

When we run the test again, it passes.

### No Show

The last item we'll cover is the case where we don't find a user in `show/2`.

Try this one on your own and see what you come up with. One possible solution will be given below.

Walking through our TDD steps, we add a test that supplies a
non-existent user id to `user_path` which returns a 404 status and an
error message.  One thing to keep in mind is that database ids tend to
start at 1 and increase forever.  Picking a large number is a reasonably safe
strategy, but there is a small chance that at some point a number that
seems large when you write the code is smack dab in the middle of the
numbers being allocated.  On the other hand, negative numbers are
perfectly valid integers, and yet never used for database ids.  So
we'll pick -1 as our "unobtainable" user id, which should always fail.

```elixir
# test/hello_phoenix_web/controllers/user_controller_test.exs line 58-ish

test "Responds with a message indicating user not found", %{conn: conn} do
  response = conn
  |> get(user_path(conn, :show, -1 ))
  |> json_response(404)

  expected = %{
    "errors" => "Resource not found"
  }

  assert response == expected
end
```

If you run the test now, you will probably just see the server throw
"Ecto.NoResultsError" or something similar, due to the `get_user!/1`
function that we *knew* would throw.  What we want is an HTTP code of
404 to notify the requester that this resource was not found, as well
as an accompanying error message.  (By the way, a good generic resource for
HTTP status codes can be found in the [MDN web
docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status).  A
corresponding resource for the status codes allowed by Plug can be
found in the [Plug.Conn.Status documentation](https://hexdocs.pm/plug/Plug.Conn.Status.html).)

Our controller action fleshes out the try/catch block we skipped over before:

```elixir
# lib/hello_phoenix_web/controllers/user_controller.ex line 12

def show(conn, %{"id" => id}) do
  try do
    user = Accounts.get_user!(id)
    render conn, "show.json", data: user
  catch
    :error, message ->
      conn
      |> put_status(:not_found)
      |> render( HelloPhoenixWeb.ErrorView, "400.json", reason: message )
  end
end
```

In the catch statement, there are two important things going on.
First we use the
[`put_status/2`](https://hexdocs.pm/plug/Plug.Conn.html#put_status/2)
function from `Plug.Conn` to set the desired error status.  As noted
above, the complete list of allowed codes can be found in the
[Plug.Conn.Status
documentation](https://hexdocs.pm/plug/Plug.Conn.Status.html).

Second, we've redirected the view to the `ErrorView` module.  If you
open that file up, you'll see a few rendering helpers for HTML,
output, but none for JSON.  We need to add a render function for
"400.json" as follows:

```elixir
# lib/hello_phoenix_web/views/error_view.ex

defmodule HelloPhoenixWeb.ErrorView do
  use HelloPhoenixWeb, :view

  def render("404.html", _assigns) do
    "Page not found"
  end

  def render("500.html", _assigns) do
    "Internal server error"
  end

  def render("400.json", %{reason: reason}) do
    message = case reason do
                %Ecto.NoResultsError{} -> "Resource not found"
                _ -> "I'm afraid I can't do that, Dave"
              end
    %{errors: message}
  end

...

```

In this render function, we only expect the `%Ecto.NoResultsError{}`
so far, so we only set that message.  As other errors crop up in our
code, we can add additional error messages.

With those implemented, our tests pass.

### No tests for the wicked

The rest of the controller is left for you to implement as practice. To
help you on your way, below is the fully fleshed out test file.  Of
course there are lots of edge cases and error conditions that are *not*
covered, but it should get you started.

```
# test/hello_phoenix_web/controllers/user_controller_test.exs

defmodule HelloPhoenixWeb.UserControllerTest do
  use HelloPhoenixWeb.ConnCase

  alias HelloPhoenix.Accounts

  @user1_attrs %{email: "grobblefruit@example.org", name: "grobblefruit", password: "surf and skate"}

  @user2_attrs %{email: "coffee@coffee.org", name: "espresso", password: "daydream of coffee"}

  @update_attrs %{email: "falada@horsehead.org", name: "goose girl", password: "covered in tar and feathers"}

  @bad_attrs %{email: "", name: "", password: ""}

  # setup creates user1 for all tests, and generates the conn
  setup do
    {:ok, user} = Accounts.create_user(@user1_attrs)
    conn = build_conn()
    {:ok, conn: conn, user: user }
  end


  test "index/2 responds with all Users", %{conn: conn, user: user} do

    response = conn
    |> get(user_path(conn, :index))
    |> json_response(200)

    expected = %{
      "data" => [
        %{ "email" => user.email, "name" => user.name }
      ]
    }

    assert response == expected

  end


  describe "create/2" do
    test "Creates, and responds with a newly created user if attributes are valid", %{conn: conn} do
      response = conn
      |> post( user_path(conn, :create), data: %{ attributes: @user2_attrs } )
      |> json_response(201)
      assert( response["data"] == %{"email"=> @user2_attrs.email,
                                   "name" => @user2_attrs.name}
      )

    end

    test "Cannot create a user with a duplicate email", %{conn: conn, user: user} do
      response = conn
      |> post( user_path(conn, :create), data: %{ attributes: %{"email" => user.email,
                                                               "name" => @user2_attrs.name,
                                                               "password" => @user2_attrs.password} } )
      |> json_response(422)
      assert( response["errors"] == %{"email"=>  ["has already been taken"]} )

    end

    test "Returns an error and does not create a user if attributes are invalid", %{conn: conn} do
      response = conn
      |> post( user_path(conn, :create), data: %{ attributes: @bad_attrs })
      |> json_response(422)
      assert( response["errors"] == %{
        "name" => ["can't be blank"],
        "email" => ["can't be blank"],
        "password" => ["can't be blank"] }
        )

    end

  end

  describe "show/2" do
    test "Responds with user details if the user is found", %{conn: conn, user: user} do
      response = conn
      |> get(user_path(conn, :show, user.id))
      |> json_response(200)

      expected = %{
        "data" =>
        %{ "email" => user.email, "name" => user.name }

      }

      assert response == expected
    end
    test "Responds with a message indicating user not found", %{conn: conn} do
      response = conn
      |> get(user_path(conn, :show, -1 ))
      |> json_response(404)

      expected = %{
        "errors" => "Resource not found"
      }

      assert response == expected
    end
  end

  describe "update/2" do
    setup do
      {:ok, user2} = Accounts.create_user(@user2_attrs)
      {:ok, user2: user2}
    end

    test "Edits, and responds with the user if attributes are valid", %{conn: conn, user2: user} do
      response = conn
      |> put(user_path(conn, :update, user.id ), data: %{ attributes: @update_attrs } )
      |> json_response(200)

      expected = %{
        "data" =>
        %{ "email" => @update_attrs.email, "name" => @update_attrs.name }
      }
      assert response == expected
    end
    test "Returns the unmodified user if attributes object is empty", %{conn: conn, user: user} do

      response = conn
      |>put( user_path(conn, :update, user.id ), data: %{ attributes: %{} } )
      |> json_response(200)

      expected = %{
        "data" =>
        %{ "email" => user.email, "name" => user.name }
      }
      assert response == expected

    end
    test "Returns an error and does not edit the user if attributes are invalid", %{conn: conn, user: user} do

      response = conn
        |> put( user_path(conn, :update, user.id ), data: %{ attributes: @bad_attrs })
        |> json_response(422)

      assert response["errors"] == %{
        "name" => ["can't be blank"],
        "email" => ["can't be blank"]
      }
      # by the way, will crash if attributes are set to nil (not a hash)
    end
  end

  describe "delete/2" do
    setup do
      {:ok, user2} = Accounts.create_user(@user2_attrs)
      {:ok, user2: user2}
    end

    test "delete/2 and responds with :ok if the user was deleted", %{conn: conn, user2: user} do
      r = conn
      |> delete( user_path(conn, :delete, user.id) )

      assert response(r, 204)
      response = conn
      |> get( user_path(conn, :show, user.id) )
      |> json_response(404)

      assert response["errors"] == "Resource not found"

    end
    test "delete/2 throws if user doesn't exist", %{conn: conn, user2: user} do
      r = conn
      |> delete( user_path(conn, :delete, user.id) )

      assert response(r, 204)

      response = conn
      |> delete( user_path(conn, :delete, user.id) )
      |> json_response(404)

      assert response["errors"] == "Resource not found"

    end
  end

end
```


Happy testing!
