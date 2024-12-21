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

## Pub workspaces (monorepos support) とは

Flutter や Dart のプロジェクトの開発をする際、複数の Flutter や Dart のパッケージを作成して、一緒にバージョンコントロールし、同一の GitHub リポジトリ、ローカルマシン上の同一ワークスペースとして開発することがあります。

- 社内に複数のプロジェクトやアプリケーションが存在し、その共通部分を切り出したい
- 単一のアプリケーションについても、各モジュール・レイヤー間の依存関係を厳密に定義したい
- riverpod, flutter_riverpod, flutter_hooks, riverpod_lint, ... のようなパッケージ開発において、各パッケージを並行して開発したい

などのモチベーションが考えられます。

以前にマルチパッケージを活用した Flutter のアーキテクチャについて過去の記事で紹介しましたので、よろしければ合わせてご覧ください：

https://zenn.dev/omiai_techblog/articles/omiai-flutter-architecture

このようなマルチパッケージ構成では、あるパッケージが他のパッケージに依存したり、各パッケージが同じ外部パッケージに依存したりすることがありますが、それらの依存関係が矛盾なく解決される必要があります。

また、本質的な問題でもありませんが、開発中に各パッケージが依存するパッケージに変更があった場合、複数パッケージそれぞれに対して `flutter pub get` や `dart pub get` を実行するのはやや面倒です。

そのため、マルチパッケージ・モノレポ構成での開発では、これまで Melos が広く使われてきました。

@[card](https://melos.invertase.dev/)

たとえば、本記事執筆時点の riverpod の GitHub リポジトリでも Melos が利用されています。

https://github.com/rrousselGit/riverpod

それに似た機能が Dart 公式でサポートされるようになったのが Pub workspaces です。

公式ドキュメントによると、エディタや IDE でマルチパッケージ構成のワークスペースを開く時、それぞれのパッケージに対して Dart analyzer が別々のコンテキストで動作することから
メモリ使用量が増加してしまう問題がありますが、Pub workspaces を利用することで解析に必要なメモリ量を削減し、パフォーマンスを向上させることもできるそうです。

## Pub workspaces (monorepos support) を利用する

Pub workspaces を利用するためには、リポジトリルートに `pubspec.yaml` を配置し、`workspace` に、ワークスペースとして扱いたいパッケージのパスを列挙します。

`melos.yaml` の `packages` 相当です。

Pub workspaces がサポートされる Dart SDK は 3.6.0 以上であることにも注意が必要です。

```yaml
name: _
publish_to: none
environment:
  sdk: ^3.6.0
workspace:
  - packages/package_a
  - packages/package_b
  - packages/package_c
```

ワークスペース内の各パッケージの `pubspec.yaml` には、`resolution: workspace` を設定します。

```yaml
environment:
  sdk: ^3.6.0
resolution: workspace
```

こうすることで、`dart (flutter) pub get` を実行した際に、下記が実行されます。

ワークスペース内のどのディレクトリで実行しても同様になります。

- プロジェクトルートの `pubspec.yaml` にある `workspace` に列挙されたパッケージの依存関係を解決した `pubspec.lock` を作成する
- プロジェクトルートの `.dart_tool/package_config.json` に、ワークスペース内のパッケージのパスとパッケージ名のマッピングを作成する
- プロジェクトルート以外のワークスペースの各パッケージの `pubspec.lock` と `.dart_tool/package_config.json` を削除する

Melos では、`melos bs` を実行すると、各パッケージの依存パッケージの解決が行われ、各パッケージに `pubspec.lock`, `.dart_tool/package_config.json` が作成されます。

また、Melos では、ワークスペース内の別のパッケージへの依存については、その参照先をローカルの相対パスで一時的に上書きするための `pubspec_overrides.yaml` も生成されます。

上記のような内容が、Pub workspaces と Melos との違いであると言えそうです。

また、Pub workspaces を利用する際、`dart pub add` や `dart pub publish` のようなコマンドは、それを実行しているディレクトリ下のパッケージについてのみ実行されます。

つまり、それらのコマンドを実行する際は `cd` コマンドでディレクトリを移動するか、または `-C` オプションでディレクトリを指定する必要があります。

```sh
# -C オプションで packages/package_a を指定して実行する。
dart pub -C packages/package_a publish

# または、cd コマンドで移動してから実行する。
cd packages/package_a
dart pub publish
```

