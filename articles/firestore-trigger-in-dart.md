---
title: "GCP を活用して Firestore トリガー関数を Dart で動かす"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "GCP", "gc24"]
published: false
published_at: 2024-02-25 00:00
---

## この記事について

:::message
この記事は [記事投稿キャンペーン2024 Google Cloud](https://zenn.dev/topics/gc24) としても投稿しています！
:::

この記事では、GCP のいろいろなサービスを活用して Firestore トリガー関数を Dart で動かす方法について説明します。

私自身、個人開発などでは Firebase をサーバサイドとしたアプリケーションを開発することが多いのですが、その際、サーバサイドで行うべき処理はこれまで TypeScript (JavaScript) で実装して Cloud Functions にデプロイしてきました。

普段は Flutter, Dart エンジニアをしており、Dart 言語が大好きなので、Dart でサーバサイドも実装することができれば嬉しいなと日々思っていたところ、それが実現できそうな方法を探ることができたのでまとめます。

GCP の Cloud Run, Eventarc, Secret Manager や Workload Identity などを活用して

- HTTP 関数を Dart で書いてデプロイする
- Firestore トリガー関数を Dart で書いてデプロイする
- GitHub Actions で、ある程度快適なデプロイワークフローを構築する

ことを目標とします。

注意点として、

- この資料で紹介する [dart_firebase_admin](https://pub.dev/packages/dart_firebase_admin) パッケージは、まだ初期段階で対応していない機能やバグが含まれている可能性があり、時には自らそのバグを追って解決する必要がある
- Firebase CLI 内部で高度に抽象化されているデプロイタスクなどを gcloud コマンドで自前で記述する必要がある

などの、根気や開拓精神も一定必要になります。

よって現時点で「今後は Firebase で使用するサーバサイドの処理は Dart で書こう！」と強く推奨する意図はありません。

が、個人的には、まだ試しきれていない機能も触ってみつつ、今後は積極的に Dart で Firebase で使用するサーバサイドの処理を書いていこうと考えていますし、すでにそのくらいは使える水準に達してきていると思っています。

