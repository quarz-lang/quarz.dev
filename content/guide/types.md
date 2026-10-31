+++
title = "Types"
description = "Where types are written, what is inferred, and how nil is handled."
weight = 2
+++

Quarz is statically typed, but types are written in declarations:
method signatures and fields. Local variables are inferred from what is
assigned to them.

```quarz
def greet(name: String, times: Int = 1) -> String
  ("Hello, #{name}! " * times).strip
end

names = ["ann", "bob"]           # Array[String], inferred
count = names.size               # Int
puts greet(names[0], count)      # => Hello, ann! Hello, ann!
```

Parameters, return types and fields carry types; a method without `->`
returns `nil`. Blocks are typed from the method they are given to, so
`{ |w| w.size }` needs no annotation. A block may still declare its
parameters' types (`|w: String|`) and its result type, written after
the parameters as a method's is and meaning the same: the block returns
that type, and its value (and every `next` value) must fit it.

```quarz
words = ["ann", "bob", "cy"]
sizes = words.map { |w| -> Int? w.size }   # Array[Int?], not Array[Int]
sizes = sizes.push(nil)
p sizes                                    # => [3, 3, 2, nil]
```

A block that takes one argument can leave it unnamed: `it` is that
argument, typed the same way.

```quarz
p [3, 1, 2].map { it * 10 }.select { it > 10 }   # => [30, 20]
```

## Nil

`nil` belongs to no other type. A value that may be missing has the
type `T?` (short for `T | Nil`), and the compiler refuses to call a
method on it until it has been checked.

```quarz
def find(users: Hash[Int, String], id: Int) -> String?
  users[id]?                   # nil when the key is missing
end

users = {1 => "ann"}
user = find(users, 2)
# user.size                    # error: String? has no method size
puts user.size if user         # fine: user is a String inside the if
puts user&.size.inspect        # => nil
puts (user || "anon").size     # => 4
```

The check is flow-sensitive: after `if user`, `return unless user`,
`user && ...` or a `case`, the variable has the narrower type. The
lookups that return `nil` in Ruby have two forms here: `h[k]` and
`arr[i]` raise when there is nothing there, and `h[k]?` and `arr[i]?`
return `nil`.

## Unions

A union type lists the types a value may have. A `case` on it is
checked for exhaustiveness, like one on an enum.

```quarz
def describe(value: Int | String | Nil) -> String
  case value
  in Int => n then "a number: #{n}"
  in String => s then "a string: #{s}"
  in nil then "nothing"
  end
end

puts describe(42)      # => a number: 42
puts describe("hi")    # => a string: hi
puts describe(nil)     # => nothing
```

## Generics

Classes and methods take type parameters in brackets. A bound,
`T <: Comparable[T]`, says what the type must support.

```quarz
def largest[T <: Comparable[T]](xs: Array[T]) -> T?
  xs.reduce { |a, b| a > b ? a : b }
end

puts largest([3, 9, 4]).inspect          # => 9
puts largest(["pear", "apple"]).inspect  # => "pear"
```

Type arguments are inferred at the call, so `largest([3, 9, 4])` needs
none. The built-in collections are generic: `Array[Int]`,
`Hash[String, Int]`, `Set[String]`, `Channel[Message]`.

## Further reading

The design document [Type System](https://github.com/quarz-lang/quarz/blob/main/docs/02-types.md)
has the full rules and the reasons behind them.
