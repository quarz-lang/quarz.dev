+++
title = "Metaprogramming"
description = "Compile-time metaprogramming: classes as values, static def, generated methods and fields, quoted blocks and values computed while compiling."
weight = 5
+++

<!-- Kept in step with quarz-lang/quarz's main (docs/14-metaprogramming.md
     is the specification). A block for a feature that has landed is a
     checked block; the generation example needs two files (the staging
     rule: a `static def` runs only from a file that requires it), so its
     library is a file=orm block that the application requires. -->

Quarz defines everything statically: there is no reopening of classes,
no `define_method`, `method_missing` or `eval`. Two things Ruby
libraries do with those are still wanted, though. An ORM adds typed
methods to a class from one line in its body (`has_many Comment`), and a
query DSL reads a block as a query rather than running it. Quarz gets
both through **compile-time execution of type-checked Quarz code**: no
macros over untyped syntax, no code built from strings.

Classes as values, `static def`, reflection, the generation of methods
and fields, blocks received as trees, and values computed while
compiling are in the language, and the tools show what was generated;
macros kept in a module come later, with what "Not included" lists. The
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
| `Quoted[P]` blocks | A block as a type-checked expression tree | Done |
| `static(call)` | A `static def`'s call computed while compiling, its value put in the program | Done |

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

A block parameter typed `Quoted[P]`, `P` a procedure type, receives the
block written at the call as a **type-checked expression tree**
(`Meta::Expr`) as well as a closure. The block is checked as an ordinary
`P`, so `views > "x"` is a type error; the method then walks the tree,
for example to build SQL, and can still run the block with `proc`. A
`Quoted` has:

- `expr`, the block as a `Meta::Expr::Block` of its parameters and
  statements. Its nodes are literals and constants whose values are
  literals (`Lit`); anything else read from outside the block, fields
  and other constants included (`Capture`); the block's parameters
  (`Arg`); the block's `self` when `P` has a receiver (`SelfRef`;
  without one, `self` is a `Capture`); calls of the methods the checker
  chose (`Call`, with a `Meta::MethodRef` naming the method); `And`,
  `Or`, `Not`; `If` (for `if`, `unless` and `?:`); array and hash
  literals; interpolated strings; and nested blocks (`&:name` becomes
  one). Each node has its `line`;
- `captures`, the values the `Capture` nodes stand for;
- `proc`, the block as a closure, and `file`, where it is written.

A small `Relation` that turns its blocks into SQL, and filters an array
with the same blocks:

```quarz
data Post = (title: String, views: Int, published: Bool)

# SQL for a block's tree: a call with no arguments reads a column (of
# `self` or of the block's parameter), and a captured value is a literal.
def to_sql(e: Meta::Expr, captures: Array[Object]) -> String
  case e
  in Meta::Expr::Block(body: body) then to_sql(body[0], captures)
  in Meta::Expr::Lit(value: v) then v.inspect
  in Meta::Expr::Capture(index: i) then captures[i].inspect
  in Meta::Expr::Call(recv: r, method: m, args: args)
    if r && !args.empty?
      op = m.name == :== ? "=" : m.name.to_s
      "#{to_sql(r, captures)} #{op} #{to_sql(args[0], captures)}"
    else
      m.name.to_s
    end
  in Meta::Expr::And(left: l, right: r)
    "(#{to_sql(l, captures)} AND #{to_sql(r, captures)})"
  in Meta::Expr::Or(left: l, right: r)
    "(#{to_sql(l, captures)} OR #{to_sql(r, captures)})"
  in Meta::Expr::Not(operand: o) then "NOT #{to_sql(o, captures)}"
  else raise "line #{e.line}: no SQL for this"
  end
end

data Selection = (sql: String, values: Array[String])

class Relation
  @rows: Array[Post]
  @conds: Array[String]

  def initialize(@rows: Array[Post], @conds: Array[String] = []); end

  # The tree gives the SQL; `proc`, the block as a closure, filters the
  # rows in memory.
  def where(&cond: Quoted[Post.() -> Bool]) -> Relation
    rows = @rows.select { |p| p.instance_exec(&cond.proc) }
    Relation.new(rows, @conds + [to_sql(cond.expr, cond.captures)])
  end

  def filter(&cond: Quoted[(Post) -> Bool]) -> Relation
    rows = @rows.select { |p| cond.proc.call(p) }
    Relation.new(rows, @conds + [to_sql(cond.expr, cond.captures)])
  end

  def map(&f: Quoted[(Post) -> String]) -> Selection
    column = to_sql(f.expr, f.captures)
    Selection.new("SELECT #{column} FROM posts#{where_sql}",
                  @rows.map { |p| f.proc.call(p) })
  end

  def sql -> String = "SELECT * FROM posts#{where_sql}"
  def to_a -> Array[Post] = @rows

  private def where_sql -> String =
    @conds.empty? ? "" : " WHERE #{@conds.join(" AND ")}"
end

posts = Relation.new([
  Post.new("Hello", 1500, true),
  Post.new("Draft", 3000, false),
  Post.new("Notes", 20, true),
])
min = 1000
popular = posts.where { published && views > min }
puts popular.sql
# => SELECT * FROM posts WHERE (published AND views > 1000)
p popular.to_a.map(&:title)     # => ["Hello"]

# A for comprehension is calls of filter and map with blocks, so it is a
# query too, with no new grammar.
s = for p in posts when p.views > min do p.title end
puts s.sql     # => SELECT title FROM posts WHERE views > 1000
p s.values     # => ["Hello", "Draft"]
```

