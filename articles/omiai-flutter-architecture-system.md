---
title: "【system パッケージ】Omiai の Flutter プロジェクトのアーキテクチャ"
emoji: "⚙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: true
publication_name: "omiai_techblog"
---

## この記事について

株式会社 Omiai の Flutter テックリードの [@kosukesaigusa](https://github.com/kosukesaigusa) です。

以前の「Omiai の Flutter プロジェクトのアーキテクチャ」という記事：

@[card](https://zenn.dev/kosukesaigusa/articles/omiai-flutter-architecture)

の続編として、`system` パッケージの具体的な実装内容について紹介します。

## system パッケージ

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

:::message
`system` のパッケージ名は、それほど一般的または典型的ではないかもしれませんが、これまでに経験したプロジェクトを参考にしています。core, service, infrastructure のような命名もあり得るかもしれません。
:::

`system` パッケージでは、3rd パーティのツールをラップして腐敗防止層のような役割をしたり、その他の基礎的・汎用的な処理を記述したりします（例：HTTP クライアント、Shared Preferences, Firebase Analytics など）。

そうすることで、`system` よりも図の中で上側のパッケージでは、direct にそれらの 3rd パーティのツールに依存することがなくなります。

`system` パッケージとして提供する API のインターフェースを適切に定義することができていれば、実際にそうすることは多くはないでしょうが、利用するツールを同等の機能をもつ別のものに置換することも容易にできるようになります。

`system` パッケージは、定められたパッケージ以外には依存せず、アプリケーションの業務知識に依存するような命名やインターフェース定義にならないように注意します。

基本的には全く別のアプリケーションでもそのまま利用できるような内容になるべきです。

一方、3rd パーティのツールには、多様な用途に対応するために、様々な機能やインターフェースが提供されています。自身のプロジェクトでは明らかに不必要な機能やインターフェースは省略したり、アプリケーションの用途に沿うようにカスタマイズしたりすることもあるかもしれません。

また、例外をどのようにハンドリングするかということも、`system` パッケージの広義のインターフェースという観点で重要です。

以下で具体的な実装内容やそのような実装にしている背景を説明します。

## Firebase Analytics の実装例

下記は `firebase_analytics` パッケージの `FirebaseAnalytics` クラスをほとんどラップしただけのクラスです。

doc comment の内容とも重複しますが、下記のような工夫をしています。

- `system` パッケージ以外が `firebase_analytics` パッケージに直接依存しないようにする
- 分析ログの送信という性質上、利用する側に `await` させず、例外ハンドリングもさせない（例外が起きても呼び出し側の処理を止めない）ようにする
- `FirebaseAnalytics` クラスの各メソッドの現状利用する予定がない引数を利用できないようにする
- 一種の腐敗防止層としての実装とする

```dart
import 'dart:async';

import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:util/util.dart';

/// firebase_analytics パッケージが提供する、Firebase Analytics の各機能をラップしたクラス。
///
/// ほとんど firebase_analytics のインターフェースの再実装のような内容になっているが、
///
/// - system パッケージ以外が firebase_analytics パッケージに直接依存しないようにする
/// - 分析ログの送信という性質上、利用する側に await させず、例外ハンドリングもさせない（例外が
/// 起きても呼び出し側の処理を止めない）ようにする
/// - [FirebaseAnalytics] クラスの各メソッドの現状利用する予定がない引数を利用できないようにする
/// - 一種の腐敗防止層としての実装とする
///
/// ことなどを目的としている。
class FirebaseAnalyticsClient {
  /// [FirebaseAnalyticsClient] を生成する。
  const FirebaseAnalyticsClient(this._firebaseAnalytics);

  final FirebaseAnalytics _firebaseAnalytics;

  /// Firebase Analytics にログを送信する。
  ///
  /// 本メソッドの返り値方は `void` で定義されており、[unawaited] で [_logEvent] を呼び
  /// 出すことで、もし例外が発生しても呼び出し側の処理を止めない（ただし、グローバルエラー
  /// ハンドラーを定義すればキャッチできる）ようにしている。
  void logEvent({
    required String name,
    required Map<String, Object>? parameters,
  }) =>
      unawaited(_logEvent(name: name, parameters: parameters));

  /// [FirebaseAnalytics.logEvent] メソッドを呼び出す。
  ///
  /// ログインイベントの送信時に何らかの例外やエラーが発生したらログ出力する。
  Future<void> _logEvent({
    required String name,
    required Map<String, Object>? parameters,
  }) async {
    try {
      // NOTE: callOptions の引数は現状利用する予定がないため、インターフェースに表していない。
      await _firebaseAnalytics.logEvent(name: name, parameters: parameters);
    }
    // NOTE: 上記の処理が何らかの例外やエラーが発生したらそれをログ出力できるように、あえて
    // 例外・エラー型を指定せずに catch (e) としている。
    // ignore: avoid_catches_without_on_clauses
    catch (e) {
      logger.e('Failed to log Firebase Analytics event: $e');
    }
  }

  /// Firebase Analytics のユーザー属性を設定する。
  ///
  /// 本メソッドの返り値方は `void` で定義されており、[unawaited] で [_setUserProperty]
  /// を呼び出すことで、もし例外が発生しても呼び出し側の処理を止めない（ただし、グローバルエラー
  /// ハンドラーを定義すればキャッチできる）ようにしている。
  void setUserProperty({required String name, required String value}) =>
      unawaited(_setUserProperty(name: name, value: value));

  /// [FirebaseAnalytics.setUserProperty] メソッドを呼び出す。
  ///
  /// ユーザー属性の設定時に何らかの例外やエラーが発生したらログ出力する。
  Future<void> _setUserProperty({
    required String name,
    required String value,
  }) async {
    try {
      // NOTE: callOptions の引数は現状利用する予定がないため、インターフェースに表していない。
      await _firebaseAnalytics.setUserProperty(name: name, value: value);
    }
    // NOTE: 上記の処理が何らかの例外やエラーが発生したらそれをログ出力できるように、あえて
    // 例外・エラー型を指定せずに catch (e) としている。
    // ignore: avoid_catches_without_on_clauses
    catch (e) {
      logger.e('Failed to set Firebase Analytics user property: $e');
    }
  }
}
```

## HTTP クライアントの実装例

`HttpClient` の実装例を紹介する前に `HttpResponse` クラスを定義します。

HTTP レスポンスの成功、失敗を freezed の sealed class で表現するクラスです。

後述の `HttpClient` では `dio` パッケージを利用しますが、HTTP のエラーレスポンスが返された場合には、`dio` パッケージで定義されている `DioException` 型の例外が発生します。

- `system` パッケージを利用するパッケージでは、`dio` パッケージに直接依存せずに例外ハンドリングしたい
- 静的解析のレベルで、HTTP レスポンスの成功、失敗をそれぞれハンドリングすることを強制したい

という理由から、Dart の本来の例外型をスローするのではなく、sealed class で表現した `HttpResponse` クラスのインスタンスを返すようにしています。

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'http_response.freezed.dart';

/// HTTP レスポンスの成功・失敗を freezed の sealed class でまとめて表現するクラス。
@freezed
sealed class HttpResponse<T> with _$HttpResponse<T> {
  /// 成功時のレスポンス。
  ///
  /// - [data] は HTTP のレスポンスボディ。
  /// - [headers] は HTTP のレスポンスヘッダ。
  const factory HttpResponse.success({
    required T data,
    required Map<String, List<String>> headers,
  }) = SuccessHttpResponse<T>;

  /// 失敗時のレスポンス。
  ///
  /// - [data] は存在する場合の HTTP のレスポンスボディ（エラーレスポンスでは一般にレスポンス
  /// ボディが存在しない場合もあるので任意）。
  /// - [e] は HTTP リクエスト時に発生した例外。
  /// - [status] は HTTP エラーの種別。
  const factory HttpResponse.failure({
    T? data,
    required Object e,
    required ErrorStatus status,
  }) = FailureHttpResponse<T>;
}

/// HTTP レスポンスの主なエラーの種別を表現する列挙型。
enum ErrorStatus {
  /// 400 Bad Request に対応するエラー。
  badRequest,

  /** 省略 */

  /// その他のエラー。
  unknown,
  ;
}
```

`HttpClient` で利用する `dio` パッケージに依存した `DioHttpClient` は下記のように定義します。

dio パッケージの README の [Extends Dio class](https://pub.dev/packages/dio#extends-dio-class) を参照してください。

```dart
class DioHttpClient with DioMixin implements Dio { /** 省略 */ }
```

`HttpClient` クラスの実装は一部省略していますが、下記の通りです。

`DioHttpClient` の `request` メソッドを呼び出して HTTP リクエストを行い下記のように結果をハンドリングします。

| 結果 | 処理 |
| ---- | ---- |
| 成功 | 成功レスポンスを格納した `HttpResponse.success` を返す |
| 失敗（`DioException` が発生） | `_httpResponseFromDioException` メソッドを通じて、失敗レスポンスを格納した `HttpResponse.failure` を返す |
| 失敗（その他の例外が発生） | 失敗レスポンスを格納した `HttpResponse.failure` を返す |

`_httpResponseFromDioException` メソッド内では、`DioException` からステータスコードやレスポンスデータなどもとに `HttpResponse.failure` を生成します。

```dart
/// HTTP リクエストを行うクライアントクラス。
class HttpClient {
  /** 省略 */

  /// Dio の HTTP クライアントインスタンス。
  final DioHttpClient _client;

  /// HTTP リクエストを行う。
  ///
  /// 返り値は [HttpResponse] であり、リクエストが成功した場合は [HttpResponse.success]
  /// が返る。
  ///
  /// リクエストが失敗した場合は、利用する側で dio パッケージに直接依存して [DioException] を
  /// 直接扱わないで済むように、[DioException] および [Exception] は内部でキャッチして、
  /// [HttpResponse.failure] を返す。
  Future<HttpResponse<dynamic>> request(/** 省略 */) async {
    try {
      final response = await _client.request<dynamic>(/** 省略 */);
      return HttpResponse.success(
        data: response.data,
        headers: response.headers.map,
      );
    } on DioException catch (e) {
      return _httpResponseFromDioException(e);
    } on Exception catch (e) {
      logger.e(e);
      return HttpResponse.failure(e: e, status: ErrorStatus.unknown);
    }
  }

  /// [DioException] から [HttpResponse.failure] を生成する。
  HttpResponse<dynamic> _httpResponseFromDioException(DioException e) {/**  省略 */}
}
```

HTTP クライアントを利用する側では下記の様に `HttpResponse` の成功、失敗を `switch` でハンドリングすることを強制されます。

```dart
Future</** 省略 */> doSomething() async {
  final response = await httpClient.request(/** 省略 */);
  switch (response) {
    case SuccessHttpResponse(:final data, :final headers):
      /** 省略 */
    case FailureHttpResponse(: final data, :final e, :final status):
      /** 省略 */
  }
}
```

## おわりに

この記事では、Omiai の Flutter プロジェクトのアーキテクチャ紹介の続編として、3rd パーティのツールをラップして腐敗防止層のような役割をしたり、その他の基礎的・汎用的な処理を記述したりする `system` パッケージについて説明しました。

Omiai の Flutter プロジェクトへの参画にご興味のある方は、ぜひ一度お気軽にお問い合わせください！
