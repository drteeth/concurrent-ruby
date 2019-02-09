## Examples

The simplest example is to use the actor as an asynchronous execution.
Although, `Promises.future { 1 + 1 }` is better suited for that purpose.

```ruby
actor = Concurrent::ErlangActor.spawn(:on_thread, name: 'addition') { 1 + 1 }
# => #<Concurrent::ErlangActor::Pid:0x000002 addition terminated normally with 2>
actor.terminated.value!                  # => 2
```

Let's send some messages and maintain some internal state 
which is what actors are good for.

```ruby
actor = Concurrent::ErlangActor.spawn(:on_thread, name: 'sum') do
  sum = 0 # internal state
  # receive and sum the messages until the actor gets :done
  while true
    message = receive
    break if message == :done
    # if the message is asked and not only told, 
    # reply with a current sum
    reply sum += message   
  end
  sum
end
# => #<Concurrent::ErlangActor::Pid:0x000003 sum running>
```

The actor can be either told a message asynchronously, 
or asked. The ask method will block until actor replies.

```ruby
# tell returns immediately returning the actor 
actor.tell(1).tell(1)
# => #<Concurrent::ErlangActor::Pid:0x000003 sum running>
# blocks, waiting for the answer 
actor.ask 10                             # => 12
# stop the actor
actor.tell :done
# => #<Concurrent::ErlangActor::Pid:0x000003 sum running>
actor.terminated.value!                  # => 12
```

### Receiving

Simplest message receive.

```ruby
actor = Concurrent::ErlangActor.spawn(:on_thread) { receive }
# => #<Concurrent::ErlangActor::Pid:0x000004 running>
actor.tell :m
# => #<Concurrent::ErlangActor::Pid:0x000004 running>
actor.terminated.value!                  # => :m
```

which also works for actor on pool, 
because if no block is given it will use a default block `{ |v| v }` 

```ruby
actor = Concurrent::ErlangActor.spawn(:on_pool) { receive { |v| v } }
# => #<Concurrent::ErlangActor::Pid:0x000005 running>
# can simply be following
actor = Concurrent::ErlangActor.spawn(:on_pool) { receive }
# => #<Concurrent::ErlangActor::Pid:0x000006 running>
actor.tell :m
# => #<Concurrent::ErlangActor::Pid:0x000006 running>
actor.terminated.value!                  # => :m
```

The received message type can be limited.

```ruby
Concurrent::ErlangActor.
  spawn(:on_thread) { receive(Numeric).succ }.
  tell('junk'). # ignored message
  tell(42).
  terminated.value!                      # => 43
```

On pool it requires a block.

```ruby
Concurrent::ErlangActor.
  spawn(:on_pool) { receive(Numeric) { |v| v.succ } }.
  tell('junk'). # ignored message
  tell(42).
  terminated.value!                      # => 43
```

By the way, the body written for on pool actor will work for on thread actor 
as well. 

```ruby
Concurrent::ErlangActor.
  spawn(:on_thread) { receive(Numeric) { |v| v.succ } }.
  tell('junk'). # ignored message
  tell(42).
  terminated.value!                      # => 43
```

The `receive` method can be also used to dispatch based on the received message.

```ruby
actor = Concurrent::ErlangActor.spawn(:on_thread) do
  while true
    receive(on(Symbol) { |s| reply s.to_s },
            on(And[Numeric, -> v { v >= 0 }]) { |v| reply v.succ },
            # put last works as else
            on(ANY) do |v| 
              reply :bad_message
              terminate [:bad_message, v]
            end)            
  end 
end
# => #<Concurrent::ErlangActor::Pid:0x000007 running>
actor.ask 1                              # => 2
actor.ask 2                              # => 3
actor.ask :value                         # => "value"
# this malformed message will terminate the actor
actor.ask -1                             # => :bad_message
# the actor is no longer alive, so ask fails
actor.ask "junk" rescue $!
# => #<Concurrent::ErlangActor::NoActor: #<Concurrent::ErlangActor::Pid:0x000007 terminated because of [:bad_message, -1]>>
actor.terminated.result                  # => [false, nil, [:bad_message, -1]]
```

And a same thing for the actor on pool. 
Since it cannot loop it will call the body method repeatedly.

```ruby
module Behaviour
  def body
    receive(on(Symbol) do |s| 
              reply s.to_s 
              body # call again  
            end,
            on(And[Numeric, -> v { v >= 0 }]) do |v| 
              reply v.succ
              body # call again 
            end,
            # put last works as else
            on(ANY) do |v| 
              reply :bad_message
              terminate [:bad_message, v]
            end)  
  end
end                                      # => :body

actor = Concurrent::ErlangActor.spawn(:on_pool, environment: Behaviour) { body }
# => #<Concurrent::ErlangActor::Pid:0x000008 running>
actor.ask 1                              # => 2
actor.ask 2                              # => 3
actor.ask :value                         # => "value"
# this malformed message will terminate the actor
actor.ask -1                             # => :bad_message
# the actor is no longer alive, so ask fails
actor.ask "junk" rescue $!
# => #<Concurrent::ErlangActor::NoActor: #<Concurrent::ErlangActor::Pid:0x000008 terminated because of [:bad_message, -1]>>
actor.terminated.result                  # => [false, nil, [:bad_message, -1]]
```

