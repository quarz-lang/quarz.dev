+++
title = "インストール"
template = "page.html"
+++

Quarz にはまだリリースされた版がありません。ソースからビルドします。

## 必要なもの

- [Rust](https://www.rust-lang.org/ja/tools/install) 1.96 以降と Cargo。
- clang 14 以降(`quarz build` に必要です。コンパイルしたプログラムは
  LLVM IR を通ります)。`quarz run` には要りません。
- 開発とテストは Linux(x86-64)で行っています。

## ビルド

```sh
git clone https://github.com/quarz-lang/quarz.git
cd quarz
cargo build --release
```

コマンドは `target/release/quarz` です。そのディレクトリを `PATH` に
加えるか、`PATH` にあるディレクトリにコピーします。

## プログラムを動かす

```sh
quarz run examples/hello.qz        # 型検査して実行
quarz check examples/shapes.qz     # 型検査だけ
quarz build examples/shapes.qz -o shapes && ./shapes   # ネイティブの実行ファイルにコンパイル
quarz repl                         # 対話セッション
```

プログラムは `quarz run` でも `quarz build` でコンパイルしても同じに
動きます。インタプリタはすぐに起動し、実行ファイルはより速く動きます。
`quarz test` は cog のテストを実行し、`quarz doc` は名前のドキュメントを
表示します。

次は[ドキュメント](@/docs/_index.ja.md)へ。
