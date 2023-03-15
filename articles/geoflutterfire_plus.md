---
title: "FlutterFire な位置情報クエリが書ける pub パッケージ geoflutterfire_plus の紹介"
emoji: "🌍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firestore"]
published: true
---

## geoflutterfire_plus パッケージ

この記事では、Flutter x Firestore で位置情報クエリがかける pub パッケージである geoflutterfire_plus を紹介し、主な使用方法をまとめたり、サンプルアプリを紹介したりします。

@[card](https://pub.dev/packages/geoflutterfire_plus)

@[card](https://github.com/KosukeSaigusa/geoflutterfire_plus)

このパッケージを使用することで、

- 中心位置（緯度経度）
- 検出半径 (km)
- （任意）その他のフィルタ条件

を入力として、Cloud Firestore に保存済みの位置情報を Geohash の仕組みを用いてリアルタイム取得（または任意のタイミングでリクエストして都度取得）をすることができます。

## geoflutterfire_plus 開発の背景

既出の同様のパッケージとして、geoflutterfire というものがあります。

@[card](https://pub.dev/packages/geoflutterfire)

しかし、このパッケージは記事執筆時点で約 1 年ほどメンテナンスが止まっており、肝心の `cloud_firestore` などの依存パッケージが最新に対応しておらず、開発中のアプリが新しいバージョンの `cloud_firestore` パッケージを使用している場合には、依存関係の解決時にコンフリクトが生じしてしまい、使用することができません。

依存パッケージを最新にする PR を作成することも検討しましたが、その流れで内部のソースコードを詳しく読んでみると、ソースコードや設計にも根本から修正した方が良いと思われる箇所が多く、それなら一からパッケージを作り直して公開しようと思ったのが開発のきっかけです。

また、依存パッケージを最新に保つメンテナンスがほぼ中心で geoflutterfire2 というパッケージもあります。

@[card](https://pub.dev/packages/geoflutterfire2)

が、基本的なコードは元の geoflutterfire パッケージと変わりません。

## Geohash について

この章では geoflutterfire_plus パッケージが位置情報クエリを動作させるために使用している Geohash という仕組みについて、参考になる情報を示しながら、かんたんに解説します。

geoflutterfire_plus パッケージをただ使用して、Flutter で位置情報関係のアプリ開発を行いたいだけの方は軽く読むか、読み飛ばすかして、パッケージ内部への興味がわいたときに詳しく読んでみても良いかもしれません。

## 基本的な機能や使い方の紹介

### 緯度経度の取り扱い

geoflutterfire_plus は `cloud_firestore` パッケージに依存しているので、緯度経度に関する情報を取り扱うときは、`cloud_firestore` パッケージで定義されている `GeoPoint` クラスを使用すると良いです。`geoflutterfire_plus` パッケージも内部でそのようにしています。

`GeoPoint` クラスは単に緯度経度をメンバにもつクラスです。

```dart
GeoPoint tokyoStation = GeoPoint(35.681236, 139.767125);
```

### 位置情報データを保存する (add, set)

### 位置情報データを更新する (update)

### 位置情報データを削除する (delete)

### 位置情報データを取得する (list)

#### リアルタイムで取得する (Stream)

#### 都度取得する (Future)

#### 任意のフィルタ条件（where 句）を追加する

## geoflutterfire パッケージとの対応（移行ガイドとしても）
