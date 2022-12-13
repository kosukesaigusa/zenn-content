---
title: "Flutter x Firestore で無限スクロールのチャット機能を実装する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firestore"]
published: true
---

## 本記事のゴール

本記事では、Flutter x Firestore で無限スクロールのチャット機能を実装する方法を説明します。

ゴールは次の動画のようなものです。

![flutterfire-infinite-scroll](/images/articles/flutterfire-infinite-scroll-chat/flutterfire-infinite-scroll.gif)

やや見にくいですが、画面上部のグレー背景のデバッグウィンドウの「取得したメッセージ」が、スクロール操作に従って、少しずつ増えている挙動を確認することができます。

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

## コントローラ (ChatController) の実装

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

## モデル (Chat) の実装

最後にチャット機能の振る舞いを記述する `Chat` クラスの説明を行います。

重要な箇所だけ抜き出したり、説明のために一部かんたんにしたりして、次のように実装しています。

```dart:lib/features/chat/chat.dart
final chatProvider =
    StateNotifierProvider.autoDispose.family<Chat, ChatState, String>(Chat.new);

/// ChatState の操作とチャット機能の振る舞いを記述したモデル。
class Chat extends StateNotifier<ChatState> {
  Chat(this._ref, this._chatRoomId) : super(const ChatState()) {
    Future<void>(() async {
      await loadMore();
      state = state.copyWith(loading: false);
    });
  }

  final AutoDisposeStateNotifierProviderRef<Chat, ChatState> _ref;

  /// チャットルームの ID。
  final String _chatRoomId;

  /// 無限スクロールで取得するメッセージ件数の limit 値。
  static const _limit = 10;

  /// この時刻以降のメッセージを新たなメッセージとしてリアルタイム取得する。
  final startDateTime = DateTime.now();

  /// 新着メッセージのサブスクリプション。
  /// リスナーで state.newMessages を更新する。
  StreamSubscription<List<Message>> get newMessagesSubscription => _ref
      .read(baseChatRepositoryProvider)
      .subscribeMessages(
        chatRoomId: _chatRoomId,
        queryBuilder: (q) => q
            .orderBy('createdAt', descending: true)
            .where('createdAt', isGreaterThanOrEqualTo: startDateTime),
      )
      .listen(_updateNewMessages);

  /// 過去のメッセージを、最後に取得した queryDocumentSnapshot 以降の
  /// limit 件だけ取得する。
  Future<void> loadMore() async {
    if (!state.hasMore) {
      state = state.copyWith(fetching: false);
      return;
    }
    if (state.fetching) {
      return;
    }
    state = state.copyWith(fetching: true);
    final qs = await _ref.read(baseChatRepositoryProvider).loadMoreMessagesQuerySnapshot(
          limit: _limit,
          chatRoomId: _chatRoomId,
          lastReadQueryDocumentSnapshot: state.lastReadQueryDocumentSnapshot,
        );
    final messages = qs.docs.map((qds) => qds.data()).toList();
    _updatePastMessages([...state.pastMessages, ...messages]);
    state = state.copyWith(
      fetching: false,
      lastReadQueryDocumentSnapshot: qs.docs.isNotEmpty ? qs.docs.last : null,
      hasMore: qs.docs.length >= _limit,
    );
  }

  /// 取得したメッセージ全体を更新する。
  void _updateMessages() {
    state = state.copyWith(messages: [...state.newMessages, ...state.pastMessages]);
  }

  /// チャットルーム画面に遷移した後に新たに取得したメッセージを更新した後、
  /// 取得したメッセージ全体も更新する。
  void _updateNewMessages(List<Message> newMessages) {
    state = state.copyWith(newMessages: newMessages);
    _updateMessages();
  }

  /// チャットルーム画面を遡って取得した過去のメッセージを更新した後、
  /// 取得したメッセージ全体も更新する。
  void _updatePastMessages(List<Message> pastMessages) {
    state = state.copyWith(pastMessages: pastMessages);
    _updateMessages();
  }
}
```

また、その状態クラスは `freezed` を用いて次のように定義しています。

```dart:lib/feature/chat/chat_state.dart
// ...省略

@freezed
class ChatState with _$ChatState {
  const factory ChatState({
    /// チャットページに入ったときの初回ローディング中かどうか。
    @Default(true) bool loading,

    /// 取得したメッセージ全体。
    @Default(<Message>[]) List<Message> messages,

    /// 取得した新着メッセージ。
    @Default(<Message>[]) List<Message> newMessages,

    /// 遡って取得した過去のメッセージ。
    @Default(<Message>[]) List<Message> pastMessages,

    /// 無限スクロールで遡って過去のメッセージを取得中かどうか。
    @Default(false) bool fetching,

    /// 無限スクロールで遡る際にまだ取得するメッセージが残っているかどうか。
    @Default(true) bool hasMore,

    /// 無限スクロールで遡って取得した最後のドキュメントのクエリスナップショット。
    QueryDocumentSnapshot<Message>? lastReadQueryDocumentSnapshot,
  }) = _ChatRoomState;
}
```