In `where`, the block's type has a receiver (`Post.() -> Bool`), so
`published` is a call on the block's `self`; `min` is a `Capture`, read
when `where` is called.

- A quoted block **captures by value at the call**: after
  `n = 1; q = r.where { |x| x == n }; n = 2`, the tree's capture is `1`,
  and `q.proc` sees `1` too.
- Only a block literal makes a `Quoted`: passing a procedure (`&f`) or
  `&:name` for such a parameter is an error; `&q` passes a `Quoted` on
  to another `Quoted` parameter, and `&q.proc` its closure to a
  procedure one.
  Inside the method the parameter is an ordinary value: `yield` to it is
  an error, and `cond.proc.call(...)` runs it.
- What the tree holds is all a quoted block may contain (with `for`
  comprehensions, which are calls). Anything else (assignments, loops,
  `case`, `return`, `raise`, ranges, keyword and splat arguments,
  lambdas...) is an error at its node:

```quarz,check=error
def keep(&cond: Quoted[(Int) -> Bool]) -> Bool = cond.proc.call(1)

keep { |x| (1..3).include?(x) }
# error: a range cannot be quoted: a block given as a `Quoted` holds
# literals, variables, `self`, calls, `&&`, `||`, `!`, `if` and blocks
```

- `Quoted[P]` is Shareable when `P` is a `shareable` procedure type
  (`&cond: Quoted[shareable Post.() -> Bool]`), whose block captures only
  Shareable values: a query type that keeps such blocks, and is itself
  Shareable, can then be a constant.

## Values computed while compiling: `static(call)`

`static(call)` runs one call of a `static def` while the program is
compiled and puts the value in the program where the expression is,
anywhere an expression goes: in a method, at the top level, in a
constant or a field's default. As with a `static` statement, the
`static def` comes from a dependency (in a script, a required file), so
the example is two files again:

```quarz,file=tables
# tables.qz
module Tables
  # The primes below `n`.
  static def self.primes(n: Int) -> Array[Int]
    (2...n).select { |i| (2...i).none? { |d| i % d == 0 } }
  end
end
```

```quarz
require_relative "tables"

PRIMES = static(Tables.primes(100))   # computed while compiling

def prime?(n: Int) -> Bool = PRIMES.include?(n)

p [PRIMES.size, prime?(97), prime?(91)]   # => [25, true, false]
```

- The receiver and positional arguments are constants: literals (and
  arrays, hashes and ranges of them), constants, classes, enum values
  and other `static(...)`; a chain is nested, `static(static(a).b)`. A
  block given to the call may read constants but no variable, `self` or
  field of the code around it.
- The result must be a value that can be baked into the program, as a
  generated method's captures are: a literal, a class, an enum variant,
  `data`, and arrays, hashes and ranges of those. A procedure or an
  object of an ordinary class is an error.
- It runs under the same restrictions as a `static` statement, each time
  the program is checked (by `check`, `run` or `build`); what it raises is an error at the expression, and what it
  prints goes to the compiler's standard error.

A variable as an argument, or a `static def` of the same file, is an
error:

```quarz,check=error
require_relative "tables"

n = 100
small = static(Tables.primes(n))
# error: `static(...)`'s receiver and arguments are constants: literals,
# constants, classes and `static(...)`
```

A query's SQL computed while compiling is then up to the ORM: a
`static def` that builds it from constant arguments can be called
through `static(...)`, and the language adds nothing for SQL itself.

## Not included

Decided against:

- Classes built at run time, macros over untyped syntax, and code
  assembled from strings: the class hierarchy stays fixed for the whole
  program, and everything that runs at compile time is checked code.
- Constant-folding of arbitrary methods: only a `static def` runs at
  compile time, from a `static` statement or `static(...)`.
- Generated fields without a default: `def_field` always takes one, so
  that every `initialize` stays complete.

Later:

- A module's `static def` used through `extend`, as a generic module
  whose parameter is the extending class (`module Macros[T <: Model]`,
  `extend Macros[Post]`).
- Reading files at compile time (`Meta.read_file`, for schemas), an
  order between the `static` statements of one cog decided on demand,
  generated nested types (`def_data`), `extend_type`, and an `inherited`
  hook.

## Further reading

The specification is
[docs/14-metaprogramming.md](https://github.com/quarz-lang/quarz/blob/main/docs/14-metaprogramming.md)
in the [repository's design documents](https://github.com/quarz-lang/quarz/tree/main/docs),
next to [Syntax](https://github.com/quarz-lang/quarz/blob/main/docs/01-syntax.md),
whose §3 and §7 state the rules this builds on.
