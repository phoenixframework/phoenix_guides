Testing Phoenix Channels
========================

Testing Phoenix Channels will help ensure that your code continues to work
as you make changes to your application. It helps guide development and
ensure there is no breakage later on down the road.

Phoenix has built in helper functions in `Phoenix.Channel.Test` that help
make testing Channels easy.

## Testing `join` authorization

Let’s add a new file at `test/channels/room_channel_test.exs`

    defmodule App.RoomChannelTest do
      use ExUnit.Case
      import Phoenix.Channel.Test
      alias App.RoomChannel

      test "room:join requires the super secure auth token" do
        {status, _socket} =
          build_socket("room:join")
          |> join(RoomChannel, %{"token" => "phoenix"}

        assert status == :ok
      end
    end

So what’s happening here?

First we’ve set this test up with `ExUnit` and imported the test helpers
with `import Phoenix.Channel.Test`.

Next, we build a socket with the topic `"room:join"` and join the
`RoomChannel` with the params `%{"token" => "phoenix"}`. We pattern match to
get the status of the response and check that it is `:ok` meaning that the
socket is authorized.

When we run `mix test` we see that we have not yet defined
`App.RoomChannel`. Let’s get this test passing by creating a file at
`web/channels/room_channel.ex`

    defmodule App.RoomChannel do
      use Phoenix.Channel

      def join("room:join", %{"token" => token}, socket) do
        {:ok, socket}
      end
    end

This will make our test pass, but it’s not quite what we need. Right now the
Channel will authorize any socket. Let’s add a test to make sure that a
channel will fail when the token is not correct.

    # Add this inside `App.RoomChannelTest`

    test "room:join returns unauthorized if token is not correct" do
      {status, message, socket} =
        build_socket(“room:join”)
        |> join(RoomChannel, %{"token" => "Not correct"}

      assert status == :error
      assert message == :unauthorized
    end

If we run mix test this will not pass since we haven't added any code. Let's
fix that.

    # In `App.RoomChannel`

    def join("room:join", %{"token" => token}, socket) do
      if token == "phoenix" do
        {:ok, socket}
      else
        {:error, :unauthorized, socket}
      end
    end

Now if we run `mix test` we will see that both tests pass. Nice job! Now you
see how to test against

## Testing `handle_in` with `reply`

Let's add some tests for handling incoming messages.

    # Add to `App.RoomChannelTest`

    test "room:info replies with the new room's info" do
      build_socket("room:info")
      |> handle_in(RoomChannel)

      assert_socket_replied("room:info", %{title: "Elixir/Phoenix discussion"})
    end

We build a socket like we did before, but this time we call
`Phoenix.Channel.handle_in` without any params. This will call the `handle_in`
function of `RoomChannel` with the socket and the socket's topic.

`assert_socket_replied` asserts that there was a reply with the topic
"room:info" and payload `%{title: "Elixir/Phoenix discussion"}`

_Note: that the payload will not be JSON encoded during the test, but will be
encoded if you're connecting in the browser._

Let's add this function to our `RoomChannel` to make this work

    def handle_in("room:info", _params, socket) do
      reply socket, "room:info", %{title: "Elixir/Phoenix discussion"}
    end

## Testing `handle_in` with `broadcast`

Let's add some tests for handling incoming messages that broadcast

    # Add to `App.RoomChannelTest`

    test "room:new_chat broadcasts the chat message" do
      chat_message = %{message: "I <3 Elixir"}

      build_socket("room:new_chat")
      |> subscribe(App.PubSub)
      |> handle_in(RoomChannel, chat_message)

      assert_socket_broadcasted("room:new_chat", chat_message)
    end

When building the socket we need to also make sure to subscribe to it, or else
there will be no one to broadcast to and the test will fail. Typically the
module you pass in is `NameOfYourApp.PubSub`.

This time we call the Channel with params for a new chat message and assert
that the channel broadcasted a new chat with the correct payload (the new
chat). We broadcast so that anyone on this channel will see that a new chat
was added.

Let's add this function to our `RoomChannel` to make this work.

    def handle_in("room:new_chat", message_params, socket) do
      # Code to create a new message in a database

      broadcast socket, "room:new_chat", message_params
    end

## Testing `handle_out`

What if we want to timestamp outgoing messages? Let's add a test and an
implementation for that.

    # Add to `App.RoomChannelTest`

    test "room:new_chat adds timestamp on the way out"
      build_socket("room:new_chat")
      |> subscribe(App.PubSub)
      |> handle_out(RoomChannel, %{message: "foo"})

      assert_socket_replied("room:new_chat", %{message: "foo", time: "now!"})
    end

Let's create the function to handle this

    def handle_out("room:new_chat", message, socket) do
      reply socket, "room:new_chat", Map.put(socket, :time, "now!")
    end

This wraps up the introduction to testing channels in Phoenix
With these helpers you should be able to
