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

依存パッケージを最新にする PR を継続して作成することも検討しましたが、その流れで内部のソースコードを詳しく読んでみると、ソースコードや設計にも根本から修正した方が良いと思われる箇所が多く、しばらくはフォークして自分なりに改善したものを使用していました。その修正がまとまったものになってきたタイミングで、一からパッケージを作り直して公開しようと思ったのが開発のきっかけです。

また、依存パッケージを最新に保つメンテナンスがほぼ中心で geoflutterfire2 というパッケージもあります。

@[card](https://pub.dev/packages/geoflutterfire2)

が、基本的なコードは元の geoflutterfire パッケージと変わりません。

## Geohash について

この章では geoflutterfire_plus パッケージが位置情報クエリを動作させるために使用している Geohash という仕組みについて、参考になる情報を示しながら、かんたんに解説します。

geoflutterfire_plus パッケージをただ使用して、Flutter で位置情報関係のアプリ開発を行いたいだけの方は軽く読むか、読み飛ばすかして、パッケージ内部への興味がわいたときに詳しく読んでみても良いかもしれません。

## 基本的な機能や使い方の紹介

この章では、geoflutterfire_plus パッケージの基本的な機能や使い方を紹介します。

説明の目的で型注釈などは冗長に書いている箇所があります。

### 緯度経度の取り扱い

geoflutterfire_plus は `cloud_firestore` パッケージに依存しているので、緯度経度に関する情報を取り扱うときは、`cloud_firestore` パッケージで定義されている `GeoPoint` クラスを使用すると良いです。`geoflutterfire_plus` パッケージも内部でそのようにしています。

`GeoPoint` クラスは単に緯度経度をメンバにもつクラスです。

```dart
// 東京駅の緯度経度。
const GeoPoint tokyoStation = GeoPoint(35.681236, 139.767125);
```

### 位置情報を Cloud Firestore で取り扱う (`GeoCollectionReference`)

位置情報を Cloud Firestore で取り扱うということは、何らかのコレクションにそのようなデータを保存するということになります。

たとえばトップレベルの locations という名前のコレクションがそれを担当するとするとき、geoflutterfire_plus パッケージが提供する `GeoCollectionReference` クラスを用いて、そのコレクションへの参照を定義することができます。

`GeoCollectionReference` のコンストラクタは、通常の `cloud_firestore` パッケージの `CollectionReference<T>` 型のコレクションへの参照を要求します。

```dart
// 通常通り CollectionReference を定義する。
final CollectionReference<Map<String, dynamic>> collectionReference =
    FirebaseFirestore.instance.collection('locations');

// GeoCollectionReference を定義する。
final GeoCollectionReference<Map<String, dynamic>> geoCollectionReference =
    GeoCollectionReference(collectionReference);
```

`withConverter` を用いて型を付けることにも対応しています。仮に、`Location` というクラスを定義して、`fromDocumentSnapshot` や `toJson` メソッドを定義しているとすると、次のようになります。

```dart
// 通常通り型付きの CollectionReference を定義する。
CollectionReference<Location> typedCollectionReference =
    FirebaseFirestore.instance.collection('locations').withConverter<Location>(
          fromFirestore: (ds, _) => Location.fromDocumentSnapshot(ds),
          toFirestore: (obj, _) => obj.toJson(),
        );

// 型付きのGeoCollectionReference を定義する。
final GeoCollectionReference<Location> typedGeoCollectionReference =
    GeoCollectionReference(typedCollectionReference);
```

この場合の `Location` クラスの内容を具体的に確認したい場合は次のファイルの該当箇所を参照してください：

@[card](https://github.com/KosukeSaigusa/geoflutterfire_plus/blob/main/example/lib/advanced/entity.dart)

### 位置情報データを定義する (`GeoFirePoint`)

geoflutterfire_plus パッケージでは、位置情報を取り扱うための `GeoFirePoint` というクラスを定義しています。

`GeoFirePoint` クラスは、`GeoPoint`（緯度経度）をコンストラクタ引数と作成することができ、`.geohash` でその Geohash 文字列を計算して返したり、`.data` で `GeoPoint` と Geohash 文字列をセットで返したりする機能が実装されています。

以下の例で示すように単に `geoFirePoint.geohash` とするだけで、該当地点の Geohash 文字列を得ることができます。

```dart
// 東京駅の緯度経度。
const GeoPoint tokyoStation = GeoPoint(35.681236, 139.767125);

// GeoPoint インスタンスを渡して GeoFirePoint を定義する。
const GeoFirePoint geoFirePoint = GeoFirePoint(tokyoStation);

// Geohash を取得する。東京駅の Geohash 文字列 'xn76urx4r' が出力される。
print(geoFirePoint.geohash);

// GeoPoint と Geohash 文字列をもつ Map<String, dynamic> を返す。
// <String, dynamic>{
//   'geopoint': GeoPoint(35.681236, 139.767125),
//   'geohash': xn76urx4r
// }
print(geoFirePoint.data);
```

### 位置情報データを保存する (add, set)

次に位置情報データを保存する方法を紹介します。

といっても、単に `GeoCollectionReference.add` メソッドを使用するだけです。内部実装的にも単に `CollectionReference.add` メソッドを使用しているのと同等です。

このとき、`GeoFirePoint.data` を用いて Geohash 文字列を保存しておくと後にその Geohash 文字列を頼りに位置情報クエリを書くことができます。

```dart
Future<DocumentReference<Map<String, dynamic>>> addGeoData() async {
  final CollectionReference<Map<String, dynamic>> collectionReference =
      FirebaseFirestore.instance.collection('locations');
  final GeoCollectionReference<Map<String, dynamic>> geoCollectionReference =
      GeoCollectionReference<Map<String, dynamic>>(collectionReference);
  const GeoPoint tokyoStation = GeoPoint(35.681236, 139.767125);
  const GeoFirePoint geoFirePoint = GeoFirePoint(tokyoStation);

  // GeoCollectionReference の add メソッドを呼ぶ。
  // geoFirePoint.data の Map<String, dynamic> のデータが
  // Cloud Firestore のドキュメントに保存される。
  return geoCollectionReference.add(geoFirePoint.data);
}
```

また、同様に `CollectionReference.doc('your-document-id').set` (`DocumentReference.set`) に対応する `GeoCollectionReference.set` メソッドも提供しています。こちらもやはり内部実装は `CollectionReference.set` メソッドを使用しているのと同等で使い方もかんたんです。

```dart
Future<void> setGeoData() async {
  // ... 省略

  // GeoCollectionReference の set メソッドを呼ぶ。
  // geoFirePoint.data の Map<String, dynamic> のデータが
  // Cloud Firestore のドキュメントに保存される。
  // cloud_firestore の [SetOptions] も同様に使用できる。
  return geoCollectionReference.set(
    id: 'your-document-id',
    data: geoFirePoint.data,
    options: SetOptions(merge: false),
  );
}
```

### 位置情報データを更新する (update)

今度は保存済みの位置情報データを更新する方法を紹介します。

位置情報データを含むフィールド名を指定し、更新したい位置情報を `GeoPoint` で与えて、`GeoCollectionReference.updatePoint` メソッドを呼びます。内部ではやはり `CollectionReference.doc('your-document-id').update` (`DocumentReference.update`) メソッドを使用しており、指定したフィールドだけが更新されます。

なお、このメソッドを使用するのに `GeoPoint` つまり緯度経度の情報のみを与えればよく、内部で自動でその地点の Geohash を計算した結果も更新します。

```dart
Future<void> updateGeoData() async {
  // 東京駅の緯度経度で更新したい。
  const GeoPoint tokyoStation = GeoPoint(35.681236, 139.767125);

  // GeoCollectionReference の update メソッドを呼ぶ。
  // 指定した 'geo' フィールドを東京駅の緯度経度で更新する。
  return geoCollectionReference.updatePoint(
    id: 'your-document-id',
    field: 'geo',
    geoPoint: tokyoStation,
  );
}
```

### 位置情報データを削除する (delete)

ドキュメントの削除は、単に `GeoCollectionReference.delete` メソッドを呼びだけです。内部ではやはり `CollectionReference.doc('your-document-id').delete` (`DocumentReference.delete`) メソッドを使用しているだけです。

```dart
Future<void> deleteGeoData() async {
  // ... 省略

  // GeoCollectionReference の delete メソッドを呼ぶ。
  // 指定したドキュメントが削除される。
  return geoCollectionReference.delete(id: 'your-document-id');
}
```

### 位置情報データを取得する (list)

#### リアルタイムで取得する (`Stream`)

#### 都度取得する (`Future`)

#### 任意のフィルタ条件（where 句）を追加する

## geoflutterfire パッケージとの対応（移行ガイドとしても）
