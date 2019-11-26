---
layout: post
title:  "Erlang In Elixir - Erlang's :global Module"
date: 2019-11-25
---

When I started working with Elixir, I had no Erlang exposure or experience. Since then, I've constantly found myself in the Erlang docs. While I don't write Erlang on a daily basis, I certainly do make use of certain functions from time to time. Recently, I was trying to globally register a GenServer process and ended up learning a ton about global processes in Elixir/Erlang. I also learned how to register and manage global processes using Erlang's `:global` module.

## Going Global

Before going into the specific problem I was solving, lets take a look at Erlang's `:global` module (http://erlang.org/doc/man/global.html).

Erlang has a global registry that is shared between all nodes running in a distributed system. Processes can be registered in this global registry, allowing all nodes to easily communicate with them.

There can only be one named instance of a global process at any given time (i.e. if my process is registered to the global registry under name `Foo`, another process can not be registered under the global registry with name `Foo`).

Each connected node in a system can interface with globally registered processes using the functions provided in the `:global` module.

## Why Global Processes Are Useful

In Elixir, it is very common to register processes with a name so they can easily be referenced across the app. Below is a very basic example of this.

Lets go into `iex`:

`iex --name node1@127.0.0.1`
```elixir
# define anonymous function to receive messages and loop
iex(node1@127.0.0.1)1>  loop = fn f ->
  receive do
    {:hello, sender} ->
      IO.puts("Hello #{inspect(sender)}. Have a nice day!")

    {:goodbye, sender} ->
      IO.puts("Goodbye #{inspect(sender)}!")
      
    _ ->
      IO.puts("I don't understand what you're trying to say...")
  end
  f.(f)
end
#Function<7.91303403/1 in :erl_eval.expr/5>

# create a new process and save its process id to variable 'greeter_pid'
iex(node1@127.0.0.1)2> greeter_pid = spawn(fn -> loop.(loop) end)
#PID<0.116.0>

# register greeter_pid with name Greeter
iex(node1@127.0.0.1)3> Process.register(greeter_pid, Greeter)
true

# verify that named process Greeter still references the same process
iex(node1@127.0.0.1)4> Process.whereis(Greeter) == greeter_pid
true

# anywhere on the current node, a message can be sent to the greeter process using its registered name
iex(node1@127.0.0.1)5> me = self()
#PID<0.104.0>

iex(node1@127.0.0.1)6> send(Greeter, {:hello, me})
Hello #PID<0.104.0>. Have a nice day!
{:hello, #PID<0.104.0>} 

iex(node1@127.0.0.1)7> send(Greeter, {:goodbye, me})
Goodbye #PID<0.104.0>!
{:goodbye, #PID<0.104.0>}
```

This works great when there is only one node running, but what happens if I want to connect another node to this one?

While keeping `node1` running, start `iex` in another window.

`iex --name node2@127.0.0.1`
```elixir
# ping node1
iex(node2@127.0.0.1)1> Node.ping(:"node1@127.0.0.1")
:pong

# verify node1 is being discovered
iex(node2@127.0.0.1)2> Node.list()
[:"node1@127.0.0.1"]

# connect to node1
iex(node2@127.0.0.1)3> Node.connect(:"node1@127.0.0.1")
true
```

Now that the 2 nodes are connected, what should happen if I send a message to the greeter process registered as Greeter on `node1`?
```elixir
iex(node2@127.0.0.1)4> me = self()
#PID<0.110.0>

iex(node2@127.0.0.1)5> send(Greeter, {:hello, me})
** (ArgumentError) argument error
    :erlang.send(Greeter, {:hello, #PID<0.110.0>})
```

**WHAT?!**

To recap, I:
  - registered a process on `node1` with the name `Greeter`
  - connected `node2` to `node1`
  - was not able to send messages to the `Greeter` process from `node2`

When I registered the process on `node1` under the name `Greeter`, it was only locally registered on that node. 

If only there was a way to register a process in a way that multiple nodes in a distributed system could easily communicate with it...

