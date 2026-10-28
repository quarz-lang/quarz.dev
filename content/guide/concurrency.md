+++
title = "Tasks and channels"
description = "Lightweight tasks, structured concurrency, channels and atoms, with data races ruled out at compile time."
weight = 4
+++

Concurrency in Quarz is built from tasks, which the runtime schedules
over the machine's cores like goroutines. A task is ordinary code: it
calls methods, blocks on IO and sleeps, and only that task waits. There
is no `async`/`await` coloring of functions.

## Tasks and scopes

`Task.scope` starts tasks and waits for all of them before it returns.
Each `spawn` returns a `Task[T]`, and `await` gives its value.

```quarz
def slow_square(n: Int) -> Int
  Task.sleep(10.ms)
  n * n
end

results = Task.scope do |scope|
  tasks = (1..4).map { |n| scope.spawn { slow_square(n) } }
  tasks.map(&:await)
end
puts results.inspect        # => [1, 4, 9, 16]
```

If a task raises, the others in the scope are cancelled and the
exception comes out of `Task.scope`, so a failure is not ignored.
`Task.spawn { }` without a scope starts a task the program does not
wait for.

## Channels

Tasks communicate through typed channels. `send` and `receive` wait as
needed, and `each` reads until the channel is closed.

```quarz
ch = Channel[Int].new(capacity: 8)

producer = Task.spawn do
  5.times { |i| ch.send(i) }
  ch.close
end

sum = 0
ch.each { |n| sum += n }    # until the channel is closed
producer.await
puts sum                    # => 10
```

`Channel.select` waits on several channels at once, and a channel with
no capacity hands each value over directly, which also serves as a
rendezvous.

## Atoms

Shared state goes in an `Atom`, which holds an immutable value and
replaces it atomically. `update` computes the new value from the old
one.

```quarz
counter = Atom.new(0)
Task.scope do |scope|
  100.times { scope.spawn { counter.update { |n| n + 1 } } }
end
puts counter.value          # => 100
```

An `Atom` in a constant takes the place of Ruby's global and class
variables.

## No data races

What a task may touch is checked by the compiler. Values that cross
between tasks, as a channel's elements, a spawned block's captures or an
`Atom`'s contents, must be `Shareable`: immutable data, `data` and enum
values whose fields are, and classes that `include Shareable`, which
the compiler holds to having no `var` field and only Shareable fields. A
Builder, or an object with a `var` field, is not, and an attempt to
share one is a compile error:

```quarz,check=error
buf = String::Builder.new
Task.spawn { buf << "x" }   # error: a String::Builder is not Shareable
```

Since no mutable object is ever reachable from two tasks, there are no
locks to take and no data races to debug.

## Further reading

The design document [Concurrency](https://github.com/quarz-lang/quarz/blob/main/docs/04-concurrency.md)
has the full rules and the reasons behind them.
