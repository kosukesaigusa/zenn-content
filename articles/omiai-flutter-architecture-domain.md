---
title: "【domain パッケージ】Omiai の Flutter プロジェクトのアーキテクチャ"
emoji: "🗄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: false
publication_name: "omiai_techblog"
---

## この記事について

以前の「Omiai の Flutter プロジェクトのアーキテクチャ」という記事：

@[card](https://zenn.dev/kosukesaigusa/articles/omiai-flutter-architecture)

@[card](https://zenn.dev/kosukesaigusa/articles/omiai-flutter-architecture-system)

@[card](https://zenn.dev/kosukesaigusa/articles/omiai-flutter-architecture-repository)

の続編として、`domain` パッケージの具体的な実装内容について紹介します。

## domain パッケージ

Omiai の Flutter プロジェクトのパッケージ構成は下図の通りです。

```mermaid
graph TD
    A[app] --> B[base_ui]
    A --> C[domain]
    A --> G[dependency_provider]
    C --> F[repository]
    F --> D[system]
    F --> G
    G --> D
```

## 実装内容

## おわりに

この記事では、Omiai の Flutter プロジェクトのアーキテクチャ紹介の続編として、`domain` パッケージについて説明しました。

Omiai の Flutter プロジェクトへの参画にご興味のある方は、ぜひ一度お気軽にお問い合わせください！