Since the behavior is stable in this case we can simplify with the `:keep` option
that will keep the receive rules until another receive is called
replacing the kept rules.

```ruby
actor = Concurrent::ErlangActor.spawn(:on_pool) do
  receive(on(Symbol) { |s| reply s.to_s },
          on(And[Numeric, -> v { v >= 0 }]) { |v| reply v.succ },
          # put last works as else
          on(ANY) do |v| 
            reply :bad_message
            terminate [:bad_message, v]
          end,
          keep: true)            
end
# => #<Concurrent::ErlangActor::Pid:0x000009 running>
actor.ask 1                              # => 2
actor.ask 2                              # => 3
actor.ask :value                         # => "value"
# this malformed message will terminate the actor
actor.ask -1                             # => :bad_message
# the actor is no longer alive, so ask fails
actor.ask "junk" rescue $!
# => #<Concurrent::ErlangActor::NoActor: #<Concurrent::ErlangActor::Pid:0x000009 terminated because of [:bad_message, -1]>>
actor.terminated.result                  # => [false, nil, [:bad_message, -1]]
```

### Actor types

There are two types of actors. 
The type is specified when calling spawn as a first argument, 
`Concurrent::ErlangActor.spawn(:on_thread, ...` or 
`Concurrent::ErlangActor.spawn(:on_pool, ...`.

The main difference is in how receive method returns.
 
-   `:on_thread` it blocks the thread until message is available, 
    then it returns or calls the provided block first. 
 
-   However, `:on_pool` it has to free up the thread on the receive 
    call back to the pool. Therefore the call to receive ends the 
    execution of current scope. The receive has to be given block
    or blocks that act as a continuations and are called 
    when there is message available.
 
Let's have a look at how the bodies of actors differ between the types:

```ruby
ping = Concurrent::ErlangActor.spawn(:on_thread) { reply receive }
# => #<Concurrent::ErlangActor::Pid:0x00000a running>
ping.ask 42                              # => 42
```

It first calls receive, which blocks the thread of the actor. 
When it returns the received message is passed an an argument to reply,
which replies the same value back to the ask method. 
Then the actor terminates normally, because there is nothing else to do.

However when running on pool a block with code which should be evaluated 
after the message is received has to be provided. 

```ruby
ping = Concurrent::ErlangActor.spawn(:on_pool) { receive { |m| reply m } }
# => #<Concurrent::ErlangActor::Pid:0x00000b running>
ping.ask 42                              # => 42
```

It starts by calling receive which will remember the given block for later
execution when a message is available and stops executing the current scope.
Later when a message becomes available the previously provided block is given
the message and called. The result of the block is the final value of the 
normally terminated actor.

The direct blocking style of `:on_thread` is simpler to write and more straight
forward however it has limitations. Each `:on_thread` actor creates a Thread 
taking time and resources. 
There is also a limited number of threads the Ruby process can create 
so you may hit the limit and fail to create more threads and therefore actors.  

Since the `:on_pool` actor runs on a poll of threads, its creations 
is faster and cheaper and it does not create new threads. 
Therefore there is no limit (only RAM) on how many actors can be created.

To simplify, if you need only few actors `:on_thread` is fine. 
However if you will be creating hundreds of actors or 
they will be short-lived `:on_pool` should be used.      

### Erlang behaviour

The actor matches Erlang processes in behaviour. 
Therefore it supports the usual Erlang actor linking, monitoring, exit behaviour, etc.

```ruby
actor = Concurrent::ErlangActor.spawn(:on_thread) do
  spawn(link: true) do # equivalent of spawn_link in Erlang
    terminate :err # equivalent of exit in Erlang    
  end
  trap # equivalent of process_flag(trap_exit, true) 
  receive  
end
# => #<Concurrent::ErlangActor::Pid:0x00000c running>
actor.terminated.value!
# => #<Concurrent::ErlangActor::Exit:0x00000d
#     @from=
#      #<Concurrent::ErlangActor::Pid:0x00000e terminated because of err>,
#     @link_terminated=true,
#     @reason=:err>
```

### TODO

*   receives
*   More erlang behaviour examples
*   Back pressure with bounded mailbox
*   _op methods
*   types of actors
*   always use timeout
*   drop and log unrecognized messages, or just terminate
*   Functions module