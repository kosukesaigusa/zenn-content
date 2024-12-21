---
title: "Pub workspaces (monorepos support) を触ってみた"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: false
---

:::message
この記事は「[Flutter Advent Calendar 2024](https://qiita.com/advent-calendar/2024/flutter)」の 22 日目の記事です🎄

21 日目：[非同期初期化が必要なRiverpodプロバイダの初期化方法](https://zenn.dev/altiveinc/articles/riverpod-provider-initialization) by 村松龍之介さん
22 日目：[Pub workspaces を使ってみた](https://zenn.dev/kosukesaigusa/articles/dart-pub-workspaces) by [Kosuke Saigusa](https://x.com/KosukeSaigusa) ← 今ここ
23 日目：by [ひであ](https://qiita.com/hidea) さん
:::

## はじめに

今年のアドベントカレンダー期間中の 2024/12/12 に Flutter SDK 3.27.0 がリリースされました！

@[card](https://medium.com/flutter/whats-new-in-flutter-3-27-28341129570c)

https://x.com/FlutterDev/status/1866942284882747471

また、同梱される Dart SDK のバージョンが 3.6.0 になり、Pub workspaces (monorepos support) がサポートされるようになりました。

@[card](https://dart.dev/tools/pub/workspaces)

この記事では、上記のドキュメントに従って Pub workspaces を使ってみた内容をサンプルプロジェクトと一緒に紹介します。

