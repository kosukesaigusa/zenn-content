---
title: "Dart のコード生成について【前編】"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: true
published_at: 2023-12-21 00:00
---

:::message
この記事は [Flutter Advent Calendar 2023](https://qiita.com/advent-calendar/2023/flutter) の 21 日目の記事です！
:::

## はじめに

この記事は「Dart のコード生成について【前編】」として、`build_runner` を用いた Dart のコード生成について、コード生成を支える仕組みをまとめます。

後日執筆予定の後編では、実際にこの記事で紹介したパッケージを使用しながら、HTTP のレスポンスボディをパースして Dart のインスタンスを作成する `fromJson` ができるコードを生成するパッケージ（`json_serializable` パッケージの一部を簡素に再実装するようなサンプル）の作成をする方法を示します。

内容は FlutterKaigi 2023 で発表したものと重なっています。

https://flutterkaigi.jp/2023/

YouTube で当日の発表アーカイブを公開していただいているので、参考にしてください。

https://www.youtube.com/watch?v=EKoI-p1UnNk

そこで使用したスライドはこちらです。

@[slideshare](9LkqbUGfpkX0oQ)

また、本記事に登場するサンプルコードの多くは、こちら：

https://github.com/kosukesaigusa/code_generation_samples

のリポジトリから見つかります。合わせてご覧ください。

## コード生成を支える仕組み

コード生成を支える仕組みとして主に次のようなパッケージがあります。

それぞれの役割と概要を以下に列挙します。

- `build_runner`: Dart でコード生成を実行するために最もよく使われパッケージ
- `build`: `build_runner` パッケージでコード生成を実行する内容やロジックを実装する
- `build_config`: `build.yaml` で規定するコード生成の設定を記述する
- `dart_style`: 生成されたコードの整形に使用する
- `analyzer`: 静的解析の仕組みで、生成対象や各種設定をパースするために使用する
- `source_gen`: analyzer パッケージや build パッケージを利用しやすくする API として使用する

以下ではそれぞれについて説明します。

### build_runner

`build_runner` パッケージは Dart でコード生成を実行するために最もよく使われるパッケージです。

https://pub.dev/packages/build_runner

`build` パッケージの `Builder` クラスに記述されたコード生成に対応しています。

`pubspec.yaml` の `dev_dependencies` に記述して、`build_runner build` のコマンドを実行することでコード生成を行うことができます。

```yaml
dev_dependencies:
  build_runner:
```

`--delete-conflicting-outputs` や `--build-filter` などのコマンドライン引数を指定することもできます。

```shell
dart run build_runner build --delete-conflicting-outputs
dart run build_runner watch --delete-conflicting-outputs
```

### build

`build` パッケージは `build_runner` パッケージでコード生成を実行する内容やロジックを実装するために使用します。

https://pub.dev/packages/build

`build` パッケージの `Builder` クラスは下記のようになっています。

```dart
/// build_runner を用いて実行するコード生成の
/// 内容やロジックを実装するクラス
abstract class Builder {
  /// コード生成のロジックを記述する
  /// 引数の BuildStep については後述する
  FutureOr<void> build(BuildStep buildStep);

  /// 入力・出力ファイルの拡張子の対応関係を記述する
  Map<String, List<String>> get buildExtensions;
}
```

`Builder` クラスの `build` メソッドの引数にも指定されている `BuildStep` クラスは下記のようになっています。実際には多くのメソッドや変数が定義されていますが、一部だけを抜粋しています。詳細は実際のソースコードを参照してください。

```dart
/// （一部のみ紹介）
/// ビルド処理の各ステップでどのように入力ファイルを読み込み、
/// どのように出力ファイルに書き込むかといった内容を取り扱う
abstract class BuildStep implements AssetReader, AssetWriter {
  /// 当該ステップで取り扱う入力ファイルを表す
  AssetId get inputId;

  /// 当該ステップで取り扱う入力ファイルのライブラリを表す
  Future<LibraryElement> get inputLibrary;

  /// 指定した [id] のファイルに [contents] を書き込む
  @override
  Future<void> writeAsString(AssetId id, FutureOr<String> contents,
      {Encoding encoding = utf8});
}
```

上記を用いることで、最も単純なコード生成のサンプルを作ることができます。

生成元のファイルの内容をコピーして、拡張子だけを変えてそのまま出力する `CopyBuilder` を下記のように記述することができます。

```dart
/// もっとも単純な [Builder] の例
/// 生成元ファイルの内容をコピーして拡張子だけ変えてそのまま出力する
class CopyBuilder implements Builder {
  /// 入力ファイルは .dart、出力ファイル は .copy.dart であることを表す
  @override
  final buildExtensions = const {'.dart': ['.copy.dart']};

  @override
  Future<void> build(BuildStep buildStep) async {
    // 入力ファイル
    final inputId = buildStep.inputId;

    // 拡張子を変えた出力ファイル
    final outputId = inputId.changeExtension('.copy.dart');

    // BuildStep.readAsString で入力ファイルの内容を読み込む
    final contents = await buildStep.readAsString(inputId);

    // BuildStep.writeAsString で出力ファイルに書き込む
    await buildStep.writeAsString(outputId, contents);
  }
}
```

各行に doc comment やインラインコメントを施したとおりですが、`.dart` ファイルの内容を、`BuildStep.readAsString(inputId)` でそのままコピーして `.copy.dart` の拡張子に変えた出力ファイル `inputId.changeExtension('.copy.dart');` 対して、`BuildStep.writeAsString` で書き込むという流れになっています。

### build_config

多くの方にとって、`json_serializable` パッケージや `freezed` パッケージなどのコード生成パッケージを利用する際に `build.yaml` に各種の設定を書いたことがあると思います。

`build_config` パッケージで定められた形式で `build.yaml` にビルドに関する各種の設定を記述できるようになります。

https://pub.dev/packages/build_config

サンプルとして先程実装した `CopyBuilder` の設定を書いてみましょう。

```yaml
# package:build_config で定められた形式でビルドに関する設定を記述する

targets:
  $default:
    builders:
      # 使用するビルダーパッケージと適用するファイルなどを指定する
      copy_builder:
        generate_for:
          include:
          - test/helper/*

builders:
  copy_builder:
    # 使用する Builder 関数が定義されている場所を記述する
    import: "package:copy_builder/builder.dart"
    builder_factories: ["copyBuilder"]
    # 入出力の拡張子を指定する
    build_extensions: {".dart": [".copy.dart"]}
    # キャッシュではなくソースツリーに出力する
    build_to: source
```

その他に適用できる設定値の一覧はパッケージの README などを参照してください。

### dart_style

`dart_style` パッケージはその名前から察することができる通り、生成したコードを整形するためのパッケージです。

https://pub.dev/packages/build_config

たとえば次のようなインデントやスペースがガタガタの `content` 文字列があるとします。

また、`const Foo(this.bar, {this.baz: 'baz'});` の部分をよく見ると、名前付き引数の `=` であるべき箇所が `:` になっているのにも気付きます。

```dart
void main() {
  const content = '''
class Foo {
  const Foo(this.bar, {this.baz: 'baz'});
  final    String bar;
      final String   baz;
}
''';
  final formatted = DartFormatter(
    fixes: [StyleFix.namedDefaultSeparator],
  ).format(content);
  print(formatted);
}
```

上記の `content` に対して `DartFormatter` の `format` メソッドを適用することで、下記のような整形されたコードが出力されます。

```dart
class Foo {
  const Foo(this.bar, {this.baz = 'baz'});
  final String bar;
  final String baz;
}
```

`DartFormatter` コンストラクタ引数の `fixes` に対して `[StyleFix.namedDefaultSeparator]` を指定することで、先程指摘した名前付き引数の `=` であるべき箇所が `:` になっている点についても、整形がうまくいっていることが分かります。

### analyzer

`analyzer` パッケージは、コード生成にあたって、静的解析のしくみを用いて生成対象や各種設定をパースするために使用されます。

https://pub.dev/packages/analyzer

たとえば、先程の `CopyBuilder` について「`@Copy` アノテーションが存在するファイルだけを対象とする」という要件を加えた `CopyBuilderForAnnotation` というのを作ってみましょう。

```dart
/// [@Copy] のアノテーションがあるファイルだけを対象としてコピーする 
/// 先程の [CopyBuilder] の派生例
class CopyBuilderForAnnotation implements Builder {
  @override
  Map<String, List<String>> get buildExtensions => {
        '.dart': ['.copy.dart'],
      };

  @override
  Future<void> build(BuildStep buildStep) async {
    final inputId = buildStep.inputId;
    final content = await buildStep.readAsString(inputId);
    final outputId = inputId.changeExtension('.copy.dart');

    // parseString は package:analyzer で定義された関数
    // package:analyzer は、静的解析によって、生成対象や各種設定をパースするための機能を提供する
    // _hasCopyAnnotation で @Copy アノテーションの存在を確認している
    final parsedStringResult = parseString(content: content);
    if (_hasCopyAnnotation(parsedStringResult)) {
      await buildStep.writeAsString(outputId, content);
    }
  }

  bool _hasCopyAnnotation(ParseStringResult parsedStringResult) {
    // 生成元のパース結果である ParseStringResult の unitMember に
    // @Copy アノテーションが存在するか判定している
    return parsedStringResult.unit.declarations.any(
      (unitMember) => unitMember.metadata
          .any((annotation) => annotation.name.name == 'Copy'),
    );
  }
}
```

`analyzer` パッケージの `parsedString` 関数を用いて読み取ったファイルの文字列をパースし、`parsedStringResult` に格納しています。

`_hasCopyAnnotation` メソッドで `parsedStringResult` に `'Copy'` の名前のアノテーションが存在しているかどうかを判定しています。

`_hasCopyAnnotation` メソッドが `true` を返す場合のみ `BuildStep.writeAsString` を実行して内容をコピーしたファイルを生成しています。

### source_gen

最後に紹介する `source_gen` パッケージは、`analyzer` パッケージや `build` パッケージの開発者フレンドリーな API を提供するものです。つまり、コード生成パッケージを開発する上で必須のものではありませんが、`source_gen` パッケージの様々な API を利用することで、`analyzer` パッケージや `build` パッケージを直接利用する機会が減り、簡潔に実装することができます。たとえば `freezed` パッケージでも活用されています。

https://pub.dev/packages/source_gen

多くの便利な API が提供されていますが、ここではサンプルでも使用している

- `PartBuilder`
- `GeneratorForAnnotation<T>`
- `TypeChecker`

の 3 つを紹介します。

#### `PartBuilder`

最初に紹介するのは `part of 'foo.dart';` のようなファイルを生成するための `Builder` を便利に記述するためのクラスです。

doc comment やインラインコメントで捕捉しているとおりですが、下記のように様々な設定を簡潔に表現することができます。

`source_gen` パッケージは `dart_style` にも依存しており、自動で `DartFormatter` による整形も施されます。

```dart
/// [PartBuilder] を使用することで package:builder の [Builder] を
/// 明示的に実装する必要がなくなる
/// part of 'foo.dart'; のようなファイルを生成する [Builder] を
/// 簡潔に定義できる
/// デフォルトで生成文字列に [DartFormatter] による整形も施す
Builder fromJsonGenerator(BuilderOptions options) {
  return PartBuilder(
    // Generator を指定する
    // また、options.config には build.yaml の option 設定が入っている
    [FromJsonGenerator(BuildYamlConfig.fromBuildYaml(options.config))],
    // 出力の拡張子を指定する
    '.from_json.dart',
    // ignore コメントなど、生成ファイルの冒頭に共通して書き込む文字列を設定できる
    header: '''
// coverage:ignore-file
// ignore_for_file: type=lint
// ignore_for_file: unused_element, ... 省略
''',
    options: options,
  );
}
```

#### `GeneratorForAnnotation<T>`

`GenerateForAnnotation<T>` は上記の `PartBuilder` の第一引数に指定する型である `source_gen` パッケージの `Generator` 型の派生で、その名の通り、特定の型 `T` のアノテーションを対象としてコードを生成するための `Generator` です。

例えば `@Foo` のアノテーションが施されたものを対象とする `Generator` は下記のように定義することができます。

```dart
//// @Foo アノテーションを対象とする [Generator] クラス
class FooGenerator extends GeneratorForAnnotation<Foo> {
  /** 省略 */

  //// @Foo アノテーションを生成対象として生成した文字列を返す
  @override
  String generateForAnnotatedElement(/** 省略 */) {/** 省略 */}
}
```

#### `TypeChecker`

`TypeChecker` を用いると、`analyzer` パッケージの、その名の通り型情報を保持する `DartType` 型に対して、`TypeChecker.isAssignableFromType(dartType)` のようにすることで、それが期待する型であるかどうかを判定するようなことができます。

例えば、コード生成の処理内で、与えられた `DartType` が特定のアノテーションに一致するかどうかを判定するような場合に役立ちます。

```dart
/// 与えられた [objectType] が @Copy アノテーションかどうかを判定する
bool isCopyAnnotation(DartType objectType) {
  // package:source_gen の TypeChecker により
  // コンパイルまたはビルド時の静的型チェックを行うことができる
  const typeChecker = TypeChecker.fromRuntime(Copy);
  return typeChecker.isAssignableFromType(objectType);
}
```

## おわりに

ほとんどの Dart, Flutter エンジニアは、きっと現在または今までに関わってきたプロジェクトで、コード生成パッケージの恩恵を受けたことがあるだろうと思います。

本記事ではそんなコード生成パッケージを支える仕組みの紹介を行いました。

後編では具体的にどのようにコード生成パッケージを開発していくかの例を示しながら解説する記事を執筆する予定です！お楽しみに！
