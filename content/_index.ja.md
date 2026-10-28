+++
title = "Quarz"
template = "index.html"

[extra]
tagline = "Ruby のような書き心地で、静的型付け・不変性・並行性の上に築かれたプログラミング言語。"
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

Ruby との違いは 3 か所だけです。引数と戻り値の型(`text: String`、
`-> Hash[String, Int]`)、`data` による不変のレコード、そしてタスク
(`Task.scope`、`spawn`、`await`)。残りは Ruby そのものです。

## 特徴

- **Ruby の文法と組み込みクラス。** `def ... end`、ブロックと `yield`、
  `case`/`in`、`#{}`、`Enumerable` と `Comparable`、例外処理は Ruby の
  ままです。
- **静的なクラスとメソッド。** クラスの再オープン、`define_method`、
  `method_missing`、`eval` はなく、メソッド呼び出しはコンパイル時に
  解決されます。既存の型への追加は、字句スコープに閉じた
  `extension`/`using` で行います。
- **既定で不変のデータ。** `String`・`Array`・`Hash`・`Set` は不変で、
  更新は新しい値を返します。参照が一意ならその場で更新されるので、
  素直なループが可変配列と同程度に速く動きます。可変が必要な場面には、
  Ruby と同じ API の Builder があります。
- **軽量タスク。** goroutine のように M:N でスケジュールされる `Task`、
  型付きの `Channel`、構造化並行性(`Task.scope`)、不変値を共有する
  `Atom`。可変オブジェクトがタスクをまたがないことを型(`Shareable`)で
  検査するので、データ競合はコンパイル時に排除されます。
- **インタプリタとコンパイラ。** `quarz run` はその場で実行し、REPL も
  使えます。`quarz build` は LLVM を通してネイティブの実行ファイルに
  コンパイルします。両者は 1 つの中間表現と 1 つのランタイムを共有し、
  挙動は同一で、違うのは速さだけです。

## さらに知る

- ソースから Quarz を[インストール](@/install.ja.md)する。
- [ドキュメント](@/docs/_index.ja.md)を読む。まずは[文法](@/docs/syntax.ja.md)と
  [Ruby との違い](@/docs/ruby-diff.ja.md)から。
- [標準ライブラリ](/std/)(英語)を見る。
- 開発は [GitHub](https://github.com/quarz-lang/quarz) で。
