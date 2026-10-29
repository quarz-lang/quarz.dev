+++
title = "Metaprogramming"
description = "Compile-time metaprogramming, being built: class objects with type arguments, static def, Meta, and quoted blocks."
weight = 5
+++

<!-- The code blocks on this page are marked check=none while the features
     are being built; make them checked blocks as each feature lands. -->

Quarz defines everything statically: there is no reopening of classes,
no `define_method`, `method_missing` or `eval`. Two things Ruby
libraries do with those are still wanted, though. An ORM adds typed
methods to a class from one line in its body (`has_many Comment`), and a
query DSL reads a block as a query rather than running it. Quarz is
getting both through **compile-time execution of type-checked Quarz
code**: no macros over untyped syntax, no code built from strings.

**This page describes work in progress.** None of it is in the language
yet; the pieces are being implemented in the order of the table, and the
code below is not checked by this site's CI until they land.

| Piece | What it gives | Status |
|---|---|---|
| `Type[T]` | A class as a value, with its type arguments | First pull request under review |
| `static def` | Methods that run at compile time | Design settled, not started |
| `Meta` and `static` statements in a class body | Generating typed methods from a library | Design settled, not started |
| `Quoted[P]` blocks | A block as a type-checked expression tree | Not started |
| Compile-time SQL (`to_sql!`) | A query's SQL as a constant | Later |

## Classes as values: `Type[T]`

A class written in an expression is a value of type `Type[T]`, with its
type arguments: `Post` is a `Type[Post]`, `Array[Int]` a
`Type[Array[Int]]`. `Class`, the type of `x.class`, is `Type[Object]`.

```quarz,check=none
def describe[T](t: Type[T]) -> String = "#{t.name} (#{t <= Model})"

puts describe(Post)          # => Post (true)
puts describe(Array[Int])    # => Array (false)
```

- `name`, `==` and `<=` (subtyping) work on any `Type[T]`. The run-time
  value knows the class only: `Array[Int] == Array[String]` is true.
- `new` is typed only where the class is known: on a class written in the
  code (`Post.new(...)`) or on `self` in a class method. A `Type[T]` held
  in a variable or a parameter has no `new`, since `T`'s `initialize` is
  not known there.
- `Class` is a type alias, so in a pattern write `in Type`, not
  `in Class`.

## Methods that run at compile time: `static def`

A `static def` is an ordinary method, type-checked and compiled like any
other, that the compiler can also **call while compiling**. Its body
must be pure: no IO, tasks, atoms or environment. A `raise` in it
becomes a compile error pointing at the call.

```quarz,check=none
module Inflect
  static def self.plural(word: String) -> String
    word.end_with?("s") ? word : word + "s"
  end
end
```

A `static def` that does not touch `Meta` (below) can be called at run
time too, so pure helpers are written once. One that uses `Meta` exists
only at compile time; calling it from ordinary code is a compile error.

## Generating methods: `Meta` and `static` statements

A class body may call a `static` class method of its ancestors or of a
module it extends. The call runs at compile time, with `self` as a
`Meta::ClassDef[Self]`: a builder for the class being defined, which can
read its fields and signatures and add methods to it.

```quarz,check=none
# In the ORM library (a cog)
abstract class Model
  static def self.has_many[T <: Model](target: Type[T]) -> Void
    fk = target.field(:"#{name.downcase}_id") ||
         raise "#{target.name} has no foreign key to #{name}"
    def_method(:"#{target.name.downcase}s") -> Relation[T] do
      target.relation.where { col(fk) == id }
    end
  end
end

# In the application
class Post < Model
  @author_id: Int
  has_many Comment       # defines Post#comments -> Relation[Comment]
end
```

The block given to `def_method` is checked where it is written, in the
library, as a method of `T`; `has_many Comment` then installs it on
`Post` with `target` and `fk` as constants. The generated method is an
ordinary one: typed, inlined and devirtualized like the rest, listed by
`quarz doc` with a note saying which line generated it. Nothing is
built from strings, and the library's names never leak into the
application's class.

Staging is by package: a class body can call `static def`s of the cogs
it depends on, not of its own cog, as with Rust's procedural macros.

## Blocks as expression trees: `Quoted[P]`

A block parameter typed `Quoted[P]` receives the block as a
**type-checked expression tree** (`Meta::Expr`) instead of a closure.
The block is first checked as an ordinary `P`, so `views > "x"` is a
type error; the receiver then walks the tree, for example to build SQL.

```quarz,check=none
data Relation[T <: Model](query: Query)
  def where(&cond: Quoted[Q[T].() -> Cond]) -> Relation[T]
  def to_a -> Array[T]
end

Post.relation.where { published && views > 1000 }.to_a
```

`for` comprehensions desugar to `flat_map`/`filter`/`map`, so a
`Relation` that takes those as `Quoted` blocks gives a query syntax with
no new grammar:

```quarz,check=none
for p in Post.relation, c in p.comments when c.approved do [p.title, c.body] end
```

## Not included

Decided against, or deferred until a need shows up:

- Classes built at run time, macros over untyped syntax, and code
  assembled from strings: the class hierarchy stays fixed for the whole
  program, and everything that runs at compile time is checked code.
- Constant-folding of arbitrary pure methods. Only a `static def` called
  on constants, with `!` (`PUBLISHED.to_sql!`), is evaluated at compile
  time; a query whose shape is decided at run time is translated at run
  time by the same code.
- `has_many :comments` by name: deriving a type from a symbol needs a
  quoting layer (`quote { }` and `${}` over `Meta::Expr`), which is
  deferred.
- Generated fields without a default: a generated field must have a
  default or be a nilable `var`, so that every `initialize` stays
  complete.
- Compile-time hooks such as `inherited`, and generated nested types.

## Further reading

The design is being written up as `docs/14-metaprogramming.md` in the
[repository's design documents](https://github.com/quarz-lang/quarz/tree/main/docs),
next to [Syntax](https://github.com/quarz-lang/quarz/blob/main/docs/01-syntax.md),
whose §3 and §7 state the rules this builds on.
