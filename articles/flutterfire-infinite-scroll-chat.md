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

重要な箇所だけ抜き出したり、説明のために一部かんたんにしたして、次のようになります。

```dart:lib/features/chat/ui/chat_room_page.dart
class ChatRoomPage extends ConsumerWidget {
  const ChatRoomPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // チャットルームの ID（サンプルアプリでは Provider 経由で Route のパスパラメータから取得するような実装になっています）。
    final chatRoomId = 'some-chat-room-id';
    // チャットコントローラ
    final controller = ref.watch(chatControllerProvider(chatRoomId));
    // 取得されたメッセージ一覧
    final messages = ref.watch(chatProvider(chatRoomId).select((s) => s.messages));
    return Scaffold(
      // messages.length の数だけ
      // _MessageItem ウィジェットを ListView.builder で並べる
      body: ListView.builder(
        // チャットコントローラの ScrollController インスタンス
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