<img src="https://media.giphy.com/media/10Jpr9KSaXLchW/giphy.gif" width="350" />

## Registering Processes Globally

This time, instead of locally registering the greeter process with a name, I will globally register it using the functions in Erlang's `:global` module.

Close all instance of `iex` and relaunch.

`iex --name node1@127.0.0.1`
```elixir
# define same anonymous function as before
iex(node1@127.0.0.1)1> loop = fn f ->
  receive do
    {:hello, sender} ->
      IO.puts("Hello #{inspect(sender)}. Have a nice day!")
    {:goodbye, sender} ->
      IO.puts("Goodbye #{inspect(sender)}!")
    _ ->
      IO.puts("I don't understand what you're trying to say...")
  end
  f.(f)
end
#Function<7.91303403/1 in :erl_eval.expr/5>

# same thing as before, register a process executing loop function
iex(node1@127.0.0.1)2> greeter_pid = spawn(fn -> loop.(loop) end)
#PID<0.123.0>

# this time, register process with :global.register_name/2
iex(node1@127.0.0.1)3> :global.register_name(Greeter, greeter_pid)
:yes

# verify that globally registered process Greeter still references the same process
iex(node1@127.0.0.1)4> :global.whereis_name(Greeter) == greeter_pid
true

# anywhere on the current node, a message can be sent to the greeter process using registered name
iex(node1@127.0.0.1)5> me = self()
#PID<0.110.0>

iex(node1@127.0.0.1)6> :global.send(Greeter, {:hello, me})
Hello #PID<0.110.0>. Have a nice day!
#PID<0.123.0>

iex(node1@127.0.0.1)7> :global.send(Greeter, {:goodbye, me})
Goodbye #PID<0.110.0>!
#PID<0.123.0>
```

What happens if I want to connect another node to this one NOW?

While keeping `node1` running, start `iex` in another window.

`iex --name node2@127.0.0.1`
```elixir
iex(node2@127.0.0.1)1> Node.ping(:"node1@127.0.0.1")
:pong

iex(node2@127.0.0.1)2> Node.list
[:"node1@127.0.0.1"]

iex(node2@127.0.0.1)3> Node.connect(:"node1@127.0.0.1")
true
```

`node2` successfully connected the same as before

Lets try to send a message to the globally registered process `Greeter` from `node2`.

```elixir
iex(node2@127.0.0.1)4> :global.send(Greeter, {:hello, me})
#PID<10914.123.0>
```

Wait... nothing printed to my console in `node2`

<img src="https://media.giphy.com/media/l0IypeKl9NJhPFMrK/giphy.gif" width="400" />

The message actually printed to `node1` since that's where the global process is running.

## Clash of The ~~Titans~~ Globally Registered Processes

Imagine in the previous example if `node2` also had a globally registered process with the same name as `node1`'s process before the two connected? (remember that there can only be one globally registered process with a given name at any time)

Close all instance of `iex` and relaunch.

`iex --name node1@127.0.0.1`
```elixir
# define our loop
iex(node1@127.0.0.1)1> loop = fn f ->
  receive do
    message -> IO.puts("message received: #{inspect(message)}")
  end
  f.(f)
end
#Function<7.91303403/1 in :erl_eval.expr/5>

iex(node1@127.0.0.1)2> pid = spawn(fn -> loop.(loop) end)
#PID<0.118.0>

iex(node1@127.0.0.1)3> :global.register_name(MyGlobalProcess, pid)
:yes

iex(node1@127.0.0.1)4> :global.send(MyGlobalProcess, :hello_from_node1)
message received: :hello_from_node1
#PID<0.118.0>
```

`iex --name node2@127.0.0.1`
```elixir
iex(node2@127.0.0.1)1> loop = fn f ->
  receive do
    message -> IO.puts("message received: #{inspect(message)}")
  end
  f.(f)
end
#Function<7.91303403/1 in :erl_eval.expr/5>

iex(node2@127.0.0.1)2> pid = spawn(fn -> loop.(loop) end)
#PID<0.118.0>

iex(node2@127.0.0.1)3> :global.register_name(MyGlobalProcess, pid)
:yes

iex(node2@127.0.0.1)4> :global.send(MyGlobalProcess, :hello_from_node2)
message received: :hello_from_node2
#PID<0.118.0>
```

