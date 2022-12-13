---
title: "Flutter x Firestore で無限スクロールのチャット機能を実装する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firestore"]
published: false
---

## 本記事のゴール

本記事では、Flutter x Firestore で無限スクロールのチャット機能を実装する方法を説明します。

ゴールは次の動画のようなものです。

TODO: あとで動画などをはる

## サンプルリポジトリ

本記事のサンプルリポジトリです：

@[card](https://github.com/KosukeSaigusa/flutter-infinite-scroll-chat)

本記事で取り上げるすべてのソースコードが含まれているので適宜参照してください。

## 実装する機能と特徴

本記事のサンプルでは、次のような機能が実装されています。

- チャットページに入ったときに、最新 10 件メッセージだけを取得して表示する
- 過去のメッセージは、一気にすべて取得することなく、画面をスクロールして遡るのに従って 10 件ずつ取得して表示する
- チャットページに入った時間以降の最新のメッセージは、リアルタイムですべて取得して表示する

一般に、無限スクロールとは、ユーザーがページの切り替えをすることなく、画面をスクロールすることでコンテンツを自動的に読み込んでいくような機能を指します。

それに加えて本記事では、チャット機能を取り上げることで、過去のコンテンツ（メッセージ）は無限スクロールで少しずつ読み込み、途中でやってくる最新のコンテンツ（メッセージ）はリアルタイムですべて取得する、という機能の両立を図る実装を紹介します。

サンプルでは Cloud Firestore を用いているので、この実装をすることで、チャット機能に必要な体験を損なうことなく、メッセージドキュメントの読み取り回数（コスト）を最適化する方法を学ぶのに役立ちます。

また、Cloud Firestore の利用の有無やチャット機能の実装かどうかにかかわらず、Flutter で無限スクロールを実装するときの参考にもなると思います。

## 登場するクラス

今回は主に、次の 3 つのクラスに分けて実装を行います。

- `ChatRoomPage` クラス：チャット画面の UI を構成するウィジェット (`ConsumerWidget`)
- `ChatController` クラス：チャット画面での各種操作を解釈し、`Chat` モデルを操作する
- `Chat` クラス：`Chat` モデルとして、チャット機能のふるまいを表現する

`riverpod`, `freezed`, `cloud_firestore` パッケージの利用を前提とした実装・説明となっているので、同様のパッケージを用いない場合は適宜読み替えてください。

## UI (ChatRoomPage) の実装

本記事ではチャット機能としてそれっぽい画面を作るウィジェットの組み方の詳細の説明は行いませんので、必要に応じてサンプルアプリを参考にしてください。

重要な箇所だけ抜き出したり、説明のために一部かんたんにしたりして、次のように実装しています。

```dart:lib/features/chat/ui/chat_room_page.dart
/// チャットルーム画面。
class ChatRoomPage extends ConsumerWidget {
  const ChatRoomPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // チャットルームの ID（サンプルアプリでは Provider 経由で Route のパスパラメータから取得するような実装になっています）。
    final chatRoomId = 'some-chat-room-id';
    // チャットコントローラ。
    final controller = ref.watch(chatControllerProvider(chatRoomId));
    // 取得されたメッセージ一覧。
    final messages = ref.watch(chatProvider(chatRoomId).select((s) => s.messages));
    return Scaffold(
      // messages.length の数だけ
      // _MessageItem ウィジェットを ListView.builder で並べる。
      body: ListView.builder(
        // チャットコントローラの ScrollController インスタンス。
        controller: controller.scrollController,
        itemBuilder: (context, index) => _MessageItem(message: messages[index]),
        itemCount: messages.length,
        reverse: true,
      ),
    );
  }
}
```

多くありませんが、注目すべきポイントは

- `ChatRoomController` クラスの `ScrollController` インスタンスを、`ListView` の `controller` 属性に指定していること
- `Chat` クラスの `messages` の数だけ、`ListView.builder` で `MessageItem` ウィジェットを並べていること

くらいでしょうか。

`ChatRoomController` クラスに記述している `ScrollController` のリスナーの設定によって、画面をある程度スクロールする次の 10 件のメッセージを取得する機能を実現しています（後述）。

`ref.watch` で `chatProvider` をリッスンしているので、`messages` 変数に変更がある（新たなメッセージが追加される）ごとにリアクティブに画面上にそれらのメッセージが反映されます。

## ChatController（コントローラ）の実装

次に `ChatController` クラスの説明を行います。行っていることはそれほど多くありません。また、場合によっては上記の UI を `StatefulWidget` にすることで同等の実装を行っても良いでしょう。

重要な箇所だけ抜き出したり、説明のために一部かんたんにしたりして、次のように実装しています。

```dart:lib/features/chat/chat_controller.dart
final chatControllerProvider = Provider.autoDispose.family<ChatController, String>(
  (ref, chatRoomId) => ChatController(ref, ref.read(chatProvider(chatRoomId).notifier)),
);

/// チャット画面での各種操作を行うコントローラ。
class ChatController {
  ChatController(this._ref, this._chat) {
    _initialize();
    _ref.onDispose(() async {
      await _newMessagesSubscription.cancel();
      scrollController.dispose();
    });
  }

  final AutoDisposeProviderRef<ChatController> _ref;

  /// チャットモデルのインスタンス。
  final Chat _chat;

  /// 新着メッセージのサブスクリプション。
  late final StreamSubscription<List<Message>> _newMessagesSubscription;

  /// メッセージを表示する ListView のコントローラ。
  late final ScrollController scrollController;

  /// 無限スクロールで取得するメッセージ件数の limit 値。
  static const _limit = 10;

  /// 画面の何割をスクロールした時点で次の _limit 件のメッセージを取得するか。
  static const _scrollValueThreshold = 0.8;
}

  /// 初期化処理。コンストラクタメソッド内でコールする。
  void _initialize() {
    _initializeScrollController();
    _initializeNewMessagesSubscription();
  }

  /// ListView の ScrollController を初期化して、
  /// 過去のメッセージを遡って取得するための Listener を設定する。
  void _initializeScrollController() {
    scrollController = ScrollController()
      ..addListener(() async {
        final scrollValue = scrollController.offset / scrollController.position.maxScrollExtent;
        if (scrollValue > _scrollValueThreshold) {
          await _chat.loadMore(limit: _limit);
        }
      });

  /// 読み取り開始時刻以降のメッセージを購読して
  /// 画面に表示する messages に反映させるリスナーを初期化する。
  void _initializeNewMessagesSubscription() {
    _newMessagesSubscription = _chat.newMessagesSubscription;
  }
}
```

まずは `ChatController` クラスのコンストラクタとメンバ変数を見ていきます。

コンストラクタ引数として `Chat` モデルのインスタンスを受け取り、コンストラクタメソッドの中でプライベートな `_initialize()` メソッドを実行しています（後述）。

その他、新着メッセージのサブスクリプションや前述のメッセージ一覧の `ListView` ウィジェットの `controller` 属性に指定する `ScrollController` 型の変数をメンバとして定義しています。

`_limit` は、無限スクロールで取得するメッセージの件数を、`_scrollValueThreshold` は画面の何割をスクロールした時点でさらにメッセージを遡って取得するかの閾値を表しています。

```dart:lib/features/chat/chat_controller.dart
/// チャット画面での各種操作を行うコントローラ。
class ChatController {
  ChatController(this._ref, this._chat) {
    _initialize();
    _ref.onDispose(() async {
      await _newMessagesSubscription.cancel();
      scrollController.dispose();
    });
  }

  final AutoDisposeProviderRef<ChatController> _ref;

  /// チャットモデルのインスタンス。
  final Chat _chat;

  /// 新着メッセージのサブスクリプション。
  late final StreamSubscription<List<Message>> _newMessagesSubscription;

  /// メッセージを表示する ListView のコントローラ。
  late final ScrollController scrollController;

  /// 無限スクロールで取得するメッセージ件数の limit 値。
  static const _limit = 10;

  /// 画面の何割をスクロールした時点で次の _limit 件のメッセージを取得するか。
  static const _scrollValueThreshold = 0.8;
}
```

`_initialize()` メソッドは次のように実装しています。

`_initializeScrollController()` コントローラメソッドでは、`ScrollController` のインスタンスを `scrollController` 変数に格納した上で、`addLister` でスクロールの変化をリッスンします。

リスナーの中で、スクロール量が閾値を超えたときに `Chat` モデルの `loadMore()` メソッドをコールすることで、次の `_limit` 件のメッセージを取得します（後述）。

`_newMessagesSubscription` 変数には単に `Chat` モデルの同名の変数 (getter) の値を格納しているだけです。

```dart:lib/features/chat/chat_controller.dart
class ChatController {
  // ... 省略

  /// 初期化処理。コンストラクタメソッド内でコールする。
  void _initialize() {
    _initializeScrollController();
    _initializeNewMessagesSubscription();
  }

  /// ListView の ScrollController を初期化して、
  /// 過去のメッセージを遡って取得するための Listener を設定する。
  void _initializeScrollController() {
    scrollController = ScrollController()
      ..addListener(() async {
        final scrollValue = scrollController.offset / scrollController.position.maxScrollExtent;
        if (scrollValue > _scrollValueThreshold) {
          await _chat.loadMore(limit: _limit);
        }
      });

  /// 読み取り開始時刻以降のメッセージを購読して
  /// 画面に表示する messages に反映させるリスナーを初期化する。
  void _initializeNewMessagesSubscription() {
    _newMessagesSubscription = _chat.newMessagesSubscription;
  }
}
```