まずは `Chat` クラスのコンストラクタとメンバ変数を見ていきます。

チャットルームの ID を引数として受け取りつつ、コンストラクタメソッドの中で `loadMore()` メソッド（後述）を実行し、終了後 `loading` を `false` にしています。この処理で、チャットページを表示したときに最初の 10 件のメッセージを取得します。

```dart:lib/features/chat/chat.dart
/// ChatState の操作とチャットルームページの振る舞いを記述したモデル。
class Chat extends StateNotifier<ChatState> {
  Chat(this._ref, this._chatRoomId) : super(const ChatState()) {
    Future<void>(() async {
      await loadMore();
      state = state.copyWith(loading: false);
    });
  }
}
```

`loadMore()` メソッドの実装は次のようになっています。

```dart:lib/features/chat/chat.dart
/// 過去のメッセージを、最後に取得した queryDocumentSnapshot 以降の _limit 件だけ取得する。
Future<void> loadMore() async {
  // 遡って取得するドキュメント（メッセージ）がこれ以上無い場合は先に進まない。
  if (!state.hasMore) {
    state = state.copyWith(fetching: false);
    return;
  }

  // 現在遡って取得している場合は先に進まない。
  if (state.fetching) {
    return;
  }

  // 遡って取得を始める。
  state = state.copyWith(fetching: true);

  // 前回最後に取得したドキュメント以降の最大 _limit 件の QueryDocumentSnapshot を取得する。
  final qs = await _ref.read(baseChatRepositoryProvider).loadMoreMessagesQuerySnapshot(
        limit: _limit,
        chatRoomId: _chatRoomId,
        lastReadQueryDocumentSnapshot: state.lastReadQueryDocumentSnapshot,
      );

  // 今回取得した最大 _limit 件のメッセージ
  final messages = qs.docs.map((qds) => qds.data()).toList();

  // 遡って取得したメッセージを追加して更新する。
  _updatePastMessages([...state.pastMessages, ...messages]);

  // 取得中のステータスを false に戻し、最後に取得した QueryDocumentSnapshot を保存し、
  // 今回取得したドキュメントの件数が _limit 件に満たない場合は hasMore を false にする。
  state = state.copyWith(
    fetching: false,
    lastReadQueryDocumentSnapshot: qs.docs.isNotEmpty ? qs.docs.last : null,
    hasMore: qs.docs.length >= _limit,
  );
}
```

各処理にコメントを付加している通りですが、少し補足します。

まず、実際にドキュメントの取得処理に進む前に、

- まだ取得することのできるドキュメントが残っているか
- 前回の取得処理の途中ではないか

をチェックします。`loadMore()` メソッドは画面が一定以上スクロールされる度にコールされるので、もう読み取るドキュメントがないのに何度も呼び出されたり、前回の `loadMore()` の処理の途中なのに再度呼び出されたりすることを避ける目的です。このようなチェックを行わないと、何度も何度も無駄な読み込み処理が走ることになってしまい、期待通りに動作しないので気をつけてください。

チェックを行った後、取得を始める際に `fetching` を `true` に更新します。

```dart:lib/features/chat/chat.dart
Future<void> loadMore() async {
  // 遡って取得するドキュメント（メッセージ）がこれ以上無い場合は先に進まない。
  if (!state.hasMore) {
    state = state.copyWith(fetching: false);
    return;
  }

  // 現在遡って取得している場合は先に進まない。
  if (state.fetching) {
    return;
  }

  // 遡って取得を始める。
  state = state.copyWith(fetching: true);
  }

  // ... 省略
}
```

いよいよ実際の取得処理です。リポジトリクラスの該当メソッド (`loadMoreMessagesQuerySnapshot()`) をコールし、結果（`QuerySnapshot<Message>` 型）をローカル変数 `qs` に格納しています。

```dart:lib/features/chat/chat.dart
Future<void> loadMore() async {
  // ... 省略

  final qs = await _ref.read(baseChatRepositoryProvider).loadMoreMessagesQuerySnapshot(
        limit: _limit,
        chatRoomId: _chatRoomId,
        lastReadQueryDocumentSnapshot: state.lastReadQueryDocumentSnapshot,
      );
  
  // ... 省略
}
```

テスト可能にするために、チャット機能のリポジトリのインターフェースである `BaseChatRepository` クラスを定義し、それを実装した `ChatRepository` クラスは Firestore に接続されています。

ここに `loadMoreMessagesQuerySnapshot()`  メソッドを定義（オーバーライド）しており、メッセージを新しい順（`createdAt` のの降順）に並べた上で、最後に取得したドキュメント（の `QueryDocumentSnapshot` である `lastReadQueryDocumentSnapshot`）が指定されている場合には、Firestore の `startAfterDocument` クエリを使用して、前回最後に取得したドキュメント以降のドキュメントから `limit` 件を取得するための `QuerySnapshot<Message>` を返します。

