---
title: "【repository パッケージ】Omiai の Flutter プロジェクトのアーキテクチャ"
emoji: "🗄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: false
# published_at: 2024-12-31 00:00
publication_name: "omiai_techblog"
---

## この記事について

株式会社 Omiai の Flutter テックリードの [@kosukesaigusa](https://github.com/kosukesaigusa) です。

以前の「Omiai の Flutter プロジェクトのアーキテクチャ」という記事：

@[card](https://zenn.dev/kosukesaigusa/articles/omiai-flutter-architecture)

@[card](https://zenn.dev/kosukesaigusa/articles/omiai-flutter-architecture-system)

の続編として、`repository` パッケージの具体的な実装内容について紹介します。

## repository パッケージ

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

`repository` パッケージでは、データソース（自社の API サーバーやローカルストレージなど）とのやり取りを記述します。

`repository` パッケージを利用する側である `domain` パッケージにとっては、通信先が自社の API サーバーであるか、それともそれ以外の API サーバーなのか、またはローカルストレージなのかということは知る必要のないようにインターフェースを定義します。

また、`system` パッケージと同様に、例外をどのようにハンドリングするかということも、`repository` パッケージの広義のインターフェースという観点で重要です。

以下で具体的な実装内容やそのような実装にしている背景を説明します。

## 実装内容

`repository` パッケージでは、下記のような実装をします。

| 実装内容 | 詳細 |
| ---- | ---- |
| RepositoryResult | Repository による通信結果の成功・失敗を表現する sealed クラス |
| Dto | Domain Transfer Object の略で、Repository による通信結果の値を表現するクラス |
| Repository | 通信処理を記述するクラス |

## RepositoryResult の例

`RepositoryResult` は、Repository による通信結果の成功、失敗を freezed の sealed class で表現するクラスです。

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'result.freezed.dart';

/// リポジトリによる通信結果を表すクラス。
///
/// 成功の場合は [SuccessRepositoryResult]、失敗の場合は [FailureRepositoryResult] が使用される。
@freezed
sealed class RepositoryResult<T> with _$RepositoryResult<T> {
  /// 通信成功の Result を生成する。
  const factory RepositoryResult.success(T data) = SuccessRepositoryResult<T>;

  /// 通信失敗の Result を生成する。
  ///
  /// - [e] には、例外オブジェクトなどが与えられる。
  /// - [reason] には、任意で HTTP のステータスコードに対応する失敗理由が与えられる。
  /// - [errorDto] には、任意でエラーレスポンスのボディが与えられる。
  const factory RepositoryResult.failure(
    Object e, {
    FailureRepositoryResultReason? reason,
    ErrorDto? errorDto,
  }) = FailureRepositoryResult;
}

/// HTTP のステータスコードに対応する失敗理由を表す列挙型。
enum FailureRepositoryResultReason {
  /// 401 Unauthorized.
  unauthorized,

  /** 省略 */
  ;
}
```

## Dto の例

Dto では、HTTP レスポンスのインターフェース相当のデータ型を定義します。

Open API 仕様のドキュメントがあれば openapi_generator などで自動生成することもできるかもしれません。

しかし、Omiai では、長年の開発・運用の歴史から、サーバサイドの技術的負債も、特に API レスポンスの仕様には常に付きまとってきます。

## Repository の例

たとえば HTTP 通信を行う部分については、`system` で定義した `HttpClient` の `Unimplemented` なインターフェースを `dependency_provider` パッケージに定義しておいて、`repository` パッケージではそれを通じて HTTP 通信を行うようにします。

## おわりに

この記事では、Omiai の Flutter プロジェクトのアーキテクチャ紹介の続編として、データソース（自社の API サーバーやローカルストレージなど）とのやり取りを記述する `repository` パッケージについて説明しました。

Omiai の Flutter プロジェクトへの参画にご興味のある方は、ぜひ一度お気軽にお問い合わせください！
