# quarz.dev

The website of the [Quarz](https://github.com/quarz-lang/quarz)
programming language, built with [Zola](https://www.getzola.org/)
(v0.23.6) and published to GitHub Pages by
`.github/workflows/deploy.yml`.

The site takes two things from a checkout of quarz, in `quarz/`:

- the TextMate grammar `editors/textmate/quarz.tmLanguage.json`, which
  colors the ```` ```quarz ```` code blocks;
- the `quarz` command, which runs the Guide's examples.

The standard library's API reference is not part of this site: quarz
publishes it at <https://api.quarz.dev/>. `/std/`, where it used to be,
is a page that sends its visitors there.

The pages themselves (the top page, Install and the Guide) are written
here, in `content/`. The language's design documents stay in quarz's
`docs/`; the Guide explains the language in short pages for its users
and links there for the details.

## Building it locally

```sh
ln -s ../quarz quarz          # or: git clone https://github.com/quarz-lang/quarz.git
(cd quarz && cargo build --release --bin quarz)
zola serve                    # or: zola build
```

`zola build` fails on a broken internal link or anchor, which the
workflow runs on every pull request. The Guide's ```` ```quarz ````
code blocks are programs, and `scripts/check_guide.qz` (written in
Quarz) runs each of them: run it after a change, as the workflow does on
every pull request and weekly, so that a change to the language that
leaves an example behind is noticed.

```sh
quarz/target/release/quarz run scripts/check_guide.qz quarz/target/release/quarz
```

A block is run by default; ```` ```quarz,check=test ```` runs it as a
test file, ```` ```quarz,check=error ```` expects the type checker
to reject it, and ```` ```quarz,check=none ```` skips the block (a
sketch of a feature not in the language yet). A block marked
```` ```quarz,file=NAME ```` is a library rather than a program: it must
pass `quarz check`, and it is saved as `NAME.qz` so that the page's
later blocks can `require_relative "NAME"` it. That is how an example
shows a library in a file of its own, which the `static def`s a class
body calls have to be. Any other option on a ```` ```quarz ```` fence
is an error.

The site uses no JavaScript, web fonts or analytics.
