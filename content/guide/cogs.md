+++
title = "Cogs"
description = "Using and publishing cogs, Quarz's packages: the manifest, the lock, minimal version selection, tests and local development."
weight = 7
+++

<!-- Kept in step with quarz-lang/quarz's main (docs/10-cogs.md is the
     specification). The walk below is what quarz's weekly job
     (cogs-network.yml) runs against the real repositories; the Quarz
     blocks depend on those cogs, so they are check=none. -->

A **cog** is a Quarz package: a git repository (or a directory in one)
with a `Cog.toml` and its source under `src/`. An application names the
cogs it uses in its own `Cog.toml`; `quarz` fetches them, resolves their
versions, records the result in `Cog.lock`, and compiles everything
together. There is no registry: a cog is identified by its git URL, and
a version is a tag. This page walks through it with two small cogs,
[quarz-lang/greet](https://github.com/quarz-lang/greet) and
[quarz-lang/greet-more](https://github.com/quarz-lang/greet-more). The
specification is
[docs/10-cogs.md](https://github.com/quarz-lang/quarz/blob/main/docs/10-cogs.md).

## An application

`quarz cog init` makes the directory an application: a `Cog.toml` and
`src/main.qz`, the file whose statements run.

```sh
$ mkdir hello && cd hello
$ quarz cog init
wrote Cog.toml for `hello` (src/main.qz is the application's main file)
$ quarz run
Hello from hello!
```

`quarz cog add` adds a dependency, by its URL and a version tag, and
fetches it:

```sh
$ quarz cog add https://github.com/quarz-lang/greet --tag v1.0.0
quarz: looking up https://github.com/quarz-lang/greet (tag v1.0.0)
quarz: fetching greet 1.0.0
added `Greet` to [dependencies] (tag = "v1.0.0")
```

```toml
# Cog.toml
[cog]
name = "hello"
version = "0.1.0"
quarz = "0.0"

[dependencies]
Greet = { git = "https://github.com/quarz-lang/greet", tag = "v1.0.0" }
```

The key, `Greet`, is the constant the application sees the cog under:
everything the cog declares is inside that module. There is no
`require`: every file of every cog is loaded, so its classes are simply
there.

```quarz,check=none
# src/main.qz
class Team < Greet::Greeter
  def initialize; super("team"); end
  greets :world      # greet's macro: defines Team#greet_world
end

puts Greet::Greeter.new("Quarz").hello   # => Hello, Quarz!
puts Team.new.greet_world                # => Hello, world!
```

`greets` is a `static def` of greet's `Greeter`: it runs while the
application is compiled and generates a method (see
[Metaprogramming](@/guide/metaprogramming.md)). A dependency's macros
can be used this way; a cog's own cannot be called from its own files,
since the compiler would have to run code it is still checking.

The usual commands work on the application around the current
directory:

```sh
$ quarz check
$ quarz run
Hello, Quarz!
Hello, world!
$ quarz build          # writes the executable ./hello
$ ./hello
Hello, Quarz!
Hello, world!
```

## The lock

`quarz cog add` (or the first command that needs the dependencies)
writes `Cog.lock`: for every cog, the version chosen, the commit, and a hash of its files.
Commit it with the application: the next build, on any machine, uses
exactly these commits and refuses a tree whose hash differs (a moved
tag, a rewritten history).

```toml
# Cog.lock (generated; its header left out)
[[cog]]
name = "greet"
module = "Greet"
source = "git+https://github.com/quarz-lang/greet"
version = "1.0.0"
rev = "6c8ae85f0446199c1ec729cdff127e690ae4878a"
hash = "h1:+nCjhZa2dOYkCgqBiruK6PWkfubGMLCXClUxDRI8PSY="
dependencies = []
```

Fetched cogs are kept in a cache (`$QUARZ_CACHE`, else
`$XDG_CACHE_HOME/quarz`, else `~/.cache/quarz`), so a second run looks
nothing up. The lock changes only when the dependencies listed in
`Cog.toml` (or in a patched cog's, below) change, or with
`quarz cog update`; `--locked` makes a build fail instead of rewriting
it (for CI), and `--offline` fetches nothing.

## Versions

A cog asks for each dependency by the lowest version it needs: its tag.
When two cogs ask for the same one, Quarz takes the higher of the two
lowest versions, never a newer one that nobody asked for (Go's
*minimal version selection*): the result depends on the manifests
alone, with no solver and no surprise upgrade.

greet-more needs greet v1.1.0, which added `Greeter#bye`. Adding it to
the application, which asked for v1.0.0, moves greet to v1.1.0:

```sh
$ quarz cog add https://github.com/quarz-lang/greet-more --tag v1.1.0
$ quarz cog tree
hello 0.1.0
  greet 1.1.0 (https://github.com/quarz-lang/greet)
  greet-more 1.1.0 (https://github.com/quarz-lang/greet-more)
    greet 1.1.0 (https://github.com/quarz-lang/greet) (*)
```

```quarz,check=none
puts GreetMore.farewell("Quarz")   # => Goodbye, Quarz. (greet-more 1.1.0)
```

Versions follow semantic versioning: two requirements must share the
major version (the minor one while the major is `0`), else the
resolution fails and names both. A `rev` in the application's own
manifest pins a dependency: one that needs a newer version is then an
error rather than an upgrade. `quarz cog outdated` lists newer tags.

## Tests

`quarz test` runs the application's tests, the files under `test/`,
which see its declarations as its own files do.

```quarz,check=none
# test/team_test.qz
require "test"

class TeamTest < Test::Case
  def test_world -> Void = assert_equal("Hello, world!", Team.new.greet_world)
end
```

```sh
$ quarz test
Loaded suite test
Started
.
Finished in 0.001152 seconds.

1 tests, 1 assertions, 0 failures, 0 errors, 0 pendings, 0 omissions, 0 notifications
```

A cog that only the tests use goes under `[dev-dependencies]` instead
(`quarz cog add URL --tag ... --dev`): the application's code does not
see it, and the application's users never fetch it. It is resolved with
the other dependencies, so a version it needs counts for `quarz run`
too.

```toml
[dev-dependencies]
Spec = { git = "https://github.com/example/spec", tag = "v0.2.0" }
```

## Writing a cog

A library cog is a repository with a `Cog.toml` and its files under
`src/`; greet's is

```toml
[cog]
name = "greet"
version = "1.1.0"
module = "Greet"
```

```quarz,check=none
# src/greeter.qz: everything here is inside `Greet`
class Greeter
  @name: String
  def initialize(@name: String); end
  def hello -> String = "Hello, #{@name}!"
  def bye -> String = "Goodbye, #{@name}."

  static def self.greets(name: Symbol) -> Void
    def_method("greet_#{name}".to_sym) do -> String
      Greeter.new(name.to_s).hello
    end
  end
end
```

- A cog's files hold declarations only (classes, modules, methods,
  constants): loading a cog runs nothing.
- Its top-level `def`s are functions of the cog, visible in all its files
  and its tests, and private to it. A function for its users is a
  `public def`: greet-more's
  ```quarz,check=none
  # src/farewell.qz
  public def farewell(name: String) -> String =
    Greet::Greeter.new(name).bye + " (#{cog.name} #{cog.version})"
  ```
  is called `GreetMore.farewell(...)` by the cogs that depend on it, a
  plain function call. Inside a cog, `cog` is its root module: `cog.name`
  and `cog.version` read its manifest, as `GreetMore.name` and
  `GreetMore.version` do outside.
- Its tests are under `test/` and see it as its users do, under its
  module (`Greet::Greeter`).
- A version is a tag `vX.Y.Z` on a commit; push the tag and users can
  depend on it. Keep `version` in `Cog.toml` the same: the tag decides
  the version, but `cog.version` reads the manifest.

## Several cogs at once

While developing a cog and an application together, `[patch]` replaces
a dependency by a local checkout, without a commit or a tag:

```toml
[patch]
Greet = { path = "../greet" }
```

A patch is the application's alone: a cog fetched from elsewhere may
not name a directory on this machine. The patched cog is not in the
lock while the patch is there, so after removing it, run once (or
`quarz cog update`) and commit the lock that brings it back; a build
with `--locked` would otherwise refuse the lock.

## What is refused

The errors a user meets say which requirement, or which cogs, and what
was wrong:

- a tag that does not exist, or a URL that does not answer (GitHub
  answers a repository that does not exist, or is private, by asking for
  credentials: "could not read Username ... terminal prompts disabled"
  means the URL is wrong, or needs your credentials set up for git);
- two versions of one cog whose majors differ, and a `rev` pin that a
  dependency outgrows;
- a cycle among cogs (`c1 -> c2 -> c1`): a cog's dependencies cannot lead
  back to it;
- a copy of the application among its dependencies.
