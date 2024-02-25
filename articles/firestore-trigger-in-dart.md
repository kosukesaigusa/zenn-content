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

## Cloud Run

Cloud Run に関する説明を引用します：

> Cloud Run は、Google のスケーラブルなインフラストラクチャ上でコンテナを直接実行できるマネージドコンピューティングプラットフォームです。
>
> コンテナイメージをビルドできるものであれば、任意のプログラミング言語で記述されたコードを Cloud Run にデプロイできます。コンテナ イメージのビルドは任意です。Go、Node.js、Python、Java、.NET Core、Ruby を使用している場合は、使用している言語のベスト プラクティスに従って、コンテナをビルドする[ソースベースのデプロイ](https://cloud.google.com/run/docs/deploying-source-code?hl=ja) オプションを使用できます。

https://cloud.google.com/run/docs/overview/what-is-cloud-run?hl=ja

昨年の Google I/O 2023 でも紹介されていましたが（GA になったのは 2022 年ですが）、Cloud Functions (2nd Gen) は裏側では Cloud Run で動くようになっています。

Cloud Functions (1st Gen) を使用する場合と比べても機能的には大差ありません。
`
参考：

https://zenn.dev/cloud_ace/articles/b30971199b392c

重要なことは、Cloud Run の「コンテナイメージをビルドできるものであれば、任意のプログラミング言語で記述されたコードをデプロイできる」という特徴により、Dart で書いた関数（サービス）を Cloud Run にデプロイして動かすことができるということです。

