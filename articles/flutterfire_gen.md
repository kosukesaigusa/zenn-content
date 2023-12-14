---
title: "【開発中】flutterfire_gen パッケージについて"
emoji: "🎅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter", "Firebase", "CloudFirestore"]
published: true
published_at: 2023-12-15 00:00
---

## はじめに

flutterfire_gen は、私が開発をしている Flutter の Cloud Firestore のためのコード生成パッケージです。

記事執筆時点の最新バージョンは `0.2.0-dev.3` で、pub.dev にも publish はしていますが unlisted の状態にしています。

https://github.com/kosukesaigusa/flutterfire_gen

まだまだ開発途中ですが、

- 私の個人のプロジェクトのアプリ開発では十分使用できるようになってきていること
- 2023 年 9 月に行われた東京 Flutter ハッカソンでも活用することで、素早い実装と、初めて一緒に開発するチームメンバーにとっても分かりやすい開発体験を可能にして、優勝することができたこと
- 2023 年 12 月の [GDG DevFest](https://gdg-tokyo.connpass.com/event/301690/) の LT でも初めて対外的に紹介することで、一部の方に興味をもっていただいたこと
  
など、そろそろ自身以外の人にも触ってみてもらう機会があっても良さそうだという思えるようになってきたので、この記事を書くことにしました。

https://x.com/TYOFlutterHack/status/1708407065146998979?s=20

本記事の内容とも重なりますが、日本語でも README を書いたのでご覧ください。

https://github.com/kosukesaigusa/flutterfire_gen/blob/main/resources/translations/ja_JP/README.md

未完成のパッケージなので、しばらくは破壊的な変更や方針転換の可能性もありますが、先行して触っていただいた方からは色々なアドバイスを頂けると嬉しいです！

GitHub Issue でのご提案やご報告はもちろん、本記事へのコメントや X 等での DM でも結構です。

https://github.com/kosukesaigusa/flutterfire_gen/issues

https://twitter.com/KosukeSaigusa

## 開発のモチベーション

Flutter と Cloud Firestore を用いて開発を行う時に、次のような課題を感じたことはないでしょうか？

- 読み込み (read)、作成 (create)、更新 (update)、削除 (delete) の各操作で必要となる最適なインターフェースが異なる
  - 読み込み時には、ドキュメント ID を必ず取得結果に含めたい
  - が、作成時にはドキュメント ID はまだ知り得ないのでインターフェースに含められない
  - また、更新時には指定したフィールドのみを更新したいので、すべて optional なパラメータとしたい
  - 読み込み、作成、更新で異なるデフォルト値を指定したいこともある
- 総じてボイラプレートコードが多い
  - 型安全な操作のために `withConverter` を用いて `CollectionReference` や `DocumentReference` をすべてのコレクション・ドキュメントに対して定義し切るのは大変（上記のように、read, create, update のインターフェースを区別するのだとしたらその分だけ定義する必要がある）
  - しばしば書き込み時に自動でサーバタイムスタンプ `FieldValue.serverTimestamp()` を設定したいフィールドがあるが、それを毎回書くのも面倒
  - Cloud Firestore の `Timestamp` 型と Dart の `DateTime` 型の変換を自分で書く必要がある
  - 取得処理や、作成、更新、削除処理も同じようなコードを異なるコレクション・ドキュメントに対して繰り返し書く

flutterfire_gen を使用すると、Cloud Firestore のドキュメントのスキーマに対応するクラスを Dart でたった一つ記述することで、上記の課題や面倒を解決できるボイラプレートコードを自動で生成することができます。

### flutterfire_gen の目的

flutterfire_gen を活用することで、できるようになることの主な例は以下の通りです。

- read, create, update (, delete) でそれぞれに最適なインターフェースを生成する
- read, create, update, delete の型安全なメソッドを生成する
- read, create, update 時に異なるデフォルト値を設定できる
- create, update 時に自動で `FieldValue.serverTimestamp()` を使用できる
- create, update 時に実際の値（例：`42`, `[1, 3, 5]`）と `FieldValue`（例：`FieldValue.increment(1)`, `FieldValue.arrayUnion([7])`）とをまとめて取り扱うインターフェースを提供する
- `JsonConverter` も使用できる
  - Cloud Firestore の `Timestamp` 型と Dart の `DateTime` 型の変換は `JsonConverter` なしで自動で行う
- その他にもいろいろ

単に、いわゆるデータクラスを生成するだけではなく、型安全な読み書きメソッドや、`FieldValue` の取り扱いなどの Cloud Firestore をより便利に・汎用的に使用することができる仕組みを生成します。

## 使い方

### 導入

導入したい Flutter プロジェクトの `pubspec.yaml` に下記のような記述をしてください。

```yaml
dependencies:
  cloud_firestore: latest

  firebase_core: latest

  # A package containing annotations for flutterfire_gen.
  flutterfire_gen_annotation: ^0.2.0-dev.1

  # A package containing utility annotations for flutterfire_gen.
  flutterfire_gen_utils: ^0.2.0-dev.1

  # Optional. Will be necessary if you use JsonConverter.
  json_annotation: latest

dev_dependencies:
  # The tool to run code-generators.
  build_runner: latest

  # The code generator.
  flutterfire_gen: ^0.2.0-dev.3
```

- [flutterfire_gen](https://pub.dev/packages/flutterfire_gen)
- [flutterfire_gen_annotation](https://pub.dev/packages/flutterfire_gen_annotation)
- [flutterfire_gen_utils](https://pub.dev/packages/flutterfire_gen_utils)

の 3 つは flutterfire_gen によるコード生成とそれに関わる機能（アノテーションや機能の拡張）を提供するパッケージです。

### `@FirestoreDocument` アノテーションでドキュメントスキーマを定義する

`todos` コレクションの Todo ドキュメントのスキーマを flutterfire_gen の記法で記述してみましょう。

要件は以下の通りです。

- `title`: Todo のタイトル
  - `String` 型
- `isCompleted`: Todo の完了状態
  - `bool` 型
  - 読み込み時にもし `null` であれば `false` とする
  - 作成時に指定しなければ `false` で作成する
- `createdAt`: Todo の作成日時
  - `DateTime?` 型
  - ドキュメントの作成時には自動で `FieldValue.serverTimestamp()` を適用したい
- `updatedAt`: Todo の更新日時
  - `DateTime?` 型
  - ドキュメントの作成および更新時には自動で `FieldValue.serverTimestamp()` を適用したい

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutterfire_gen_annotation/flutterfire_gen_annotation.dart';

part 'todo.flutterfire_gen.dart';

@FirestoreDocument(path: 'todos/{todoId}')
class Todo {
  const Todo({
    required this.title,
    required this.isCompleted,
    required this.createdAt,
    required this.updatedAt,
  });

  final String title;

  @ReadDefault(false)
  @CreateDefault(false)
  final bool isCompleted;

  @alwaysUseFieldValueServerTimestampWhenCreating
  final DateTime? createdAt;

  @alwaysUseFieldValueServerTimestampWhenCreating
  @alwaysUseFieldValueServerTimestampWhenUpdating
  final DateTime? updatedAt;
}
```

まずは Cloud Firestore のドキュメントに対応するクラス `Todo` に `@FirestoreDocument` のアノテーションを施します。

```dart
@FirestoreDocument(path: 'todos/{todoId}')
class Todo { /** 省略 */ }
```

`@FirestoreDocument` アノテーションの必須パラメータである `path` には、該当するドキュメントまでのパスを以下のようなルールで記述してください。

- スラッシュ区切りでコレクション名とドキュメント ID を交互に書く
- ドキュメント ID は `{}` で囲む
- ドキュメント ID は `Id` で終わる文字列とする
  - `Id` 以前の文字列がドキュメント名として認識されます
  - 例：`users/{userId}` は「`user` コレクションの `user` ドキュメント」のスキーマ定義であることを意味する

サブコレクションを用いたネストしたパスも同様に定義することができます。

例：

```dart
@FirestoreDocument(path: 'chatRooms/{chatRoomId}/chatMessages/{chatMessageId}')
class ChatMessage { /** 省略 */ }
```

コンストラクタパラメータの部分はコード生成のロジックでは参照していません（`required` を指定しようとしまいと、デフォルト値を設定しようとしまいと、生成されるコードに影響はありません）。コンパイルエラーの起きない方法で記述してください。

```dart
@FirestoreDocument(path: 'todos/{todoId}')
class Todo {
  const Todo({
    required this.title,
    required this.isCompleted,
    required this.createdAt,
    required this.updatedAt,
  });

  /** 省略 */
}
```

メンバ変数の定義も通常の Dart の文法に従って行ってください。様々なアノテーションに対応しています。

```dart
@FirestoreDocument(path: 'todos/{todoId}')
class Todo {
  /** 省略 */

  final String title;

  @ReadDefault(false)
  @CreateDefault(false)
  final bool isCompleted;

  @alwaysUseFieldValueServerTimestampWhenCreating
  final DateTime? createdAt;

  @alwaysUseFieldValueServerTimestampWhenCreating
  @alwaysUseFieldValueServerTimestampWhenUpdating
  final DateTime? updatedAt;
}
```

flutterfire_gen では read, update, create 時にそれぞれ異なるデフォルト値を設定することができます。

```dart
@ReadDefault(false)
@CreateDefault(false)
final bool isCompleted;
```

たとえばこの `isCompleted` フィールドは

- read 時に当該フィールドに値がなければ（または `null` であれば）デフォルトで `false` にする
- create 時に当該フィールドの値を指定しなければデフォルトで `false` を書き込む

のように処理されます。

`@alwaysUseFieldValueServerTimestampWhenCreating` や `@alwaysUseFieldValueServerTimestampWhenUpdating` のアノテーションを使用すると、create 時や update 時に当該フィールドに自動的に `FieldValue.serverTimestamp()` が与えられるようになります。

```dart
@alwaysUseFieldValueServerTimestampWhenCreating
final DateTime? createdAt;

@alwaysUseFieldValueServerTimestampWhenCreating
@alwaysUseFieldValueServerTimestampWhenUpdating
final DateTime? updatedAt;
```

### flutterfire_gen でコードを生成する

コード生成を実行するためには下記のコマンドを実行してください。

```sh
flutter pub run build_runner build --delete-conflicting-outputs
```

また、生成元ファイルの拡張子の直前に `.flutterfire_gen` を追加したファイルが生成されるので、生成元ファイルには下記のような `part 'todo.flutterfire_gen.dart';` の記述がされている必要があります。

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutterfire_gen_annotation/flutterfire_gen_annotation.dart';

part 'todo.flutterfire_gen.dart';
```

### 生成された `Query` クラスを利用する

上記の `@FirestoreDocument` アノテーションを施した `Todo` クラスに対してコード生成を行うと、生成結果に `TodoQuery` というクラスが含まれています。`TodoQuery` には

read

- `fetchDocuments`: `todos` コレクションの複数のドキュメントを取得する
- `subscribeDocuments`: `todos` コレクションの複数のドキュメントのリアルタイム更新を取得する
- `fetchDocument`: `todos` コレクションの指定したドキュメントを取得する
- `subscribeDocument`: `todos` コレクションの指定したドキュメントのリアルタイム更新を取得する

create/update

- `add`: `todos` コレクションに新しいドキュメントを作成する
- `set`: `todos` コレクションの指定したドキュメントにデータをセットする
- `update`: `todos` コレクションの指定したドキュメントを更新する

delete

- `delete`: `todos` コレクションの指定したドキュメントを削除する

のような基本的な読み書きメソッドが定義されています。

さらに、それらは

- read で得られる値としての `ReadTodo` 型
- create で作成する際のインターフェースとしての `CreateTodo` 型
- update で更新する際のインターフェースとしての `UpdateTodo` 型

が定義されて型安全性が担保されています。

たとえば、`@FirestoreDocument` アノテーションを施した `Todo` クラスには特にドキュメント ID の文字列フィールドを定義していませんが、`TodoQuery` のメソッドを通じて得られる `ReadTodo` 型のインスタンスには、自動的に non-nullable な `String todoId` が含まれるようになっています。

一方で、create 時にはこれから作成するドキュメントの ID は知り得ないので、`todoId` はそのインターフェースに含まれません。

そして、update では指定したフィールドだけを更新したいので、`UpdateTodo` 型が提供するドキュメントの更新のためのインターフェースのパラメータはすべて optional です。

`Todo` クラスを一つ定義してコード生成を実行するだけで、読み書きの操作のそれぞれで異なる最適な型と、基本的な読み書きのメソッドが自動で生成されるのが flutterfire_gen を使用することで得られる大きな恩恵です。

### read (get/list)

read 操作は下記のようにとてもシンプルに書くことができます。`FirebaseFirestore.instance` と繰り返し書くことも、`CollectionReference` や `DocumentReference` に `withConverter` を施して型安全な操作をするためのコードを自分で書く必要も全くありません。それらのボイラプレートコードはすべて flutterfire_gen が生成します。

```dart
final query = TodoQuery();

Future<List<ReadTodo>> fetchTodos() => query.fetchDocuments();

Stream<List<ReadTodo>> subscribeTodos() => query.subscribeDocuments();

Future<ReadTodo?> fetchTodo(String todoId) =>
    query.fetchDocument(todoId: todoId);

Stream<ReadTodo?> subscribeTodo(String todoId) =>
    query.subscribeDocument(todoId: todoId);
```

読み込みクエリに `where` 句や `orderBy` 句を追加するのにも対応しています。各メソッドの任意引数の `queryBuilder` を持ちいて下記のように各種の条件を追加するだけです。

```dart
final query = TodoQuery();

Future<List<ReadTodo>> fetchTodos() => query.fetchDocuments(
      queryBuilder: (query) => query
          .where('isCompleted', isEqualTo: false)
          .orderBy('createdAt', descending: true),
    );
```

上で説明した通り、`Todo` クラスの定義をした際には書く必要のなかった `todoId` が確実に取得できています。

```dart
Future<List<ReadTodo>> fetchTodos() async {
  final todos = await query.fetchDocuments();
  for (final todo in todos) {
    print(todo.todoId);
  }
  return todos;
}
```

### create

create 時には、型安全な操作のために `CreateTodo` という専用のインターフェースが提供されています。

```dart
final query = TodoQuery();

Future<DocumentReference<CreateTodo>> addTodo(String title) =>
    query.add(createTodo: CreateTodo(title: title));

Future<DocumentReference<CreateTodo>> addCompletedTodo(String title) =>
    query.add(createTodo: CreateTodo(title: title, isCompleted: true));
```

Todo の `title` が必須のパラメータとなっています。`isCompleted` が任意のパラメータになっているのは、`Todo` クラスを定義した際に `@CreateDefault(false)` のアノテーションを施したためです。よって、特に指定しなければ `isCompleted` は `false` でドキュメントが作成されます。

また、`createdAt`, `updatedAt` はインターフェースに登場しませんが、内部で自動で `FieldValue.serverTimestamp()` が適用されています。このあたりを何も意識しなくて良いのも flutterfire_gen が自動でそれらのコードを生成している恩恵です。

### update

update 時にも、`UpdateTodo` という専用のインターフェースが提供されています。

指定したフィールドのみを更新したいので、`UpdateTodo` に定義されているのはすべて任意引数です。

```dart
final query = TodoQuery();

Future<void> updateCompletionStatus({
  required String todoId,
  required bool isCompleted,
}) =>
    query.update(
      todoId: todoId,
      updateTodo: UpdateTodo(isCompleted: isCompleted),
    );
```

上記は、指定した Todo ドキュメントの完了状態 (`isCompleted`) を更新する関数です。

ここでも create と同様に、`updatedAt` には内部で自動で `FieldValue.serverTimestamp()` が適用されています。

### 発展

#### JsonConverter

[json_annotation](https://pub.dev/packages/json_annotation) パッケージの `JsonConverter` を適用することも可能です。

たとえば、下記の `visibility` フィールドには `@_visibilityConverter` の `JsonConverter` のアノテーションが施されており、

- Dart では `enum` の `Visibility` 型
- Cloud Firestore では `String` 型

として扱うための変換をすることができます。

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutterfire_gen_annotation/flutterfire_gen_annotation.dart';
import 'package:json_annotation/json_annotation.dart';

part 'repository.flutterfire_gen.dart';

@FirestoreDocument(path: 'repositories/{repositoryId}')
class Repository {
  Repository({
    required this.visibility,
  });

  @_visibilityConverter
  final Visibility visibility;
}

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

const _visibilityConverter = _VisibilityConverter();

class _VisibilityConverter implements JsonConverter<Visibility, String> {
  const _VisibilityConverter();

  @override
  Visibility fromJson(String json) => Visibility.fromString(json);

  @override
  String toJson(Visibility visibility) => visibility.name;
}
```

#### FieldValue

Cloud Firestore で値を作成したり更新したりする際には、フィールドに `42` や `[1, 3, 5]` のように具体的な値を与えることもできますが、

- `num` 型のフィールドに対して、現在の値から相対的な値を指定するための `FieldValue.increment(1)`
- `array` 型のフィールドに対して、既になければ指定した値を追加するための `FieldValue.arrayUnion([7])` や、指定した値があれば取り除く `FieldValue.arrayRemove([5])`

のように `FieldValue` を使用した指定方法もあります。

そのような `FieldValue` で指定する可能性があるフィールドには `@allowFieldValue` アノテーションを使用して、下記のように定義することができます。

```dart
@allowFieldValue
final int fieldValueAllowedInt;

@allowFieldValue
final List<String> fieldValueAllowedList;
```

そうすると、`CreateFoo` や `UpdateFoo` のインターフェースとして

- `int` 型の代わりに `FirestoreData<int>` 型
- `List<String>` 型の代わりに `FirestoreData<List<String>>` 型

を使用することになります。

`FirestoreData` 型は [flutterfire_gen_utils](https://pub.dev/packages/flutterfire_gen_utils) パッケージで定義された、sealed class で、下記の 2 つ

- `ActualValue`: `42` や `[1, 3, 5]` のような具体的な値を指定するための型
- `FieldValueData`: `FieldValue` で指定するための型

をまとめています。

よって、たとえば `Counter` ドキュメントの `count` という整数フィールドを更新する際には、下記のように実際の値と `FieldValue` とで更新を実行することができます。

```dart
final query = CounterQuery();

Future<void> updateCount(String counterId, int count) => query.update(
      counterId: counterId,
      updateCounter: UpdateCounter(count: ActualValue<int>(count)),
    );

Future<void> incrementCount(String counterId) => query.update(
      counterId: counterId,
      updateCounter:
          UpdateCounter(count: FieldValueData<int>(FieldValue.increment(1))),
    );
```

## 活用例

さて、以上までで flutterfire_gen によるスキーマ定義の方法、各種アノテーション、生成される Query クラスの仕様などについて説明しました。

ここでは実際に flutterfire_gen を使用した小さな Todo アプリをサンプルとして取り上げながら説明を加えます。

同じくコード生成の仕組みを活用する [riverpod_generator](https://pub.dev/packages/riverpod_generator) と併用しています。

https://github.com/kosukesaigusa/flutterfire_gen_todo

https://pub.dev/packages/riverpod_generator

サンプルアプリでは次のような機能が実装されています。

- Todo 一括取得（`orderBy` 句あり）
- Todo 作成
- Todo 更新
- Todo 削除
- pull to refresh

![todo list](/images/articles/flutterfire_gen/todo_list.gif)

UI については GIF を見るだけで十分察することができる通り、特に説明することはありませんが、このようです：

```dart
class TodoListPage extends ConsumerWidget {
  const TodoListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todoList = ref.watch(todoListProvider);
    return Scaffold(
      appBar: AppBar(title: const Text('Todo List')),
      body: todoList.when(
        data: (todos) => RefreshIndicator(
          onRefresh: () => ref.refresh(todoListProvider.future),
          child: ListView.builder(
            itemCount: todos.length,
            itemBuilder: (context, index) {
              final todo = todos[index];
              return ListTile(
                key: ValueKey(todo.todoId),
                title: Text(todo.title),
                subtitle: Text(todo.todoId),
                leading: Checkbox(
                  value: todo.isCompleted,
                  onChanged: (value) async {
                    if (value == null) {
                      return;
                    }
                    await ref
                        .read(todoListProvider.notifier)
                        .updateCompletionStatus(
                          todoId: todo.todoId,
                          isCompleted: value,
                        );
                    ref.invalidate(todoListProvider);
                  },
                ),
                trailing: IconButton(
                  onPressed: () async {
                    await ref
                        .read(todoListProvider.notifier)
                        .delete(todo.todoId);
                    ref.invalidate(todoListProvider);
                  },
                  icon: const Icon(Icons.delete),
                ),
              );
            },
          ),
        ),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text(e.toString())),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          await ref
              .read(todoListProvider.notifier)
              .addTodo('Todo ${DateTime.now()}');
          ref.invalidate(todoListProvider);
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

驚くべきなのは、UI 以外に記述が必要なソースコードは、riverpod_generator の簡潔さも相まって下記のみで完了してしまうということです。

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutterfire_gen_annotation/flutterfire_gen_annotation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'todo.flutterfire_gen.dart';
part 'todo.g.dart';

@riverpod
TodoQuery todoQuery(TodoQueryRef _) => TodoQuery();

@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<ReadTodo>> build() => ref.watch(todoQueryProvider).fetchDocuments(
        queryBuilder: (query) => query.orderBy('createdAt', descending: true),
      );

  Future<DocumentReference<CreateTodo>> addTodo(String title) =>
      ref.read(todoQueryProvider).add(createTodo: CreateTodo(title: title));

  Future<void> updateCompletionStatus({
    required String todoId,
    required bool isCompleted,
  }) =>
      ref.read(todoQueryProvider).update(
            todoId: todoId,
            updateTodo: UpdateTodo(isCompleted: isCompleted),
          );

  Future<void> delete(String todoId) =>
      ref.read(todoQueryProvider).delete(todoId: todoId);
}

@FirestoreDocument(path: 'todos/{todoId}')
class Todo {
  const Todo({
    required this.title,
    required this.isCompleted,
    required this.createdAt,
    required this.updatedAt,
  });

  final String title;

  @ReadDefault(false)
  @CreateDefault(false)
  final bool isCompleted;

  @alwaysUseFieldValueServerTimestampWhenCreating
  final DateTime? createdAt;

  @alwaysUseFieldValueServerTimestampWhenCreating
  @alwaysUseFieldValueServerTimestampWhenUpdating
  final DateTime? updatedAt;
}
```

今までに説明した通り、`@FirestoreDocument` アノテーションで `Todo` ドキュメントを定義するだけで、`TodoQuery` というクラスに基本的に読み書きメソッドが定義されているので、これを `@riverpod` で `todoQueryProvider` として提供できるようにしています。

```dart
@riverpod
TodoQuery todoQuery(TodoQueryRef _) => TodoQuery();
```

そのため上記のコードでは直接 `FirebaseFirestore.instance` や `withConverter` を自分で繰り返し書くこともなければ、自動で `FieldValue.serverTimestamp()` を適用したいフィールドが作成・更新のインターフェースに登場することもありません。

読み取り時には必ず取得したドキュメント `todoId` が `ReadTodo` インスタンスに含まれていますし、ドキュメントの作成時には、`isCompleted` は任意パラメータ・デフォルト `false` で定義がされています。

それによって、この Todo アプリに必要な取得・作成・更新・削除のすべて振る舞いが、下記のみで簡潔に記述し切ることができています。

```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<ReadTodo>> build() => ref.watch(todoQueryProvider).fetchDocuments(
        queryBuilder: (query) => query.orderBy('createdAt', descending: true),
      );

  Future<DocumentReference<CreateTodo>> addTodo(String title) =>
      ref.read(todoQueryProvider).add(createTodo: CreateTodo(title: title));

  Future<void> updateCompletionStatus({
    required String todoId,
    required bool isCompleted,
  }) =>
      ref.read(todoQueryProvider).update(
            todoId: todoId,
            updateTodo: UpdateTodo(isCompleted: isCompleted),
          );

  Future<void> delete(String todoId) =>
      ref.read(todoQueryProvider).delete(todoId: todoId);
}
```

## 今後の展望

flutterfire_gen の直近の展望・課題としては以下のようなものを考えています。

- 生成されるコードやアノテーションのより良いインターフェースを模索する
  - 例：現状はスキーマ定義専用のクラスとして `Todo` という名前をに使用して、そこに `Read`, `Create`, `Update` のような接頭辞を書くインターフェースにつけているので、もっともぴったりな名前である `Todo` が使えないという考え方もある
- パッケージ内部の doc comment を充実させる（現状はまだまだ適当です）
- 生成ロジックのユニットテストを強固に網羅的に書く
- バッチ書き込みに対応したメソッドも生成する
- 最近公開された [dart_firebase_admin](https://pub.dev/packages/firebase_admin) 向けのコードも生成できるようにする
- ...など

先行して触ってみてくださる方がいれば、ご意見やフィードバックをいただけると嬉しいです！

## おわりに

2 年以上前から似たような範囲をカバーするものに [cloud_firestore_odm](https://pub.dev/packages/cloud_firestore_odm) があったり、すでに各人のボイラプレートコードのテンプレートなどを活用して同様のことを実現したりしているという声があったりもしますが、flutterfire_gen では、生成されたコード自体も読みやすく、生成物を自分でカスタマイズして使うこともしやすいといったところもポイントです。

密にフィードバックをくださる方、または一緒に開発や改善をすることにご興味のある方などがいらっしゃいましたら X でご連絡いただいても嬉しいです！

https://twitter.com/KosukeSaigusa

flutterfire_gen をよろしくお願いします！

https://github.com/kosukesaigusa/flutterfire_gen
