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
