# quarz.dev

The website of the [Quarz](https://github.com/quarz-lang/quarz)
programming language, built with [Zola](https://www.getzola.org/)
(v0.23.6) and published to GitHub Pages by
`.github/workflows/deploy.yml`.

The site takes three things from a checkout of quarz, in `quarz/`:

- the language's documents, `docs/NN-name.md` and `.ja.md`, which
  `scripts/import_docs.qz` (written in Quarz, with its tests in
  `scripts/docs_test.qz`) turns into `content/docs/` (not committed),
  giving each heading the anchor GitHub gives it;
- the TextMate grammar `editors/textmate/quarz.tmLanguage.json`, which
  colors the ```` ```quarz ```` code blocks;
- the standard library's API reference, `quarz doc --std --html`, put
  under `/std/`.

## Building it locally

```sh
ln -s ../quarz quarz          # or: git clone https://github.com/quarz-lang/quarz.git
(cd quarz && cargo build --release --bin quarz)
quarz/target/release/quarz run scripts/import_docs.qz quarz "$(git -C quarz rev-parse HEAD)"
quarz/target/release/quarz test scripts/docs_test.qz
zola serve                    # or: zola build && quarz/target/release/quarz doc --std --html -o public/std
```

`zola build` fails on a broken internal link or anchor, which is how the
workflow checks the imported docs' links on every pull request.

The site uses no JavaScript, web fonts or analytics. Pages in English
are `name.md`, their Japanese versions `name.ja.md`.
