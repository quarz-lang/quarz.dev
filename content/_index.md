+++
title = "Quarz"
template = "index.html"

[extra]
tagline = "A programming language that feels like Ruby, built on static typing, immutability and concurrency."
+++

```quarz
data WordCount(word: String, count: Int)
  include Comparable[WordCount]
  def <=>(other: WordCount) -> Int = other.count <=> count
end

def count_words(text: String) -> Hash[String, Int]
  text.downcase.scan(/[a-z']+/).tally
end

# Start a lightweight task per file and wait for all of them
counts = Task.scope do |scope|
  ARGV.map { |path| scope.spawn { count_words(File.read(path)) } }
      .map(&:await)
end

total = counts.reduce({}) { |acc, h| acc.merge(h) { |_, a, b| a + b } }

total.map { |word, count| WordCount.new(word, count) }
     .sort
     .first(10)
     .each { |wc| puts "#{wc.word}: #{wc.count}" }
```

In the code above, only three things differ from Ruby: types
(`text: String`, `-> Hash[String, Int]`, `Comparable[WordCount]`),
immutable records with `data`, and tasks (`Task.scope`, `spawn`,
`await`). The rest is Ruby.

## Features

- **Ruby's syntax and built-in classes.** `def ... end`, blocks and
  `yield`, `case`/`in`, `#{}`, `Enumerable` and `Comparable`, and
  exceptions are as they are in Ruby.
- **Static classes and methods.** No reopening of classes, no
  `define_method`, `method_missing` or `eval`: calls are resolved at
  compile time. Existing types are extended with `extension`/`using`,
  confined to a lexical scope.
- **Immutable data by default.** `String`, `Array`, `Hash` and `Set` are
  immutable, and an update returns a new value; when the value has a
  single reference it is updated in place, so a loop that updates a
  collection stays linear (amortized O(1) per update). Builders give
  Ruby's mutable API where it is needed.
- **Lightweight tasks.** `Task`s scheduled M:N like goroutines, typed
  `Channel`s, structured concurrency (`Task.scope`) and `Atom`s that share
  immutable values between tasks. The type system (`Shareable`) keeps
  mutable objects from crossing tasks, so data races are ruled out at
  compile time.
- **An interpreter and a compiler.** `quarz run` runs a program at once,
  and `quarz repl` gives a REPL; `quarz build` compiles it to a native
  executable through LLVM. Both share one intermediate representation
  and one runtime, and give the same results: the test suite runs every
  example and over a hundred test programs on both and compares their
  output.

## Learn more

- [Install](@/install.md) Quarz from source.
- Read the [guide](@/guide/_index.md): the language in six short pages.
- Browse the [standard library's API reference](/std/).
- Follow the development on [GitHub](https://github.com/quarz-lang/quarz).