Both `node1` and `node2` now have globally registered processes named `MyGlobalProcess`.

What happens after connecting `node2` to `node1`?

```elixir
iex(node2@127.0.0.1)5> Node.connect(:"node1@127.0.0.1")
true
```

Looks like it still connected.

If I look at `node1` I see

```elixir
iex(node1@127.0.0.1)5>
17:27:58.702 [info]  global: Name conflict terminating {MyGlobalProcess, #PID<11477.118.0>}
```

I can check if the orignally spawned global process is still alive on each node

```elixir
iex(node1@127.0.0.1)5> Process.alive?(pid)
true
```

```elixir
iex(node2@127.0.0.1)6> Process.alive?(pid)
false
```

One instance of the globally registered process was selected at random and stopped (in this case it was the globally registered process on `node2`).

## My Use Case

Okay so now you know more about Erlang global processes than you ever wanted to. How did I end up down this rabbit hole you may ask? I'm only giving a high level example of the problem I was trying to solve.

I was running a distributed Elixir application. I needed to implement a process that would poll messages from a third party queueing service so I decided to use a GenServer for this specific case. 

I also determined that I could take advantage of Erlang's global process registration since I only wanted one instance of this process for my entire application and knew Erlang's global process name rules would help me enforce this.

I felt like I was definitely on the right track, especially when I noticed GenServer supports global name registration out of the box (https://hexdocs.pm/elixir/GenServer.html#module-name-registration).

```elixir 
{:ok, pid} = GenServer.start_link(MyGlobal, [], name: {:global, MyGlobal})
```

This would register our new GenServer, `MyGlobal`, in the global registry with the name `MyGlobal`.

## It Worked... Until It Didn't

This was working great, until I had this GenServer process running on multiple nodes. When the nodes were connecting, one of my GenServer processes was stopping (as expected), but my supervision tree was restarting.

My first instinct was to try to trap the exit terminating my global GenServer process (added trap_exit flag in processes's `init` method w/ `Process.flag(:trap_exit, true)`). That didn't work. I understood the general concept of how Erlang was handling the global process name resolution, but I didn't realize that one of the global processes actually receives a `kill` signal. I was actually pretty surprised that this was the default way that global processes were cleaned up. I at least figured the process would receive an `:EXIT` signal to allow for error trapping.

I hoped that there was an alternative to how this name conflict was resolved so that I could gracefully handle the shutdown of the duplicate named global process. So I headed to the Erlang docs!

## Resolving Global Name Clashes Using `Resolve` Callback

So just to recap, if two unconnected nodes are running their own globally registered process `Foo`, and they connect,
one of the global processes will be sent a `kill` signal.

Luckily, this is not the only option for how Erlang will resolve these conflicts. 

When registering a GenServer with a global name (using `GenServer.start_link/3`'s `:name` option), `:global.register_name/2` is used. This was the same function I used in the `iex` example above. 

There is another function, `:global.register_name/3`, that allows a callback as an additional argument. This callback function is referred to as `Resolve`.

From the Erlang docs:
> When new nodes are added to the network, they are informed of the globally registered names that already exist. The network is also informed of any global names in newly connected nodes. If any name clashes are discovered, function Resolve is called. Its purpose is to decide which pid is correct. If the function crashes, or returns anything other than one of the pids, the name is unregistered. This function is called once for each name clash.

So this `Resolve` function tells Erlang how to handle name clashes amongst globally registered processes.

There are 3 options:

1. `random_exit_name/3` (default)  - randomly selects one of the pids for registration and kills the other one. This is the equivalent of calling `Process.exit(pid, :kill)` on one of the global processes. This is the default `Resolve` callback, but keep in mind that abrubptly shutting down the process with `kill` makes it difficult to gracefully handle shutdown.

2. `random_notify_name/3` - randomly selects one of the pids and sends the message `{global_name_conflict, Name}` to the other id. This is a less intense way of resolving the name clash. This is also useful because it allows you to do any clean up and gracefully shut down your process instead of having it `kill`ed.

3. `notify_all_name/3` - Unregisters both pids and sends the message `{global_name_conflict, Name, OtherPid}` to both processes.

I'm just quickly going to demonstrate what happens when `random_notify_name/3` is specified as the `Resolve` function when registering the global process. 

`iex --name node2@127.0.0.1`
```elixir
# define our loop
iex(node1@127.0.0.1)1> loop = fn f ->
  receive do
    message -> IO.puts("message received: #{inspect(message)}")
  end
  f.(f)
end
#Function<7.91303403/1 in :erl_eval.expr/5>

iex(node1@127.0.0.1)2> pid = spawn(fn -> loop.(loop) end)
#PID<0.118.0>

iex(node1@127.0.0.1)3> :global.register_name(MyGlobalProcess, pid, &:global.random_notify_name/3)
:yes

iex(node1@127.0.0.1)4> :global.send(MyGlobalProcess, :hello_from_node1)
message received: :hello_from_node1
#PID<0.118.0>
```

`iex --name node2@127.0.0.1`
```elixir
iex(node2@127.0.0.1)1> loop = fn f ->
  receive do
    message -> IO.puts("message received: #{inspect(message)}")
  end
  f.(f)
end
#Function<7.91303403/1 in :erl_eval.expr/5>

iex(node2@127.0.0.1)2> pid = spawn(fn -> loop.(loop) end)
#PID<0.118.0>

iex(node2@127.0.0.1)3> :global.register_name(MyGlobalProcess, pid, &:global.random_notify_name/3)
:yes

iex(node2@127.0.0.1)4> :global.send(MyGlobalProcess, :hello_from_node2)
message received: :hello_from_node2
#PID<0.118.0>
```

There is a global process registered under the name `MyGlobalProcess` on unconnected nodes `node1` and `node2`.

Lets see what happens when the nodes are connected when both of these global processes were registered with `random_notify_name/3` as their Resolve function.

```elixir
iex(node2@127.0.0.1)5> Node.connect(:"node1@127.0.0.1")
true
message received: {:global_name_conflict, MyGlobalProcess}
```

It looks like `node2`'s process was the randomly selected global process to receive the notification message. 

Note that using this new `Resolve` callback didn't actually kill either of the global processes. I would need to manually shut down whichever process receives the `{:global_name_conflict, global_process_name}` message.

## Bringing It All Together

I was excited that GenServer supported global name registration in the `:name` option of `start_link/3`, but as previously mentioned, this uses `:global.random_exit_name/3` as the `Resolve` when I wanted to use `:global.random_notify_name/3`. My workaround for this was by globally registering the process in the `init/1` callback.

In addition, I also added a handle_info function to handle graceful shutdown of the GenServer upon receiving `:global_name_conflict` message.

```elixir
defmodule MyGlobal do
  use GenServer, restart: :transient

  def start_link(_opts \\ []) do 
    GenServer.start_link(__MODULE__, [])
  end

  def init(_opts) do
    # explicitly pass desired Resolve callback and match on result
    case :global.register_name(__MODULE__, self(), &:global.random_notify_name/3) do
      :yes ->
        # do some stuff
        {:ok, []}

      :no ->
        # do some other stuff
        :ignore
    end
  end

  def handle_info({:global_name_conflict, _module_name}, state) do
    # do some graceful shutdown stuff

    {:stop, :normal, state}
  end

  ...
end
```

## Embrace The Erlang

This was a really fun problem solving experience and all of the information here comes from a combination of Elixir/Erlang docs, Elixir/Erlang source code, and trial and error. I really appreciate having the power of Erlang at my fingertips and having the ability to seemlessly use it while writing Elixir! 

<img src="https://media.giphy.com/media/QoesEe6tCbLyw/giphy.gif" width="500" />
