+++
title = "Immutable data"
description = "Collections are values; updates return new ones, in place when nothing else holds them."
weight = 3
+++

`String`, `Array`, `Hash` and `Set` are immutable. A method that would
modify its receiver in Ruby returns a new value instead, and the old one
is unchanged. A bang method such as `xs.sort!` is shorthand for
`xs = xs.sort`: it rebinds the variable rather than changing the value.
There is no `<<` on an array.

```quarz
xs = [1, 2, 3]
ys = xs.push(4)             # a new array; xs is unchanged
puts xs.inspect             # => [1, 2, 3]
puts ys.inspect             # => [1, 2, 3, 4]

h = {"a" => 1}
puts h.put("b", 2).inspect  # => {"a" => 1, "b" => 2}
puts h.inspect              # => {"a" => 1}
```

This is what makes values safe to share between tasks (see
[Tasks and channels](@/guide/concurrency.md)) and to keep in an `Atom`.

## Updates in place

Immutable does not mean slow. Quarz counts references, and when a
collection has only one, an update modifies it in place instead of
copying it. The usual accumulator loop costs the same as it would with
a mutable array: amortized O(1) per update, linear overall.

```quarz
def squares(n: Int) -> Array[Int]
  acc: Array[Int] = []
  n.times { |i| acc = acc.push(i * i) }   # acc is the only reference: updated in place
  acc
end

puts squares(5).inspect     # => [0, 1, 4, 9, 16]
```

Write `acc = acc.push(x)` rather than keeping the old value around; as
long as nothing else refers to the collection, the update costs no more
than it would on a mutable array.

## Builders

For code that is naturally written with mutation, such as building a
string piece by piece or an algorithm that needs a scratch array, there
are Builders, with the API of Ruby's mutable types. `build` gives one to
a block and returns the finished value.

```quarz
words = %w[b a c]
index = Hash.build do |b|         # b: Hash::Builder[String, Int]
  words.each_with_index { |w, i| b[w] = i }
end
puts index.inspect                # => {"b" => 0, "a" => 1, "c" => 2}

s = String.build { |b| 3.times { |i| b << i.to_s << "," } }
puts s                            # => 0,1,2,
```

`String::Builder`, `Array::Builder` and `Hash::Builder` can also be
made with `new`, and `to_s`, `to_a` and `to_h` hand their contents over
without copying. A Builder is mutable, so it stays in the task that made
it: the type system does not let one cross to another task.

## Further reading

The design document [Data Model and Immutability](https://github.com/quarz-lang/quarz/blob/main/docs/03-data-model.md)
has the full rules and the reasons behind them.
