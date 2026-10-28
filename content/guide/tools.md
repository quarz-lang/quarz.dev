+++
title = "Tools"
description = "Running, compiling, testing and documenting programs, and packaging them as cogs."
weight = 5
+++

One command, `quarz`, does everything. See [Install](@/install.md) for
building it.

## Run and build

```sh
quarz run hello.qz            # type-check and run
quarz check hello.qz          # type-check only
quarz build hello.qz -o hello # compile to a native executable
quarz repl                    # an interactive session
```

`quarz run` interprets the program and starts at once; `quarz build`
compiles it through LLVM into an executable that runs much faster. Both
go through the same type checker and the same runtime, so a program
behaves the same either way.

Errors are reported at compile time with the line and the reason, as
for a type mismatch or a `case` that misses a variant:

```text
app.qz:7:12: error: argument `name` of greet(name: String, times: Int = ...) -> String expects String but found String?
  puts greet(names.first, total)
             ^^^^^^^^^^^
```

## Exceptions

Exceptions are Ruby's: `raise`, `rescue`, `ensure`, `retry`, with the
exception classes of the standard library.

```quarz
begin
  Integer("x")
rescue ArgumentError => e
  puts "not a number: #{e.message}"
ensure
  puts "done"
end
```

## Tests

`quarz test` runs the tests of a cog or a script. A test is a `test_`
method of a class that inherits `Test::Case`, in the shape of Ruby's
Test::Unit, with typed assertions. The compiler finds the tests, and
`--parallel` runs them in parallel, which is safe since mutable state
never crosses a task.

```quarz,check=test
require "test"

class PushTest < Test::Case
  def test_push_returns_a_new_array
    xs = [1, 2, 3]
    assert_equal([1, 2, 3, 4], xs.push(4))
    assert_equal([1, 2, 3], xs)
  end
end
```

## Documentation

A comment written directly above a declaration is its documentation, in
Markdown. A code block in it is an example that `quarz test` compiles
and runs, so examples cannot go stale. `quarz doc Array#push` prints a
name's documentation in the terminal, and `quarz doc --html` generates
a site like [the standard library's](/std/).

## Cogs

A package is a *cog*: a git repository with a `Cog.toml` at its root
and its sources under `src/`. There is no registry; a dependency is
named by its repository and pinned to a commit in `Cog.lock`.

```toml
[cog]
name    = "webkit"
version = "0.3.1"

[dependencies]
Httpx = { git = "https://github.com/foo/httpx", tag = "v1.4.0" }
```

A cog's declarations live in its own module (`Httpx::Client`), with no
`require` needed. `quarz cog add` writes a dependency into the manifest,
and `quarz cog update` re-resolves the lock file.

## Further reading

The design documents have the details:
[Execution](https://github.com/quarz-lang/quarz/blob/main/docs/05-execution.md)
for `quarz run` and `quarz build`,
[Testing](https://github.com/quarz-lang/quarz/blob/main/docs/12-testing.md),
[Documentation Comments](https://github.com/quarz-lang/quarz/blob/main/docs/13-doc-comments.md)
and [Cogs](https://github.com/quarz-lang/quarz/blob/main/docs/10-cogs.md).
