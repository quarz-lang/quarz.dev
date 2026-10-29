+++
title = "Metaprogramming"
description = "Compile-time metaprogramming: classes as values, static def, generated methods and fields, and (being built) quoted blocks."
weight = 5
+++

<!-- Kept in step with quarz-lang/quarz's main (docs/14-metaprogramming.md
     is the specification). A block for a feature that has landed is a
     checked block; the generation example needs two files (the staging
     rule: a `static def` runs only from a file that requires it), so its
     library is a file=orm block that the application requires. Blocks for
     features not built yet are sketches, check=none, until they land. -->

Quarz defines everything statically: there is no reopening of classes,
no `define_method`, `method_missing` or `eval`. Two things Ruby
libraries do with those are still wanted, though. An ORM adds typed
methods to a class from one line in its body (`has_many Comment`), and a
query DSL reads a block as a query rather than running it. Quarz gets
both through **compile-time execution of type-checked Quarz code**: no
macros over untyped syntax, no code built from strings.

**This page describes work in progress.** Classes as values,
`static def`, reflection and the generation of methods and fields are in
the language, and the tools show what was generated; the rest is being
built in the order of the table. The
specification is
[docs/14-metaprogramming.md](https://github.com/quarz-lang/quarz/blob/main/docs/14-metaprogramming.md).

| Piece | What it gives | Status |
|---|---|---|
| `Type[T]` | A class as a value, with its type arguments | Done |
| `static def` | Methods that run at compile time | Done |
| `static` statements and `Meta` reflection | A class body's call of a library's `static def`, which reads the class's fields and methods | Done |
| `def_method`, `def_class_method`, `def_field`, `doc` | Generating typed methods and fields | Done |
| Tools for generated members | `quarz doc` showing them, `quarz check --expand`, `static` statements in the REPL | Done |
| A module's `static def` through `extend` | Macros kept in a module | Later (an error today) |
| `Quoted[P]` blocks | A block as a type-checked expression tree | In progress |
| `static(expr)` and compile-time SQL | A query's SQL as a constant | Later (`static(...)` is reserved) |

## Classes as values: `Type[T]`

A class written in an expression is a value of type `Type[T]`, with its
type arguments: `Post` is a `Type[Post]`, `Array[Int]` a
`Type[Array[Int]]`. `Class`, the type of `x.class`, is `Type[Object]`.

```quarz
abstract class Model
end
class Post < Model
end

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
other, that the compiler can also **call while compiling**, from a
`static` statement in a class body (below). One that neither reflects on
classes nor generates members is an ordinary method too, so a pure
helper is written once:

```quarz
module Inflect
  static def self.plural(word: String) -> String
    word.end_with?("s") ? word : word + "s"
  end
end

puts Inflect.plural("comment")   # => comments, at run time
```

What a `static def` reaches when it runs at compile time is restricted
so that a compilation depends only on the sources: no IO, files,
environment, tasks, channels, atoms, randomness or clock (`puts`, `p`
and `warn` are allowed and print to the compiler's standard error), and
ten seconds at most. One that reflects on a class, generates members or
takes the class builder `Meta::ClassDef` (below) exists only at compile
time: calling it from ordinary code is a compile error.

## Generating methods: `static` statements and `Meta`

A class body may hold, besides its declarations, calls of a `static def`
class method of the class or one of its ancestors. Each call runs at
compile time, after every file is declared and before the method bodies
are checked, with `self` a `Meta::ClassDef[Self]`: a `Type[Self]` (its
name, `==`, `<=` and the class's class methods) that can also read the class's
fields and method signatures (`fields`, `field`, `methods`, `method?`,
`includes?`) and generate members of it.

```quarz,file=orm
# orm.qz, the library: a cog the application depends on, or a file it
# requires
abstract class Model
  @id: Int
  def initialize(@id: Int); end
  def id -> Int = @id

  static def self.has_many[T <: Model](target: Type[T]) -> Void
    fk = target.field("#{name.downcase}_id".to_sym) ||
         raise "#{target.name} has no foreign key to #{name}"
    table = "#{target.name.downcase}s"
    doc "The SQL that finds this #{name}'s #{table}."
    def_method("#{table}_sql".to_sym) do
      "SELECT * FROM #{table} WHERE #{fk.name} = #{id}"
    end
  end

  static def self.counter(field: Symbol) -> Void
    def_field(field, Int, 0, var: true, reader: true)
  end
end
```

```quarz
# The application
require_relative "orm"

class Comment < Model
  @post_id: Int
  def initialize(@id: Int, @post_id: Int); end
end

class Post < Model
  has_many Comment     # defines Post#comments_sql -> String
  counter :views       # defines @views (0) and Post#views
  def view -> Void = @views += 1
end

post = Post.new(7)
post.view
puts post.comments_sql   # => SELECT * FROM comments WHERE post_id = 7
p post.views             # => 1
```

- **The block is the method's body.** `def_method(name) { ... }` makes a
  method of the class whose body is the block: its `self` is an instance
  of the class (here, any `Model`), and its parameters' and result's
  types come from the block (`def_method(:pair, [:x, :y]) { |x: Int, y:
  String| ... }`; a block that only raises gives the result type
  `Nothing`, so it can stand for any method, an abstract one's
  implementation included). `def_class_method` makes a class method the
  same way, `doc` gives the next generated method its documentation, and
  `def_field(name, type, default)` adds a field with a default, made
  assignable by `var: true` and given accessors by `reader:` and
  `writer:`.
- **Checked where it is written.** The block is type-checked in the
  library, once, under the `static def`'s constraints (`T <: Model`), so
  it sees the library's names, not the class's. A value it reads from the
  `static def` (`table`, `fk`) is computed at compile time and baked into
  the generated method as a constant: `nil`, booleans, numbers, strings,
  symbols, regexps, classes, enum variants, `data` values (a
  `Meta::Field` is one), and arrays, hashes and ranges of these.
- **An ordinary member.** The generated method is typed, inlined and
  compiled like a written one; it can implement an abstract method. A
  name the class already has is an error at the statement, and so are
  `initialize`, `deinitialize` and `new`.
- **Errors are compile errors.** An exception the statement does not
  rescue is reported at the statement, with its message and the
  compile-time backtrace: `has_many Tag` on a `Tag` without `post_id`
  says "`has_many` raised RuntimeError: Tag has no foreign key to Post".
- **Seeing what was generated.** `quarz check --expand` prints the
  generated members as declarations under the statement that made them,
  and `quarz doc` documents a generated method with its `doc`, naming the
  statement it came from. For the application above:

  ```text
  class Post  # app.qz:9
    # generated by `has_many Comment` at app.qz:10
    # The SQL that finds this Post's comments.
    def comments_sql -> String = <block at orm.qz:13>

    # generated by `counter :views` at app.qz:11
    var @views: Int = 0
    attr_reader :views
  end
  ```

  In the REPL, a class's `static` statements run when the input that
  declares it is entered, calling the `static def`s of earlier inputs
  and of required files.

Staging is by package: a `static` statement calls only a `static def` of
a cog its cog depends on, of the standard library, or, in a script, of a
file it requires; one of its own cog (or, in a script, of its own file)
is an error, as with Rust's procedural macros. A module's `static def`
reached through `extend` is refused for now: the planned form is a
generic module whose parameter is the extending class
(`module Macros[T <: Model]` with `extend Macros[Post]`).

## Blocks as expression trees: `Quoted[P]`

This part is being built; the code below is a sketch. A block parameter
typed `Quoted[P]` receives the block as a
**type-checked expression tree** (`Meta::Expr`) instead of a closure.
The block is first checked as an ordinary `P`, so `views > "x"` is a
type error; the receiver then walks the tree, for example to build SQL.

```quarz,check=none
data Relation[T <: Model](query: Query)
  def where(&cond: Quoted[Q[T].() -> Cond]) -> Relation[T]
    Relation[T].new(query.and(cond.expr))     # keeps the tree, runs nothing
  end
  def to_a -> Array[T] = Db.current.run(query).map { |row| T.from_row(row) }
end

Post.relation.where { published && views > 1000 }.to_a
```

`for` comprehensions desugar to `flat_map`/`filter`/`map`, so a
`Relation` that takes those as `Quoted` blocks gives a query syntax with
no new grammar:

```quarz,check=none
for p in Post.relation when p.views > 1000 do p.title end
# => Post.relation.filter { |p| p.views > 1000 }.map { |p| p.title }
```

## Not included

Decided against, or deferred until a need shows up:

- Classes built at run time, macros over untyped syntax, and code
  assembled from strings: the class hierarchy stays fixed for the whole
  program, and everything that runs at compile time is checked code.
- Constant-folding of arbitrary pure methods. Only a `static def` called
  on constants, and asked to (`static(expr)`, reserved and not built yet),
  will be evaluated at compile time; a query whose shape is decided at run
  time is translated at run time by the same code.
- `has_many :comments` by name: deriving a type from a symbol needs a
  quoting layer (`quote { }` and `${}` over `Meta::Expr`), which is
  deferred.
- Generated fields without a default: `def_field` always takes one, so
  that every `initialize` stays complete.
- Reading files at compile time (schemas), compile-time hooks such as
  `inherited`, and generated nested types (`def_data`): later, if a need
  shows up.

## Further reading

The specification is
[docs/14-metaprogramming.md](https://github.com/quarz-lang/quarz/blob/main/docs/14-metaprogramming.md)
in the [repository's design documents](https://github.com/quarz-lang/quarz/tree/main/docs),
next to [Syntax](https://github.com/quarz-lang/quarz/blob/main/docs/01-syntax.md),
whose §3 and §7 state the rules this builds on.
