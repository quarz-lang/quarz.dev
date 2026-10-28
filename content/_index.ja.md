+++
title = "Quarz"
template = "index.html"

[extra]
tagline = "Ruby のような書き心地と、静的型付け・不変性・並行性を備えたプログラミング言語。"
+++

```quarz
data WordCount(word: String, count: Int)
  include Comparable[WordCount]
  def <=>(other: WordCount) -> Int = other.count <=> count
end

def count_words(text: String) -> Hash[String, Int]
  text.downcase.scan(/[a-z']+/).tally
end

# ファイルごとに軽量タスクを起動し、すべての終了を待つ
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

Ruby と違うのは、型(`text: String`、`-> Hash[String, Int]`、
`Comparable[WordCount]`)、`data` による不変のレコード、タスク
(`Task.scope`、`spawn`、`await`)の 3 点だけです。それ以外は Ruby の
ままです。

## 特徴

- **Ruby の文法と組み込みクラス。** `def ... end`、ブロックと `yield`、
  `case`/`in`、`#{}`、`Enumerable` と `Comparable`、例外処理は Ruby の
  ままです。
- **静的なクラスとメソッド。** クラスの再オープン、`define_method`、
  `method_missing`、`eval` はなく、メソッド呼び出しはコンパイル時に
  解決されます。既存の型を拡張するには、効果が字句スコープに限られる
  `extension`/`using` を使います。
- **データは既定で不変。** `String`・`Array`・`Hash`・`Set` は不変で、
  更新は新しい値を返します。参照が一意ならその場で更新されるので、
  ループで更新を繰り返しても、計算量は線形に収まります(1 回の更新は
  償却 O(1))。可変のデータが必要な場面には、Ruby と同じ API の Builder
  があります。
- **軽量タスク。** goroutine のように M:N でスケジュールされる `Task`、
  型付きの `Channel`、構造化並行性(`Task.scope`)、不変の値を入れて
  タスク間で共有できる `Atom`。可変オブジェクトがタスクをまたがない
  ことを型(`Shareable`)で検査するので、データ競合はコンパイル時に
  排除されます。
- **インタプリタとコンパイラ。** `quarz run` はプログラムをすぐに実行し、
  `quarz repl` で対話的にも使えます。`quarz build` は LLVM を使って
  ネイティブの実行ファイルにコンパイルします。両者は中間表現と
  ランタイムを共有し、同じ結果を出します。テストスイートはすべての
  サンプルと 100 を超えるテストプログラムを両方で動かし、出力を
  比べています。

## 詳しく知る

- ソースから Quarz を[インストール](@/install.ja.md)する。
- [ドキュメント](@/docs/_index.ja.md)を読む。まずは[文法](@/docs/syntax.ja.md)と
  [Ruby との違い](@/docs/ruby-diff.ja.md)から。
- [標準ライブラリ](/std/)(英語)を見る。
- 開発は [GitHub](https://github.com/quarz-lang/quarz) で行っています。
