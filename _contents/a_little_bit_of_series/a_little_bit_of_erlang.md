---
title: A Little Bit of Erlang
date: '2025-11-01'
---

Let's do hello world.

```erlang
%% File: hello.erl
%% Define hello module with a greet/0 function.

-module(hello).
-export([greet/0])

greet() ->
  io:format("Hello, world!~n").
```


```sh
$ erl
1> c(hello).                            # Compile the hello module.
2> hello:greet().                       # Call the greet function.
Hello, world!
ok
```

## Communication between nodes

On Machine 1.

```erlang
-module(hello).
-export([start/0]).

start() ->
    register(hello, spawn(?MODULE, loop_and_wait_for_message, [])).

loop_and_wait_for_message() ->
    receive
      hi ->
        greet(),
        loop_and_wait_for_message()
    end.

greet() ->
  io:format("Hello, world!~n").

```

```sh
$ erl -name node1@pc1 -setcookie thisisasecret
1> c(hello).
2> hello:start().
true
```

Start the node 2 (could be on the same or another PC) and send a `hi` message to the first.

```sh
$ erl -name node2@pc2 -setcookie thisisasecret    # Being on the same network with matching cookie make them able to communicate.
1> {hello, 'node1@pc1'} ! hi.                     # Send to the hello process on the node1.
```


Back on Machine 1, we see:

```sh
Hello, world!
```

## Replying back

Just add where you are along with the message in a tuple (`{..}`) so the receiving end know where to reply back:

```erlang
-module(hello).
-export([start/0]).

start() ->
    register(hello, spawn(?MODULE, loop_and_wait_for_message, [])).

loop_and_wait_for_message() ->
    receive
      {hi, From} ->
        greet(From),
        From ! {reply, self()},
        loop_and_wait_for_message()

      {reply, From} ->
        greet(From),
        loop_and_wait_for_message()
    end.

greet(From) ->
  io:format("Hello from ~p!~n", [From]).
```

```sh
# Node 1
1> hello:start().
```

```sh
# Node 2
1> hello:start().
2> {hello, 'node1@bq'} ! {hi, whereis(hello)}.
```

Then we see the messages:

```sh
# Node 1
Hello from <10359.100.0>!                 # 10359.100.0 is the process from node B

# Node 2
Hello from <10358.100.0>!                 # 10358.100.0 is the process from node A
```

## In the wild

What have just been shown is **Distributed Erlang**. Your Erlang processes can communicate from this built-in protocol or via a custom raw TCP, HTTP, through a message broker, and a lot more. You can spawn millions of processes in a single or multiple machines and have they discover each other and work together.

Build on top of Distributed Erlang and its inherent capability of message passing, we can achieve all kind of process and node communications and supervisions with auto scaling and self-healing mechanisms (maybe via the famous [OTP](https://en.wikipedia.org/wiki/Open_Telecom_Platform) framework) with no need of an orchestrator like Kubernetes or Swarm.

## Additional resources

1. [Learn You Some Erlang for Great Good!](https://learnyousomeerlang.com/)
2. [Stuff Goes Bad: Erlang in Anger](https://www.erlang-in-anger.com/)
3. [Frank Hebert's Blog](https://ferd.ca/)
