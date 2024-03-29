---
title: "FlutterFire な位置情報クエリが書ける pub パッケージ geoflutterfire_plus の紹介"
emoji: "🌍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firestore"]
published: true
---

## geoflutterfire_plus パッケージ

この記事では、著者が開発・リリースした、Flutter x Firestore で位置情報クエリがかける pub パッケージである geoflutterfire_plus を紹介し、主な使用方法をまとめたり、サンプルアプリを紹介したりします。

@[card](https://pub.dev/packages/geoflutterfire_plus)

@[card](https://github.com/KosukeSaigusa/geoflutterfire_plus)

このパッケージを使用することで、

- 中心位置（緯度経度）
- 検出半径 (km)
- （任意）その他のフィルタ条件

を入力として、Cloud Firestore に保存済みの位置情報を Geohash の仕組みを用いてリアルタイム取得（または任意のタイミングでリクエストして都度取得）をすることができます。

次の GIF 画像のような、地図上で位置情報データを探すようなアプリで広く用いることができます。

![example](https://user-images.githubusercontent.com/13669049/223426938-392e3c65-fe92-4a7c-aad8-82ae6d296bce.gif)

## geoflutterfire_plus 開発の背景

既出の同様のパッケージとして、geoflutterfire というものがあります。

@[card](https://pub.dev/packages/geoflutterfire)

しかし、このパッケージは記事執筆時点で約 1 年ほどメンテナンスが止まっており、肝心の `cloud_firestore` などの依存パッケージが最新に対応しておらず、開発中のアプリが新しいバージョンの `cloud_firestore` パッケージを使用している場合には、依存関係の解決時にコンフリクトが生じしてしまい、使用することができません。

![geoflutterfire_pubspec.png](/images/articles/geoflutterfire_plus/geoflutterfire_pubspec.png)

依存パッケージを最新にする PR を継続して作成することも検討しましたが、その流れで内部のソースコードを詳しく読んでみると、ソースコードや設計にも根本から修正した方が良いと思われる箇所が多く、しばらくはフォークして自分なりに改善したものを使用していました。その修正がまとまったものになってきたタイミングで、一からパッケージを作り直して公開しようと思ったのが開発のきっかけです。

また、依存パッケージを最新に保つメンテナンスがほぼ中心で geoflutterfire2 というパッケージもあります。

@[card](https://pub.dev/packages/geoflutterfire2)

が、基本的なコードは元の geoflutterfire パッケージと変わりません。

## Geohash と Geo query について

この章では geoflutterfire_plus パッケージが位置情報クエリを動作させるために使用している Geohash と Geo query いう仕組みについて、参考になる情報を示しながら、かんたんに解説します。

geoflutterfire_plus パッケージをただ使用して、Flutter で位置情報関係のアプリ開発を行いたいだけの方は軽く読むか、読み飛ばすかして、パッケージ内部への興味がわいたときに詳しく読んでみても良いかもしれません。

Geohash と Geo query について参考になるリンクには以下のようなものがあります。

@[card](https://firebase.google.com/docs/firestore/solutions/geoqueries)

@[card](https://qiita.com/yabooun/items/da59e47d61ddff141f0c)

@[card](https://www.fuzina.com/blog/2019/12/10/geohash%E3%81%A7%E7%AF%84%E5%9B%B2%E6%A4%9C%E7%B4%A2%E3%82%92%E8%A1%8C%E3%81%86.html)

@[card](https://yuroyoro.hatenablog.com/entry/20100115/1263526125)

位置情報検索を行うにあたり、率直に考えれば、緯度経度の情報を保存し、2 点間の距離を計算しながら与えられた中心位置から検出範囲内にあるものをフィルタするクエリを書くことになるでしょう。

しかし、Cloud Firestore では、ある複合クエリに対して 1 つの範囲条件しか適用することができないので、緯度経度を別々のフィールドとして保存していると、「緯度がこの範囲で、経度がこの範囲」といったクエリを書くことができません。

そこで、緯度経度の値の組を文字列に変換して扱う仕組みである Geohash が用いられます。

Geohash は位置情報の緯度経度をエンコードして Base32 文字列に変換したものです。一般にエンコードというと、入力と出力はバラバラである（つまり、複数の出力からそれぞれの入力やその関係を予想できない）ことを想像しますが、Geohash には、物理的な地点が近いもののほど、Geohash 文字列も似た並びになるという特徴があります。アルゴリズムの詳細を追うとその理由はよく理解できます。

たとえば、東京駅の緯度経度を (35.681236, 139.767125) で与えると、それに対応する Geohash 文字列は `xn76urx66` となります。渋谷駅は緯度経度が (35.658034, 139.701636) で、Geohash 文字列は `xn76fgreh` です。

Geohash は緯度経度から計算されますが、そのアルゴリズムの内部で一部の情報を落としているので、Geohash が示す位置情報は特定の点ではなく、精度に応じた長方形領域です。緯度経度の桁数を詳しく与えれば与えるほど、Geohash 文字列の精度（桁数）が上がります。

地球は球形であるため実際には地点（緯度）によって異なりますが、Geohash の精度（桁数）と、その Geohash が与える赤道上での最大の誤差範囲を表す長方形領域の大きさ（縦：南北方向 [km] x 横：東西方向 [km]）は次のように対応しています。

|  桁数  |  長方形領域  |
| ---- | ---- |
|  1  |  約 5,000km x 約 5,000km  |
|  2  |  約 1,250km x 約 625km  |
|  3  |  約 156km x 約 156km  |
|  4  |  約 39.1km x 約 19.5km  |
|  5  |  約 4.89km x 約 4.89km  |
|  6  |  約 1.22km x 約 0.61km  |
|  7  |  約 153m x 約 153m  |
|  8  |  約 38.2m x 約 19.1m  |
|  9  |  約 4.77m x 約 4.77m  |

7 桁程度の精度が確保できるなら、十分位置情報系のサービスに使用できそうなことが分かります。

位置情報クエリを行う際、geoflutterfire_plus のパッケージの内部では、この Geohash 文字列に対して `startAt`, `endAt` クエリを行うことで指定した中心地点から指定された半径以内のドキュメントを取得するような実装になっています。

@[card](https://github.com/KosukeSaigusa/geoflutterfire_plus/blob/e6c1bfa23efb28865f2e6637d152f01188adb1ce/lib/src/geo_collection_reference.dart#L303C10-L316)

## 基本的な機能や使い方の紹介

この章では、geoflutterfire_plus パッケージの基本的な機能や使い方を紹介します。

説明の目的で型注釈などは冗長に書いています。

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
// Location クラスの定義。詳細は下記リンクを確認してください。
class Location {
  // ... 省略
}

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

`Location` クラスの実装内容の例を具体的に確認したい場合は次のファイルの該当箇所を参照してください：

@[card](https://github.com/KosukeSaigusa/geoflutterfire_plus/blob/e6c1bfa23efb28865f2e6637d152f01188adb1ce/example/lib/advanced/entity.dart#L4)

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

ドキュメントの削除は、単に `GeoCollectionReference.delete` メソッドを呼ぶだけです。内部ではやはり `CollectionReference.doc('your-document-id').delete` (`DocumentReference.delete`) メソッドを使用しています。よって、無理に geoflutterfire_plus のこのメソッドを使用せずとも、cloud_firestore パッケージの `DocumentReference.delete` メソッドを使用しても構いません。

```dart
Future<void> deleteGeoData() async {
  // ... 省略

  // GeoCollectionReference の delete メソッドを呼ぶ。
  return geoCollectionReference.delete(id: 'your-document-id');
}
```

### 位置情報データを取得する (list)

いよいよ位置情報データの取得の方法を紹介します。

上述までの方法で Cloud Firestore のあるコレクションに、特定の名前のフィールド（上の例では `geo` というフィールド）に、`GeoPoint` の値（緯度経度）と Geohash 文字列が Cloud Firestore の map 型（≒ Dart の Map 型）で保存されているはずです。

それらの保存済みの位置情報データの中から、指定した中心位置 (`center`) から指定した半径 R [km] (`radiusInIKm`) 以内に位置するドキュメントを取得する方法として、geoflutterfire_plus パッケージでは次の 2 つのメソッドを提供しています。

- `Future` 型で都度取得するメソッド：`GeoCollectionReference.fetchWithin`
- `Stream` 型で購読するメソッド：`GeoCollectionReference.subscribeWithin`

#### 都度取得する (`Future`)

返り値は `Future<List<DocumentSnapshot<T>>>` 型です。総称型の `T` は `GeoCollectionReference` にどのような型をつけているかによります。特に型を付けていなければ `Map<String, dynamic>` になるでしょう。`withConverter` を使っていればそれで付けた型になります。

必須のパラメータは以下の通りです。

- 中心位置：`center`（`GeoFirePoint` 型）
- 検出半径 (km)：`radiusInKm`（`double` 型）
- フィールド名：`field`（`String` 型）
- `T` 型オブジェクトから `GeoPoint` インスタンスを作成する関数：`geopointFrom`（`GeoPoint Function(T obj)` 型）

最後の `GeoPoint Function(T obj)` 型の `geopointFrom` というパラメータはやや複雑なので説明を加えます。

これは、`T` 型のオブジェクト（上で述べた通り `withConverter` などで `GeoCollectionReference` に型をつけていればその型が、付けていなければ `Map<String, dynamic>` 型になるでしょう）から、`GeoPoint` 型のインスタンスを生成する関数を求めるものです。

たとえば、型を付けていないとき、対象ドキュメントの `geo` という map 型のフィールドに `geopoint` というキー名で `GeoPoint` 型の値を保存しているならば、次のように記述します。

```dart
GeoPoint Function(Map<String, dynamic> data) geopointFrom =
    (data) => (data['geo'] as Map<String, dynamic>)['geopoint'] as GeoPoint;
```

たとえば、ドキュメントに `Location` という型がついていて、`Location.geo.geopoint` で `GeoPoint` 型の値を得られるときは次のように記述します。

```dart
GeoPoint Function(Location location) geopointFrom =
    (location) => location.geo.geopoint;
```

よって、locations コレクションのドキュメント（`geo` フィールドに `GeoFirePoint.data` 相当の map 型のデータが保存されている想定）から、東京駅を中心に、半径 50 km 以内に位置するデータだけを取得する方法は下記のようになります。

```dart
Future<List<DocumentSnapshot<Map<String, dynamic>>>> fetchGeoData() async {
  const GeoPoint center = GeoPoint(35.681236, 139.767125);
  final CollectionReference collectionReference =
      FirebaseFirestore.instance.collection('locations');
  return GeoCollectionReference(collectionReference).fetchWithin(
    center: const GeoFirePoint(center),
    radiusInKm: 50,
    field: 'geo',
    geopointFrom: (data) =>
        (data['geo'] as Map<String, dynamic>)['geopoint'] as GeoPoint,
    strictMode: true,
  );
}
```

任意のパラメータとして `bool` 方の `strictMode` というものがあります。デフォルトで `false` ですが、`false` の場合は実際の検出範囲から `1.02` 倍のバッファをもたせた半径を検出範囲とします。`true` の場合はバッファを設けず指定した通りの半径で検出を行います。

ちなみに、この `Future` 型の都度取得のメソッドは geoflutterfire_plus で新たに機能追加したものです。

次章の `Stream` 型を返す購読以外と異なり、リクエストしたタイミングで都度取得するので、読み取りコストや体験の最適化につながる場合に使用してください。

パッケージの doc comment でも各パラメータの意味を説明しているので参考にしてください。

#### リアルタイムで取得する (`Stream`)

次はリアルタイムに取得した結果を `Stream` 型で返す `GeoCollectionReference.subscribeWithin` メソッドの紹介です。

返り値は `Stream<List<DocumentSnapshot<T>>>` 型です。

使用方法は都度取得の `GeoCollectionReference.fetchWithin` と全く同じです。

```dart
Stream<List<DocumentSnapshot<Map<String, dynamic>>>> subscribeGeoData() async {
  const GeoPoint center = GeoPoint(35.681236, 139.767125);
  final CollectionReference collectionReference =
      FirebaseFirestore.instance.collection('locations');
  return GeoCollectionReference(collectionReference).subscribeWithin(
    center: const GeoFirePoint(center),
    radiusInKm: 50,
    field: 'geo',
    geopointFrom: (data) =>
        (data['geo'] as Map<String, dynamic>)['geopoint'] as GeoPoint,
    strictMode: true,
  );
}
```

たとえば、検出の中心位置や検出半径をユーザーインターフェースで操作しながらリアルタイムで位置情報データを取得していくような場合に使用してください。

#### 任意のフィルタ条件（where 句）を追加する

上述までで、基本的な位置情報データの都度取得・購読の方法を紹介しました。

geoflutterfire_plus の `GeoCollectionReference.fetchWithin`, `GeoCollectionReference.subscribeWithin` メソッドでは、`Query<T>? Function(Query<T> query)?` 型の `queryBuilder` という任意のパラメータで、任意のフィルタ条件を追加することができます。

たとえば上述までと同様の位置情報クエリに、`isVisible` というフィールドが `true` のドキュメントだけをフィルタしたい場合、次のように `queryBuilder` パラメータに、`Query<T>` 型の `query` という値を受け取って、任意のフィルタ条件（where 句など）を追加した新たな `Query<T>` 型の値を返すような関数を指定します。

```dart
Future<List<DocumentSnapshot<Map<String, dynamic>>>> fetchVisibleGeoData() async {
  const center = GeoPoint(35.681236, 139.767125);
  final collectionReference =
      FirebaseFirestore.instance.collection('locations');
  return GeoCollectionReference(collectionReference).fetchWithin(
    center: const GeoFirePoint(center),
    radiusInKm: 50,
    field: 'geo',
    geopointFrom: (data) =>
        (data['geo'] as Map<String, dynamic>)['geopoint'] as GeoPoint,
    // where('isVisible', isEqualTo: true) のフィルタ条件を追加する
    queryBuilder: (query) => query.where('isVisible', isEqualTo: true),
    strictMode: true,
  );
}
```

注意点として、このようなクエリは複合インデックスを必要とするので、もしそのようなインデックスが生成されていない状態で実行すると "[cloud_firestore/failed-precondition] The query requires an index..." というエラーメッセージとともにエラーが発生するでしょう。コンソール上のそのエラーメッセージに、対応するインデックスを作成するためのリンクが表示されるはずなので、それをクリックしてインデックスを生成した後に再度実行するとその通りに動作するはずです。

## geoflutterfire パッケージとの対応（移行ガイドとしても）

後日追記します。

## geoflutterfire_plus を用いたサンプルアプリ

この章では、geoflutterfire_plus パッケージと google_maps_flutter パッケージを用いたアプリを作る方法をかんたんに追ってみます。アプリ上に表示した Google Maps 上に geoflutterfire_plus パッケージの機能で取得した位置情報データをリアルタイムに表示していくようなアプリです。

基本的には geoflutterfire_plus リポジトリの example プロジェクトの内容の抜粋なので、詳細は下記リンクで確認することもできます。

@[card](https://github.com/KosukeSaigusa/geoflutterfire_plus/tree/main/example)

pubspec.yaml に最新バージョンの geoflutterfire_plus パッケージと google_maps_flutter パッケージを追加してください。cloud_firestore, firebase_core, rxdart も同様です。

```yaml:pubspec.yaml
dependencies:
  cloud_firestore: <latest-version-here>
  firebase_core: <latest-version-here>
  flutter:
    sdk: flutter
  geoflutterfire_plus: <latest-version-here>
  google_maps_flutter: <latest-version-here>
  rxdart: <latest-version-here>
```

google_maps_flutter パッケージについては、パッケージの README をよく確認し、[GCP の maps-platform](https://cloud.google.com/maps-platform/) から API キーを取得して、iOS, Android のそれぞれで必要な設定を済ませてください。

@[card](https://pub.dev/packages/google_maps_flutter)

サンプルアプリの位置情報データは、Cloud Firestore のトップレベルの locations コレクションに、次のようなスキーマで保存している前提とします。

前述の位置情報データの追加のセクションの説明の通りに `geo` というフィールドに Geohash 文字列と `GeoPoint` 型の値（`GeoFirePoint.data` 相当）が保存されています（ついでに `name` と `isVisible` というフィールドを追加しました）。

![schema.png](/images/articles/geoflutterfire_plus/schema.png)

エントリポイント周りは次のとおりです。`Example` という `StatefulWidget` がマップを表示する画面です。

```dart:main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(const App());
}

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        sliderTheme: SliderThemeData(
          overlayShape: SliderComponentShape.noOverlay,
        ),
      ),
      home: const Example(),
    );
  }
}
```

`State` クラスのメンバ（状態）としては主に

- 取得された Google Maps 上のマーカ：`_markers`（`Set<Marker>` 型）
- クエリ条件の購読と保持を行うオブジェクト：`_geoQueryCondition`（`BehaviorSubject<_GeoQueryCondition>` 型）
  - `_GeoQueryCondition` 型は下記の条件を内包しています
    - 検出半径：`_radiusInKm`（`double` 型）
    - Google Maps 上のカメラ位置：`_cameraPosition`（`CameraPosition` 型）
- クエリ結果の購読：`_stream`（`Stream<List<DocumentSnapshot<Map<String, dynamic>>>>` 型）

があります。

検出半径 (km) とカメラ位置（= 位置情報クエリの中心）を変化させながら、それをクエリ条件 (`_GeoQueryCondition`) `BehaviorSubject` に流し、リアルタイムで反映されるその結果として得られた位置情報データを flutter_google_maps パッケージの `Marker` 型の集合 (`Set`) である `_markers` として更新していき、それを表示するということです。

`_geoQueryCondition.switchMap` によって、最新検出半径や中心位置の検索条件による位置情報クエリの `Stream` を作成します。`GeoCollectionReference.subscribeWithin` メソッドの使い方は上で説明した通りです。

`dispose` メソッドをオーバーライドして `_geoQueryCondition` をクローズすることも忘れないでください。

```dart:main.dart
/// 検出半径とカメラ位置をクエリ条件として一緒に取り扱うためのクラス。
class _GeoQueryCondition {
  _GeoQueryCondition({
    required this.radiusInKm,
    required this.cameraPosition,
  });

  final double radiusInKm;
  final CameraPosition cameraPosition;
}

class Example extends StatefulWidget {
  const Example({super.key});

  @override
  ExampleState createState() => ExampleState();
}

class ExampleState extends State<Example> {
  /// Google Maps 上のマーカ一覧。
  Set<Marker> _markers = {};

  /// クエリ条件を購読しつつ、最新のクエリ条件を保持するための [BehaviorSubject]。
  final _geoQueryCondition = BehaviorSubject<_GeoQueryCondition>.seeded(
    _GeoQueryCondition(
      radiusInKm: _initialRadiusInKm,
      cameraPosition: _initialCameraPosition,
    ),
  );

  /// クエリ結果の購読。this._geoQueryCondition を使用したいので late final で定義する。
  late final Stream<List<DocumentSnapshot<Map<String, dynamic>>>> _stream =
      _geoQueryCondition.switchMap(
    (geoQueryCondition) =>
        GeoCollectionReference(_collectionReference).subscribeWithin(
      center: GeoFirePoint(
        GeoPoint(
          _cameraPosition.target.latitude,
          _cameraPosition.target.longitude,
        ),
      ),
      radiusInKm: geoQueryCondition.radiusInKm,
      field: 'geo',
      geopointFrom: (data) =>
          (data['geo'] as Map<String, dynamic>)['geopoint'] as GeoPoint,
      strictMode: true,
    ),
  );

  @override
  void dispose() {
    _geoQueryCondition.close();
    super.dispose();
  }
}
```

`build` メソッドで、flutter_google_maps パッケージの `GoogleMap` ウィジェットを表示します。

`void Function(GoogleMapController)?` 型である `onMapCreated` でクエリ結果である `_stream` の購読を開始し、新しいクエリ結果を得るたびに `_updateMarkersByDocumentSnapshots` メソッドでマーカ一覧である `_markers` を更新して `setState` します。

`void Function(CameraPosition)?` 型である `onCameraMove` では、移動したカメラ位置 (`CameraPosition`) が都度得られるので、それを `_geoQueryCondition.add` して新しいクエリ条件として `BehaviorSubject` に流します。

こうすることで、リアルタイムで中心位置を変えながら位置情報をリアルタイムで取得していくことができます。

```dart:main.dart
// ... 省略

class ExampleState extends State<Example> {
  // ... 省略
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Stack(
        children: [
          GoogleMap(
            // ... 一部のプロパティは省略
            initialCameraPosition: _initialCameraPosition,
            onMapCreated: (_) =>
                _stream.listen(_updateMarkersByDocumentSnapshots),
            markers: _markers,
            onCameraMove: (cameraPosition) {
              _geoQueryCondition.add(
                _GeoQueryCondition(
                  radiusInKm: _radiusInKm,
                  cameraPosition: cameraPosition,
                ),
              );
            },
          ),
          // ... 省略
        ],
      ),
    );
  }

  /// Updates [_markers] by fetched geo [DocumentSnapshot]s.
  void _updateMarkersByDocumentSnapshots(
    List<DocumentSnapshot<Map<String, dynamic>>> documentSnapshots,
  ) {
    final markers = <Marker>{};
    for (final ds in documentSnapshots) {
      final id = ds.id;
      final data = ds.data();
      if (data == null) {
        continue;
      }
      final name = data['name'] as String;
      final geoPoint =
          (data['geo'] as Map<String, dynamic>)['geopoint'] as GeoPoint;
      markers.add(_createMarker(id: id, name: name, geoPoint: geoPoint));
    }
    setState(() {
      _markers = markers;
    });
  }

  /// Creates a [Marker] by fetched geo location.
  Marker _createMarker({
    required String id,
    required String name,
    required GeoPoint geoPoint,
  }) =>
      Marker(
        markerId: MarkerId('(${geoPoint.latitude}, ${geoPoint.longitude})'),
        position: LatLng(geoPoint.latitude, geoPoint.longitude),
        infoWindow: InfoWindow(title: name),
        onTap: () {/** 省略 */},
      );
}
```

検出半径を更新したい場合にも、次のようなメソッドがコールされるようにすれば良いでしょう。

```dart:main.dart
// ... 省略

class ExampleState extends State<Example> {
  // ... 省略

  /// 検出半径を更新する。
  void updateRadius(double radius) => _geoQueryCondition.add(
    _GeoQueryCondition(
      radiusInKm: value,
      cameraPosition: _cameraPosition,
    ),
  );
}
```

## さいごに

geoflutterfire_plus パッケージと使い方の紹介をしてきました。パッケージに更新があり次第更新していく予定です。

今後の開発のモチベーションになりますので、pub.dev のパッケージページへ LIKE 👍 や GitHub リポジトリへのスターをいただけると嬉しいです！

@[card](https://pub.dev/packages/geoflutterfire_plus)

@[card](https://github.com/KosukeSaigusa/geoflutterfire_plus)

パッケージに関するご不明点や見つけたバグ、Pull Request なども GitHub の該当リポジトリ上でお待ちしております。
