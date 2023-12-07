---
title: "Dart のドキュメンテーションについて"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: true
published_at: 2023-12-06 00:00
publication_name: team_soda
---

## ＼[スニダンを開発している SODA inc.の Advent Calendar](https://qiita.com/advent-calendar/2023/soda-inc) 7 日目の記事です!!!／

昨日は [@rinchsan](https://twitter.com/rinchsan) さんによる[「pprotein でボトルネックを探して ISUCON で優勝する」](https://zenn.dev/team_soda/articles/20231206000000)でした！

明日は [@miyapana](https://qiita.com/kanata333) さんによる記事です！

お楽しみに！

https://qiita.com/advent-calendar/2023/soda-inc

## はじめに

Dart や Flutter のエンジニア（またはこの記事を読むその他のエンジニア）の皆さんは、普段どのくらい、どのように Dart のドキュメンテーションを書いていますか？Dart のドキュメンテーションについて調べたり考えたりしたことはありますか？ドキュメンテーションについてどんなことを知っていますか？

この記事では Effective Dart: Documentation

https://dart.dev/effective-dart/documentation

を参照しながら Dart のドキュメンテーションについての知識を深めます。

また、これまであまり Dart のドキュメンテーションを書いてこなかった方や、書く意義や書き方について考えることがあまりなかった方にとって、その意識を変えるきっかけになれば良いなと思います！

## Effective Dart 冒頭

Effective Dart: Documentation: の冒頭を引用します。

> It’s easy to think your code is obvious today without realizing how much you rely on context already in your head. People new to your code, and even your forgetful future self won’t have that context. A concise, accurate comment only takes a few seconds to write but can save one of those people hours of time.
>
> We all know code should be self-documenting and not all comments are helpful. But the reality is that most of us don’t write as many comments as we should. It’s like exercise: you technically can do too much, but it’s a lot more likely that you’re doing too little. Try to step it up.

まとめると下記のようになるでしょうか。

> いま眼の前にある自分の書いたコードをはっきりと理解できるように思えていても、それは既にあなたの頭の中にあるコンテキストに大いに依存していることを忘れがちです。
> そのコードに新しく触れるメンバーや、将来の自分自身はそのコンテキストを持ち合わせてはいないでしょう。簡潔で正確なコメントは数秒で書くことができますが、それが他の誰かのまたは将来の自分の何時間もの時間を節約することになるかもしれません。
>
> コードはそれ自体が自己説明的であるべき、つまりコメントやドキュメンテーション無しでも十分に分かりやすいものであるべきではありますがし、全てのコメントが役に立つわけでもないかもしれませんが、しかし、現実には、ほとんどの人が本来書くべきコメントを書いていないのが現状です。それは運動と同じで、やり過ぎることは可能ですが、実際には十分に行っていないことの方がずっと一般的です。もう少しコメントを書くように心掛けてみましょう。

「十分に洗練された設計で責務がはっきりしていて命名が完璧なコードを書くことができれば、理想的には、コメントなど必要ない」という考えも一理あるとも思いますが、そのようなことは現実にはほとんどありません。

コメントにかかる数秒〜数分の労力は、その何十倍・何百倍分ものチームメンバーや将来の自分を助けてくれることになります。

## comment と doc comment

Effective Dart では次に comment と doc comment との違いを説明しています。

comment は `//` ではじめて、doc comment としてドキュメンテーション化はしない内容、たとえばその処理や該当行の文脈を補足するような内容を書くと良いでしょう。

doc comment は `///` ではじめます。ただの comment とは全く質の異なるドキュメンテーションコメントです。

たとえば皆さんのエディタや IDE で、あるクラスや関数にカーソルを合わせると下記のような吹き出しが現れます。

![doc comment](/images/articles/dart-documentation.dart/doc_comment.png)

上記は `StatelessWidget` にカーソルを当てた際に現れる文章ですが、これがまさに doc comment です。

試しに `StatelessWidget` を command + クリックしてコードジャンプしてみると分かりますが、そのような Flutter SDK が提供する API や特にその抽象的な実装は、もはや Dart のソースコードの分量の何倍・何十倍の量のドキュメンテーションが書かれていることが分かります。

皆さんも開発中には常に、Flutter や Dart の SDK や使用しているパッケージのクラス・メソッド・変数名などを command + クリックして、その内部実装を読むと同時に、ドキュメンテーションコメントを読むことでしょう。

また、<https://api.flutter.dev> で始まる下記のような Web ページ：

https://api.flutter.dev/flutter/widgets/StatelessWidget-class.html

を見かけ、参考にすることもよくあるでしょう。

実はこれは、

```shell
dart doc
```

という Dart のドキュメンテーション生成のコマンドを実行されるものです。

`dart doc` コマンドは、パッケージ内の `///` で書かれた doc comment に対して、美しいリファレンスの Web ページを生成します。

以上のようなことから、`///` で書く doc comment は、私たちもやりがちな雑なメモ書き程度のコメントとは全く質の異なる、とても重要なドキュメンテーションであることを理解できると思います。

また、私たちは毎日の開発の中でそれらの恩恵を絶えず受けていることに思いを馳せましょう。

## ドキュメンテーションは正しいことばで書く

さて、ドキュメンテーションの重要性や、普段自分が受けている恩恵について理解が深まったところですが、皆さんは普段どのようにドキュメンテーションを書いているでしょうか？またはこれからどのように書くと良いでしょうか？

いくつかの具体的な作法は後述しますが、ここではまず「正しいことばで書く」ということを強調しておきます。

Effective Dart でも下記のように書かれています。

> Capitalize the first word unless it’s a case-sensitive identifier. End it with a period (or “!” or “?”, I suppose). This is true for all comments: doc comments, inline stuff, even TODOs. Even if it’s a sentence fragment.

翻訳すると、

> 最初の単語は大文字で始めること。ただし、ケースセンシティブな識別子の場合は除く。コメントはピリオド（または「！」や「？」といった記号）で終わらせる。これはすべてのコメントに当てはまる：ドキュメントコメント、インラインのもの、TODOさえも。たとえ文の断片であってもそうする。

です。

上記ではわざわざ

- 大文字で始めること
- ピリオドやクエスチョンマークで終わること
- doc comment だろうとインラインコメントだろうと TODO コメントだろうとすべてそうであるべきであること

などの従うべき方針が細かく説明されています。

日本語でドキュメンテーションを書く場合には、主語・述語のはっきりとした内容で、英語がピリオドで終わるようにと言っているのだから日本語では句点「。」で終わるような、正しいことばで書くべきでしょう。

皆さんの書くドキュメンテーション（コメント）は正しいことばで書けているでしょうか？主語の抜けた、カタコトの、雑で正しくない日本語にはなっていないでしょうか？

Flutter, Dart SDK や著名なパッケージのドキュメンテーションは例外なく正しいことばで書かれています。

たとえば、句点「。」で終わるかどうかまでを厳密にチームの規約としたりレビュー基準にしたりするのは少し堅苦しいかもしれませんが、繰り返し、「ドキュメンテーションは雑なメモ書き程度のものとは違う」ということが十分に理解できれば、今日から書くべきドキュメンテーションの作法は、より正しく美しいものになるでしょう。

## (public) API を書いているのだという意識を持つ

「Effective Dart 冒頭」を読むだけでは、コメントを書くということは「今はちょっと面倒だけど、将来の助けになるからやるべきもの」のような解釈に終わるかもしれません。そんな「面倒な部屋の片付け」くらいの認識では、コメントを書くモチベーションも上がらないかもしれません。

そこで、あなたは仕事やその他のプロジェクトでコードを書いている時には同時に「(public) API を書いているのだ」という意識を持つと良いでしょう。

Dart の lint ルールに `public_member_api_docs` というものがあります。

https://dart.dev/tools/linter-rules/public_member_api_docs

これは、すべての public メンバーにドキュメンテーションを書くことを強制するルールです。

public な、つまり `_` から始まらないすべてのクラス、メソッド・関数、変数に対して doc comment を書くことを強制します。

**BAD:**

```dart
class Bad {
  void meh() { }
}
```

**GOOD:**

```dart
/// A good thing.
abstract class Good {
  /// Start doing your thing.
  void start() => _start();

  _start();
}
```

さすがに仕事で開発しているアプリケーションコード（例えば snkrdunk プロジェクト）にこのルールを適用するのは難しいかもしれませんが、実は皆さんが使っている著名な OSS パッケージの多くでこの lint ルールは採用されています。

たとえば Riverpod のプロジェクトでは、下記のようにルートに配置し Dart の全 lint ルールを列挙した `all_lint_rules.yaml`:

https://github.com/rrousselGit/riverpod/blob/e626a0ebf71587be63ad663d80521d22407895c0/all_lint_rules.yaml#L160

に含まれる `public_member_api_docs` のルールを、

`analysis_options.yaml`:

https://github.com/rrousselGit/riverpod/blob/master/analysis_options.yaml

で `false` にすることなく使用しています。

つまり、Riverpod のパッケージ内のコードでは、すべての public メンバーに API ドキュメンテーションが施されているということです。

※ ちなみに Riverpod パッケージの中でも、アプリケーションコードに相当する `example` プロジェクトでは `public_member_api_docs` を無効化しています。

https://github.com/rrousselGit/riverpod/blob/e626a0ebf71587be63ad663d80521d22407895c0/packages/riverpod/example/analysis_options.yaml#L13

たとえば私が公開している `geoflutterfire_plus` パッケージでも同様の構成をしています。

https://github.com/kosukesaigusa/geoflutterfire_plus

さて、Dart の lint ルールに `public_member_api_docs` というものが存在することや、著名な OSS パッケージの多くで採用されていることを知ったことで、これを今日から眼の前のアプリケーションコードに適用するのは難しいかもしれませんが、API ドキュメンテーションを書く意義をイメージしやすくなったのではないでしょうか。

たとえば皆さんが仕事で開発するアプリの「〇〇機能の開発タスク」として書くコードは、言い換えれば「〇〇機能の Dart の (public) API を実装している」ということです。

それが重要なまたは汎用的な機能であればあるほど、他の色々なファイルから呼び出されることでしょう。

時間軸の観点でも、チームメンバーや自分の将来の別の開発タスクで、そのとき書いた〇〇機能を依存・活用したり、もしくはデグレが起こらないように気をつける対象として認識したりすることになります。

設計も責務も命名も実装内容も完璧で、さらにその動作を保証するユニットテストも書かれていた上で、さらに実装意図や文脈、使い方などの知識も補足しているドキュメンテーションコメントがそこにあったならば、それを参照することになった未来のチームメンバーや自分がとても嬉しく、安心して素早く開発ができるはずです。

逆に、責務も命名もイマイチでユニットテストも書かれておらず、実装意図や文脈、使い方を補足する情報もないコード、つまり負債と言うべきコードに気を配りながら、思わぬところで不具合が発生しないように怖怖と開発せざるを得なかったような経験も、多くの方経験してきていると思います。

ドキュメンテーションさえ書けばすべてが解決されるなんてことはありませんが、設計や命名やユニットテストと同様に、「とりあえずいまは動く」を超えた信用できるものづくりのために非常に重要なファクターです。

## 具体的な書き方

Effective Dart でも具体的な書き方が、典型的な例とともに説明されています。

### 最初の一文で要約しよう

> DO separate the first sentence of a doc comment into its own paragraph
>
> Add a blank line after the first sentence to split it out into its own paragraph. If more than a single sentence of explanation is useful, put the rest in later paragraphs.
>
> This helps you write a tight first sentence that summarizes the documentation. Also, tools like dart doc use the first paragraph as a short summary in places like lists of classes and members.

最初の一文を独立した段落として、要約した内容を書こうというアドバイスです。

**GOOD:**

```dart
/// Deletes the file at [path].
///
/// Throws an [IOError] if the file could not be found. Throws a
/// [PermissionError] if the file is present but could not be deleted.
void delete(String path) {
  ...
}
```

**BAD:**

```dart
/// Deletes the file at [path]. Throws an [IOError] if the file could not
/// be found. Throws a [PermissionError] if the file is present but could
/// not be deleted.
void delete(String path) {
  ...
}
```

### 関数やメソッドは 3 人称単数の動詞で始めよう

> PREFER starting function or method comments with third-person verbs

英語で関数やメソッドに対するドキュメンテーションを書く時には、3 人称単数の動詞で始めることが推奨されています。

```dart
/// Returns `true` if every element satisfies the [predicate].
bool all(bool predicate(T element)) => ...

/// Starts the stopwatch if not already running.
void start() {
  ...
}
```

上記の doc comment を日本語で書くなら下記のようになるでしょう。句点「。」で終わるようにもしています。

```dart
/// すべての要素が [predicate] を満たす場合に `true` を返す。
bool all(bool predicate(T element)) => ...

/// まだ開始されていない場合に、ストップウォッチタイマーをスタートする。
void start() {
  ...
}
```

### 真偽値以外の変数は名詞句として書こう

> PREFER starting a non-boolean variable or property comment with a noun phrase
>
> The doc comment should stress what the property is. This is true even for getters which may do calculation or other work. What the caller cares about is the result of that work, not the work itself.

真偽値以外の変数名に対するドキュメンテーションは、名詞っぽい書き方をすることが推奨されています。

```dart
/// The current day of the week, where `0` is Sunday.
int weekday;

/// The number of checked buttons on the page.
int get checkedCount => ...
```

日本語で書くならば、下記のような体言止めのような表現にするのが良いかもしれません。

```dart
/// 日曜日を `0` とした 1 週間における現在の曜日の整数値。
int weekday;

/// 当該ページにおけるチェック済みボタンの数。
int get checkedCount => ...
```

### 真偽値は「かどうか。」で書こう

> PREFER starting a boolean variable or property comment with “Whether” followed by a noun or gerund phrase
>
> The doc comment should clarify the states this variable represents. This is true even for getters which may do calculation or other work. What the caller cares about is the result of that work, not the work itself.

真偽値については「かどうか」を意味する “Whether” で始めると良いとされています。

```dart
/// Whether the modal is currently displayed to the user.
bool isVisible;

/// Whether the modal should confirm the user's intent on navigation.
bool get shouldConfirm => ...

/// Whether resizing the current browser window will also resize the modal.
bool get canResize => ...
```

日本語だと「〜かどうか。」というドキュメンテーションにすると良さそうです。

```dart
/// 現在モーダルがユーザーに表示されているかどうか。
bool isVisible;

/// 画面遷移を行う際に、ユーザーの意図を確認すべきかどうか。
bool get shouldConfirm => ...

/// 現在のブラウザウィンドウのサイズ変更がモーダルのサイズ変更にも影響するかどうか。
bool get canResize => ...
```

### サンプルコードも含めてみよう

> CONSIDER including code samples in doc comments

下記のように doc comment 内に \`\`\` で囲んだコードブロックを記述して、その API の典型的な使用方法を示すサンプルコードを含めるのも有効です。\`\`\`dart とすることで、doc comment 内のサンプルコードにも Dart のシンタックスハイライトが効くようになります。

```dart
/// Returns the lesser of two numbers.
///
/// ```dart
/// min(5, 3) == 3
/// ```
num min(num a, num b) => ...
```

### 角括弧 `[]` で参照を表現しよう

> DO use square brackets in doc comments to refer to in-scope identifiers
>
> If you surround things like variable, method, or type names in square brackets, then dart doc looks up the name and links to the relevant API docs. Parentheses are optional, but can make it clearer when you’re referring to a method or constructor.

doc comment 内に角括弧 `[]` で囲んだ内容は、当該スコープから見える範囲であれば参照が効くようになります。つまりエディタや IDE でその角括弧 `[]` の内容を command + クリックすると、その定義までコードジャンプをしたり、カーソルを合わせることでその doc comment を読んだりすることができます。

```dart
/// Throws a [StateError] if ...
/// similar to [anotherMethod()], but ...
```

```dart
/// Similar to [Duration.inDays], but handles fractional days.
```

```dart
/// To create a point, call [Point.new] or use [Point.polar] to ...
```

### 省略語は避けよう

> AVOID abbreviations and acronyms unless they are obvious
>
> Many people don’t know what “i.e.”, “e.g.” and “et al.” mean. That acronym that you’re sure everyone in your field knows may not be as widely known as you think.

「例」を意味する "e.g." などは書きがちですが、省略する必要がないものは省略することなく素直に書きましょう。

### Markdown 記法で書こう

これまでに挙げた例でも既にそのようにしていましたが、doc comment は Markdown 記法に対応しています。

正しく Markdown 記法を活用すれば、エディタや IDE でカーソルを合わせた際に現れる文書や `dart doc` コマンドで生成される文書がより適切に表現されます。

doc comment が対応している Markdown 記法の例の詳細はこちら：

https://dart.dev/effective-dart/documentation#markdown

も確認してみてください。

また、VS Code で正しい Markdown を書くための Lint を適用するには、下記のような拡張機能も有名です。

https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint

## ドキュメンテーションの具体的な内容

さて、これまでにドキュメンテーションの意義や重要性、書き方について解説してきましたが、具体的にどのような内容をドキュメンテーションとして書くと良いのでしょうか。

Effective Dart でも、含めるべき内容について事細かに解説されているわけではありません。

そこでこの記事では、多くの人に読まれている API ドキュメントや、あなたが読みやすいと思う API ドキュメントを思い起こして、そこに記述されている要素を考えてみることにしましょう。

たとえば GitHub API や Stripe API などのドキュメントは多くの方が読んだことがあるでしょう。

https://docs.github.com/en/rest?apiVersion=2022-11-28

https://stripe.com/docs/api

上記は両方 REST API のドキュメントですが、それらに含まれる次のような内容は、Dart の doc comment に書くべき内容としても通じているはずです。

- API が提供する概念や機能の名前と説明
- インターフェースの説明
  - 入力パラメータはなにか
    - どのような型か
    - 必須か任意か
    - その概念はどんなものか
  - 出力はなにか
    - 処理の結果、何が返ってくるのか
- どのような例外やエラーが発生し得るか
- 使用方法や典型的な入出力の例示

## おわりに

主に Effective Dart: Documentation を参照しながら、Dart のドキュメンテーションについて書きました。

snkrdunk の開発チームでは特にドキュメンテーションについて規約を設けているわけではありませんが、個人的に積極的に doc comment を書くように心がけたり、PR レビュー時に doc comment を見かけるとポジティブなリアクションをしたりすることで、徐々に充実した doc comment をソースコード内や PR レビューで見かけたりすることが増えてきた実感もあります。

この記事をきっかけに皆さんの Dart のドキュメンテーションに対する意識が深まり、ドキュメンテーションを書く動きが活発になっていけば良いなと思っています！
