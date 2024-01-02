---
title: "Dart のコード生成について【後編】"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: true
published_at: 2023-12-25 00:00
---

:::message
この記事は [Dart のコード生成について【前編】](https://zenn.dev/kosukesaigusa/articles/dart-code-generation-1) の続編記事です！
:::

## はじめに

この記事は、先日行為回した「Dart のコード生成について【前編】」

https://zenn.dev/kosukesaigusa/articles/dart-code-generation-1

の続編として、Dart のコード生成パッケージを実際に作成する方法を説明する記事です。

例として、HTTP のレスポンスボディをパースして Dart のインスタンスを作成する `fromJson` ができるコードを生成するパッケージ（`json_serializable` パッケージの一部を簡素に再実装するようなサンプル）を作成していきます。

内容は FlutterKaigi 2023 で発表したものと重なっています。

https://flutterkaigi.jp/2023/

YouTube で当日の発表アーカイブを公開していただいているので、参考にしてください。

https://www.youtube.com/watch?v=EKoI-p1UnNk

そこで使用したスライドはこちらです。

@[slideshare](9LkqbUGfpkX0oQ)

また、本記事に登場するサンプルコードの多くは、こちら：

https://github.com/kosukesaigusa/code_generation_samples

のリポジトリから見つかります。合わせてご覧ください。

## コード生成パッケージを自作する

### 仕様を決める

まずは作成するコード生成パッケージの仕様を決めます。下記のようにすることにしましょう。

- HTTP のレスポンスボディをパースして Dart のインスタンスを作成する `fromJson` ができるコードを生成するパッケージを
- 自作の `@Default` アノテーションを施して、当該フィールドが `null` の場合にデフォルト値を与えることができる
- スネークケースのフィールドをキャメルケースに変換できる
- `JsonConverter` の `fromJson` が適用できる

`json_serializable` パッケージの一部を簡素に再実装するような内容と言えます。

### 構成

次に構成を考えます。構成に唯一の正解はありませんが、コード生成パッケージを自作する上での構成は一般的には下記のようになることが多いと思います。

```sh
packages
├── from_json_generator # コードを生成するパッケージ
│   └── lib
│       ├── from_json_generator.dart
│       └── src
│           ├── from_json_generator.dart
│           ├── templates
│           └── その他の生成ロジックなど
└── from_json_generator_annotation # アノテーションを定義するパッケージ
```

最初に注目すべきは、コードを生成するパッケージ `from_json_generator` と、そのアノテーションを定義するパッケージ `from_json_generator_annotation` とを分離していることです。

前者はコード生成を実行するためだけに用いるので使用する側は `dev_dependencies` に記述して使用すれば十分です。

たとえば `freezed` パッケージ、`freezed_annotation` パッケージが別々のパッケージとして記述されているのと同様です。

`from_json_generator` パッケージには `パッケージ.dart` というファイル名でそのパッケージで実行するコード生成関数を定義するファイルを公開します。`builder.dart` などとしている例も見受けられますが、その場合 `dart pub publish` の実行時に下記のような警告が表示されます。

```sh
The name of "lib/builder.dart", "builder", should match the name of the package, "from_json_generator".
```

`from_json_generator` パッケージの

- `lib/from_json_generator.dart` には、`Builder` の定義
- `lib/src/from_json_generator.dart` には、`Generator` の定義（`Builder` とファイル名が同名でやや分かりづらいので、もちろん変えても問題ありません）
- `templates` には、コード生成の各種設定をパースし終わった情報を流し込んで生成文字列を形作る雛形
- `その他の生成ロジック` には、コード生成の各種設定をパースする処理など

を記述していきます。

### `pubspec.yaml`

`pubspec.yaml` は下記のようになります。[Dart のコード生成について【前編】](https://zenn.dev/kosukesaigusa/articles/dart-code-generation-1) で説明した各パッケージに加え、`from_json_generator_annotation` パッケージを相対パスで指定しています。

```yaml
name: from_json_generator
description: from_json_generator
version: 0.0.1
homepage: https://github.com/kosukesaigusa/dart_from_json_generator
publish_to: none

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  analyzer: ^6.1.0
  build: ^2.4.1
  from_json_generator_annotation:
    path: ../from_json_generator_annotation
  json_annotation: ^4.8.1
  source_gen: ^1.4.0

dev_dependencies:
  build_runner: ^2.4.6
```

### `build.yaml`

`build.yaml` の設定も行います。

サンプルリポジトリでは、`test/helper/` の下にコード生成を確認す対象を配置しているので `generate_for` の `include` にその記述をしています。

また、`from_json_generator` のビルダー設定の最後の `defaults` > `options` > `field_rename` に `snake` と記述すると、レスポンスボディのスネークケースのフィールド名をキャメルケースに変換することにしています。

```yaml
targets:
  $default:
    builders:
      from_json_generator:
        generate_for:
          include:
            - test/helper/*

builders:
  from_json_generator:
    import: "package:from_json_generator/from_json_generator.dart"
    builder_factories: ["fromJsonGenerator"]
    build_extensions: {".dart": [".from_json.dart"]}
    build_to: source
    defaults:
      options:
        field_rename: snake # スークケースをキャメルケースに変換する場合に snake とする
```

### 生成対象と生成結果の例

これ以降ではコード生成パッケージの作成のための詳しい説明が行われていきます。その前に、今回作成するコード生成パッケージでは、どのようなものを生成対象として、どのようなものが生成されることを期待するのかを整理して見通しを良くしておきましょう。

生成対象のコードとしては下記のようなものを想定しています。GitHub の GET Repository API のレスポンスボディを抜粋したような内容です。

```dart
@FromJson()
class Repository {
  Repository({
    required this.id,
    required this.starGazersCount,
    required this.visibility,
  });

  final String id;

  @Default(0) final int starGazersCount;

  @_visibilityConverter final Visibility visibility;
}
```

`String id` はリポジトリの ID、`int starGazersCount` はリポジトリのスター数を表しています。

`Visibility visibility` はリポジトリの公開設定を表しており、HTTP レスポンスとしては `"public"` or `"private"` のどちらかの文字列として定義されていますが、Dart では `enum` を使って下記のような `Visibility` 型とすることにしています。

```dart
enum Visibility {
  public,
  private,
  ;

  factory Visibility.fromString(String visibilityString) {
    switch (visibilityString) {
      case 'public':
        return Visibility.public;
      case 'private':
        return Visibility.private;
    }
    throw ArgumentError('visibility is not valid: $visibilityString');
  }
}
```

また、

- HTTP レスポンスでの `String` 型
- Dart に変換後の `Visibility` 型

を変換するための `JsonConverter` も下記のように定義しておきます。今回のサンプルでは `toJson` をする仕様は想定していないので実装しません。

```dart
const _visibilityConverter = _VisibilityConverter();

class _VisibilityConverter implements JsonConverter<Visibility, String> {
  const _VisibilityConverter();

  @override
  Visibility fromJson(String json) => Visibility.fromString(json);

  @override
  String toJson(Visibility _) => throw UnimplementedError();
}
```

これによって生成されるコードとしては下記のようなものを想定しています。

```dart
// coverage:ignore-file
// ignore_for_file: type=lint
// ignore_for_file: unused_element, deprecated_member_use, deprecated_member_use_from_same_package, use_function_type_syntax_for_parameters, unnecessary_const, avoid_init_to_null, invalid_override_different_default_values_named, prefer_expression_function_bodies, annotate_overrides, invalid_annotation_target, unnecessary_question_mark

part of 'repository.dart';

// **************************************************************************
// FromJsonGenerator
// **************************************************************************

Repository _$RepositoryFromJson(Map<String, dynamic> json) => Repository(
      id: json['id'] as String,
      starGazersCount: json['starGazersCount'] as int? ?? 0,
      visibility: _visibilityConverter.fromJson(json['visibility'] as String),
    );
```

生成対象と生成結果のイメージはついたでしょうか？

それでは実際の実装方法に話を進めていきます。

### アノテーションの定義

上で取り上げた生成対象の例について、生成元のコードに付されている 3 つのアノテーションを確認しましょう。

- `FromJson()`: このアノテーションが付されたクラスを生成対象とみなします
- `Default(0)`: HTTP レスポンスに `starGazersCount` のフィールドが含まれていなかった場合に、`fromJson` 時のデフォルト値として `0` を使用します
- `@_visibilityConverter`: `fromJson` 時に上記の `_VisibilityConverter` の `fromJson` メソッドを適用して `enum` の `Visibility` 型に変換します

`FromJson` アノテーションは下記のように定義されます。

アノテーションの定義に慣れていなければどんな特別な記法をするのだろうか？と思うかもしれませんが、至って普通の Dart のクラス定義をするだけです。下記のようにするだけで `FromJson()` アノテーションを使用することができます。

HTTP レスポンスのスネークケースのフィールドをキャメルケースに変換したければ、`FromJson(convertSnakeToCamel: true)` として使用します。

また、`const fromJson = FromJson()` のようにコンスとコンストラクタの結果を `fromJson` という変数に格納して定義しておけば、使用する側は `@fromJson` のように小文字始まりでアノテーションを付すこともできます。riverpod_generator パッケージで `keepAlive` しない場合には小文字で `@riverpod` と記述すれば十分で、`keepAlive` を指定するときには大文字で `@Riverpod(keepAlive: true)` と書くようになっているのと同じです。

```dart
/// コード生成の対象であることを示すアノテーション
class FromJson {
  const FromJson({this.convertSnakeToCamel = false});
  /// スネークケースのフィールド名をキャメルケースに変換するかどうか
  final bool convertSnakeToCamel;
}
```

`fromJson` 時のデフォルト値を与えるアノテーションは下記のように定義することができます。

```dart
/// レスポンスデータのパース時に、フィールドが null である場合の
/// デフォルト値を指定するアノテーション
class Default {
  const Default(this.defaultValue);

  final Object? defaultValue;
}
```

### `Builder` (`PartBuilder`) の定義

いよいよ、コード生成ロジックのエントリポイントである `Builder` を定義します。`build` パッケージが提供する `Builder` クラスを使用してももちろん構いませんが、`source_gen` が `analyzer` パッケージと合わせて開発者フレンドリーで使いやすい API を提供してくれているのでその `PartBuilder` を使用します。`PartBuilder` については、[Dart のコード生成について【前編】](https://zenn.dev/kosukesaigusa/articles/dart-code-generation-1) でも紹介しています。

`PartBuilder` の第一引数には後に説明している `FromJsonGenerator` を指定しています。

また、`BuildYamlConfig` も後に説明しますが、`build.yaml` の `options` の設定内容を与える `BuilderOptions options` を受け取ってビルド時の設定に反映するためのクラスです。

名前付き引数の `header` には、生成コードの冒頭に共通して記述すべき lint ルールの ignore 設定などを与えています。

```dart
/// Builds generators for `build_runner` to run.
Builder fromJsonGenerator(BuilderOptions options) {
  return PartBuilder(
    [FromJsonGenerator(BuildYamlConfig.fromBuildYaml(options.config))],
    '.from_json.dart',
    header: '''
// coverage:ignore-file
// ignore_for_file: type=lint
// ignore_for_file: unused_element, deprecated_member_use, deprecated_member_use_from_same_package, use_function_type_syntax_for_parameters, unnecessary_const, avoid_init_to_null, invalid_override_different_default_values_named, prefer_expression_function_bodies, annotate_overrides, invalid_annotation_target, unnecessary_question_mark
''',
    options: options,
  );
}
```

また、生成コードの

```dart
// **************************************************************************
// FromJsonGenerator
// **************************************************************************
```

の部分は `PartBuilder` を使用するとデフォルトで生成コードの直前に付されます。`json_serializable` や `freezed` のコード生成パッケージの生成結果のファイルにも見かけたことがあるのではないでしょうか？

### `BuildYamlConfig` の定義

`BuildYamlConfig` は下記のように自作したクラスで、`.fromBuildYaml` の factory コンストラクタで、`build.yaml` のオプション設定の `field_rename` に `snake` という文字列が設定されていた場合に、HTTP レスポンスのスネークケースのフィールドをキャメルケースに変換するようにしています。

```dart
/// Configuration values defined in build.yaml.
class BuildYamlConfig {
  /// Creates a [BuildYamlConfig].
  const BuildYamlConfig({required this.convertSnakeToCamel});

  /// Decode the options from a build.yaml.
  factory BuildYamlConfig.fromBuildYaml(Map<String, dynamic> yaml) =>
      BuildYamlConfig(
        convertSnakeToCamel: (yaml['field_rename'] as String?) == 'snake',
      );

  /// Whether to convert field name snake_case to camelCase.
  final bool convertSnakeToCamel;
}
```

### `GeneratorForAnnotation` の定義

今度はコード生成のエントリポイントである `fromJsonGenerator` に指定された `GeneratorForAnnotation` を下記のように定義します。総称型の `T` に `FromJson` を指定しているので、`@FromJson()` アノテーションが付されたクラスを生成対象とする `Generator` です。

`GeneratorForAnnotation<T>` については、[Dart のコード生成について【前編】](https://zenn.dev/kosukesaigusa/articles/dart-code-generation-1) でも紹介しています。

```dart
/// A generator for [FromJson] annotation.
class FromJsonGenerator extends GeneratorForAnnotation<FromJson> {
  /// Creates a new instance of [FromJsonGenerator].
  const FromJsonGenerator(this._buildYamlConfig);

  /// A [BuildYamlConfig] instance.
  final BuildYamlConfig _buildYamlConfig;

  @override
  String generateForAnnotatedElement(
    Element element,
    ConstantReader annotation,
    BuildStep buildStep,
  ) {
    if (element is! ClassElement) {
      throw InvalidGenerationSourceError(
        '@FromJson can only be applied on classes. '
        'Failing element: ${element.name}',
        element: element,
      );
    }

    final annotation = const TypeChecker.fromRuntime(FromJson)
        .firstAnnotationOf(element, throwOnUnresolved: false)!;

    final config = CodeGenerationConfig(
      className: element.name,
      fieldConfigs: element.fields.map(parseFieldElement).toList(),
      convertSnakeCaseToCamelCase: annotation.decodeField<bool>(
        'convertSnakeToCamel',
        decode: (obj) => obj.toBoolValue()!,
        orElse: () => _buildYamlConfig.convertSnakeToCamel,
      ),
    );

    final buffer = StringBuffer()..writeln(FromJsonTemplate(config));

    return buffer.toString();
  }
}
```

このクラスにコード生成のロジックの骨格が記述されていることになります。

各部を抜粋しながら内容を確認していきます。

まずは冒頭で生成対象として読み込もうとしている `element` が `analyzer` パッケージの `ClassElement` であることを確認しています。`@FromJson` アノテーションがクラスに付されていることを確認するためです。

`ClassElement` でない場合には `source_gen` パッケージの `InvalidGenerationSourceError` をスローして当該 `BuildStep` を中断します。

```dart
class FromJsonGenerator extends GeneratorForAnnotation<FromJson> {
  /** 省略 */

  @override
  String generateForAnnotatedElement(
    Element element,
    ConstantReader annotation,
    BuildStep buildStep,
  ) {
    if (element is! ClassElement) {
      throw InvalidGenerationSourceError(
        '@FromJson can only be applied on classes. '
        'Failing element: ${element.name}',
        element: element,
      );
    }

    /** 省略 */
  }
}
```

下記の部分では `source_gen` パッケージの `TypeChecker` を用いて、生成対象の `ClassElement` に付されたアノテーションの中から `FromJson` に一致するものを抽出しています。

結果を格納した `annotation` 変数は `analyzer` パッケージの `DartObject` 型です。

```dart
@override
String generateForAnnotatedElement(
  Element element,
  ConstantReader annotation,
  BuildStep buildStep,
) {
  /** 省略 */

  final annotation = const TypeChecker.fromRuntime(FromJson)
      .firstAnnotationOf(element, throwOnUnresolved: false)!;

  /** 省略 */
}
```

## おわりに
