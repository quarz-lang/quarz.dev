+++
title = "インストール"
template = "page.html"
+++

Quarz のリリースはまだありません。ソースからビルドします。

## 必要なもの

- [Rust](https://www.rust-lang.org/ja/tools/install) 1.96 以降と Cargo。
- clang 14 以降(`quarz build` に必要です。`quarz build` は LLVM IR を
  生成し、clang でコンパイルします)。`quarz run` には要りません。
- x86-64 の Linux。CI はこの環境でだけ実行しており、ほかの OS では
  検証していません。

## ビルド

```sh
git clone https://github.com/quarz-lang/quarz.git
cd quarz
cargo build --release
```

できあがったコマンドは `target/release/quarz` にあります。そのディレクトリを
`PATH` に加えるか、`PATH` にあるディレクトリにコピーします。

## プログラムを動かす

```sh
quarz run examples/hello.qz        # 型検査して実行
quarz check examples/shapes.qz     # 型検査だけ
quarz build examples/shapes.qz -o shapes && ./shapes   # ネイティブの実行ファイルにコンパイル
quarz repl                         # 対話セッション
```

`quarz run` で実行しても、`quarz build` でコンパイルしても、
プログラムの動作は同じです。インタプリタはすぐに起動し、実行ファイルは
より速く動きます。`quarz test` は cog のテストを実行し、`quarz doc` は
名前を指定してドキュメントを表示します。

次は[ドキュメント](@/docs/_index.ja.md)へ。
