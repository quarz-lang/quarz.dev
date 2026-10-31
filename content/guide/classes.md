+++
title = "Classes, data and enums"
description = "Classes with declared fields, immutable records, sum types, modules and extensions."
weight = 1
+++

A Quarz program looks like a Ruby program. The difference is that
classes, fields and methods are declared once and fixed: there is no
reopening of classes, no `define_method`, no `method_missing` and no
`eval`, so every call is resolved when the program is compiled.

## Classes

Fields are declared with their types. A field is immutable unless it
says `var`; an immutable field is assigned only in `initialize`.

```quarz
class Account
  @id: Int                  # immutable field: assigned only in initialize
  var @balance: Int = 0     # mutable field
  attr_reader :id, :balance

  def initialize(@id: Int); end

  def deposit(amount: Int)
    raise ArgumentError, "amount must be positive" unless amount > 0
    @balance += amount
  end

  def to_s -> String = "account #{id}: #{balance}"
end

acct = Account.new(1)
acct.deposit(50)
puts acct                   # => account 1: 50
```

Parameters and return types are written on methods; a method without
a return type, like `deposit`, is `-> Void`: it returns nil.
`def initialize(@id: Int)` assigns the field from the parameter, and a
one-line method is written with `=`. Everything else here
(`attr_reader`, `raise ... unless`, `#{}` interpolation, `puts` calling
`to_s`) is Ruby.

## Data

`data` declares an immutable record: fields, a constructor, readers,
`==`, `to_s` and `with` come for free.

```quarz
data Point = (x: Int, y: Int)

p1 = Point.new(1, 2)
p2 = p1.with(y: 10)         # a copy with one field changed
puts p2                     # => Point(x: 1, y: 10)
puts p1 == Point.new(1, 2)  # => true
```

A `data` type can have a body with methods, like a class:
`data Line(from: Point, to: Point) ... end`.

## Enums

An `enum` is a type with a fixed set of variants, each of which may carry
fields. `case`/`in` on it must cover every variant; a missing one is a
compile error.

```quarz
enum Shape
  variant Circle(radius: Float)
  variant Rect(width: Float, height: Float)

  def area -> Float
    case self                 # every variant must be covered
    in Circle(radius:) then Math::PI * radius ** 2
    in Rect(width:, height:) then width * height
    end
  end
end

shapes = [Shape::Circle.new(1.0), Shape::Rect.new(2.0, 3.0)]
puts shapes.map(&:area).sum   # => 9.141592653589793
```

Pattern matching is Ruby 3's: `in Circle(radius:)` binds the field,
and array, hash, pin and guard patterns work as in Ruby.

## Modules

Modules are namespaces and mixins, as in Ruby. A module can declare
`abstract` methods, and a class that includes it must define them: the
compiler checks it, so a module is also an interface.

```quarz
data Version(major: Int, minor: Int)
  include Comparable[Version]
  def <=>(other: Version) -> Int
    [major, minor] <=> [other.major, other.minor]
  end
end

puts Version.new(1, 2) < Version.new(1, 10)   # => true
```

## Extensions

Since classes cannot be reopened, a method is added to an existing type
with an `extension` inside a module, and used where the module is
activated with `using`. The method is visible only in the file, class or
method that says `using`, so a library's extensions never leak into the
programs that use it.

```quarz
module Shout
  extension String
    def shout -> String = upcase + "!"
  end
end

using Shout
puts "hello".shout          # => HELLO!
```

## Further reading

The design document [Syntax](https://github.com/quarz-lang/quarz/blob/main/docs/01-syntax.md)
has the full rules and the reasons behind them.