```dart:lib/repositories/chat.dart
/// チャット機能関係の、データソースが Firestore であるリポジトリの実装クラス。
class ChatRepository implements BaseChatRepository {
  @override
  Future<QuerySnapshot<Message>> loadMoreMessagesQuerySnapshot({
    required int limit,
    required String chatRoomId,
    required QueryDocumentSnapshot<Message>? lastReadQueryDocumentSnapshot,
  }) {
    var query = messagesRef(chatRoomId: chatRoomId).orderBy('createdAt', descending: true);
    final qds = lastReadQueryDocumentSnapshot;
    if (qds != null) {
      // 最後に取得したドキュメント以降から取得するためのクエリ。
      query = query.startAfterDocument(qds);
    }
    return query.limit(limit).get();
  }
}
```

`loadMore()` メソッドの最後では、上記で取得された `QuerySnapshot<Message>` からメッセージ一覧で state を更新する他、最後に state の

1. `fetching` を `false` に戻す
2. 最後に読み取ったドキュメントを記録する
3. まだ読み取れるドキュメントが残っているか判断して記録する

ための操作を行います。

2 は、今回取得した `QuerySnapshot<Message>` が空でない場合、そのドキュメント一覧の最後の要素 (`qs.docs.last`) を保持するようすれば良いでしょう。

3 は、今回取得した `QuerySnapshot<Message>` に含まれるドキュメントの数が `_limit` 件と等しいかどうかで判断できます。

```dart:lib/features/chat/chat.dart
Future<void> loadMore() async {
  // ... 省略

  final qs = await _ref.read(baseChatRepositoryProvider).loadMoreMessagesQuerySnapshot(
        limit: _limit,
        chatRoomId: _chatRoomId,
        lastReadQueryDocumentSnapshot: state.lastReadQueryDocumentSnapshot,
      );

  // 取得した新たな最大 _limit 件のメッセージで state を更新する。
  final messages = qs.docs.map((qds) => qds.data()).toList();
  _updatePastMessages([...state.pastMessages, ...messages]);

  state = state.copyWith(
    // fetching を false に戻す。
    fetching: false,
    // 最後に読み取ったドキュメントを記録する。
    lastReadQueryDocumentSnapshot: qs.docs.isNotEmpty ? qs.docs.last : null,
    // 取得できた _limit 件に満たない場合は hasMore: false となる（実際には `==` でも同等）。
    hasMore: qs.docs.length >= _limit,
  );
}
```

以上が、チャットルームの過去のメッセージをスクロールしながら随時取得したものを `state.messages` に反映していくための処理です。

一方で、チャットルームに入ってから受信する新着メッセージは、スクロールに関係なく、すべてリアルタイムに `state.messages` に反映していく必要があります。

これは、`Chat` クラスをインスタンス化した現在時刻を基準に、それよりも新しいメッセージが取得され次第発火するリスナーを定義し、それをもとに `state.messages` を更新していけば良いでしょう。

```dart:lib/features/chat/chat.dart
class Chat extends StateNotifier<ChatState> {
  /// 新着メッセージのサブスクリプション。
  /// リスナーで state.newMessages を更新する。
  StreamSubscription<List<Message>> get newMessagesSubscription => _ref
      .read(baseChatRepositoryProvider)
      .subscribeMessages(
        chatRoomId: _chatRoomId,
        queryBuilder: (q) => q
            .orderBy('createdAt', descending: true)
            .where('createdAt', isGreaterThanOrEqualTo: startDateTime),
      )
      .listen(_updateNewMessages);
}
```

リポジトリクラスの `subscribeMessages()` メソッドは次のように定義しています。

```dart:lib/repositories/chat.dart
class ChatRepository implements BaseChatRepository {
  @override
  Stream<List<ChatRoom>> subscribeChatRooms({
    Query<ChatRoom>? Function(Query<ChatRoom> query)? queryBuilder,
    int Function(ChatRoom lhs, ChatRoom rhs)? compare,
  }) {
    Query<ChatRoom> query = chatRoomsRef;
    if (queryBuilder != null) {
      query = queryBuilder(query)!;
    }
    return query.snapshots().map((qs) {
      final result = qs.docs.map((qds) => qds.data()).toList();
      if (compare != null) {
        result.sort(compare);
      }
      return result;
    });
  }
}
```

以上により、

- チャット画面をスクロールすることで順次取得されたメッセージは `state.pastMessages` に保持されながら
- チャット画面を表示して以降の新着メッセージはリアルタイムですべて `state.newMessages` に保持されながら

取得したメッセージ全体を保持する `state.messages` を更新していくような実装ができました。

`state.fetching` や `state.hasMore` のフラグを管理しながら無駄な読み込み（意図しない無限読み込み）のないように実装することも重要です。

## 最後に

長めの記事となりましたが最後までお読みいただきありがとうございました。

詳細や、他の実装（Riverpod の使い方、型安全な Firestore の `CollectionReference` や `DocumentReference` の定義、その他の汎用コード）も含めて下記のリポジトリも参考にしていただけると幸いです。

@[card](https://github.com/KosukeSaigusa/flutter-infinite-scroll-chat)

続編として、上記のチャット機能の振る舞いを記述した `Chat` クラスのパブリックメソッドに対するユニットテストを書く記事の執筆も予定しているのでもうしばらくお待ちください！
